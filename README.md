# Lucene PostingsFormat At-a-Glance

English / [日本語](./README_ja.md)

_Last updated: 2022-05-28_ (commit `f5c1f11`)

This is an at-a-glance overview of [Apache Lucene](https://lucene.apache.org/)'s default `PostingsFormat`, which encodes inverted indices into low-level binary format, written for advanced users (and myself).

**NOTE:** The contents are NOT related to any Lucene release version but specific revision (commit). It will be updated on an irregular basis, also very fine details are often omitted. Please refer the official documentation or source code (the latter is the best) for more detailed and/or up-to-date information.

If you are also interested in inverted index construction on memory, [Lucene Indexing Chain for Inverted Index](./indexing_chain.md) could be helpful.

## Overview

Basically, `PostingsFormat` (i.e., inverted index) composes of two components: term dictionary and postings list.

1. Term dictionary composes of:
   - .tmd file [Term Metadata](#term-metadata)
   - .tim file [Term Dictionary](#term-dictionary)
   - .tip file [Term Index](#term-index)
2. Postings list composes of:
   - .doc file [Frequencies and Skip data](#frequencies-and-skip-data)
   - .pos file [Positions](#positions)
   - .pay file [Payloads and Offsets](#payloads-and-offsets)


## Term Metadata

Overview of the term metadata format (.tmd file).

```
+--------+-----------+------------+------------+-----+-----------------+----------------+--------+
| Header | NumFields | FieldStats | FieldStats | ... | TermIndexLength | TermDictLength | Footer |
+--------+-----------+------------+------------+-----+-----------------+----------------+--------+
                     |------- ( # of fields ) -------|
```

- Header (`CodecHeader`)
- NumFields (`VInt`) : Numbef of fields in this index.
- [FieldStats](#fieldstats) : The field level statistics and metadata.
- TermIndexLength (`Long`) : Whole length of [Term Index](#term-index).
- TermDictLength (`Long`) : Whole length of [Term Dictionary](#term-dictionary).
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

- FieldNumber (`VInt`) : Field number.
- NumTerms (`VLong`) : Number of unique terms for the field.
- RootCodeLength (`VInt`): The length of following RootCode.
- RootCode (`Bytes`) : 
- SumTotalTermFreq (`VLong`): Sum of total term frequencies; omitted when only documents are indexed.
- SumDocFreq (`VLong`) : Sum of document frequencies for terms in the field.
- DocCount (`VInt`) : Number of documents which have the field.
- MinTermLength (`VInt`) : The length of following MinTerm.
- MinTerm (`Bytes`): Minimum (first) term for the field.
- MaxTermLength (`VInt`): The length of following MaxTerm.
- MaxTerm (`Bytes`): Maximum (last) term for the field.
- IndexStartFP (`VLong`): The file pointer to [Term Index](#term-index) for the field.
- FSTHeader (`CodecHeader`)
- FSTMetadata


## Term Dictionary

Overview of the term dictionary format. (.tim file)

```
+--------+-----------+-----------+-----------+-----+--------+
| Header | NodeBlock | NodeBlock | NodeBlock | ... | Footer |
+--------+-----------+-----------+-----------+-----+--------+
         |------------ ( # of blocks ) ------------|
```

- Header (`CodecHeader`)
- [NodeBlock](#nodeblock) : Block-packed terms data.
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

- BlockHeader (`VInt`) : Block metadata (e.g. number of entries the block contains).
- SuffixLength (`VLong`) : The length of following Suffix.
- Suffix (`Bytes`): Concatenated suffixes for the all terms that are packed in the block.
- SuffixLengthsLength (`VInt`) : The length of following SuffixLengths.
- SuffixLengths (`Byte`) or `Bytes`) : The lengths of suffixes of the all terms that are packed in the block.
- StatsLength (`VInt`) : Total length of following TermStats.
- [TermStats](#termstats) : The term level statistics.
- MetaLength (`VInt`) : Total length of following TermMetadata.
- [TermMetadata](#termmetadata)

### TermStats

```
+-----------------+---------+----------------+
| SingletonCount? | DocFreq | TotalTermFreq? |
+-----------------+---------+----------------+
```

- SingletonCount (`VInt`) : Number of singleton terms (having DF==1, TTF==1) preceding the term.
- DocFreq (`VInt`) : Document frequency of the term.
- TotalTermFreq (`VLong`) : Total term frequency of the term; ommitted when only documents are indexed.

### TermMetadata

```
+------------+-----------------+-------------+-------------+---------------------
| DocStartFP | SingletonDocID? | PosStartFP? | PayStartFP? | LastPosBlockOffset?
+------------+-----------------+-------------+-------------+---------------------

--+-------------+
  | SkipOffset? |
--+-------------+
```

- DocStartFP (`VLong`) : The file pointer to the start of the doc ids for this term in .doc file
- SingletonDocID (`VInt`) : Document id if there is only one posting for the term.
- PosStartFP (`VLong`) : The file pointer to the start of the positions for this term in .pos file; omitted when positions are not indexed.
- PayStartFP (`VLong`) : The file pointer to the start of the payloads for this term in .pay file; omitted when neither offsets nor payloads are indexed.
- LastPosBlockOffset (`VLong`) : The file offset for the last position for the last block; omitted when there are less positions than the block size.
- SkipOffset (`VLong`) : The relative file offset for the start of the skip list to DocStartFP; omitted when there are less docs than the block size.


## Term Index

Overview of the term index file format. (.tip file)

```
+--------+----------+----------+----------+-----+--------+
| Header | FSTIndex | FSTIndex | FSTIndex | ... | Footer |
+--------+----------+----------+----------+-----+--------+
         |---------- ( # of fields ) -----------|
```

- Header (`CodecHeader`)
- FSTIndex (`Bytes`) : Binary encoded fst index for a field.
- Footer (`CodecFooter`)


## Frequencies and Skip data

Overview of the document and term frequencies file format. (.doc file)

```
+--------+------------------------+------------------------+-----+--------+
| Header | (TermFreqs, SkipData?) | (TermFreqs, SkipData?) | ... | Footer |
+--------+------------------------+------------------------+-----+--------+
         |------------------- ( # of terms ) --------------------|
```

- Header (`CodecHeader`)
- [TermFreqs](#termfreqs) : The document id deltas and term frequencies.
- [SkipData](#skipdata) : The skip list data for faster retrieval.
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

- PackedDocDelta (`PackedInts`) : Block compressed document id deltas in each block.
- PackedFreq (`PackedInts`) : Block compressed term frequency deltas in each block; omitted when term frequencies are not indexed.
- DocDelta (`VInt`) : Document id delta.
- Freq (`VInt`) : Term frequency delta; omitted when term frequencies are not indexed.

### SkipData

```
+------------------------------+------------------------------+-----+-----------+
| (SkipLevelLength, SkipLevel) | (SkipLevelLength, SkipLevel) | ... | SkipDatum |
+------------------------------+------------------------------+-----+-----------+
|------------------- ( # of skip levels - 1 ) ----------------------|
```

- SkipLevelLength (`VLong`) : The length of following SkipLevel.
- [SkipLevel](#skiplevel)
- [SkipDatum](#skipdatum) : Skip datum for level 0.

### SkipLevel

```
+-----------+-------------------------------+-------------------------------+-----+
| SkipDatum | (SkipDatum, ChildSkipLevelFP) | (SkipDatum, ChildSkipLevelFP) | ... | 
+-----------+-------------------------------+-------------------------------+-----+
            |--- ( maximum number of skip level for the number of docs seen ) ----|
```

- [SkipDatum](#skipdatum)
- ChildSkipLevelFP (`VLong`) : The file pointer of its direct child skip level data.

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

- SkipDocDelta (`VInt`) : The last document id delta in each block.
- SkipDocFPDelta (`VLong`) : The file pointer of each block in .doc file.
- SkipPosFPDelta (`VLong`) : The file pointer of each related block in .pos file; omitted when positions are not indexed.
- SkipPosBlockOffset (`VInt`) : The offset value inside the related block in .pos file; omitted when positions are not indexed.
- SkipPayBlockLength (`VInt`) : The sum of the payload lengths of each related block in .pay file; omitted when payloads are not indexed.
- SkipPayFPDelta (`VLong`) : The file pointer of each related block in .pay file; omitted when offsets/payloads are not indexed.
- ImpactLength (`VInt`) : The total length of following Impacts.
- [Impact](#impact) : The competitive frequency and norm data for faster top-k retrieval.

### Impact

```
+----------------------+-----------------------+
| CompetitiveFreqDelta | CompetitiveNormDelta? |
+----------------------+-----------------------+
```

- CompetitiveFreqDelta (`VInt`)
- CompetitiveNormDelta (`ZLong`)


## Positions

Overview of the positions file format. (.pos file)

```
+--------+---------------+---------------+---------------+-----+--------+
| Header | TermPositions | TermPositions | TermPositions | ... | Footer |
+--------+---------------+---------------+---------------+-----+--------+
         |------------------- ( # of terms ) ------------------|
```

- Header (`CodecHeader`)
- [TermPositions](#termpositions) : The term positions data that composes of fixed-size block compressed part and residual part.
- Footer (`CodecFooter`)

### TermPositions

```
+----------------+----------------+-----+------------------+------------------+-----+
| PackedPosDelta | PackedPosDelta | ... | ResidualPosDelta | ResidualPosDelta | ... |
+----------------+----------------+-----+------------------+------------------+-----+
|---------- ( # of pos blocks ) --------|------- ( # of remaining positions ) ------|
```

- PackedPosDelta (`PackedInts`) : Block compressed position deltas in each block.
- [ResidualPosDelta](#residualposdelta) : The residual position deltas that are encoded as `VInt`.

### ResidualPosDelta

```
+----------+----------------+----------+--------------+---------------+
| PosDelta | PayloadLength? | Payload? | OffsetDelta? | OffsetLength? |
+----------+----------------+----------+--------------+---------------+
```

- PosDelta (`VInt`) : Position delta.
- PayloadLength (`VInt`) : The length of following Payload; omitted when payloads are not indexed.
- Payload (`Bytes`) : Payload data; ommitted when payloads are not indexed.
- OffsetDelta (`VInt`) : Start offset delta; omitted when offsets are not indexed.
- OffsetLength (`VInt`) : The length of the offset (end offset - start offset); omitted when offsets are not indexed.


## Payloads and Offsets

Overview of the payloads and offsets file format. (.pay file)

```
+--------+-------------------------------+------------------------------+-----+--------+
| Header | (TermPayloads?, TermOffsets?) | (TermPayloads?, TermOffsets) | ... | Footer |
+--------+-------------------------------+------------------------------+-----+--------+
         |---------------------- ( # of terms ) ------------------------------|
```

- Header (`CodecHeader`)
- [TermPayloads](#termpayloads) : Payload data; ommitted when payloads are not indexed.
- [TermOffsets](#termoffsets) : Offsets data; omitted when offsets are not indexed.
- Footer (`CodecFooter`)

### TermPayloads

```
+--------------------------------------------------------+--------------------------------------------------------+-----+
| (PackedPayloadLengths, SumPayloadLengths, PayloadData) | (PackedPayloadLengths, SumPayloadLengths, PayloadData) | ... |
+--------------------------------------------------------+--------------------------------------------------------+-----+
|------------------------------------------- ( # of pos blocks ) -------------------------------------------------------|
```

- PackedPayloadLengths (`PackedInts`) : Block compressed payload lengths in each block.
- SumPayloadLengths (`VInt`) : The sum of payload lengths in each block.
- PayloadData (`Bytes`) : Concatenated payload data in each block.

### TermOffsets

```
+------------------------------------------+------------------------------------------+-----+
| (PackedOffsetDelta, PackedOffsetLengths) | (PackedOffsetDelta, PackedOffsetLengths) | ... |
+------------------------------------------+------------------------------------------+-----+
|----------------------------------- ( # of pos blocks ) -----------------------------------|
```

- PackedOffsetDelta (`PackedInts`) : Block compressed start offset deltas in each block.
- PackedOffsetLengths (`PackedInts`) : Block compressed offset lengths (end offset - start offset) in each block.
