# Serialisation formats
- Type: New Product
- Related compontent: RDF in SAFE Client Libs (accompanying RFC)

# Summary

This is an accompanying RFC to the parent `RDF in SAFE Client Libs` RFC and defines the different types of serialisation in the implementation of RDF in Safe Client Libs. This RFC must be read and is useful only in conju

# Conventions
- The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

# Detailed design

There are two types of serialisation that are required for this RFC. 
- Serialisation for storage 
- User-friendly serialisation

## Serialisation for storage 

Serialisation for storage refers to the serialisation of the data into a stream of bytes so that it can be stored in the available data types. Initially, we MAY rely on the serialisation format provided by Redland to serialize statements / rdf models to store them onto the network. This will most-likely change in the future for improved efficiency.

The Redland API provides a key-value storage system, [storage-hashes](http://librdf.org/docs/api/redland-storage-module-hashes.html).  Triples in an RDF graph are populated in Redland's storage-hashes as follows. SPO (subject + predicate + object) triples are stored in as key-value pairs, hashing and indexing them multiple times.

The list of hashes are the following:

```
sp2o -> key: subject + predicate, value: object

po2s -> key: predicate + object, value: subject

so2p -> key: subject + object, value: predicate
```

The triples themselves are serialised in a simple binary format, with prefixes like `M<stringLength><stringLiteral>` or `U<uriLength><uriValue>`. These hashes SHOULD be stored as key-value pairs in the existing MutableData structure. As statements are added to an RdfGraph, appropriate `EntryActions` MUST be added to a vector which can be applied directly in a `mutate_mdata_entries` operation. Note that Redland's storage-hashes structure supports duplicate keys. This MUST be properly translated into MutableData entries that allows only unique keys.

For example: Consider an empty RDF graph to which a triple `Dolly says hi` is added.

The key-value pairs in the RDF storage-hash would be as follows.

| Key        | Value |
|------------|-------|
| Dolly-says |  hi   |
| says-hi    | Dolly |
| Dolly-hi   | says  |

Adding another triple `Dolly says bye` would insert three more key-value pairs.

| Key        | Value |
|------------|-------|
| Dolly-says |  hi   |
| says-hi    | Dolly |
| Dolly-hi   | says  |
| Dolly-says |  bye  |
| says-bye   | Dolly |
| Dolly-bye  | says  |

When translated into mutable data, it MUST be stored as follows:

| Key        | Value     |
|------------|-----------|
| Dolly-says | [hi, bye] |
| says-hi    | Dolly     |
| Dolly-hi   | says      |
| says-bye   | Dolly     |
| Dolly-bye  | says      |

So essentially, when inserting a key-value pair we SHOULD check if the key already exists. If it does not, add the key-value pair to the entries. If it exists, fetch the existing values, add the new value to the list and apply an `EntryAction::Update` operation.

Similarly, when deleting a triple from the graph, the entry for a key is fetched, and the desired value is removed from the list. The `EntryAction::Update` operation should be applied with the remaining values for the value field.

## User-friendly serialisation

User-friendly serialisation on the other hand refers to the serialisation of an Rdf Model into a user-readable format such as Turtle, JSON-LD etc. The functions for these operations are as defined in the [serialisation section](https://github.com/nbaksalyar/rfcs/blob/master/text/0000-rdf-in-client-libs/0000-rdf-in-client-libs.md#serialisation) of the primary RFC.