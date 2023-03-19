# Lucene Indexing Chain for Inverted Index

_Last updated: 2023-03-19_ (commit `0782535`)

These diagrams describe the relationship between the classes that play a key role in creating inverted indexes on memory. (Note: For brevity, fine details are omitted.)

## Class diagram

![](./image/lucene_index_classes.png)

- `DocumentWriterPerThread`: Document writer.
- `IndexingChain`: Default document indexing chain.
- `IndexingChain.PerField`: Performs indexing for a field.
- `TermsHash`: Stores indexed tokens in a hash table.
- `TermsHashPerField`: Stores streams of information per term.

## Sequence diagram

![](./image/lucene_index_sequence.png)

[1] FreqProxTermsWriter

[2] FreqProxTermsWriterPerField
