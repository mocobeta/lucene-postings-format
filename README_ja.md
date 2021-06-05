# Lucene PostingsFormat 概観図

[English](./README.md) / 日本語

_Last updated: 2021-06-05_ (commit `beafd11`)

これは，[Apache Lucene](https://lucene.apache.org/) のデフォルト `PostingsFormat` - 転置インデックスをバイナリエンコードした，低レベル表現 - の概観図で，上級ユーザー（また，著者自身）に向けて書かれたものです。

**NOTE:** 以下のコンテンツは，どのLuceneリリースバージョンとも対応するものではなく，ソースコードリポジトリの特定のリビジョン（コミット）と紐づいています。更新は不定期に行われ，また細部はしばしば省略されています。より詳細や最新の情報については，公式ドキュメントないしソースコード（後者がベストです）を参照してください。

## 概要

`PostingsFormat` （転置インデックス）はおおまかに，2つのコンポーネント - ターム辞書とポスティングスリスト - から構成されます。

1. ターム辞書はさらに，これらのファイルから構成されます:
   - .tmd file [Term Metadata](#term-metadata)
   - .tim file [Term Dictionary](#term-dictionary)
   - .tip file [Term Index](#term-index)
2. ポスティングスリストはさらに，これらのファイルから構成されます:
   - .doc file [Frequencies and Skip data](#frequencies-and-skip-data)
   - .pos file [Positions](#positions)
   - .pay file [Payloads and Offsets](#payloads-and-offsets)


## Term Metadata

term metadata フォーマット (.tmd file)

```
+--------+-----------+------------+------------+-----+-----------------+----------------+--------+
| Header | NumFields | FieldStats | FieldStats | ... | TermIndexLength | TermDictLength | Footer |
+--------+-----------+------------+------------+-----+-----------------+----------------+--------+
                     |------- ( # of fields ) -------|
```

- Header (`CodecHeader`)
- NumFields (`VInt`) : インデックス中の全フィールド数。
- [FieldStats](#fieldstats) : フィールドレベルの統計情報とメタデータ。
- TermIndexLength (`Long`) : このインデックスの [Term Index](#term-index) 長さ総和。
- TermDictLength (`Long`) : このインデックスの [Term Dictionary](#term-dictionary) 長さ総和。
- Footer (`CodecFooter`)

### FieldStats

```
+-------------+----------+----------------+----------+-------------------+--
| FieldNumber | NumTerms | RootCodeLength | RootCode | SumTotalTermFreq? |
+-------------+----------+----------------+----------+-------------------+--

--+------------+----------+---------------+---------+---------------+---------+--
  | SumDocFreq | DocCount | MinTermLength | MinTerm | MaxTermLength | MaxTerm |
--+------------+----------+---------------+---------+---------------+---------+--

--+--------------+-----------+-------------+
  | IndexStartFP | FSTHeader | FSTMetadata |
--+--------------+-----------+-------------+
```

- FieldNumber (`VInt`) : フィールド番号。
- NumTerms (`VLong`) : フィールド中のターム異なり数。
- RootCodeLength (`VInt`): 後続のRootCodeの長さ。
- RootCode (`Bytes`) : 
- SumTotalTermFreq (`VLong`): ターム出現頻度（TF）の総和。ドキュメントIDのみインデックスする時は省略される。
- SumDocFreq (`VLong`) : ドキュメント頻度（DF）の総和。
- DocCount (`VInt`) : このフィールドをもつドキュメント数。
- MinTermLength (`VInt`) : 後続のMinTermの長さ。
- MinTerm (`Bytes`): このフィールドに含まれる最小（最初）のターム。
- MaxTermLength (`VInt`): 後続のMaxTermの長さ。
- MaxTerm (`Bytes`): このフィールドに含まれる最大（最後）のターム。
- IndexStartFP (`VLong`): このフィールドに対応する， [Term Index](#term-index) へのファイルポインタ。
- FSTHeader (`CodecHeader`)
- FSTMetadata


## Term Dictionary

term dictionary フォーマット (.tim file)

```
+--------+-----------+-----------+-----------+-----+--------+
| Header | NodeBlock | NodeBlock | NodeBlock | ... | Footer |
+--------+-----------+-----------+-----------+-----+--------+
         |------------ ( # of blocks ) ------------|
```

- Header (`CodecHeader`)
- [NodeBlock](#nodeblock) : ブロック単位にまとめられたタームのデータ。
- Footer (`CodecFooter`)

### NodeBlock

```
+-------------+--------------+--------+---------------------+---------------
| BlockHeader | SuffixLength | Suffix | SuffixLengthsLength | SuffixLengths 
+-------------+--------------+--------+---------------------+---------------

--+-------------+-----------+-----------+-----------+-----
  | StatsLength | TermStats | TermStats | TermStats | ... 
--+-------------+-----------+-----------+-----------+-----
                |------------- ( # of terms ) ------------

--+----------------+--------------+--------------+--------------+-----+
  | MetadataLength | TermMetadata | TermMetadata | TermMetadata | ... |
--+----------------+--------------+--------------+--------------+-----+
--|                |----------------- ( # of terms ) -----------------|
```

- BlockHeader (`VInt`) : このブロックのメタデータ（例：ブロックが含むターム数）
- SuffixLength (`VLong`) : 後続のSuffixの長さ。
- Suffix (`Bytes`): このブロックに含まれる，すべてのタームのサフィックスを結合したバイト列。
- SuffixLengthsLength (`VInt`) : 後続のSuffixLengthsの長さ。
- SuffixLengths (`Byte`) or `Bytes`) : このブロックに含まれる，すべてのタームのサフィックス長のリスト。
- StatsLength (`VInt`) : 後続のTermStatsの長さ。
- [TermStats](#termstats) : タームレベルの統計情報。
- MetaLength (`VInt`) : 後続のTermMetadataの長さ。
- [TermMetadata](#termmetadata)

### TermStats

```
+-----------------+---------+----------------+
| SingletonCount? | DocFreq | TotalTermFreq? |
+-----------------+---------+----------------+
```

- SingletonCount (`VInt`) : このタームの前に出現する，シングルトンターム(DF==1, TTF==1)の数.
- DocFreq (`VInt`) : このタームのドキュメント頻度（DF）。
- TotalTermFreq (`VLong`) : このタームの総出現頻度。ドキュメントIDのみインデックスするときは省略される。

### TermMetadata

TBD


## Term Index

term index フォーマット (.tip file)


```
+--------+----------+----------+----------+-----+--------+
| Header | FSTIndex | FSTIndex | FSTIndex | ... | Footer |
+--------+----------+----------+----------+-----+--------+
         |---------- ( # of fields ) -----------|
```

- Header (`CodecHeader`)
- FSTIndex (`Bytes`) : フィールドごとの，バイナリエンコードされたFSTインデックス。
- Footer (`CodecFooter`)


## Frequencies and Skip data

document and term frequencies フォーマット (.doc file)

```
+--------+------------------------+------------------------+-----+--------+
| Header | (TermFreqs, SkipData?) | (TermFreqs, SkipData?) | ... | Footer |
+--------+------------------------+------------------------+-----+--------+
         |------------------- ( # of terms ) --------------------|
```

- Header (`CodecHeader`)
- [TermFreqs](#termfreqs) : ドキュメントIDの差分列とターム出現頻度。
- [SkipData](#skipdata) : 高速な検索のための，スキップリストデータ。
- Footer (`CodecFooter`)


### TermFreqs

```
+-------------------------------+-------------------------------+-----
| (PackedDocDelta, PackedFreq?) | (PackedDocDelta, PackedFreq?) | ... 
+-------------------------------+-------------------------------+-----
|------------------------- ( # of doc blocks ) -----------------------

--+-------------------+-------------------+-------------------+-----+
  | (DocDelta, Freq?) | (DocDelta, Freq?) | (DocDelta, Freq?) | ... |
--+-------------------+-------------------+-------------------+-----+
--|------------------- (# of remaining docs ) ----------------------|

```

- PackedDocDelta (`PackedInts`) : ブロック圧縮されたドキュメントIDの差分列。
- PackedFreq (`PackedInts`) : ブロック圧縮されたターム出現頻度（TF）の差分列。ターム出現頻度をインデックスしない時は省略される。
- DocDelta (`VInt`) : ドキュメントIDの差分。
- Freq (`VInt`) : ターム出現頻度（TF）の差分。ターム出現頻度をインデックスしない時は省略される。

### SkipData

```
+------------------------------+------------------------------+-----+-----------+
| (SkipLevelLength, SkipLevel) | (SkipLevelLength, SkipLevel) | ... | SkipDatum |
+------------------------------+------------------------------+-----+-----------+
|------------------- ( # of skip levels - 1 ) ----------------------|
```

- SkipLevelLength (`VLong`) : 後続のSkipLevelの長さ。
- [SkipLevel](#skiplevel)
- [SkipDatum](#skipdatum) : レベル0のスキップデータ。

### SkipLevel

```
+-----------+-------------------------------+-------------------------------+-----+
| SkipDatum | (SkipDatum, ChildSkipLevelFP) | (SkipDatum, ChildSkipLevelFP) | ... | 
+-----------+-------------------------------+-------------------------------+-----+
            |--- ( maximum number of skip level for the number of docs seen ) ----|
```

- [SkipDatum](#skipdatum)
- ChildSkipLevelFP (`VLong`) : 子スキップレベルデータへのファイルポインタ。

### SkipDatum

```
+--------------+----------------+-----------------+---------------------+---------------------+-----------------+--
| SkipDocDelta | SkipDocFPDelta | SkipPosFPDelta? | SkipPosBlockOffset? | SkipPayBlockLength? | SkipPayFPDelta? |
+--------------+----------------+-----------------+---------------------+---------------------+-----------------+--

--+--------------+--------+--------+--------+-----+
  | ImpactLength | Impact | Impact | Impact | ... |
--+--------------+--------+--------+--------+-----+
                 |------- ( # of impacts ) -------|
```

- SkipDocDelta (`VInt`) : このブロックに含まれる，最後のドキュメントID（差分）。
- SkipDocFPDelta (`VLong`) : このブロックに対応する .doc ファイルへのファイルポインタ。
- SkipPosFPDelta (`VLong`) : このブロックに対応する .pos ファイルへのファイルポインタ。ターム出現位置をインデックスしない時は省略される。
- SkipPosBlockOffset (`VInt`) : このブロックに対応する .pos ファイル中のオフセット値。ターム出現位置をインデックスしない時は省略される。
- SkipPayBlockLength (`VInt`) : このブロックに対応する .pay ファイル中のタームペイロード長さ総和。ペイロードをインデックスしない時は省略される。
- SkipPayFPDelta (`VLong`) : このブロックに対応する .pay ファイルへのファイルポインタ。ペイロードをインデックスしない時は省略される。
- ImpactLength (`VInt`) : 後続のImpactsの長さ。
- [Impact](#impact) : 高速なTop-k検索のための付加データ（competitive frequency and norm data）。

### Impact

```
+----------------------+-----------------------+
| CompetitiveFreqDelta | CompetitiveNormDelta? |
+----------------------+-----------------------+
```

- CompetitiveFreqDelta (`VInt`)
- CompetitiveNormDelta (`ZLong`)


## Positions

positions フォーマット (.pos file)

```
+--------+---------------+---------------+---------------+-----+--------+
| Header | TermPositions | TermPositions | TermPositions | ... | Footer |
+--------+---------------+---------------+---------------+-----+--------+
         |------------------- ( # of terms ) ------------------|
```

- Header (`CodecHeader`)
- [TermPositions](#termpositions) : ターム位置情報。固定ブロックサイズに圧縮されたパートと，その余剰パートからなる。
- Footer (`CodecFooter`)

### TermPositions

```
+----------------+----------------+-----+------------------+------------------+-----+
| PackedPosDelta | PackedPosDelta | ... | ResidualPosDelta | ResidualPosDelta | ... |
+----------------+----------------+-----+------------------+------------------+-----+
|---------- ( # of pos blocks ) --------|------- ( # of remaining positions ) ------|
```

- PackedPosDelta (`PackedInts`) : ブロック圧縮されたターム出現位置情報の差分列。
- [ResidualPosDelta](#residualposdelta) : `VInt` でエンコードされた余剰のターム出現位置情報。

### ResidualPosDelta

```
+----------+----------------+----------+--------------+---------------+
| PosDelta | PayloadLength? | Payload? | OffsetDelta? | OffsetLength? |
+----------+----------------+----------+--------------+---------------+
```

- PosDelta (`VInt`) : ターム出現位置（差分）。
- PayloadLength (`VInt`) : 後続のPayloadの長さ。ペイロードをインデックスしない時は省略される。
- Payload (`Bytes`) : ペイロードデータ。ペイロードをインデックスしない時は省略される。
- OffsetDelta (`VInt`) : 開始文字オフセット（差分）。文字オフセットをインデックスしない時は省略される。
- OffsetLength (`VInt`) : 文字オフセット長（終了文字オフセットから開始文字オフセットを引いたもの）。 文字オフセットをインデックスしない時は省略される。


## Payloads and Offsets

payloads and offsets フォーマット (.pay file)

```
+--------+-------------------------------+------------------------------+-----+--------+
| Header | (TermPayloads?, TermOffsets?) | (TermPayloads?, TermOffsets) | ... | Footer |
+--------+-------------------------------+------------------------------+-----+--------+
         |---------------------- ( # of terms ) ------------------------------|
```

- Header (`CodecHeader`)
- [TermPayloads](#termpayloads) : ペイロードデータ。ペイロードをインデックスしない時は省略される。
- [TermOffsets](#termoffsets) : 文字オフセットデータ。文字オフセットをインデックスしない時は省略される。
- Footer (`CodecFooter`)

### TermPayloads

```
+--------------------------------------------------------+--------------------------------------------------------+-----+
| (PackedPayloadLengths, SumPayloadLengths, PayloadData) | (PackedPayloadLengths, SumPayloadLengths, PayloadData) | ... |
+--------------------------------------------------------+--------------------------------------------------------+-----+
|------------------------------------------- ( # of pos blocks ) -------------------------------------------------------|
```

- PackedPayloadLengths (`PackedInts`) : ブロック圧縮された，ペイロード長さのリスト。
- SumPayloadLengths (`VInt`) : このブロック中のペイロード長さ総和。
- PayloadData (`Bytes`) : このブロック中のすべてのペイロードを結合したバイト列。

### TermOffsets

```
+------------------------------------------+------------------------------------------+-----+
| (PackedOffsetDelta, PackedOffsetLengths) | (PackedOffsetDelta, PackedOffsetLengths) | ... |
+------------------------------------------+------------------------------------------+-----+
|----------------------------------- ( # of pos blocks ) -----------------------------------|
```

- PackedOffsetDelta (`PackedInts`) : ブロック圧縮された開始文字オフセットの差分列。
- PackedOffsetLengths (`PackedInts`) : ブロック圧縮された文字オフセット長（終了文字オフセットから開始文字オフセットを引いたもの）のリスト。
