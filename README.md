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
	1. [平面](#planes)


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
* マップ内のすべてのブラシベース、モデル（プロップ）ベース、および非可視（論理）エンティティの位置とプロパティ  
* BSP木と可視性テーブル（マップジオメトリ内のプレイヤーの位置の特定と、マップの見える範囲をできる限り効率的にレンダリングするのに使用）  

また、マップファイルにはレベルで使用されているカスタムテクスチャやモデルをマップのPakfile lumpの中に任意で埋め込むこともできます（下記参照）。  

BSPファイルに格納されていない情報として、マルチプレイヤーゲーム（[Counter-Strike: Source]や[Half-Life 2: Deathmatch]など）でマップを読み込んだ後に表示されるマップの説明テキスト（*mapname.txt*に格納されています）と、ノンプレイヤーキャラクター（NPC, マップをナビゲートする必要がある）が使用するAIナビゲーションファイル（*mapname.nav*に格納されています）があります。Souce Engineのファイルシステムの仕組み上、これらの外部ファイルはBSPファイルの[Pakfile](#Pakfile) lumpに埋め込まれることもありますが、通常はそうではありません。  

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

2番目の整数は、BSPファイル形式のバージョン（BSPVERSION）です。 Source ゲームの場合、この値はVampire: The Masquerade – Bloodlinesを除いて19から21の範囲です（下記の表を参照）。他のエンジン（HL1、Quakeシリーズなど）のBSPファイル形式は、全く異なるバージョン番号の範囲を使用することに注意してください。


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
		<td>256bit XOR暗号化</td>
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
		<td>28</td>
		<td>Titanfall</td>
		<td>大幅に変更されている</td>
	</tr>
</table>


ゲーム固有のBSP形式の詳細については、[Source BSP File Format/Game-Specific]を参照してください。  


<h3 id="lumpstructures">Lump構造体</h3>  

次に、16バイトの`lump_t`構造体の配列が続きます。HEADER_LUMPSは64と定義されているため、全部で64個のエントリがあります。ただし、ゲームやバージョンによっては定義されていないものや空のものがあります。  

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

最初の2つの整数はbspファイルの先頭からのバイトオフセットとそのLumpに含まれるデータブロックのバイト長を表します。続いて、そのLumpの形式のバージョン番号（通常は0）と、通常0, 0, 0, 0である4バイトの識別子があります。圧縮されたLumpの場合、fourCCには非圧縮のLumpデータサイズが整数で示されています（詳細は[Lumpの圧縮](#lumpcompression)の節を参照）。lump_t配列の未使用のメンバについてはすべての要素が0に設定されています。  
Lumpのオフセット（と、それに対応するデータ）は最も近い4バイトの境界に切り上げられています。ただし、Lumpの長さについてはこの限りではありません。  


<h3 id="lumptypes">Lumpの種類</h3>

`lump_t`配列が指すデータの種類は、配列内の位置によって定義されます。例えば、配列の最初のLump **(Lump 0)** は常にBSPファイルのエンティティデータです（下記参照）。BSPファイル内の実際のデータの位置は、そのLumpのoffsetとlengthによって定義されるため、ファイル内で特定の順番に並んでいる必要はありません。例えば、エンティティデータはLump配列の最初にあるにも関わらず通常はBSPファイルの最後に格納されます。したがって、lump_tヘッダの配列はLumpデータに関するディレクトリのようなものであり、Lumpデータはファイル内を自由に配置することができます。  

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
		<td>BSP木の節ノード</td>
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
		<td>BSP木の葉ノード</td>
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
		<td>プレイヤーが入れる葉ノード</td>
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
		<td>水中の葉ノードのためのデータ</td>
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
		<td>葉ノードから水までの距離</td>
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
		<td>葉ノードごとのアンビエントライト<br>サンプル（HDR）</td>
	</tr>
	<tr><td>56</td><td>
		<img src="https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png" 
		     alt="Half-Life 2 Icon" title="Half-Life 2 Icon">
		<a href="https://developer.valvesoftware.com/wiki/Source_2006" 
		   title="Episode One (engine branch) - Valve Developer Community">Source 2006</a>
		</td>
		<td>LUMP_LEAF_AMBIENT_LIGHTING</td>
		<td>葉ノードごとのアンビエントライト<br>サンプル（LDR）</td>
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
既知のエントリについて、データLumpの構造を以下に説明します。Lumpの多くは単純な構造の配列で、またいくつかはその内容に応じて可変長です。各データの最大サイズまたはエントリ数は*bspfile.h*でもMAX_MAP_\*として定義されています。

最後に、ヘッダはマップリビジョン番号を表す整数で終わります。この数値は、マップのvmfファイルのリビジョン番号（`mapversion`）に基いています。これは、Hammer Editorで保存される度に増加する値です。

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

圧縮に関して、2つのスペシャルケースがあります。`LUMP_PAKFILE`は決して圧縮されず、`LUMP_GAME_LUMP`の各ゲームLumpは個別に圧縮されます。圧縮されたゲームLumpのサイズは、現在のゲームLumpのオフセットを次のエントリのオフセットから引くことで決めることができます。このため、最後のゲームLumpは常にオフセットを含む空のダミーです。
<br><br>

<h1 id="lumpheader">Lump</h1>
<h2 id="planes">平面</h3>

BSPジオメトリの基底は、BSP木構造全体の分割面として使われる平面によって定義されます。


以下翻訳中……  

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
[hl2]: https://developer.valvesoftware.com/w/images/4/41/Icon_hl2.png "Half-Life 2 Icon"
[tf2]: https://developer.valvesoftware.com/w/images/8/84/Tf2-16px.png "Team Fortress 2 Icon"
[ld2]: https://developer.valvesoftware.com/w/images/9/93/L4D2-16px.png "Left 4 Dead 2 Icon"
[l4d]: https://developer.valvesoftware.com/w/images/c/c0/L4D-16px.png "Left 4 Dead Icon"
[confirm]: https://developer.valvesoftware.com/w/images/2/2e/Confirm.png "Confirm"
