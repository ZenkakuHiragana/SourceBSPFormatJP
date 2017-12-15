### SourceBSPFormatJP  
Valve Developer Communityにある[Source BSP File Formatのページ]を日本語訳したもの（Google 翻訳 + 分かりやすく）<br>
面倒くさがった結果機械翻訳もびっくりな謎の訳が出来上がっているかもしれませんがご容赦ください……<br>
また、読んでてよく分からなかった部分には(?)を置いてます。

***
# Source BSP File Format  
*この記事は2005年10月の[Rof]による[「The Source Engine BSP File Format」]に基づいており、サービスが停止する前にGeocitiesから取得されています。元のバージョンへのミラーが[ここ]にあります。*  

### 目次
1. [前書き](#introduction)  
1. [概要](#overview)  
1. [BSPファイルのヘッダ](#bspheader)  
	1. [バージョン](#versions)  
	1. [Lump構造体](#lumpstructures)  
	1. [Lumpの種類](#lumptypes)  
	1. [Lumpの圧縮](#lumpcompression)  
1. [Lump](#lumpheader)
	1. [平面](#plane)
	1. [頂点](#vertex)
	1. [辺](#edge)
	1. [Surfedge](#surfedge)
	1. [面と元の面](#faceandoriginalface)
	1. [ブラシとブラシ側面](#brushandbrushside)
	1. [ノードとリーフ](#nodeandleaf)
	1. [リーフ面とリーフブラシ](#leaffaceandleafbrush)
	1. [テクスチャ](#textures)
		1. [Texinfo](#texinfo)
		1. [Texdata](#texdata)
		1. [TexdataStringDataとTexdataStringDataTable](#texstring)
	1. [Model](#model)
	1. [可視性](#visiblity)
	1. [エンティティ](#entity)
	1. [Gamelump](#gamelump)
		1. [Static Prop](#staticprops)
		1. [その他](#othergame)
	1. [Displacement](#displacement)
		1. [DispInfo](#dispinfo)
		1. [DispVerts](#dispverts)
		1. [DispTris](#disptris)
	1. [Pakfile](#pakfile)
	1. [Cubemap](#cubemap)
	1. [オーバーレイ](#overlay)
	1. [ライティング](#lighting)
	1. [アンビエントライティング](#ambientlighting)
	1. [オクルージョン](#occlusion)
	1. [Physics](#physics)
	1. [その他](#other)


<h2 id="introduction">前書き</h2>

このドキュメントでは、Source Engineで使用される[BSP]ファイルの構造について説明します。このファイル形式は、[Half-Life 1のエンジン（GoldSrc）]のものと似ていますが、同じではありません。GoldSrcのBSPファイルはQuake、Quake II、QuakeWorldおよびQuake III Arenaのフォーマットに基づいています。このため、Max McGuireの記事 [Quake 2 BSP File Format]は全体的な構造と、類似している部分の構造を理解する上で非常に役に立ちます。  

このドキュメントは、Rofが[VMEX]（Half-Life 2 BSPファイルの逆コンパイラ）の作成中に彼が書いたノートを拡張したものです。したがって、[マップの逆コンパイル]（BSPファイルをHammer Map Editorで読み込み可能な[VMF]ファイルに変換すること）を実行するために必要な部分に焦点を当てています。 

ここにあるほとんどの情報は、上記のMax McGuireの記事とSource SDKのソースコード（特に、C言語のヘッダーファイルである*public/bspfile.h*）、そしてRof自身がVMEXの作成中に行った実験から得られたものです。  

このドキュメントは読者がC/C++、ジオメトリ、およびSourceのマッピング用語についてある程度の理解があることを想定しています。コード（主にC言語の構造体）は`等幅フォント`で書かれています。また、ここにある構造体は明瞭性と一貫性のため、SDKヘッダファイルにある実際の定義から変更されていることがあります。 


<h2 id="overview">概要</h2>

BSPファイルには、マップをレンダリングして遊ぶためにSource Engineで必要とされる大多数の情報が含まれています。これには以下のような情報が含まれます。 

* レベル内のすべてのポリゴンのジオメトリ  
* 各ポリゴンが描画される時に用いるテクスチャの名前と方向  
* ゲーム中にプレイヤーや他のアイテムの物理的挙動をシミュレートするために使われるデータ
* マップ内のすべてのブラシベース、モデル（プロップ）ベース、不可視（論理）エンティティの位置とプロパティ  
* BSP木と可視性テーブル（マップジオメトリ内のプレイヤーの位置の特定と、マップの見える範囲を可能な限り効率的にレンダリングするのに使用）  

また、マップファイルにはレベルで使用されているカスタムテクスチャやモデルをマップのPakfile lumpの中に任意で埋め込むこともできます（下記参照）。  

BSPファイルに格納されていない情報として、マルチプレイヤーゲーム（[Counter-Strike: Source]や[Half-Life 2: Deathmatch]など）でマップを読み込んだ後に表示されるマップの説明テキスト（*mapname.txt*に格納されています）と、ノンプレイヤーキャラクター（NPC、マップをナビゲートする必要がある）が使用するAIナビゲーションファイル（*mapname.nav*に格納されています）があります。Souce Engineのファイルシステムの仕組み上、これらの外部ファイルはBSPファイルの[Pakfile](#pakfile) lumpに埋め込まれることもありますが、通常はそうではありません。  

公式マップのファイルは、[Steam Game Cache File（GCF）]形式で保存され、ゲームエンジンによりSteamファイルシステムを通じてアクセスされます。GCFファイルから中身を抽出してSteam外から閲覧するには、Nemesisの[GCFScape]を使用します。[VPK]ファイル形式を使用している新しいゲームでは通常、マップはオペレーティングシステムのファイルシステムに直接保存されます。  

BSPファイルのデータは、PC/Macではリトルエンディアンで、PS3/Xbox360ではビッグエンディアンで保存されます。リトルエンディアンのファイルをJavaなどのビッグエンディアン形式のプラットフォームで読み込む場合は、バイトスワップが必要です（逆も同様です）。  


<h2 id="bspheader">BSPファイルのヘッダ</h2>

BSPファイルはヘッダから始まります。この構造は、ファイルがValve Source Engine BSPファイルであることと、フォーマットのバージョンを示して、その後にファイル内の各データ(*lump*と呼ばれる)の場所、長さ、およびバージョンが最大64個分続きます。最後に、マップの修正回数が書かれています。  

ヘッダの構造はSDKの *public/bspfile.h* というヘッダファイルに記載されています。このファイルは、このドキュメント全体を通して広範囲にわたって参照されています。また、ヘッダは合計で1036バイトです。  

![Alien Swarm Icon] Alien Swarmでは、この構造体はBSPHeader_tに名前が変更されています。  
``` C++  
struct dheader_t
{
	int	ident;                // BSPファイルの識別子
	int	version;              // BSPファイルのバージョン
	lump_t	lumps[HEADER_LUMPS];  // lumpディレクトリの配列
	int	mapRevision;          // マップの修正回数
};
```  
`ident`は、次のように定義された4バイトのマジックナンバーです。  
``` C++  
// リトルエンディアン "VBSP"   0x50534256
#define IDBSPHEADER	(('P'<<24)+('S'<<16)+('B'<<8)+'V')
```  
したがって、ファイルの最初の4バイトは常に`VBSP`（ASCII形式）です。これらのバイトはファイルをValve BSPファイルとして識別します。他のBSPファイルの形式では、異なるマジックナンバーを使用します（例えば、id SoftwareのQuake Engineを用いたゲームは`IBSP`で始まります）。[GoldSrc]のBSP形式では、マジックナンバーはまったく使用されません。また、マジックナンバーの順序はファイルのエンディアンを決定するためにも使用できます。`VBSP`はリトルエンディアンに、`PSBV`はビッグエンディアンに使用されます。  

2番目のint値は、BSPファイル形式のバージョン（BSPVERSION）です。 Source ゲームの場合、この値はVampire: The Masquerade – Bloodlinesを除いて19から21の範囲です（下記の表を参照）。他のエンジン（Half-Life 1、Quakeシリーズなど）のBSPファイル形式は、全く異なるバージョン番号の範囲を使用することに注意してください。


<h3 id="versions">バージョン</h3>

この表は、Source Engineを用いたいくつかのゲームで使用されているBSPバージョンの概要を示しています。 

<table>
	<tr>
		<th align="center">バージョン</th>
		<th align="center">ゲーム</th>
		<th align="center">備考</th>
	</tr>
	<tr>
		<td>17</td>
		<td>
			<a href="http://en.wikipedia.org/wiki/Vampire:_The_Masquerade_%E2%80%93_Bloodlines" 
			   title="Vampire: The Masquerade – Bloodlines - Wikipedia（英語）">
				Vampire: The Masquerade – Bloodlines
			</a>
		</td>
		<td>dface_tに変更あり</td>
	</tr>
	<tr>
		<td>17～18</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2" 
			   title="Half-Life 2 - Valve Developer Community">
				Half-Life 2 (Beta)
			</a>
		</td>
		<td>ベータ版のリーク</td>
	</tr>
	<tr>
		<td>19</td>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/Sin_Episodes" 
			   title="Sin Episodes - Valve Developer Community">
				Sin Episodes
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td rowspan="4">19～20</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2" 
			   title="Half-Life 2: Valve Developer Community">
				Half-Life 2
			</a>
		</td>
		<td rowspan="4">リリース時は19で、Source 2007/2009アップデートからは<br>部分的に20</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2:_Deathmatch" 
			   title="Half-Life 2: Deathmatch - Valve Developer Community">
				Half-Life 2: Deathmatch
			</a>
		</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/c/c7/Css.png" 
			     alt="Counter-Strike: Source Icon" title="Counter-Strike: Source Icon">
			<a href="https://developer.valvesoftware.com/wiki/Counter-Strike:_Source" 
			   title="Counter-Strike: Source - Valve Developer Community">
				Counter-Strike: Source
			</a>
		</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/c/ce/Icon_dods.png" 
			     alt="Day of Defeat: Source Icon" title="Day of Defeat: Source Icon">
			<a href="https://developer.valvesoftware.com/wiki/Day_of_Defeat:_Source" 
			   title="Day of Defeat: Source - Valve Developer Community">
				Day of Defeat: Source
			</a>
		</td>
	</tr>
	<tr>
		<td rowspan="13">20</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2:_Episode_One" 
			   title="Half-Life 2: Episode One - Valve Developer Community">
				Half-Life 2: Episode One
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2:_Episode_Two" 
			   title="Half-Life 2: Episode Two - Valve Developer Community">
				Half-Life 2: Episode Two
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
			     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Half-Life_2:_Lost_Coast" 
			   title="Half-Life 2: Lost Coast - Valve Developer Community">
				Half-Life 2: Lost Coast
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/1/1c/Garry%5C%27s_Mod_logo.png"
			     alt="Garry's Mod Icon" title="Garry's Mod Icon" width="16" height="16">
			<a href="https://developer.valvesoftware.com/wiki/Garry%27s_Mod"
			   title="Garry's Mod - Valve Developer Community">
				Garry's Mod
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png"
			     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon" width="16" height="16">
			<a href="https://developer.valvesoftware.com/wiki/Garry%27s_Mod"
			   title="Team Fortress 2 - Valve Developer Community">
				Team Fortress 2
			</a>
		</td>
		<td>
			最近のマップはStaticPropLump_tに変更あり（version = 10）<br>
			ゲームlump、エンティティ情報、PAKファイルはLZMA圧縮
		</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/f/fc/Portal-16px.png"
			     alt="Portal Icon" title="Portal Icon">
			<a href="https://developer.valvesoftware.com/wiki/Portal"
			   title="Portal - Valve Developer Community">
				Portal
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/c/c0/L4D-16px.png"
			     alt="Left 4 Dead Icon" title="Left 4 Dead Icon">
			<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead"
			   title="Left 4 Dead - Valve Developer Community">
				Left 4 Dead
			</a>
		</td>
		<td>dworldlight_tとStaticPropLump_tに変更あり（version = 8）</td>
	</tr>
	<tr>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/Zeno_Clash"
			   title="Zeno Clash - Valve Developer Community">
				Zeno Clash
			</a>
		</td>
		<td>StaticPropLump_tに変更あり（version = 7）</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/c/c5/DMMaM.png"
			     alt="Dark Messiah Icon" title="Dark Messiah Icon">
			<a href="https://developer.valvesoftware.com/wiki/Dark_Messiah"
			   title="Dark Messiah - Valve Developer Community">
				Dark Messiah
			</a>
		</td>
		<td>dshader_t、StaticPropLump_t、texinfo_t、dgamelump_tに<br>変更あり</td>
	</tr>
	<tr>
		<td>
			<a href="http://en.wikipedia.org/wiki/Vindictus"
			   title="Vindictus - Wikipedia（英語）">
				Vindictus
			</a>
		</td>
		<td>多くの変更された構造体を使用</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/3/33/Icon_theShip.png"
			     alt="The Ship Icon" title="The Ship Icon">
			<a href="https://developer.valvesoftware.com/wiki/The_Ship"
			   title="The Ship - Valve Developer Community">
				The Ship
			</a>
		</td>
		<td>StaticPropLump_tに変更あり</td>
	</tr>
	<tr>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/Bloody_Good_Time"
			   title="Bloody Good Time - Valve Developer Community">
				Bloody Good Time
			</a>
		</td>
		<td>StaticPropLump_tに変更あり</td>
	</tr>
	<tr>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/Black_Mesa_(Source)"
			   title="Black Mesa (Source) - Valve Developer Community">
				Black Mesa: Source
			</a>
		</td>
		<td>StaticPropLump_tに変更あり（version = 11）</td>
	</tr>
	<tr>
		<td rowspan="7">21</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png"
			     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_2"
			   title="Left 4 Dead 2 - Valve Developer Community">
				Left 4 Dead 2
			</a>
		</td>
		<td>lump_tと古いdbrushside_t(?)に変更あり</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/c/c9/AS-16px.png"
			     alt="Alien Swarm Icon" title="Alien Swarm Icon">
			<a href="https://developer.valvesoftware.com/wiki/Alien_Swarm"
			   title="Alien Swarm - Valve Developer Community">
				Alien Swarm
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/3/35/Csgo.png"
			     alt="Counter-Strike: Global Offensive Icon" title="Counter-Strike: Global Offensive Icon">
			<a href="https://developer.valvesoftware.com/wiki/Counter-Strike:_Global_Offensive"
			   title="Counter-Strike: Global Offensive - Valve Developer Community">
				Counter-Strike: Global Offensive
			</a>
		</td>
		<td>StaticPropLump_tに変更あり（version = 10）</td>
	</tr>
	<tr>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/2/2d/Dear_Esther.png"
			     alt="Dear Esther Icon" title="Dear Esther Icon">
			<a href="https://developer.valvesoftware.com/wiki/Dear_Esther"
			   title="Dear Esther - Valve Developer Community">
				Dear Esther
			</a>
		</td>
		<td>StaticPropLump_tに変更あり</td>
	</tr>
	<tr>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/Insurgency"
			   title="Insurgency - Valve Developer Community">
				Insurgency
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>
			<a href="https://developer.valvesoftware.com/wiki/The_Stanley_Parable"
			   title="The Stanley Parable - Valve Developer Community">
				The Stanley Parable
			</a>
		</td>
		<td></td>
	</tr>
	<tr>
		<td>Tactical Intervention</td>
		<td>256ビット XOR暗号化</td>
	</tr>
	<tr>
		<td>22</td>
		<td rowspan="2">
			<img src="https://developer.valvesoftware.com/w/images/0/0b/Dota2-16px.png"
			     alt="Dota 2 Icon" title="Dota 2 Icon">
			<a href="https://developer.valvesoftware.com/wiki/Dota_2"
			   title="Dota 2 - Valve Developer Community">
				Dota 2
			</a>
		</td>
		<td>早期ベータ版 dbrushside_tとddispinfo_tに変更あり</td>
	</tr>
	<tr>
		<td>23</td>
		<td>dbrushside_t、ddispinfo_t、doverlay_tに変更あり</td>
	</tr>
	<tr>
		<td>27</td>
		<td>Contagion</td>
		<td></td>
	</tr>
	<tr>
		<td>29</td>
		<td>Titanfall</td>
		<td>大幅に変更されている</td>
	</tr>
</table>


ゲーム固有のBSP形式の詳細については、[Source BSP File Format/Game-Specific]を参照してください。  


<h3 id="lumpstructures">Lump構造体</h3>  

次に、16バイトの`lump_t`構造体の配列が続きます。HEADER_LUMPSは64と定義されているため、全部で64個の要素があります。ただし、ゲームやバージョンによっては定義されていないものや空のものもあります。  

各`lump_t`は*bspfile.h*で定義されています。  
``` C++
struct lump_t
{
	int	fileofs;	// ファイルへのオフセット（バイト単位）
	int	filelen;	// Lumpの長さ（バイト単位）
	int	version;	// Lumpの形式のバージョン
	char	fourCC[4];	// Lump識別子
};
```
最初の2つのint値はbspファイルの先頭からのバイトオフセットとそのLumpに含まれるデータブロックのバイト長を表します。続いて、そのLumpの形式のバージョン番号（通常は0）と、通常0, 0, 0, 0である4バイトの識別子があります。圧縮されたLumpの場合、fourCCには非圧縮のLumpデータサイズが整数で示されています（詳細は[Lumpの圧縮](#lumpcompression)の節を参照）。lump_t配列の未使用のメンバについてはすべての要素が0に設定されています。  
Lumpのオフセット（と、それに対応するデータ）は最も近い4バイトの境界に切り上げられています。ただし、Lumpの長さについてはこの限りではありません。  


<h3 id="lumptypes">Lumpの種類</h3>

`lump_t`配列が指すデータの種類は、配列内の位置によって定義されます。例えば、配列の最初のLump **(Lump 0)** は常にBSPファイルのエンティティデータです（下表参照）。BSPファイル内の実際のデータの位置は、そのLumpのoffsetとlengthによって定義されるため、ファイル内で特定の順番に並んでいる必要はありません。例えば、エンティティデータはLump配列の最初にあるにも関わらず通常はBSPファイルの最後に格納されます。したがって、lump_tヘッダの配列はLumpデータに関するディレクトリのようなものであり、Lumpデータはファイル内を自由に配置することができます。  

配列内におけるLumpの順番は以下のように定義されます。  

<table>
	<tr>
		<th align="center">番号</th>
		<th align="center">エンジン</th>
		<th align="center">名前</th>
		<th align="center">目的</th>
	</tr>
	<tr><td>0</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_ENTITIES</td>
		<td>マップ上のエンティティ</td>
	</tr>
	<tr><td>1</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PLANES</td>
		<td>平面の配列</td>
	</tr>
	<tr><td>2</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_TEXDATA</td>
		<td>テクスチャ名へのインデックス</td>
	</tr>
	<tr><td>3</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_VERTEXES</td>
		<td>頂点の配列</td>
	</tr>
	<tr><td>4</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_VISIBLITY</td>
		<td>圧縮された可視性に関するビット配列</td>
	</tr>
	<tr><td>5</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_NODES</td>
		<td>BSP木のノード</td>
	</tr>
	<tr><td>6</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_TEXINFO</td>
		<td>面のテクスチャ配列</td>
	</tr>
	<tr><td>7</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_FACES</td>
		<td>面の配列</td>
	</tr>
	<tr><td>8</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LIGHTING</td>
		<td>ライトマップのサンプル</td>
	</tr>
	<tr><td>9</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_OCCLUSION</td>
		<td>オクルージョンポリゴンと頂点</td>
	</tr>
	<tr><td>10</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LEAFS</td>
		<td>BSP木の葉ノード（リーフ）</td>
	</tr>
	<tr><td>11</td><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_FACEIDS</td>
		<td>dfaceとHammerの面IDの関連付けと、<br><a href="">detail prop</a>を配置する時の乱数の種に使用</td>
	</tr>
	<tr><td>12</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Orange Box - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_EDGES</td>
		<td>辺の配列</td>
	</tr>
	<tr><td>13</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_SURFEDGES</td>
		<td>辺へのインデックス</td>
	</tr>
	<tr><td>14</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_MODELS</td>
		<td>ブラシモデル</td>
	</tr>
	<tr><td>15</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_WORLDLIGHTS</td>
		<td>エンティティLumpから変換された内部の<br>ワールドライト(?)</td>
	</tr>
	<tr><td>16</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LEAFFACES</td>
		<td>各leafの面へのインデックス</td>
	</tr>
	<tr><td>17</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LEAFBRUSHES</td>
		<td>各leafのブラシへのインデックス</td>
	</tr>
	<tr><td>18</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_BRUSHES</td>
		<td>ブラシの配列</td>
	</tr>
	<tr><td>19</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_BRUSHSIDES</td>
		<td>ブラシサイドの配列</td>
	</tr>
	<tr><td>20</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_AREAS</td>
		<td>エリアの配列</td>
	</tr>
	<tr><td>21</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_AREAPORTALS</td>
		<td>エリア間のポータル</td>
	</tr>
	<tr><td rowspan="3">22</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PORTALS</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/2/2e/Confirm.png" alt="Confirm" title="Confirm">
			<b>確認:</b> 隣接するポリゴン間の境界を定義<br>するポリゴン?
		</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_UNUSED0</td>
		<td>未使用</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PROPCOLLISION</td>
		<td>静的プロップの凸包リスト</td>
	</tr>
	<tr><td rowspan="3">23</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_CLUSTERS</td>
		<td>プレイヤーが入れるリーフ</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_UNUSED1</td>
		<td>未使用</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PROPHULLS</td>
		<td>静的プロップの凸包</td>
	</tr>
	<tr><td rowspan="3">24</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PORTALVERTS</td>
		<td>ポータルポリゴンの頂点</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_UNUSED2</td>
		<td>未使用</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PROPHULLVERTS</td>
		<td>静的プロップの凸包の頂点</td>
	</tr>
	<tr><td rowspan="3">25</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_CLUSTERPORTALS</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/2/2e/Confirm.png" alt="Confirm" title="Confirm">
			<b>確認:</b> 隣接するクラスタ間の境界を定義<br>するポリゴン?
		</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_UNUSED3</td>
		<td>未使用</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PROPTRIS</td>
		<td>静的プロップの凸包三角形<br>へのインデックスとカウント(?)</td>
	</tr>
	<tr><td>26</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_DISPINFO</td>
		<td>Displacementの面配列</td>
	</tr>
	<tr><td>27</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_ORIGINALFACES</td>
		<td>分割前のブラシ面の配列</td>
	</tr>
	<tr><td>28</td><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_PHYSDISP</td>
		<td>Displacementの物理衝突判定データ</td>
	</tr>
	<tr><td>29</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PHYSCOLLIDE</td>
		<td>物理衝突判定データ</td>
	</tr>
	<tr><td>30</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_VERTNORMALS</td>
		<td>面の平面の法線(?)</td>
	</tr>
	<tr><td>31</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_VERTNORMALINDICES</td>
		<td>面の平面の法線(?)へのインデックス</td>
	</tr>
	<tr><td>32</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_DISP_LIGHTMAP_ALPHAS</td>
		<td>Displacementのライトマップのアルファ(?)<br>（Source 2006からは未使用もしくは空）</td>
	</tr>
	<tr><td>33</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_DISP_VERTS</td>
		<td>Displacementの表面メッシュの頂点</td>
	</tr>
	<tr><td>34</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_DISP_LIGHTMAP_SAMPLE_POSITIONS</td>
		<td>Displacementのライトマップサンプル位置</td>
	</tr>
	<tr><td>35</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_GAME_LUMP</td>
		<td>ゲーム固有のデータLump</td>
	</tr>
	<tr><td>36</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LEAFWATERDATA</td>
		<td>水中のリーフのためのデータ</td>
	</tr>
	<tr><td>37</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PRIMITIVES</td>
		<td>水ポリゴンデータ</td>
	</tr>
	<tr><td>38</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PRIMVERTS</td>
		<td>水ポリゴン頂点</td>
	</tr>
	<tr><td>39</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PRIMINDICES</td>
		<td>水ポリゴン頂点へのインデックスの配列</td>
	</tr>
	<tr><td>40</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PAKFILE</td>
		<td>埋め込まれた非圧縮のZip形式のファイル</td>
	</tr>
	<tr><td>41</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_CLIPPORTALVERTS</td>
		<td>クリップされたポータルポリゴンの頂点</td>
	</tr>
	<tr><td>42</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_CUBEMAPS</td>
		<td>env_cubemapの位置の配列</td>
	</tr>
	<tr><td>43</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_TEXDATA_STRING_DATA</td>
		<td>テクスチャ名のデータ</td>
	</tr>
	<tr><td>44</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_TEXDATA_STRING_TABLE</td>
		<td>LUMP_TEXDATA_STRING_DATA<br>へのインデックス配列</td>
	</tr>
	<tr><td>45</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_OVERLAYS</td>
		<td>info_overlayのデータ配列</td>
	</tr>
	<tr><td>46</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_LEAFMINDISTTOWATER</td>
		<td>リーフから水までの距離</td>
	</tr>
	<tr><td>47</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_FACE_MACRO_TEXTURE_INFO</td>
		<td>面のマクロテクスチャ情報</td>
	</tr>
	<tr><td>48</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_DISP_TRIS</td>
		<td>Displacementの表面の三角形</td>
	</tr>
	<tr><td rowspan="2">49</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source" 
		   title="Source - Valve Developer Community">Source 2004</a>
		</td>
		<td>LUMP_PHYSCOLLIDESURFACE</td>
		<td>圧縮されたWin32固有のHavok地形面衝突<br>判定データ(?) 廃止されたため使われない</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PROP_BLOB</td>
		<td>静的プロップの三角形と文字列のデータ(?)</td>
	</tr>
	<tr><td>50</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_WATEROVERLAYS</td>
		<td>
			<img src="https://developer.valvesoftware.com/w/images/2/2e/Confirm.png" alt="Confirm" title="Confirm">
			<b>確認:</b> 水面の上にinfo_overlayが存在する?
		</td>
	</tr>
	<tr><td rowspan="2">51</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LIGHTMAPPAGES</td>
		<td>Xboxでのライティング関係の別の実装</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_LEAF_AMBIENT_INDEX_HDR</td>
		<td>LUMP_LEAF_AMBIENT_LIGHTING_HDRへの<br>インデックス</td>
	</tr>
	<tr><td rowspan="2">52</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LIGHTMAPPAGEINFOS</td>
		<td>Xboxでのライティング関係の別の<br>インデックス</td>
	</tr>
	<tr><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_LEAF_AMBIENT_INDEX</td>
		<td>LUMP_LEAF_AMBIENT_LIGHTINGへの<br>インデックス</td>
	</tr>
	<tr><td>53</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LIGHTING_HDR</td>
		<td>HDRライトマップサンプル</td>
	</tr>
	<tr><td>54</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_WORLDLIGHTS_HDR</td>
		<td>エンティティLumpから変換された内部の<br>HDRワールドライト</td>
	</tr>
	<tr><td>55</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LEAF_AMBIENT_LIGHTING_HDR</td>
		<td>リーフごとのアンビエントライト<br>サンプル（HDR）</td>
	</tr>
	<tr><td>56</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LEAF_AMBIENT_LIGHTING</td>
		<td>リーフごとのアンビエントライト<br>サンプル（LDR）</td>
	</tr>
	<tr><td>57</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_XZIPPAKFILE</td>
		<td>Xboxで、XZipバージョンのpakファイル<br>廃止されている</td>
	</tr>
	<tr><td>58</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_FACES_HDR</td>
		<td><i>HDRマップは異なる面データを持つことが<br>ある</i></td>
	</tr>
	<tr><td>59</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_MAP_FLAGS</td>
		<td>レベル全体の拡張フラグ<br>すべてのレベルに存在するわけではない</td>
	</tr>
	<tr><td>60</td><td>
		<img src="https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png" 
		     alt="Team Fortress 2 Icon" title="Team Fortress 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2007" 
		   title="Orange Box - Valve Developer Community">Source 2007</a>
		</td>
		<td>LUMP_OVERLAY_FADES</td>
		<td>オーバーレイのフェード距離</td>
	</tr>
	<tr><td>61</td><td>
		<img src="https://developer.valvesoftware.com/w/images/c/c0/L4D-16px.png" 
		     alt="Left 4 Dead Icon" title="Left 4 Dead Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2008</a>
		</td>
		<td>LUMP_OVERLAY_SYSTEM_LEVELS</td>
		<td>システムレベル設定（このオーバーレイを<br>描画するための最小/最大CPU&GPU）</td>
	</tr>
	<tr><td>62</td><td>
		<img src="https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png" 
		     alt="Left 4 Dead 2 Icon" title="Left 4 Dead 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Left_4_Dead_(engine_branch)" 
		   title="Left 4 Dead (engine branch) - Valve Developer Community">Source 2009</a>
		</td>
		<td>LUMP_PHYSLEVEL</td>
		<td><b>To do</b></td>
	</tr>
	<tr><td>63</td><td>
		<img src="https://developer.valvesoftware.com/w/images/c/c9/AS-16px.png" 
		     alt="Alien Swarm Icon" title="Alien Swarm Icon">
		<a href="https://developer.valvesoftware.com/wiki/Alien_Swarm_(engine_branch)" 
		   title="Alien Swarm (engine branch) - Valve Developer Community">Source 2010</a>
		</td>
		<td>LUMP_MULTIBLEND</td>
		<td>Displacementのマルチブレンド情報</td>
	</tr>
</table>

Lump 53～56はバージョン20以上のBSPファイルでのみ使用されます。Lump 22～25はバージョン20では使用されていません。
既知の要素について、データLumpの構造を以下に説明します。Lumpの多くは単純な構造の配列で、またいくつかはその内容に応じて可変長です。各データの最大サイズまたは要素数は*bspfile.h*でもMAX_MAP_\*として定義されています。

最後に、ヘッダはマップリビジョン番号を表すint値で終わります。この数値は、マップのvmfファイルのリビジョン番号（`mapversion`）に基いています。これは、Hammer Editorで保存される度に増加する値です。

ヘッダのすぐ後ろには、最初のデータLumpが続いています。実際には最初のデータLumpはLump 1（平面データ配列）ですが、これは前のリスト中ののoffsetフィールドで指定された任意のLumpです。


<h3 id="lumpcompression">Lumpの圧縮</h3>

PlayStation 3やXbox 360などのコンソールプラットフォーム用のBSPファイルは通常、LZMAで圧縮されたLumpを格納しています。この場合、Lumpデータは次のヘッダで始まります（*public/tier1/lzmaDecoder.h*より）。
``` C++
struct lzma_header_t
{
	unsigned int	id;
	unsigned int	actualSize;		// 常にリトルエンディアン
	unsigned int	lzmaSize;		// 常にリトルエンディアン
	unsigned char	properties[5];
};
```
`id`は以下のように定義されています。
``` C++
// リトルエンディアン "LZMA"
#define LZMA_ID	(('A'<<24)|('M'<<16)|('Z'<<8)|('L'))
```
圧縮に関して、2つのスペシャルケースがあります。`LUMP_PAKFILE`は決して圧縮されず、`LUMP_GAME_LUMP`の各Gamelumpは個別に圧縮されます。圧縮されたGamelumpのサイズは、現在のGamelumpのオフセットを次のもののオフセットから引くことで決めることができます。このため、最後のGamelumpは常にオフセットを含む空のダミーです。
<br><br>

<h1 id="lumpheader">Lump</h1>
<h2 id="plane">平面</h3>

BSPジオメトリの基礎は、BSP木構造全体の分割面として使われる平面によって定義されます。

平面Lump **(Lump 1)** は`dplane_t`構造体の配列です。
``` C++
struct dplane_t
{
	Vector	normal;	// 法線ベクトル
	float	dist;	// 原点からの距離
	int	type;	// 平面軸識別子(?)
};
```
[Vector]型は以下のように定義される3次元ベクトルです。
``` C++
struct Vector
{
	float x;
	float y;
	float z;
};
```
float型は4バイトの長さを持つため、1つの平面につきサイズは20バイトで、平面Lumpのサイズは20の倍数です。

平面は、その面に垂直な単位ベクトル（長さが1.0のベクトル）である法線ベクトルを示す要素`normal`によって表現されます。平面の位置は、マップの原点（0, 0, 0）から平面上の最も近い点までの距離`dist`によって与えられます。

数学的には、平面は方程式  
	Ax + By + Cz = D  
を満たす点(x, y, z)の集合として記述されます。ここで、点(A, B, C)は平面の法線ベクトル、すなわち`normal`で、Dは`dist`です。各平面は無限に広がっていて、マップ全体を平面の上（F = 0）、平面の前（F > 0）、平面の後ろ（F < 0）の3つに分割します。

平面は特定の向きを持ち、それによって平面の前と後ろが対応づけられます。また、平面の向きを反転するにはA、B、C、Dの各要素を反転します。

この構造体の`type`メンバは座標軸に垂直な平面を示すフラグが含まれているようですが、通常は使用されません。

マップには最大で65536枚の平面が存在します（`MAX_MAP_PLANES`）。


<h2 id="vertex">頂点</h2>

頂点Lump **(Lump 3)** はマップジオメトリのブラシのすべての頂点（コーナー）座標の配列です。各頂点は3つの浮動小数点数からなる`Vector`型で表され、1つにつき12バイトのサイズを持ちます。

2つの面で頂点が正確に一致する場合、頂点はそれらの面で共有されることに注意してください。

マップには最大で65536個の頂点が存在します（`MAX_MAP_VERTS`）。


<h2 id="edge">辺</h2>

辺Lump **(Lump 12)** はdedge_t構造体の配列です。
``` C++
struct dedge_t
{
	unsigned short	v[2];	// 頂点インデックス
};
```
各辺は単なる頂点インデックス（頂点Lumpの配列へのインデックス）の組です。辺は2頂点間の直線として定義されます。通常、辺LumpはSurfedge配列を介して参照されます（下記）。

頂点と同じように、隣接する面の間で辺を共有することができます。

マップには最大で256000本の辺が存在します（`MAX_MAP_EDGES`）。


<h2 id="surfedge">Surfedge</h2>

「Surface edge lump」の略であると推定されるSurfedge Lump **(Lump 13)** は（符号付きの）整数の配列です。Surfedgeはやや複雑な方法で辺の配列を参照するために使用されます。Surfedge配列は正または負の値を取ります。この数値の絶対値は辺の配列へのインデックスです。もしこの数値が正の数である場合は、辺が最初の頂点から2番めの頂点の方向に定義されていることを意味しています。負の数の場合は、2番めの頂点から最初の頂点へ向かうように定義されています。

この方法により、Surfedge配列は辺を特定の方向に向かうように参照できます（辺を方向づけする理由については、以下の面の項目を参照してください）。

1つのマップに付き512000個のSurfedgeまでという制限があります（`MAX_MAP_SURFEDGES`）。ただし、Surfedgeの数は必ずしもマップの辺の数と同じではありません。


<h2 id="faceandoriginalface">面と元の面</h2>

面Lump **(Lump 7)** にはプレイヤーの視点をレンダリングするためにゲームエンジンによって使用されるマップの主要なジオメトリを含まれます。面LumpにはBSP分割処理を行った後の面が含まれています。したがって、面Lumpに含まれる面はHammer Editorで作成されたブラシの面には直接対応しません。また、面は常に平らな凸多角形ですが、同一直線上にある頂点を含むことができます。

面Lumpはマップファイルの中でも複雑な構造の1つです。このLumpは、56バイトの長さを持つ`dface_t`の配列です。
``` C++
struct dface_t
{
	unsigned short	planenum;			// 平面番号
	byte		side;				// 面の向きと平面番号による平面の向きが反対なら1
	byte		onNode;				// ノードにあるなら1、リーフにあるなら0
	int		firstedge;			// Surfedge Lumpへのインデックス
	short		numedges;			// 面を構成するSurfedgeの数
	short		texinfo;			// テクスチャ情報
	short		dispinfo;			// Displacement情報
	short		surfaceFogVolumeID;		// ?
	byte		styles[4];			// 切り替え可能なライティングの情報
	int		lightofs;			// ライトマップLumpへのオフセット
	float		area;				// 面の面積（単位はHammer units^2）
	int		LightmapTextureMinsInLuxels[2];	// テクスチャライティング情報
	int		LightmapTextureSizeInLuxels[2];	// テクスチャライティング情報
	int		origFace;			// この面の分割元となった元の面
	unsigned short	numPrims;			// プリミティブ
	unsigned short	firstPrimID;
	unsigned int	smoothingGroups;		// ライトマップスムージンググループ
};
```
最初のメンバである`planenum`は平面番号（この面と位置の合う平面Lumpへのインデックス）です。この時、参照された平面がこの面と同じ方向を向いている場合、`side`は0です。そうでない場合は、非ゼロです。

[![facexray](https://developer.valvesoftware.com/w/images/f/ff/Bsp_geometry_s.gif)](https://developer.valvesoftware.com/wiki/File:Bsp_geometry_s.gif)

(3つのタイプのジオメトリデータを用いたd1_trainstation_01のX線ビューをアニメーションしたもの。[フルサイズで見るにはこちら。](https://developer.valvesoftware.com/w/images/f/f8/Bsp_geometry.gif))

`firstedge`はSurfedge配列へのインデックスです。`firstedge`番目のSurfedgeからそれに続く`numedges`個のSurfedgeが面を構成する辺を定義します。上で述べたように、Surfedge配列の値が正か負かは、辺の配列に格納された対応する頂点の組をどちらの方向にたどるかを示します。したがって、面を構成する頂点は面に向かって見ると時計回りの順になるように参照されます。これによって面のレンダリングが容易になり、視点から離れた面を高速に間引きすることができます。

`texinfo`はTexInfo配列へのインデックスで（以下参照）、面に描かれるべきテクスチャを示します。`dispinfo`はDispInfo配列へのインデックスで、0以上の場合は面はDisplacementであり、Displacementの境界を定義します。そうでない場合は、`dispinfo`は-1です。`surfaceFogVolumeID`はプレイヤーの視点が水中にあるか水面を見ている時にフォグを描画することに関係しているように思われます。

`origFace`はこの面の分割される元となった「元の面（Original Face）」へのインデックスです。`numPrims`と`firstPrimID`は「非ポリゴンプリミティブ」（以下参照）の描画に関連しています。`dface_t`構造体の他のメンバは、面のライティング情報を参照するために使用されます（下記のライティングLumpを参照）。

面の数は65536枚に制限されています（`MAX_MAP_FACES`）。

元の面Lump **(Lump 27)** は面Lumpと同じ構造を持ちますが、BSP分割処理が行われる前の面の配列が含まれています。したがって、これらの面は面の配列よりもマップのコンパイル前に存在する元のブラシ面に近く、また面の数はより少ないです。元の面の`origFace`はすべて0です。また、元の面の配列の要素数も最大で65536枚です。

面と元の面の両方がカリングされます。つまり、マップのコンパイル前に存在する多くの面（主にマップの境界より外側の方を向いている面）が配列から削除されます。


<h2 id="brushandbrushside">ブラシとブラシ側面</h2>

ブラシLump **(Lump 18)** にはコンパイル前のVMFファイルに含まれていたすべての[ブラシ]が含まれています。面とは異なり、ブラシは辺や頂点の代わりに平面を使う[空間領域構成法（CSG）]で定義されています。Source BSPファイルにはブラシとブラシ側面のLumpが存在するため、この情報が存在しないGoldSrcのファイルよりもかなり容易に逆コンパイルできます。このLumpは12バイトの長さを持つ`dbrush_t`構造体の配列です。
``` C++
struct dbrush_t
{
	int	firstside;	// 最初のブラシ側面（へのインデックス）
	int	numsides;	// ブラシ側面の数
	int	contents;	// コンテンツフラグ
};
```
最初のint値`firstside`はブラシ側面Lumpへのインデックスで、`firstside`番目のブラシ側面から`numsides`枚のブラシ側面がこのブラシのすべての側面を構成します。`contents`にはこのブラシの内容を決定するビットフラグが格納されています。値はビットOR演算されたもので、フラグは*public/bspflags.h*で定義されています。

<table>
	<tr>
		<th align="center">名前</th>
		<th align="center">値</th>
		<th align="center">備考</th>
	</tr>
	<tr><td><code>CONTENTS_EMPTY</code></td><td>0</td><td>空の空間</td></tr>
	<tr><td><code>CONTENTS_SOLID</code></td><td>0x1</td><td>固体の中では目が見えない</td></tr>
	<tr><td><code>CONTENTS_WINDOW</code></td><td>0x2</td><td>半透明だが、水ではない（つまりガラス）</td></tr>
	<tr><td><code>CONTENTS_AUX</code></td><td>0x4</td><td></td></tr>
	<tr><td><code>CONTENTS_GRATE</code></td><td>0x8</td><td>アルファテストされた「鉄格子」のテクスチャで、弾丸と視線は通過するが、固体は通過しない</td></tr>
	<tr><td><code>CONTENTS_SLIME</code></td><td>0x10</td><td></td></tr>
	<tr><td><code>CONTENTS_WATER</code></td><td>0x20</td><td></td></tr>
	<tr><td><code>CONTENTS_MIST</code></td><td>0x40</td><td></td></tr>
	<tr><td><code>CONTENTS_OPAQUE</code></td><td>0x80</td><td>AIの視線を遮断する</td></tr>
	<tr><td><code>CONTENTS_TESTFOGVOLUME</code></td><td>0x100</td><td>透けて見えないもの（固体ではないかもしれないにもかかわらず）</td></tr>
	<tr><td><code>CONTENTS_UNUSED</code></td><td>0x200</td><td>未使用</td></tr>
	<tr><td><code>CONTENTS_UNUSED6</code></td><td>0x400</td><td>未使用</td></tr>
	<tr><td><code>CONTENTS_TEAM1</code></td><td>0x800</td><td rowspan="2">チームごとにプレイヤーやオブジェクトの衝突判定を区別するために使用される</td></tr>
	<tr><td><code>CONTENTS_TEAM2</code></td><td>0x1000</td></tr>
	<tr><td><code>CONTENTS_IGNORE_NODRAW_OPAQUE</code></td><td>0x2000</td><td><code>SURF_NODRAW</code>を持つ面では<code>CONTENTS_OPAQUE</code>を無視する</td></tr>
	<tr><td><code>CONTENTS_MOVEABLE</code></td><td>0x4000</td><td><code>MOVETYPE_PUSH</code>のエンティティ（ドア、足場など）にあたる</td></tr>
	<tr><td><code>CONTENTS_AREAPORTAL</code></td><td>0x8000</td><td>以下のフラグは不可視で、ブラシを侵食しない</td></tr>
	<tr><td><code>CONTENTS_PLAYERCLIP</code></td><td>0x10000</td><td></td></tr>
	<tr><td><code>CONTENTS_MONSTERCLIP</code></td><td>0x20000</td><td></td></tr>
	<tr><td><code>CONTENTS_CURRENT_0</code></td><td>0x40000</td><td rowspan="6">CURRENT系は他のフラグに追加されるもので、複数追加されることもある</td></tr>
	<tr><td><code>CONTENTS_CURRENT_90</code></td><td>0x80000</td></tr>
	<tr><td><code>CONTENTS_CURRENT_180</code></td><td>0x100000</td></tr>
	<tr><td><code>CONTENTS_CURRENT_270</code></td><td>0x200000</td></tr>
	<tr><td><code>CONTENTS_CURRENT_UP</code></td><td>0x400000</td></tr>
	<tr><td><code>CONTENTS_CURRENT_DOWN</code></td><td>0x800000</td></tr>
	<tr><td><code>CONTENTS_ORIGIN</code></td><td>0x1000000</td><td>エンティティをBSP処理する前に削除される</td></tr>
	<tr><td><code>CONTENTS_MONSTER</code></td><td>0x2000000</td><td>ブラシに与えられるフラグではなく、ゲーム中に関わる</td></tr>
	<tr><td><code>CONTENTS_DEBRIS</code></td><td>0x4000000</td><td></td></tr>
	<tr><td><code>CONTENTS_DETAIL</code></td><td>0x8000000</td><td>VisLeaf処理の後にブラシに追加される</td></tr>
	<tr><td><code>CONTENTS_TRANSLUCENT</code></td><td>0x10000000</td><td>面に透明度がある場合に自動的に付与</td></tr>
	<tr><td><code>CONTENTS_LADDER</code></td><td>0x20000000</td><td></td></tr>
	<tr><td><code>CONTENTS_HITBOX</code></td><td>0x40000000</td><td>トレースで正確なヒットボックスを使うためのもの(?)</td></tr>
</table>

これらのフラグの一部は、以前のゲームエンジンから引き継がれているように見え、Sourceのマップでは使われていないものもあります。また、これらのフラグはマップのリーフの中身について説明するためにも使われます（下記参照）。`CONTENTS_DETAIL`フラグはマップのコンパイル前に[func_detail]エンティティであったブラシをマークするために使われます。

ブラシの配列の要素数は8192個に制限されています（`MAX_MAP_BRUSHES`）。

ブラシ側面Lump **(Lump 19)** は8バイトの構造体の配列です。
``` C++
struct dbrushside_t
{
	unsigned short	planenum;	// リーフの外側を向いた平面
	short		texinfo;	// テクスチャ情報
	short		dispinfo;	// Displacementの情報
	short		bevel;		// 側面が斜めなら1
};
```
`planenum`は平面の配列へのインデックスで、そのブラシ側面に対応する平面を示します。`texinfo`と`dispinfo`はテクスチャとDisplacement情報のLumpへの参照を示します。`bevel`は普通のブラシ側面なら0ですが、斜面の場合は1です（衝突判定に使われるものと思われます）。

面の配列とは異なり、ワールドの外を向いたブラシ側面はカリング（削除）されません。その代わり、コンパイル処理中にテクスチャ情報が`tools/[toolsnodraw]`に変更されます。ここで注意すべきことは、ブラシをレンダリングするのに使われる対応する面の配列の要素と、ブラシとブラシ側面とを関連付ける直接的な方法が存在しないことです。ブラシ側面はすべてのプレイヤーとワールドブラシとの衝突判定を計算するためにエンジンによって使われます（VPhysicsオブジェクトは代わりにLump 29が使われます）。

ブラシ側面は最大で65536枚です（`MAX_MAP_BRUSHSIDES`）。また、ブラシごとのブラシ側面の最大数は128枚です（`MAX_BRUSH_SIDES`）。


<h2 id="nodeandleaf">ノードとリーフ</h2>

ノード配列 **(Lump 5)** とリーフ配列 **(Lump 10)** はマップのBSP木（Binary Space Partition Tree）を定義します。BSP木はマップのジオメトリに対するプレイヤーの視点の位置と、可視性情報（下記参照）からマップのどの部分が描画されるかを素早く決定するためにエンジンによって使用されます。

ノードとリーフは木構造を構築します。各リーフはマップのボリュームを定義したものを表し、各ノードはすべての子ノードの総ボリュームを表します。

各ノードには必ず2つの子ノードがあり、子ノードは別のノードかリーフです。子ノードさらに2つの別の子ノードを持っていて、これは木のすべての分岐がリーフを指すまで続きます。また、各ノードは平面の配列内にある平面も参照します。プレイヤーの視点を決定する時、エンジンは視点がどのリーフの中にあるかを探します。この時、根ノード（ノード0）が参照している平面と視点の座標を比較し、平面の前に視点がある場合は最初の子ノードへ、後ろにある場合は2番めの子ノードへ移動します。比較するノードがリーフになるまでこれを続けることで、視点がどのリーフの中に存在するかを判断することができます。したがって、エンジンは視点の位置が見つかるまでBSP木を走査します。そして、リーフは親ノードの平面によって定義される重ならない凸なボリュームとしてマップのボリュームを分割します。

BSP木がどのように構築されるかについての詳細は、[「BSP for dummies」]の記事を参照してください。

ノード配列は32バイトの構造体で構成されています。
``` C++
struct dnode_t
{
	int		planenum;	// 平面の配列へのインデックス
	int		children[2];	// 負の値は -(leafs + 1)番目のリーフを表す
	short		mins[3];	// 視錐台カリング用
	short		maxs[3];
	unsigned short	firstface;	// 面の配列へのインデックス
	unsigned short	numfaces;	// 両面をカウントする
	short		area;		// すべての子ノードが同じエリアの場合はエリアへのインデックス
					// そうでない場合は-1
	short		paddding;	// 32バイトの長さを持つpad(?)
};
```
`planenum`は平面の配列の要素を表します。`children[]`メンバはこのノードが持つ2つの子ノードです。もしこの値が正なら、ノードへのインデックスで、負なら、 *-1-child*はリーフの配列へのインデックスを表します（例えば、-100は99番目のリーフを参照します）。

`mins[]`と`maxs[]`メンバはノードを囲むバウンディングボックスの座標です。`firstface`と`numfaces`はこのノードに含まれているマップの面を表す面の配列へのインデックスです。0の場合は面が含まれていません。`area`の値はこのノードにおけるマップのエリアです（下記参照）。マップには最大65536個のノードが存在します（`MAX_MAP_NODES`）。

リーフ配列は要素が56バイトの長さを持つ配列です。
``` C++
struct dleaf_t
{
	int			contents;		// ブラシをすべてビットORしたもの（不必要？）
	short			cluster;		// リーフ内のクラスタの数
	short			area:9;			// リーフ内のエリアの数
	short			flags:7;		// フラグ
	short			mins[3];		// 視錐台カリング用
	short			maxs[3];
	unsigned short		firstleafface;		// リーフ面へのインデックス
	unsigned short		numleaffaces;
	unsigned short		firstleafbrush;		// リーフブラシへのインデックス
	unsigned short		numleafbrushes;
	short			leafWaterDataID;	// 水中でない場合は-1
 
	//!!! バージョン19以前のマップは以下のコメントブロックを外す
	/*
	CompressedLightCube	ambientLighting;	// エンティティ用 計算済みのライティング情報
	short			padding;		// 4バイトの境界のためのpadding(?)
	*/
};
```
リーフの構造は子と平面への参照を持たないことを除いてノードのそれと似ています。追加の要素として`contents`フラグ（前述のブラシLumpを参照）と`cluster`（クラスタの数、下記参照）があります。`contents`フラグはリーフ内のブラシの中身を示すフラグです。`area`と`flags`は16ビットの空間を共有していて、下位9ビットが`area`、上位7ビットが`flags`です。`area`はエリアナンバーを、`flags`はリーフに関するフラグを示します。`firstleafface`と`numleaffaces`はリーフ面の配列へのインデックスで、リーフ内に面がある時にどの面があるかを表します。`firstleafbrush`と`numleafbrushes`も同様にリーフブラシの配列を通してリーフ内にあるブラシへのインデックスを表します。

`ambientLighitng`という要素は24バイトの`CompressedLightCube`構造体で、リーフ内にあるオブジェクトのライティングに関するものです。バージョン17のBSPファイルはdleaf_t構造体がアンビエントライティングのデータを省くように変更されていて、リーフごとのサイズが32バイトになっています。同じ構造体はバージョン20のBSPファイルでも用いられていて、LDRとHDRのためのアンビエントライティングの情報はおそらく新しいLumpであるLump 55とLump56に格納されています。

すべてのリーフは凸多面体で、親ノードの平面により定義されます。リーフは重ならず、マップ内の任意の点はただ1つのリーフの中に属します。固体のブラシで満たされていないリーフはプレイヤーが入ることが可能で、そのようなリーフはクラスタ番号が設定されています。これは可視性の情報と一緒に使用されます（下記）。

マップには通常複数の独立したBSP木が存在します。各木はモデル配列（下記）の要素に対応し、モデル配列の要素には各木の根ノードへの参照があります。最初のBSP木はworldspawnでモデルで、レベル全体のジオメトリです。続くBSP木はマップの各ブラシエンティティのモデルです。

BSP木の作成はマップのコンパイルの第一段階でVBSPプログラムにより行われます。マップの制作者によるHINTブラシやfunc_detailの使用と、すべてのブラシの注意深い配置は、どのようにBSP木が作られマップがリーフに分割されるかに影響を及ぼすことがあります。


<h2 id="leaffaceandleafbrush">リーフ面とリーフブラシ</h2>

リーフ面Lump **(Lump 16)** はunsigned short型の配列で、各要素は面の配列へのインデックスです。リーフブラシLump（これもunsigned short型の配列です） **(Lump 17)** も同様のことをブラシに対して行います。これらの最大サイズはどちらも65536個です（`MAX_MAP_LEAFFACES`、`MAX_MAP_LEAFBRUSHES`）。


<h2 id="textures">テクスチャ</h2>

マップのテクスチャ情報は様々なLumpに分割されています。Texinfo Lumpが最も基本的なもので、面やブラシ側面の配列から参照され、他のテクスチャ関係のLumpを参照します。


<h3 id="texinfo">Texinfo</h3>

Texinfo Lump **(Lump 6)** は`texinfo_t`構造体の配列です。
``` C++
struct texinfo_t
{
	float	textureVecs[2][4];	// [s/t][xyz オフセット]
	float	lightmapVecs[2][4];	// [s/t][xyz オフセット] - 長さの単位はtexels/area
	int	flags;			// miptex フラグ オーバーライド
	int	texdata;		// テクスチャ名やサイズなどへのポインタ
}
```
各Texinfoのサイズは72バイトです。

最初のfloat型配列は、ワールドジオメトリ上にレンダリングされる時のテクスチャの方向とスケーリングを表す2つのベクトルです。 2つのベクトル**s**、**t**は、テクスチャピクセル座標空間における左から右へ、下から上への方向をワールドにマッピングするものです。各ベクトルには、x、y、z成分と、ワールドに対するテクスチャの「シフト」のオフセットがあります。 ベクトルの長さは、各方向へのテクスチャのスケーリングを表します。

テクスチャピクセル（または[テクセル]）の2次元座標（[u, v]）は、次式によって面上の点のワールド座標（x、y、z）にマッピングされる。

<i>u = tv<sub>0,0</sub> * x + tv<sub>0,1</sub> * y + tv<sub>0,2</sub> * z + tv<sub>0,3</sub></i>

<i>v = tv<sub>1,0</sub> * x + tv<sub>1,1</sub> * y + tv<sub>1,2</sub> * z + tv<sub>1,3</sub></i>

（すなわち、その方向へのオフセットと頂点のベクトルとの内積です。ここで、tv<sub>A, B</sub>は<code>textureVecs[A][B]</code>です。）

さらに、（u, v）を計算してからグラフィックスカードに送るテクスチャ座標に変換するには、uとvをテクスチャの幅と高さでそれぞれ割ります。

`lightmapVecs`は、テクスチャのライトマップサンプルをワールドに同様にマッピングします。

`flags`には*bspflags.h*で定義されているビットフラグが含まれます。

名前 | 値 | 備考
:--- | :--- | :---
`SURF_LIGHT`	| 0x1    | 値は光の強さを保持する
`SURF_SKY2D`	| 0x2    | 描画しない、2Dスカイボックスを描画することを示し、3Dスカイボックスは描画しない
`SURF_SKY`	| 0x4    | 描画しないが、スカイボックスに追加する
`SURF_WARP`	| 0x8    | 乱流ワープ(?)
`SURF_TRANS`	| 0x10   | テクスチャは半透明
`SURF_NOPORTAL`	| 0x20   | この表面にはポータルを置くことができない
`SURF_TRIGGER`	| 0x40   | Xboxでオクルーダーに不具合が生じるため回避策としてトリガーサーフェスを消すためのもの
`SURF_NODRAW`	| 0x80   | テクスチャの参照をしないようにするもの
`SURF_HINT`	| 0x100  | BSPの境界面を作る
`SURF_SKIP`	| 0x200  | 完全に無視し、閉じてないブラシを可能にする
`SURF_NOLIGHT`	| 0x400  | ライティングを計算しない
`SURF_BUMPLIGHT`| 0x800  | バンプマップ付きの表面のために3つのライトマップを計算する
`SURF_NOSHADOWS`| 0x1000 | 影を落とさない
`SURF_NODECALS`	| 0x2000 | デカールを受け取らない
`SURF_NOCHOP`	| 0x4000 | この表面は分割されない
`SURF_HITBOX`	| 0x8000 | この面はヒットボックスの一部である

フラグは、テクスチャの.vmtファイルの内容から派生しているように見え、そのテクスチャの特殊なプロパティを指定します。


<h3 id="texdata">Texdata</h3>

最後に、`texdata`はTexdata配列へのインデックスで、実際のテクスチャを指定します。

Texinfoのインデックス（面やブラシ側面から参照される）は-1が与えられることがあります。これはテクスチャ情報がその面に関連付けられていないことを示し、テクスチャのタイプにSKIP、CLIP、もしくはINVISIBLEを指定したブラシ面をコンパイルすると発生します。

Texdata配列 **(Lump 2)** は以下の構造体で構成されています。
``` C++
struct dtexdata_t
{
	Vector	reflectivity;		// RGB反射率
	int	nameStringTableID;	// TexdataStringTableへのインデックス
	int	width, height;		// 元画像
	int	view_width, view_height;
};
```
`reflectivity`ベクトルは、マテリアルの.vtfファイルから取り出されたテクスチャの反射率のRGB成分に対応します。これは、テクスチャ表面からどのように光が反射するかのラジオシティ（照明）の計算におそらく使用されます。`nameStringTableID`はTexdataStringTableへのインデックスです（下記）。 他のメンバは、テクスチャのソースイメージに関連しています。


<h3 id="texstring">TexdataStringDataとTexdataStringTable</h3>

TexdataStringTable **(Lump 44)** はint型の配列で、TexdataStringData **(Lump 43)** へのインデックスです。TexdataStringData Lumpはヌル終端文字列で表されたテクスチャ名を連結したものです。

マップには最大12288個のTexinfoが存在します（`MAX_MAP_TEXINFO`）。Texdataの制限は最大2048個です（`MAX_MAP_TEXDATA`）。また、TexdataStringDataのサイズは最大256000バイトです（`MAX_MAP_TEXDATA_STRING_DATA`）。そして、テクスチャ名は最大128文字までです（`TEXTURE_NAME_LENGTH`）。


<h2 id="model">Model</h2>

ModelとはBSPファイル形式の用語で、しばしば「bmodel」とも呼ばれるブラシと面の集合のことです。Source SDKで「studiomodel」と呼ばれるHammer Editorで使用されるプロップモデルの方と混同しないように注意してください。

Model Lump **(Lump 14)** は24バイトの`dmodel_t`構造体で構成されています。
``` C++
struct dmodel_t
{
	Vector	mins, maxs;		// バウンディングボックス
	Vector	origin;			// サウンドやライティング用
	int	headnode;		// ノード配列へのインデックス
	int	firstface, numfaces;	// 面の配列へのインデックス
};
```
`mins`と`maxs`はModelのバウンディングボックスを示す点です。`origin`が設定されている場合、Modelの原点座標がその点であることを意味します。`headnode`はこのModelを表すBSP木の根ノードを示すノード配列へのインデックスです。`firstface`と`numfaces`は面の配列へのインデックスで、Modelを構成する面を示します。

この配列にある最初のModel（Model 0）は常に「worldspawn」（エンティティ以外のマップ全体のジオメトリとfunc_detailブラシの集合）です。続くModelはブラシエンティティに関連付けられるもので、エンティティLumpから参照されます。

マップには最大で1024個のModelが存在します（`MAX_MAP_MODELS`）。これにはworldspawnのModelも含みます。


<h2 id="visiblity">可視性</h2>

可視性Lump **(Lump 4)** はこれまでに解説したものとはやや異なった形式のLumpです。これを理解するためには、Source Engineの可視性システムがどのように機能するかについての議論が必要です。

[ノードとリーフ](#nodeandleaf)の節で述べたように、マップ内の任意の点はリーフと呼ばれる凸ボリュームに分類されます。マップ内の（外側の空間に触れていない）ブラシで覆われていないリーフは、プレイヤーの視点を含む可能性があります。このようなプレイヤーの入れるリーフ（*[visleaf]* とも呼ばれる）にはクラスタ番号が割り当てられます。Source BSPファイルでは1つの進入可能なリーフには1つのクラスタ番号が対応しています。

（用語がここでは少し分かりにくくなっています。「Quake 2 BSP File Format」の記事によると、Q2 Engineでは各クラスタに複数の隣接するリーフが存在する可能性があります。したがって、クラスタはリーフの集まりとみなせるためクラスタという名前で呼ばれています。この状況は、Sourceのマップをコンパイルする時にも発生することがあります。しかしながら、[VVIS]のコンパイル処理が終了した後、これらの隣接するリーフ（と、それらの親ノード）は通常1つのリーフに統合されます。古いSourceのマップ（[Counter-Strike: Global Offensive]より前）では、クラスタごとにリーフは1つしかないようですが、いくつかのCS:GOのマップでは最終的にコンパイルされたBSP内で、複数のリーフが同じクラスタに属することがあります。これは、ほとんどのCS:GOマップの3Dスカイボックスで特に起こると思われ、de_cbbleやde_nukeなどの最近改装されたマップの主なプレイ可能領域から見ることができます。）

各クラスタは、プレイヤーが存在する可能性を持つマップ内のボリュームです。マップを素早くレンダリングするために、ゲームエンジンは現在のクラスタから見えるクラスタについてのみジオメトリを描画します。プレイヤーのいるクラスタから完全に見えないクラスタを描く必要はありません。クラスタ間の可視性の計算はVVISコンパイルツールの役目で、その結果得られるデータは可視性Lumpに格納されます。

エンジンがあるクラスタが見えると認識すると、リーフデータはそのクラスタに存在するすべての面を参照し、結果としてそのクラスタの内容をレンダリングすることができます。

データはビットベクトルの配列として格納されます。各クラスタについて、他のどのクラスタが見えるかのリストはn番目のビットがn番目のクラスタに対応するビット配列（1なら見える、0なら見えない）として配列に格納されます。これはクラスタの[Potentially Visiblity Set（PVS）]として知られています。このデータはサイズが大きいため、各ビットベクトルは0ビットのランレングス符号化グループによって圧縮されます。

また、各クラスタに対して[Potentially Audible Set（PAS）]の配列も作成されます。これはあるクラスタで発生する音をどのクラスタで聞くことができるかを表します。PASは、現在のクラスタのPVS内にあるすべてのクラスタのPVSビットを統合することで作成されるようです。

可視性Lumpは以下のように定義されます。
``` C++
struct dvis_t
{
	int	numclusters;
	int	byteofs[numclusters][2]
};
```
最初のint値はマップ内の総クラスタ数です。その後にはint型配列が続き、この配列はLumpの先頭から各クラスタのPVSビット配列の開始点とPAS配列の開始点へのオフセットです。配列の後には圧縮されたビットベクトルがあります。

ランレングス圧縮の復号化は次のように動作します。特定のクラスタのPVSを見つけるためには、`byteofs[]`配列内のオフセットによって指定されたバイトから開始します。PVSバッファの現在のバイトが0の場合、次のバイトの値に8をかけた値がスキップするクラスタの数で、これはそのクラスタからは見えないクラスタです。現在のバイトが0でない場合は、設定されているビットがそのクラスタから見えるクラスタに対応します。これがマップ内の総クラスタ数の分だけ続きます。

ビットベクトルを展開するCコードの例は「Quake 2 BSP File Format」のドキュメントにあります。

可視性Lumpの最大サイズは0x1000000バイトです（`MAX_MAP_VISIBLITY`）。すなわち、16MBです。


<h2 id="entity">エンティティ</h2>

**関連:** [Patching levels with lump files]

エンティティLump **(Lump 0)** はエンティティのデータをコンパイル前の[VMF]ファイルにある[KeyValue]フォーマットに非常によく似た形式で格納するASCIIテキストバッファです。一般的な形式は次の通りです。
``` C++
{
	"world_maxs" "480 480 480"
	"world_mins" "-480 -480 -224"
	"maxpropscreenwidth" "-1"
	"skyname" "sky_wasteland02"
	"classname" "worldspawn"
}
{
	"origin" "-413.793 -384 -192"
	"angles" "0 0 0"
	"classname" "info_player_start"
}
{
	"model" "*1"
	"targetname" "secret_1"
	"origin" "424 -1536 1800"
	"Solidity" "1"
	"StartDisabled" "0"
	"InputFilter" "0"
	"disablereceiveshadows" "0"
	"disableshadows" "0"
	"rendermode" "0"
	"renderfx" "0"
	"rendercolor" "255 255 255"
	"renderamt" "255"
	"classname" "func_brush"
}
```
エンティティは中括弧（`[`と`]`）に囲まれて定義され、引用符で囲まれたキーと値のペアを各行にリストします。最初のエンティティは常に[worldspawn]です。`classname`プロパティはエンティティの種類を指定し、エンティティの名前がHammerで定義されていれば[`targetname`]がその名前を指します。`model`プロパティはアスタリスク（\*）で始まる場合は少し特殊で、続く数字は[ブラシエンティティ]のModelに対応するModel配列（上記）へのインデックスです。それ以外の場合、`model`プロパティはコンパイルされた[モデル]の名前を表します。他のキーと値のペアは、Hammerで設定されたエンティティのプロパティに対応します。

![noteicon] **注釈:**    エンティティのうちのいくつか（func_detail、env_cubemap、info_overlay、prop_staticを含む）は[内部的なもの]で、通常は[ワールド]に吸収されるためコンパイル処理中にエンティティLumpから削除されます。

Source Engineのバージョンに応じて、エンティティLumpには4096（Source 2004）から16384（Alien Swarm）個のエンティティを含むことができます（`MAX_MAP_ENTITIES`）。これらの制限は、エンジンの実際の[エンティティの制限]とは無関係です。各キーは最大32文字まで（`MAX_KEY`）、値は最大1024文字までです（`MAX_VALUE`）。


<h2 id="gamelump">Gamelump</h2>

Gamelump **(Lump 35)** は、Source Engineを使用したゲーム固有のマップデータに使用されるように意図されているため、以前に定義されたフォーマットを変更することなくファイルフォーマットを拡張することができます。GamelumpはGamelumpヘッダから始まります。
``` C++
struct dgamelumpheader_t
{
	int lumpCount;	// Gamelumpの数
	dgamelump_t gamelump[lumpCount];
};
```
Gamelumpディレクトリ配列は次のように定義されます。
``` C++
struct dgamelump_t
{
	int		id;		// Gamelump ID
	unsigned short	flags;		// フラグ
	unsigned short	version;	// Gamelumpバージョン
	int		fileofs;	// このGamelumpへのオフセット
	int		filelen;	// 長さ
};
```
Gamelumpはどのデータが格納されているかを定義する4バイトの`id`メンバによって識別され、データのバイト位置と長さは`fileofs`と`filelen`で与えられます。`fileofs`はBSPファイル先頭からの相対的なオフセットであり、Gamelumpのオフセットとは関係ないことに注意してください。ただし、Portal 2のコンソール版に関しては例外で、そこでは`fileofs`はGamelumpからの相対的なオフセットです。


<h3 id="staticprops">Static Prop</h3>

興味深いのは、「scrp」（ASCII表記、10進数で1936749168）というGamelump IDを用いる[prop_static]エンティティの格納に使われるGamelumpです。他のほとんどのエンティティとは異なり、Static PropはエンティティLumpに格納されません。Sourceで用いるGamelumpの形式は*public/gamebspfile.h*で定義されています。

Static Prop Gamelumpの最初の要素は辞書で、int値のカウントに続いてマップで使用されるモデル（Prop）の名前のリストがあります。
``` C++
struct StaticPropDictLump_t
{
	int	dictEntries;
	char	name[dictEntries];	// モデル名
};
```
`name`の要素はそれぞれ128文字で、ヌル文字でこの文字数まで埋められています。

辞書に続くのはリーフ配列です。
``` C++
struct StaticPropLeafLump_t
{
	int leafEntries;
	unsigned short	leaf[leafEntries];
};
```
おそらくこの配列は各Propが配置されているリーフを見つけるためにリーフLumpへインデックスをつけるために使用されます。また、prop_staticはいくつかのリーフにまたがって配置される場合があります。

次に、`StaticPropLump_t`構造体の数を示すint値に続いてその構造体が多く続きます。
``` C++
struct StaticPropLump_t
{
	// v4
	Vector		Origin;		 // 原点座標
	QAngle		Angles;		 // 方向（ピッチ ロール ヨー）
	unsigned short	PropType;	 // モデル名の辞書へのインデックス
	unsigned short	FirstLeaf;	 // リーフ配列へのインデックス
	unsigned short	LeafCount;
	unsigned char	Solid;		 // 固体性(?)の種類
	unsigned char	Flags;
	int		Skin;		 // モデルスキン番号
	float		FadeMinDist;
	float		FadeMaxDist;
	Vector		LightingOrigin;  // ライティング用
	// v5から
	float		ForcedFadeScale; // フェード距離スケール
	// v6とv7のみ
	unsigned short  MinDXLevel;      // 見えるための最低DirectXバージョン
	unsigned short  MaxDXLevel;      // 見えるための最大DirectXバージョン
        // v8から
	unsigned char   MinCPULevel;
	unsigned char   MaxCPULevel;
	unsigned char   MinGPULevel;
	unsigned char   MaxGPULevel;
        // v7から
        color32         DiffuseModulation; // インスタンスごとの、色とアルファのモジュレーション
        // v10から
        float           unknown; 
        // v9から
        bool            DisableX360;     // trueならXbox360で表示しない
};
```
Propの座標は`Origin`メンバによって、向き（ピッチ ロール ヨー）は3つのfloat型ベクトルである`Angles`で与えられます。`PropType`は上で与えられたPropモデル名の辞書へのインデックスです。他の要素はマップのBSP構造におけるPropの位置、ライティング、Hammerで設定された他のエンティティプロパティに対応します。Gamelumpの指定されたバージョン（`dgamelump_t.version`を参照）が十分に高い場合は、さらに要素が存在します（`ForcedFadeScale`など）。バージョン4と5のStatic Prop GamelumpはHL2の公式マップで使用されています。TF2からはバージョン6が現れました。Left 4 Deadのマップの一部ではバージョン7が使用され、[Zeno Clash]のマップでは変更が加えられたバージョン7が使用されています。バージョン8は主に[Left 4 Dead]で使用され、バージョン9は[Left 4 Dead 2]で使用されます。Tactical Interventionでは新しいバージョンであるバージョン10が現れます。


<h3 id="othergame">その他</h3>

Source BSPファイルで使われる他のGamelumpは、Detail prop gamelump（dprp）、Detail prop lighting gamelump（LDRはdplt、HDRはdplh）です。これらはDisplacementに特定のテクスチャを指定した場合に自動的に現れる[prop_detail]エンティティ（草むらなど）に使用されます。

Gamelumpのサイズには特に制限はないようです。


<h2 id="displacement">Displacement</h2>

DisplacementサーフェスはBSPファイルの中で最も複雑な部分であり、ここで説明されるのはそのフォーマットの一部のみです。DisplacementのデータはいくつかのデータLumpに分割されていますが、それらの基本的な参照はDispInfo Lump **(Lump 26)** によるものです。DispInfoは面、元の面、ブラシ側面の配列から参照されます。


<h3 id="dispinfo">DispInfo</h3>

``` C++
struct ddispinfo_t
{
	Vector			startPosition;		// 方向付けのために用いられる開始位置
	int			DispVertStart;		// LUMP_DISP_VERTSへのインデックス
	int			DispTriStart;		// LUMP_DISP_TRISへのインデックス
	int			power;			// サーフェスのサイズを示す（2^power - 1）
	int			minTess;		// 許容される最小のテセレーション(?)
	float			smoothingAngle;		// ライティング スムージング角度
	int			contents;		// サーフェス コンテンツ
	unsigned short		MapFace;		// このDisplacementがどの面から得られたかを示すインデックス
	int			LightmapAlphaStart;	// ddisplightmapalphaへのインデックス
	int			LightmapSamplePositionStart;	// LUMP_DISP_LIGHTMAP_SAMPLE_POSITIONSへのインデックス
	CDispNeighbor		EdgeNeighbors[4];	// NEIGHBOREDGE_ の定義によりインデックスされる
	CDispCornerNeighbors	CornerNeighbors[4];	// CORNER_ の定義によりインデックスされる
	unsigned int		AllowedVerts[10];	// アクティブな頂点(?)
};
```
この構造体は176バイトの長さを持ちます。`startPosition`はDisplacementの最初のコーナーの座標です。`DispVertStart`と`DispTriStart`はDispVerts LumpとDispTris Lumpへのインデックスです。`power`はDisplacementの分割数を表します。許容値は2、3、4で、これらの値はDisplacementの各辺を4，8、16本に分割することに対応しています。この構造体は`EdgeNeighbors`および`CornerNeighbors`メンバを介してこのDisplacementの側面や角に隣接するDisplacementも参照します。隣接するDisplacementの順序には複雑な規則があります。詳細は*bspfile.h*のコメントを参照してください。`MapFace`は面の配列へのインデックスで、このDisplacementに変換される元になった面です。この面にはテクスチャ、Displacement全体の物理的位置、Displacementの境界を設定するために使用されます。


<h3 id="dispverts">DispVerts</h3>

DispVerts Lump **(Lump 33)** にはDisplacementの頂点データが含まれていて、次のように与えられます。
``` C++
struct dDispVert
{
	Vector	vec;	// Displacementのボリュームを定めるベクトル場
	float	dist;	// Displacement距離.
	float	alpha;	// 「頂点ごとの」アルファ値
};
```
`vec`はDisplacementの各頂点について、元の（平坦な）位置からの変位を表すベクトルを正規化したものです。`dist`は変位の距離です。`alpha`はその頂点でのテクスチャのアルファブレンド値です。

<code>power</code>が<i>p</i>のDisplacementは、DispVertStartから(2<sup><i>p</i></sup> + 1)<sup>2</sup>個のDispVertsを参照します。


<h3 id="disptris">DispTris</h3>

DispTris Lump **(Lump 48)** にはDisplacementのメッシュの特定の三角形のプロパティに関する「三角形タグ」もしくはフラグが含まれています。
``` C++
struct dDispTri
{
	unsigned short Tags;	// Displacementの三角形タグ
};
```
フラグが示すものは以下の通りです。

名前 | 値
:--- | :---
`DISPTRI_TAG_SURFACE` | 0x1
`DISPTRI_TAG_WALKABLE` | 0x2
`DISPTRI_TAG_BUILDABLE` | 0x4
`DISPTRI_FLAG_SURFPROP1` | 0x8
`DISPTRI_FLAG_SURFPROP2` | 0x10

<code>power</code>が<i>p</i>のDisplacementには2×(2<sup><i>p</i></sup>)<sup>2</sup>個のDispTrisがあります。それらはおそらくその位置が歩行可能かどうかなど、Displacementを構成する各三角形のプロパティを示すために使用されます。

DispInfoは1つのマップにつき2048個の制限があり、DispVertsとDispTrisの制限は2048個すべてのDisplacementの`power`が4である場合の個数に制限されています（つまり、もっとも細かく分割された場合の個数です）。

Displacementに関連する他のデータは、DispLightmapAlphas Lump **(Lump 32)** とDispLightmapSamplePos Lump **(Lump 34)** であり、Displacementのライティングに関連していると思われます。


<h2 id="pakfile">Pakfile</h2>

Pakfile Lump **(Lump 40)** はBSPファイルに埋め込まれた複数のファイルを格納できる特別なLumpです。通常、マップ内のenv_cubemapエンティティからの反射マップを格納するための特別なテクスチャ（.vtf）ファイルとマテリアル（.vmt）ファイルが含まれています。これらのファイルは[`buildcubemaps`]コンソールコマンドが実行された時にビルドされてPakfile Lumpに格納されます。Pakfileにはマップで使用されるカスタムテクスチャやPropのようなものも任意で含めることができ、それらは[BSPZIP]プログラム（もしくは[Pakrat]のような代替プログラム）を用いてBSPファイル内に配置されます。また、これらのファイルはゲームエンジンのファイルシステムに統合されて、外部のファイルが使用される前に優先的に読み込まれます。

Pakfile Lumpの形式は、圧縮が指定されていない場合（つまり、個々のファイルが非圧縮形式で保存されている場合）はZip圧縮ユーティリティで用いられる形式と同じです。Pakfile Lumpを展開すると、WinZipなどのプログラムで開くことができるようになります。

ヘッダファイル*public/zip_uncompressed.h*は、Pakfile Lumpに存在する構造体を定義します。Lumpの最後の要素は`ZIP_EndOfCentralDirRecord`構造体です。これはその構造体の直前に、Pakに存在する各ファイルに対して1つずつある`ZIP_FileHeader`構造体の配列を指します。これらのヘッダはそれぞれ、ファイルのデータが後ろに続いている`ZIP_LocalFileHeader`構造体を指し示します。

Pakfile Lumpは通常、BSPファイルの最後の要素です。


<h2 id="cubemap">Cubemap</h2>

Cubemap Lump **(Lump 42)** は16バイトの`dcubemapsample_t`構造体の配列です。
``` C++
struct dcubemapsample_t
{
	int		origin[3];	// 最も近い整数に丸められたライトの位置
	int	        size;		// Cubemapの解像度（0: デフォルト）
};
```
<code>dcubemapsample_t</code>構造体は、マップ内のenv_cubemapエンティティの場所を定義します。<code>origin</code>メンバには、Cubemapの整数座標x、y、zが含まれ、<code>size</code>メンバは2<sup><code>size</code> - 1</sup>ピクセルの正方形として指定されるCubemapの解像度で、0の場合はデフォルトのサイズである6（32×32ピクセル）になります。ファイルには最大1024個のCubemapが存在します（<code>MAX_MAP_CUBEMAPSAMPLES</code>）。

`buildcubemap`コンソールコマンドが実行されると、各Cubemapエンティティの位置でマップのスナップショットが6つ（各方向について1つずつ）撮影されます。これらのスナップショットはマルチフレームテクスチャファイル（.vtf）に格納され、Pakfile Lump（上記）に追加されます。テクスチャ名は`cX_Y_Z.vtf`で、（X, Y, Z）はCubemapの（整数）座標です。

環境マッピングされたマテリアルを含む面（例えば光沢のあるテクスチャ）は、マテリアル名を介してCubemapを参照します。（例えば）<code>walls/shiny.vmt</code>と名前の付いたマテリアルは書き換えられて（新しいTexinfoとTexdataが作成されて）、変更されたマテリアル名である<code>maps/<i>mapname</i>/walls/shiny_X_Y_Z.vmt</code>を参照するようになります。ここで（X、Y、Z）は 前述したようにCubemapの座標です。この.vmtファイルはPakfileにも格納され、$envmapプロパティを使用してCubemapの.vtfファイルを参照します。

バージョン20のファイルにはさらに`cX_Y_Z_hdr.vtf`がPakfile Lumpに追加されます。これにはRGBA16161616F形式（チャンネルごとに16ビット）のHDRテクスチャファイルが含まれています。


<h2 id="overlay">オーバーレイ</h2>

単純なデカール（infodecalエンティティ）とは異なり、info_overlayはエンティティLumpから削除されて、オーバーレイLump **(Lump 45)** に分けて保存されます。この構造体はHammerのエンティティのプロパティをほぼ正確に反映しています。
``` C++
struct doverlay_t
{
	int		Id;
	short		TexInfo;
	unsigned short	FaceCountAndRenderOrder;
	int		Ofaces[OVERLAY_BSP_FACE_COUNT];
	float		U[2];
	float		V[2];
	Vector		UVPoints[4];
	Vector		Origin;
	Vector		BasisNormal;
};
```
`FaceCountAndRenderOrder`メンバは2つの部分に分かれています。下位14ビットはオーバーレイが表示される面の数で、上位2ビットはオーバーレイの表示順序です（重なったオーバーレイの場合）。`Ofaces`は要素数64の配列で（`OVERLAY_BSP_FACE_COUNT`）、オーバーレイが表示される面へのインデックスが格納されています。他の要素はオーバーレイのテクスチャ、スケール、向きを設定します。1つのファイルには最大512個のオーバーレイが存在します（`MAX_MAP_OVERLAYS`）。また、Dota 2ではオーバーレイ数の制限が大幅に増加しています。


<h2 id="lighting">ライティング</h2>

ライティングLump **(Lump 8)** はマップの面の静的ライトマップサンプルを格納するために使用されます。各ライトマップサンプルは、テクスチャピクセルの色と乗算する色合いであり、様々な強度の照明を生成します。これらのライトマップはマップコンパイル時のVRAD処理中に作成され、`dface_t`構造体から参照されます。現在のライティングLumpのバージョンは1です。

`dface_t`では`styles[]`配列で定義された最大4つのライトスタイルを持つことができます（ただし値255はライトスタイルがないことを示す）。面の各方向についてのルクセル数は、2つの`LightmapTextureSizeInLuxels[]`メンバの値（+ 1）によって与えられ、面ごとのルクセルの総和は次のようになります。

`(LightmapTextureSizeInLuxels[0] + 1) * (LightmapTextureSizeInLuxels[1] + 1)`

面はそれぞれ、`dface_t`の`lightofs`メンバによってライティングLump内のオフセットを持ちます（もし、スカイボックスやnodrawなどの不可視のテクスチャであるなどの理由でその面にライティング情報が使われていない場合、`lightofs`は-1です）。ライトマップサンプルの総数は（*ライトスタイル数*）×（*ルクセル数*）で、各サンプルは`ColorRGBExp32`構造体で与えられます。
``` C++
struct ColorRGBExp32
{
	byte r, g, b;
	signed char exponent;
};
```
この構造体から、それぞれの色成分に2<sup><code><i>exponent</i></code></sup>をかけることで標準のRGB形式を得ることができます。バンプマップ付きのテクスチャを持つ面の場合、おそらくバンプマップを計算するためのサンプルを含むためライトマップサンプルの数は通常の4倍になります。

`lightofs`で参照されるサンプルグループの直前には、面のライティングの平均値がライトスタイルごとに、`styles[]`配列で与えられた順番とは逆の順番になって存在しています。

バージョン20のBSPファイルには、同じサイズの2つ目のライティングLump **(Lump 53)** が含まれています。これは、各ライトマップサンプルに対してより正確な（より高精度の）HDRデータを格納するものと推定されています。フォーマットは現在不明ですが、1サンプルにつき32ビットです。

ライティングLumpの最大サイズは0x1000000バイトです（`MAX_MAP_LIGHTING`）。すなわち、16MBです。


<h2 id="ambientlighting">アンビエントライティング</h2>

アンビエントライティングLump **(Lump 55とLump 56)** は、BSPバージョン20以降に存在します。Lump 55はHDRライティングに使用され、Lump 56はLDRライティングに使用されます。これらのLumpは、Volumetric [Ambient Lighting]情報（例えば、NPC、ビューモデル、静的でないPropなどのエンティティのためのライティング情報）を格納するために使われます。バージョン20より前はこのデータをリーフLumpの`dleaf_t`構造体に格納していましたが、この新しいLumpよりもはるかに低い精度でした。

アンビエントライティングLumpは両方とも`dleafambientlighting_t`構造体の配列です。
``` C++
struct dleafambientlighting_t
{
	CompressedLightCube	cube;
	byte x;		// 固定小数点で、リーフのバウンディングボックスの割合
	byte y;		// 固定小数点で、リーフのバウンディングボックスの割合
	byte z;		// 固定小数点で、リーフのバウンディングボックスの割合
	byte pad;	// 未使用
};
```
各リーフは`dleafambientlighting_t`構造体のうちのいくつかに関連付けられています。各構造体は`x`、`y`、`z`メンバによって指定された位置にある周囲光データのキューブを含みます。これらの座標はリーフが持つバウンディングボックスの割合として格納されます。つまり、xが0の場合はリーフの最西端で、255の場合は最東端、128の場合は中心を表します。

各サンプルのライティングデータは、`CompressedLightCube`構造体で表されます。これは、前のセクションで説明した`ColorRGBExp32`構造体が6つ格納された配列です。
``` C++
struct CompressedLightCube
{
	ColorRGBExp32 m_Color[6];
};
```
配列中の各ライティングサンプルは、3D空間内の各座標軸方向から受ける光の量に対応します。

コンパイル時に、[VRAD]は各リーフで周囲光のサンプルを取る位置ランダムに生成し、各サンプル点についてのライティング情報を`dleafambientlighting_t`構造体へ格納します。各リーフと周囲光のサンプルを関連付けるために、アンビエントライティングインデックスLump **(Lump 51とLump 32)** が用いられます。Lump 51はHDRについてのアンビエントライティングインデックス情報を、Lump 52はLDRについての情報を格納します。

アンビエントライティングインデックスLumpは`dleafambientindex_t`構造体の配列です。
``` C++
struct dleafambientindex_t
{
	unsigned short ambientSampleCount;
	unsigned short firstAmbientSample;
};
```
この配列のN番目の`dleafambientindex_t`構造体は常に`dleaf_t`配列のN番目に対応します。`ambientSampleCount`フィールドは対応するリーフに関連する周囲光の数で、`firstAmbientSample`はアンビエントライティング配列へのインデックスで、これは関連付けられたリーフの最初の周囲光サンプルを参照します。


<h2 id="occlusion">オクルージョン</h2>

オクルージョンLump **(Lump 9)** にはポリゴンジオメトリと[func_occluder]エンティティで使用されるいくつかのフラグが含まれています。他のブラシエンティティとは異なり、func_occluderは[エンティティLump](#entity)で「model」キーを使用しません。代わりに、そのブラシはコンパイル処理中にエンティティから分離され、`occludernum`という数値としてオクルーダーキーが割り当てられます。`tools/[toolsoccluder]`または`tools/[toolstrigger]`のテクスチャが付いたブラシの側面は、オクルーダーキーとともに保存され、このLumpにいくつかの追加情報が格納されます。

このLumpは3つに分割され、オクルーダーの総数に、一定のサイズの`doccluderdata_t`フィールドの配列が続いたものから始まります。次のパートは別の整数値で始まり、こちらはオクルーダーの総ポリゴン数です。続いてその数だけ`doccluderpolydata_t`フィールドの配列が続きます。3つ目のパートはオクルーダーの総頂点数を表す整数から始まり、頂点インデックスが続きます。
``` C++
struct doccluder_t
{
	int			count;
	doccluderdata_t		data[count];
	int			polyDataCount;
	doccluderpolydata_t	polyData[polyDataCount];
	int			vertexIndexCount;
	int			vertexIndices[vertexIndexCount];
};
```
`doccluderdata_t`構造体にはオクルーダーのフラグと寸法、そしてその領域が含まれています。`firstpoly`は`polycount`を含む`doccluderpolydata_t`への最初のインデックスです。
``` C++
struct doccluderdata_t
{
	int	flags;
	int	firstpoly;	// doccluderpolysへのインデックス
	int	polycount;	// ポリゴンの数
	Vector	mins;	        // 全頂点の最小値
	Vector	maxs;	        // 全頂点の最大値
	// v1から
	int	area;
};
```
オクルーダーポリゴンは`doccluderpolydata_t`構造体に格納されていて、`firstvertexindex`フィールドを含みます。これはオクルーダーの頂点配列へのインデックスで、オクルーダーの頂点配列の要素は[頂点Lump](#vertex) **(Lump 3)** の配列へのインデックスとなっています。頂点インデックスの総数は`vertexcount`に格納されます。
``` C++
struct doccluderpolydata_t
{
	int	firstvertexindex;	// doccludervertindicesへのインデックス
	int	vertexcount;		// 頂点インデックスの数
	int	planenum;
};
```


<h2 id="physics">Physics</h2>

Physcolldie Lump **(Lump 29)** にはワールドの物理的なデータが含まれています。

このLumpはひと続きの*model*で構成されていて、各*model*は以下から構成されます。
* `dphysmodel_t`ヘッダ
	``` C++
	struct dphysmodel_t
	{
		int modelIndex;  // おそらくこの物理モデルを適用するモデルへのインデックス？
		int dataSize;    // コリジョンデータセクションの合計サイズ
		int keydataSize; // テキストセクションのサイズ
		int solidCount;  // コリジョンデータセクションの数
	};
	```
* 一連のコリジョンデータセクション（`compactsurfaceheader_t`を含む）
* テキストセクション

このLumpは`modelIndex`が-1に設定された`dphysmodel_t`構造体で終了します。

最後の2つの部分は、[PHY]ファイル形式と同じに見えます。すなわち、正確な内容は不明です。`compactsurfaceheader_t`構造体には各コリジョンデータセクション（ヘッダの残りも含む）のサイズが含まれているので、このLumpは次のように解析できます。
``` C++
   while(true) {
       header = readHeader();
       if(header.modelIndex == -1)
           break;
       
       for(int k = 0; k < header.solidCount; k++) {
           size = read4ByteInt();
           collisionData = readBytes(size);
       }
       
       textData = readBytes(header.keydataSize);
   }
```


<h2 id="other">その他</h2>

**To do:** これらの情報の一部は推測に基づくものである可能性があるため、更なる調査が必要です。

* Worldlights Lump **(Lump 15)** にはワールドにあるそれぞれのスタティックライトエンティティに関する情報が含まれていて、移動するエンティティの半動的ライティングを提供するために使用されているようです。
* Area Lump **(Lump 20)** は、Areaportal Lump **(Lump 21)** を参照し、func_areaportalおよびfunc_areaportalwindowエンティティとともに使用することでマップをレンダリングするかしないかを切り替えられる仕切りを定義します。
* Portal Lump **(Lump 22)** 、Cluster Lump **(Lump 23)** 、PortalVerts Lump **(Lump 24)** 、ClusterPortals **(Lump 25)** 、ClipPortalVerts **(Lump 41)** はコンパイルのVVIS処理でどのクラスタがあるクラスタから見ることができるかを確認するために使用されます。クラスタはマップ内のプレイヤーが進入可能なリーフ ボリュームです（上記）。「ポータル」はクラスタまたはリーフが隣接する部分のポリゴン境界のことです。この情報の大部分はVRADプログラムによってスタティックライティングを計算するためにも用いられ、その後はBSPファイルから削除されます。
* PhysCollide Lump **(Lump 29)** とPhysCollideSurface Lump **(Lump 49)** はゲームエンジンでエンティティの衝突に関する物理シミュレーションに関連しているようです。
* VertNormal Lump **(Lump 30)** とVertNormalIndices Lump **(Lump 31)** は面のライトマップのスムージングに関連している可能性があります。
* FaceMacroTextureInfo Lump **(Lump 47)** は、マップ内の面の数と同じ数の要素を持ったshort値の配列です。この要素に-1（0xFFFF）以外のものが含まれている場合、その面はTexDataStringTableのテクスチャ名へのインデックスを持っています。VRADでは、対応するテクスチャはワールドエクステント(?)にマッピングされ、その面のライトマップのモジュレーションとして使用されます。すべての面に適用されるベースマクロテクスチャ（<code>materials/macro/<i>mapname</i>/base.vtf</code>に位置する）が見つかることもあります。VTMBのマップだけがマクロテクスチャを使用しているようです。
* LeafWaterData Lump **(Lump 36)** とLeafMinDistToWater Lump **(Lump 46)** は、水のボリュームに関してプレイヤーの位置を決定するために用いられるようです。
* Primitives Lump **(Lump 37)** とPrimVerts Lump **(Lump 38)** は、「非ポリゴンプリミティブ」に関することに使用されます。これらはもともと水のメッシュを分割するためだけに用いられていたため、SDK Sourceでは「waterstrips」、「waterverts」、「waterindices」と呼ばれることもあります。現在は、面を構成する辺に「T字型接合」（2つの頂点からなる直線上に頂点がある状態）が含まれる場合に隣接する面との間にクラックが発生するのを防ぐために使用されています。PrimIndices Lumpは面の頂点間の三角形のセットを定義して、面をテセレーションします。そして、それらはPrimitives Lumpから参照されます。Primitive Lumpは面Lumpによって参照されます。現在のマップでは、PrimVerts Lumpは全く使用されていないようです（[参考]）。
* HDRライティング情報を含んでいるバージョン20のファイルは、さらに4つの追加のLumpを持っていますが、現在その内容は正確には分かっていません。Lump 53は常に標準のライティングLump **(Lump 8)** と同じサイズであり、おそらく各ライトマップサンプルについて精度の高いデータを含んでいます。Lump 54はWorldlight Lump **(Lump 15)** と同じサイズであり、おそらくライトエンティティにおけるHDR関連のデータを含みます。


[Source BSP File Formatのページ]: https://developer.valvesoftware.com/wiki/Source_BSP_File_Format "Source BSP File Format - Valve Developer Community"
[Rof]: https://developer.valvesoftware.com/wiki/User:Rof "User:Rof - Valve Developer Community"
[「The Source Engine BSP File Format」]: http://web.archive.org/web/20050426034532/http://www.geocities.com/cofrdrbob/bspformat.html "The Source Engine BSP File Format - Web Archive"
[ここ]: http://www.bagthorpe.org/bob/cofrdrbob/bspformat.html "The Source Engine BSP File Format"
[BSP]: https://developer.valvesoftware.com/wiki/BSP "BSP - Valve Developer Community"
[Half-Life 1のエンジン（GoldSrc）]: https://developer.valvesoftware.com/wiki/GoldSrc "GoldSrc - Valve Developer Community"
[Quake 2 BSP File Format]: http://www.flipcode.com/archives/Quake_2_BSP_File_Format.shtml "flipcode - Quake 2 BSP File Format"
[VMEX]: https://developer.valvesoftware.com/wiki/VMEX "VMEX - Valve Developer Community"
[マップの逆コンパイル]: https://developer.valvesoftware.com/wiki/Decompiling_Maps "Decompiling Maps - Valve Developer Community"
[VMF]: https://developer.valvesoftware.com/wiki/VMF "VMF - Valve Developer Community"
[Counter-Strike: Source]: https://developer.valvesoftware.com/wiki/Counter-Strike:_Source "Counter-Strike: Source - Valve Developer Community"
[Half-Life 2: Deathmatch]: https://developer.valvesoftware.com/wiki/Half-Life_2:_Deathmatch "Half-Life 2: Deathmatch - Valve Developer Community"
[Steam Game Cache File（GCF）]: https://developer.valvesoftware.com/wiki/Game_Cache_File "GCF - Valve Developer Community"
[GCFScape]: https://developer.valvesoftware.com/wiki/GCFScape "GCFScape - Valve Developer Community"
[VPK]: https://developer.valvesoftware.com/wiki/VPK "VPK - Valve Developer Community"
[Alien Swarm Icon]: https://developer.valvesoftware.com/w/images/c/c9/AS-16px.png "Alien Swarm Icon"
[GoldSrc]: https://developer.valvesoftware.com/wiki/GoldSrc "GoldSrc - Valve Developer Community"
[Source BSP File Format/Game-Specific]: https://developer.valvesoftware.com/wiki/Source_BSP_File_Format/Game-Specific "Source BSP File Format/Game-Specific - Valve Developer Community"
[Vector]: https://developer.valvesoftware.com/wiki/Vector "Vector - Valve Developer Community"
[ブラシ]: https://developer.valvesoftware.com/wiki/Brush "Brush - Valve Developer Community"
[空間領域構成法（CSG）]: https://ja.wikipedia.org/wiki/Constructive_Solid_Geometry "Constructive Solid Geometry - Wikipedia（日本語）"
[func_detail]: https://developer.valvesoftware.com/wiki/Func_detail "func_detail - Valve Developer Community"
[toolsnodraw]: https://developer.valvesoftware.com/wiki/Tool_textures#nodraw "Tool textures - Valve Developer Community"
[「BSP for dummies」]: http://web.archive.org/web/20050426034532/http://www.planetquake.com/qxx/bsp/ "BSP for Dummies - WebArchive.org"
[テクセル]: https://developer.valvesoftware.com/wiki/Texel "Texel - Valve Developer Community"
[u, v]: https://developer.valvesoftware.com/wiki/UV_map "UV map - Valve Developer Community"
[visleaf]: https://developer.valvesoftware.com/wiki/Visleaves "Visleaves - Valve Developer Community"
[VVIS]: https://developer.valvesoftware.com/wiki/VVIS "VVIS - Valve Developer Community"
[Counter-Strike: Global Offensive]: https://developer.valvesoftware.com/wiki/Counter-Strike:_Global_Offensive "Counter-Strike: Global Offensive - Valve Developer Community"
[Potentially Visiblity Set（PVS）]: https://developer.valvesoftware.com/wiki/PVS "PVS - Valve Developer Community"
[Potentially Audible Set（PAS）]: https://developer.valvesoftware.com/wiki/PAS "PAS - Valve Developer Community"
[Patching levels with lump files]: https://developer.valvesoftware.com/wiki/Patching_levels_with_lump_files "Patching levels with lump files - Valve Developer Community"
[KeyValue]: https://developer.valvesoftware.com/wiki/Keyvalue "Keyvalue - Valve Developer Community"
[worldspawn]: https://developer.valvesoftware.com/wiki/Worldspawn "worldspawn - Valve Developer Community"
[`targetname`]: https://developer.valvesoftware.com/wiki/Targetname "targetname - Valve Developer Community"
[ブラシエンティティ]: https://developer.valvesoftware.com/wiki/Brush_entity "Brush entity - Valve Developer Community"
[モデル]: https://developer.valvesoftware.com/wiki/Model "Model - Valve Developer Community"
[noteicon]: https://developer.valvesoftware.com/w/images/c/cc/Note.png "Note Icon"
[内部的なもの]: https://developer.valvesoftware.com/wiki/Internal_entity "Internal entity - Valve Developer Community"
[ワールド]: https://developer.valvesoftware.com/wiki/The_world "The world - Valve Developer Community"
[エンティティの制限]: https://developer.valvesoftware.com/wiki/Entity_limit "Entity limit - Valve Developer Community"
[prop_static]: https://developer.valvesoftware.com/wiki/Prop_static "prop_static - Valve Developer Community"
[Zeno Clash]: https://developer.valvesoftware.com/wiki/Zeno_Clash "Zeno Clash - Valve Developer Community"
[Left 4 Dead]: https://developer.valvesoftware.com/wiki/Left_4_Dead "Left 4 Dead - Valve Developer Community"
[Left 4 Dead 2]: https://developer.valvesoftware.com/wiki/Left_4_Dead_2 "Left 4 Dead 2 - Valve Developer Community"
[prop_detail]: https://developer.valvesoftware.com/wiki/Prop_detail "prop_detail - Valve Developer Community"
[`buildcubemaps`]: https://developer.valvesoftware.com/wiki/Cubemaps#Building "Cubemap - Valve Developer Community"
[BSPZIP]: https://developer.valvesoftware.com/wiki/BSPZIP "BSPZIP - Valve Developer Community"
[Pakrat]: https://developer.valvesoftware.com/wiki/Pakrat "Pakrat - Valve Developer Community"
[Ambient Lighting]: https://developer.valvesoftware.com/wiki/Ambient_light "Ambient light - Valve Developer Community"
[VRAD]: https://developer.valvesoftware.com/wiki/VRAD "VRAD - Valve Developer Community"
[func_occluder]: https://developer.valvesoftware.com/wiki/Func_occluder "func_occluder - Valve Developer Community"
[toolsoccluder]: https://developer.valvesoftware.com/wiki/Tool_textures#occluder "Tool textures: Valve Developer Community"
[toolstrigger]: https://developer.valvesoftware.com/wiki/Tool_textures#trigger "Tool textures: Valve Developer Community"
[PHY]: https://developer.valvesoftware.com/wiki/PHY "PHY - Valve Developer Community"
[参考]: http://web.archive.org/web/20071110230828/http://www.chatbear.com/board.plm?a=viewthread&b=4991&t=137,1118051039,3296&s=0&id=862840 "What are \"waterindices\"? - Source Coding \[VERC Network Forums\] - WebArchive.org"
