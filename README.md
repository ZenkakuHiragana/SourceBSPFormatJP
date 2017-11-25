### SourceBSPFormatJP  
Valve Developer Communityにある[Source BSP File Formatのページ](https://developer.valvesoftware.com/wiki/Source_BSP_File_Format)を日本語訳したもの（Google 翻訳 + 校正）  
  
****  
# Source BSP File Format  
*この記事は2005年10月のRofによる「The Source Engine BSP File Format」に基づいており、サービスが停止する前にGeocitiesから取得されています。元のバージョンへのミラーが[ここ](http://www.bagthorpe.org/bob/cofrdrbob/bspformat.html)にあります。*  

### 目次  
1. [前書き](#%E5%89%8D%E6%9B%B8%E3%81%8D)
2. [概要](#%E6%A6%82%E8%A6%81)
3. [BSPファイルのヘッダー](#BSP%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%81%AE%E3%83%98%E3%83%83%E3%83%80%E3%83%BC)

## 前書き  
このドキュメントでは、Source Engineで使用されるBSPファイルの構造について説明します。このファイル形式は、[Half-Life 1のエンジン（GoldSrc）](https://developer.valvesoftware.com/wiki/GoldSrc)のものと似ていますが、同じではありません。GoldSrcのBSPファイルはQuake、Quake II、QuakeWorldおよびQuake III Arenaのフォーマットに基づいています。このため、Max McGuireの記事 [Quake 2 BSP File Format](http://www.flipcode.com/archives/Quake_2_BSP_File_Format.shtml)は全体的な構造と、類似している部分の構造を理解する上で非常に役に立ちます。  

このドキュメントは、Rofが[VMEX](https://developer.valvesoftware.com/wiki/VMEX)（Half-Life 2 BSPファイルの逆コンパイラ）の作成中に彼が書いたノートを拡張したものです。したがって、[マップの逆コンパイル](https://developer.valvesoftware.com/wiki/Decompiling_Maps)（BSPファイルをHammer Map Editorで読み込み可能な[VMF](https://developer.valvesoftware.com/wiki/VMF)ファイルに変換すること）を実行するために必要な部分に焦点を当てています。 

ここにあるほとんどの情報は、上記のMax McGuireの記事とSource SDKのソースコード（特に、C言語のヘッダーファイルである*public/bspfile.h*）、そしてRof自身がVMEXの作成中に行った実験から得られたものです。  

このドキュメントは読者がC/C++、ジオメトリ、およびSourceのマッピング用語についてある程度の理解があることを想定しています。コード（主にC言語の構造体）は`固定幅フォント`で書かれています。また、ここにある構造体は明瞭性と一貫性のため、SDKヘッダーファイルにある実際の定義から変更されていることがあります。  
  
## 概要  
BSPファイルには、マップをレンダリングして遊ぶためにSource Engineで必要とされる大多数の情報が含まれています。これには以下のような情報が含まれます。  
* レベル内のすべてのポリゴンのジオメトリ  
* 各ポリゴンが描画される時に用いるテクスチャの名前と方向  
* ゲーム中にプレイヤーや他のアイテムの物理的挙動をシミュレートするために使われるデータ
* マップ内のすべてのブラシベース、モデル（プロップ）ベース、および非可視（論理）エンティティの位置とプロパティ  
* BSP木と可視性テーブル（マップジオメトリ内のプレイヤーの位置の特定と、マップの見える範囲をできる限り効率的にレンダリングするのに使用）  

また、マップファイルにはレベルで使用されているカスタムテクスチャやモデルをマップのPakfile lumpの中に任意で埋め込むこともできます（下記参照）。  

BSPファイルに格納されていない情報として、マルチプレイヤーゲーム（[Counter-Strike: Source](https://developer.valvesoftware.com/wiki/Counter-Strike:_Source)や[Half-Life 2: Deathmatch](https://developer.valvesoftware.com/wiki/Half-Life_2:_Deathmatch)など）でマップを読み込んだ後に表示されるマップの説明テキスト（*mapname.txt*に格納されています）と、ノンプレイヤーキャラクター（NPC, マップをナビゲートする必要がある）が使用するAIナビゲーションファイル（*mapname.nav*に格納されています）があります。Souce Engineのファイルシステムの仕組み上、これらの外部ファイルはBSPファイルの[Pakfile](#Pakfile) lumpに埋め込まれることもありますが、通常はそうではありません。  

公式マップのファイルは、[Steam Game Cache File（GCF）](https://developer.valvesoftware.com/wiki/Game_Cache_File)形式で保存され、ゲームエンジンによりSteamファイルシステムを通じてアクセスされます。GCFファイルから中身を抽出してSteam外から閲覧するには、Nemesisの[GCFScape](https://developer.valvesoftware.com/wiki/GCFScape)を使用します。[VPK](https://developer.valvesoftware.com/wiki/VPK)ファイル形式を使用している新しいゲームでは通常、マップはオペレーティングシステムのファイルシステムに直接保存されます。  

BSPファイルのデータは、PC/Macではリトルエンディアンで、PS3/Xbox360ではビッグエンディアンで保存されます。リトルエンディアンのファイルをJavaなどのビッグエンディアン形式のプラットフォームで読み込む場合は、バイトスワップが必要です（逆も同様です）。  

## BSPファイルのヘッダー  
BSPファイルはヘッダーから始まります。この構造は、ファイルがValve Source Engine BSPファイルであることと、フォーマットのバージョンを示して、その後にファイル内の各データ(*lump*と呼ばれる)の場所、長さ、およびバージョンが最大64個分続きます。最後に、マップの修正回数が書かれています。  

ヘッダーの構造はSDKの *public/bspfile.h* というヘッダーファイルに記載されています。このファイルは、このドキュメント全体を通して広範囲にわたって参照されています。また、ヘッダーは合計で1036バイトです。  

![Alien Swarm](https://developer.valvesoftware.com/w/images/c/c9/AS-16px.png) Alien Swarmでは、この構造体はBSPHeader_tに名前が変更されています。  
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
したがって、ファイルの最初の4バイトは常に"VBSP"（ASCII形式）です。これらのバイトはファイルをValve BSPファイルとして識別します。他のBSPファイルの形式では、異なるマジックナンバーを使用します（例えば、id SoftwareのQuake Engineを用いたゲームは"IBSP"で始まります）。GoldSrcのBSP形式では、マジックナンバーはまったく使用されません。また、マジックナンバーの順序はファイルのエンディアンを決定するためにも使用できます。"VBSP"はリトルエンディアンに、"PSBV"はビッグエンディアンに使用されます。  

2番目の整数は、BSPファイル形式のバージョン（BSPVERSION）です。 Source ゲームの場合、この値はVampire: The Masquerade – Bloodlinesを除いて19から21の範囲です（下記の表を参照）。他のエンジン（HL1、Quakeシリーズなど）のBSPファイル形式は、全く異なるバージョン番号の範囲を使用することに注意してください。  


### バージョン  
この表は、Source Engineを用いたいくつかのゲームで使用されているBSPバージョンの概要を示しています。 

| バージョン | ゲーム | 備考 |  
| - | - | - |  
| 17 | Vampire: The Masquerade – Bloodlines | dface_tに変更あり |  

以下翻訳中……  
