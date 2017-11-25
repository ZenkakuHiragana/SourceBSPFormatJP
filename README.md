### SourceBSPFormatJP  
Valve Developer Communityにある[Source BSP File Formatのページ]を日本語訳したもの（Google 翻訳 + 校正）  
  
****  
# Source BSP File Format  
*この記事は2005年10月の[Rof]による[「The Source Engine BSP File Format」]に基づいており、サービスが停止する前にGeocitiesから取得されています。元のバージョンへのミラーが[ここ]にあります。*  

### 目次
1. [前書き](#introduction)
1. [概要](#overview)
1. [BSPファイルのヘッダー](#bspheader)
    1. [バージョン](#versions)


<h2 id="introduction">前書き</h2>

このドキュメントでは、Source Engineで使用される[BSP]ファイルの構造について説明します。このファイル形式は、[Half-Life 1のエンジン（GoldSrc）]のものと似ていますが、同じではありません。GoldSrcのBSPファイルはQuake、Quake II、QuakeWorldおよびQuake III Arenaのフォーマットに基づいています。このため、Max McGuireの記事 [Quake 2 BSP File Format]は全体的な構造と、類似している部分の構造を理解する上で非常に役に立ちます。  

このドキュメントは、Rofが[VMEX]（Half-Life 2 BSPファイルの逆コンパイラ）の作成中に彼が書いたノートを拡張したものです。したがって、[マップの逆コンパイル]（BSPファイルをHammer Map Editorで読み込み可能な[VMF]ファイルに変換すること）を実行するために必要な部分に焦点を当てています。 

ここにあるほとんどの情報は、上記のMax McGuireの記事とSource SDKのソースコード（特に、C言語のヘッダーファイルである*public/bspfile.h*）、そしてRof自身がVMEXの作成中に行った実験から得られたものです。  

このドキュメントは読者がC/C++、ジオメトリ、およびSourceのマッピング用語についてある程度の理解があることを想定しています。コード（主にC言語の構造体）は`等幅フォント`で書かれています。また、ここにある構造体は明瞭性と一貫性のため、SDKヘッダーファイルにある実際の定義から変更されていることがあります。 


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


<h2 id="bspheader">BSPファイルのヘッダー</h2>
  
BSPファイルはヘッダーから始まります。この構造は、ファイルがValve Source Engine BSPファイルであることと、フォーマットのバージョンを示して、その後にファイル内の各データ(*lump*と呼ばれる)の場所、長さ、およびバージョンが最大64個分続きます。最後に、マップの修正回数が書かれています。  

ヘッダーの構造はSDKの *public/bspfile.h* というヘッダーファイルに記載されています。このファイルは、このドキュメント全体を通して広範囲にわたって参照されています。また、ヘッダーは合計で1036バイトです。  

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
// little-endian "VBSP"   0x50534256
#define IDBSPHEADER	(('P'<<24)+('S'<<16)+('B'<<8)+'V')
```  
したがって、ファイルの最初の4バイトは常に"VBSP"（ASCII形式）です。これらのバイトはファイルをValve BSPファイルとして識別します。他のBSPファイルの形式では、異なるマジックナンバーを使用します（例えば、id SoftwareのQuake Engineを用いたゲームは"IBSP"で始まります）。[GoldSrc]のBSP形式では、マジックナンバーはまったく使用されません。また、マジックナンバーの順序はファイルのエンディアンを決定するためにも使用できます。"VBSP"はリトルエンディアンに、"PSBV"はビッグエンディアンに使用されます。  

2番目の整数は、BSPファイル形式のバージョン（BSPVERSION）です。 Source ゲームの場合、この値はVampire: The Masquerade – Bloodlinesを除いて19から21の範囲です（下記の表を参照）。他のエンジン（HL1、Quakeシリーズなど）のBSPファイル形式は、全く異なるバージョン番号の範囲を使用することに注意してください。  


### バージョン  
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
		<td rowspan="4">リリース時は19で、Source 2007/2009<br>アップデートからは部分的に20</td>
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
			Game lump、エンティティ情報、PAKファイルはLZMA圧縮
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
		<td>dshader_t, StaticPropLump_t, texinfo_t, dgamelump_tに変更あり</td>
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
