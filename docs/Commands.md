# RediSearch Full Command Documentation

## FT.CREATE 

### Format:
```
  FT.CREATE {index} 
    [NOOFFSETS] [NOFIELDS]
    [STOPWORDS {num} {stopword} ...]
    SCHEMA {field} [TEXT [NOSTEM] [WEIGHT {weight}] | NUMERIC | GEO] [SORTABLE] [NOINDEX] ...
```

### Description:
Creates an index with the given spec. The index name will be used in all the key names
so keep it short!

!!! warning "Note on field number limits"
        
        RediSearch supports up to 1024 fields per schema, out of which at most 128 can be TEXT fields.

        On 32 bit builds, at most 64 fields can be TEXT fields.

        Notice that the more fields you have, the larger your index will be, as each additional 8 fields require one extra byte per index record to encode.

        You can always use the NOFIELDS option and not encode field information into the index, for saving space, if you do not need filtering by text fields. This will still allow filtering by numeric and geo fields.
        

### Parameters:

* **index**: the index name to create. If it exists the old spec will be overwritten

* **NOOFFSETS**: If set, we do not store term offsets for documents (saves memory, does not allow exact searches or highlighting).
  Implies `NOHL`

* **NOHL**: Conserves storage space and memory by disabling highlighting support. If set, we do
  not store corresponding byte offsets for term positions. `NOHL` is also implied by `NOOFFSETS`.

* **NOFIELDS**: If set, we do not store field bits for each term. Saves memory, does not allow filtering by specific fields.

* **NOFREQS**: If set, we avoid saving the term frequencies in the index. This saves
  memory but does not allow sorting based on the frequencies of a given term within
  the document.

* **STOPWORDS**: If set, we set the index with a custom stopword list, to be ignored during indexing and search time. {num} is the number of stopwords, followed by a list of stopword arguments exactly the length of {num}. 

    If not set, we take the default list of stopwords. 

    If **{num}** is set to 0, the index will not have stopwords.

* **SCHEMA {field} {options...}**: After the SCHEMA keyword we define the index fields. 
They can be numeric, textual or geographical. For textual fields we optionally specify a weight. The default weight is 1.0.

    ### Field Options


    * **SORTABLE**
    
        Numeric or text field can have the optional SORTABLE argument that allows the user to later [sort the results by the value of this field](/Sorting) (this adds memory overhead so do not declare it on large text fields).
  
    * **NOSTEM**

        Text fields can have the NOSTEM argument which will disable stemming when indexing its values. 
        This may be ideal for things like proper names.
  
    * **NOINDEX**

        Fields can have the `NOINDEX` option, which means they will not be indexed. 
        This is useful in conjunction with `SORTABLE`, to create fields whose update using PARTIAL will not cause full reindexing of the document. If a field has NOINDEX and doesn't have SORTABLE, it will just be ignored by the index.

### Complexity
O(1)

### Returns:
OK or an error

---


## FT.ADD 

### Format:

```
FT.ADD {index} {docId} {score} 
  [NOSAVE]
  [REPLACE [PARTIAL]]
  [LANGUAGE {language}] 
  [PAYLOAD {payload}]
  [IF {condition}]
  FIELDS {field} {value} [{field} {value}...]
```

### Description

Add a document to the index.

### Parameters:

- **index**: The Fulltext index name. The index must be first created with FT.CREATE

- **docId**: The document's id that will be returned from searches. 
  Note that the same docId cannot be added twice to the same index

- **score**: The document's rank based on the user's ranking. This must be between 0.0 and 1.0. 
  If you don't have a score just set it to 1

- **NOSAVE**: If set to true, we will not save the actual document in the index and only index it.

- **REPLACE**: If set, we will do an UPSERT style insertion - and 
ete an older version of the document if it exists. 

- **PARTIAL** (only applicable with REPLACE): If set, you do not have to specify all fields for reindexing. Fields not given to the command will be loaded from the current version of the document. Also, if only non indexable fields, score or payload are set - we do not do a full reindexing of the document, and this will be a lot faster.

- **FIELDS**: Following the FIELDS specifier, we are looking for pairs of  `{field} {value}` to be indexed.
  Each field will be scored based on the index spec given in FT.CREATE. 
  Passing fields that are not in the index spec will make them be stored as part of the document, or ignored if NOSAVE is set 

- **PAYLOAD {payload}**: Optionally set a binary safe payload string to the document, 
  that can be evaluated at query time by a custom scoring function, or retrieved to the client.

- **IF {condition}**: (Applicable only in conjunction with `REPLACE` and optionally `PARTIAL`). 
  Update the document only if a boolean expression applies to the document **before the update**, 
  e.g. `FT.ADD idx doc 1 REPLACE IF "@timestamp < 23323234234". 

  The expression is evaluated atomically before the update, ensuring that the update will happen only if it is true.

  See [Aggregations](/Aggregations) for more details on the expression language. 

- **LANGUAGE language**: If set, we use a stemmer for the supplied language during indexing. Defaults to English. 
  If an unsupported language is sent, the command returns an error. 
  The supported languages are:

    > "arabic",  "danish",    "dutch",   "english",   "finnish",    "french",
    > "german",  "hungarian", "italian", "norwegian", "portuguese", "romanian",
    > "russian", "spanish",   "swedish", "tamil",     "turkish"
    > "chinese"

  If indexing a chinese language document, you must set the language to `chinese`
  in order for the chinese characters to be tokenized properly.

### Adding Chinese Documents

When adding Chinese-language documents, `LANGUAGE chinese` should be set in
order for the indexer to properly tokenize the terms. If the default language
is used then search terms will be extracted based on punctuation characters and
whitespace. The Chinese language tokenizer makes use of a segmentation algorithm
(via [Friso](https://github.com/lionsoul2014/friso)) which segments texts and
checks it against a predefined dictionary. See [Stemming](/Stemming) for more
information.

### Complexity

O(n), where n is the number of tokens in the document

### Returns

OK on success, or an error if something went wrong.

A special status `NOADD` is returned if an `IF` condition evaluated to false.

!!! warning "FT.ADD with  REPLACE and PARTIAL"
        
        By default, FT.ADD does not allow updating the document, and will fail if it already exists in the index.

        However, updating the document is possible with the REPLACE and REPLACE PARTIAL options.

        **REPLACE**: On its own, sets the document to the new values, and reindexes it. Any fields not given will not be loaded from the current version of the document.

        **REPLACE PARTIAL**: When both arguments are used, we can update just part of the document fields, and the rest will be loaded before reindexing. Not only that, but if only the score, payload and non-indexed fields (using NOINDEX) are updated, we will not actually reindex the document, just update its metadata internally, which is a lot faster and does not create index garbage.
----

## FT.ADDHASH

### Format

```
 FT.ADDHASH {index} {docId} {score} [LANGUAGE language] [REPLACE]
```

### Description

Add a document to the index from an existing HASH key in Redis.

### Parameters:

- **index**: The Fulltext index name. The index must be first created with FT.CREATE

-  **docId**: The document's id. This has to be an existing HASH key in redis that will hold the fields 
    the index needs.

- **score**: The document's rank based on the user's ranking. This must be between 0.0 and 1.0. 
  If you don't have a score just set it to 1

- **REPLACE**: If set, we will do an UPSERT style insertion - and delete an older version of the document if it exists.

- **LANGUAGE language**: If set, we use a stemmer for the supplied language during indexing. Defaults to English. 
  If an unsupported language is sent, the command returns an error. 
  The supported languages are:

  > "arabic",  "danish",    "dutch",   "english",   "finnish",    "french",
  > "german",  "hungarian", "italian", "norwegian", "portuguese", "romanian",
  > "russian", "spanish",   "swedish", "tamil",     "turkish"

### Complexity

O(n), where n is the number of tokens in the document

### Returns

OK on success, or an error if something went wrong.

----

## FT.INFO

### Format

```
FT.INFO {index} 
```

### Description

Return information and statistics on the index. Returned values include:

* Number of documents.
* Number of distinct terms.
* Average bytes per record.
* Size and capacity of the index buffers.

Example:

```
127.0.0.1:6379> ft.info wik{0}
 1) index_name
 2) wikipedia
 3) fields
 4) 1) 1) title
       2) type
       3) FULLTEXT
       4) weight
       5) "1"
    2) 1) body
       2) type
       3) FULLTEXT
       4) weight
       5) "1"
 5) num_docs
 6) "502694"
 7) num_terms
 8) "439158"
 9) num_records
10) "8098583"
11) inverted_sz_mb
12) "45.58
13) inverted_cap_mb
14) "56.61
15) inverted_cap_ovh
16) "0.19
17) offset_vectors_sz_mb
18) "9.27
19) skip_index_size_mb
20) "7.35
21) score_index_size_mb
22) "30.8
23) records_per_doc_avg
24) "16.1
25) bytes_per_record_avg
26) "5.90
27) offsets_per_term_avg
28) "1.20
29) offset_bits_per_record_avg
30) "8.00
```

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE

### Complexity

O(1)

### Returns

Array Response. A nested array of keys and values.

---

## FT.SEARCH 

### Format

```
FT.SEARCH {index} {query} [NOCONTENT] [VERBATIM] [NOSTOPWORDS] [WITHSCORES] [WITHPAYLOADS] [WITHSORTKEYS]
  [FILTER {numeric_field} {min} {max}] ...
  [GEOFILTER {geo_field} {lon} {lat} {raius} m|km|mi|ft]
  [INKEYS {num} {key} ... ]
  [INFIELDS {num} {field} ... ]
  [RETURN {num} {field} ... ]
  [SUMMARIZE [FIELDS {num} {field} ... ] [FRAGS {num}] [LEN {fragsize}] [SEPARATOR {separator}]]
  [HIGHLIGHT [FIELDS {num} {field} ... ] [TAGS {open} {close}]]
  [SLOP {slop}] [INORDER]
  [LANGUAGE {language}]
  [EXPANDER {expander}]
  [SCORER {scorer}]
  [PAYLOAD {payload}]
  [SORTBY {field} [ASC|DESC]]
  [LIMIT offset num]
```

### Description

Search the index with a textual query, returning either documents or just ids.

### Parameters

- **index**: The index name. The index must be first created with `FT.CREATE`.
- **query**: the text query to search. If it's more than a single word, put it in quotes.
  Refer to [query syntax](/Query_Syntax) for more details. 
- **NOCONTENT**: If it appears after the query, we only return the document ids and not 
  the content. This is useful if redisearch is only an index on an external document collection
- **RETURN {num} {field} ...**: Use this keyword to limit which fields from the document are returned.
  `num` is the number of fields following the keyword. If `num` is 0, it acts like `NOCONTENT`.
- **SUMMARIZE ...**: Use this option to return only the sections of the field which contain the matched text.
  See [Highlighting](/Highlight) for more detailts
- **HIGHLIGHT ...**: Use this option to format occurrences of matched text. See [Highligting](/Highlight) for more
  details
- **LIMIT first num**: If the parameters appear after the query, we limit the results to 
  the offset and number of results given. The default is 0 10
- **INFIELDS {num} {field} ...**: If set, filter the results to ones appearing only in specific
  fields of the document, like title or url. num is the number of specified field arguments
- **INKEYS {num} {field} ...**: If set, we limit the result to a given set of keys specified in the list. 
  the first argument must be the length of the list, and greater than zero.
  Non existent keys are ignored - unless all the keys are non existent.
- **SLOP {slop}**: If set, we allow a maximum of N intervening number of unmatched offsets between phrase terms. (i.e the slop for exact phrases is 0)
- **INORDER**: If set, and usually used in conjunction with SLOP, we make sure the query terms appear in the same order in the document as in the query, regardless of the offsets between them. 
- **FILTER numeric_field min max**: If set, and numeric_field is defined as a numeric field in 
  FT.CREATE, we will limit results to those having numeric values ranging between min and max.
  min and max follow ZRANGE syntax, and can be **-inf**, **+inf** and use `(` for exclusive ranges. 
  Multiple numeric filters for different fields are supported in one query.
- **GEOFILTER {geo_field} {lon} {lat} {radius} m|km|mi|ft**: If set, we filter the results to a given radius 
  from lon and lat. Radius is given as a number and units. See [GEORADIUS](https://redis.io/commands/georadius) for more details. 
- **NOSTOPWORDS**: If set, we do not filter stopwords from the query. 
- **WITHSCORES**: If set, we also return the relative internal score of each document. this can be
  used to merge results from multiple instances
- **WITHSORTKEYS**: Only relevant in conjunction with **SORTBY**. Returns the value of the sorting key, right after the id and score and /or payload if requested. This is usually not needed by users, and exists for distributed search coordination purposes.
- **VERBATIM**: if set, we do not try to use stemming for query expansion but search the query terms verbatim.
- **LANGUAGE {language}**: If set, we use a stemmer for the supplied language during search for query expansion.
  If querying documents in Chinese, this should be set to `chinese` in order to
  properly tokenize the query terms. 
  Defaults to English. If an unsupported language is sent, the command returns an error. See FT.ADD for the list of languages.
- **EXPANDER {expander}**: If set, we will use a custom query expander instead of the stemmer. [See Extensions](/Extensions).
- **SCORER {scorer}**: If set, we will use a custom scoring function defined by the user. [See Extensions](/Extensions).
- **PAYLOAD {payload}**: Add an arbitrary, binary safe payload that will be exposed to custom scoring functions. [See Extensions](/Extensions).
- **WITHPAYLOADS**: If set, we retrieve optional document payloads (see FT.ADD). 
  the payloads follow the document id, and if `WITHSCORES` was set, follow the scores.
- **SORTBY {field} [ASC|DESC]**: If specified, and field is a [sortable field](/Sorting), the results are ordered by the value of this field. This applies to both text and numeric fields.

### Complexity

O(n) for single word queries (though for popular words we save a cache of the top 50 results).

Complexity for complex queries changes, but in general it's proportional to the number of words and the number of intersection points between them.

### Returns

**Array reply,** where the first element is the total number of results, and then pairs of document id, and a nested array of field/value. 

If **NOCONTENT** was given, we return an array where the first element is the total number of results, and the rest of the members are document ids.

---

## FT.AGGREGATE 

### Format 

```
FT.AGGREGATE  {index_name}
  {query_string}
  [WITHSCHEMA] [VERBATIM]
  [LOAD {nargs} {property} ...]
  [GROUPBY {nargs} {property} ...
    REDUCE {func} {nargs} {arg} ... [AS {name:string}]
    ...
  ] ...
  [SORTBY {nargs} {property} [ASC|DESC] ... [MAX {num}]]
  [APPLY {expr} AS {alias}] ...
  [LIMIT {offset} {num}] ...
  [FILTER {expr}] ...
```

### Description

Run a search query on an index, and perform aggregate transformations on the results, extracting statistics etc form them. See [the full documentation on aggregations](/Aggregations/) for further details.

### Parameters

* **index_name**: The index the query is executed again.

* **query_string**: The base filtering query that retrieves the documents. It follows **the exact same syntax** as the search query, including filters, unions, not, optional, etc.

* **LOAD {nargs} {property} …**: Load document fields from the document HASH objects. This should be avoided as a general rule of thumb. Fields needed for aggregations should be stored as **SORTABLE**, where they are available to the aggregation pipeline with very load latency. LOAD hurts the performance of aggregate queries considerably, since every processed record needs to execute the equivalent of HMGET against a redis key, which when executed over millions of keys, amounts to very high processing times. 

* **GROUPBY {nargs} {property}**: Group the results in the pipeline based on one or more properties. Each group should have at least one reducer (See below), a function that handles the group entries, either counting them, or performing multiple aggregate operations (see below).
    * **REDUCE {func} {nargs} {arg} … [AS {name}]**: Reduce the matching results in each group into a single record, using a reduction function. For example COUNT will count the number of records in the group. See the Reducers section below for more details on available reducers. 
    
          The reducers can have their own property names using the `AS {name}` optional argument. If a name is not given, the resulting name will be the name of the reduce function and the group properties. For example, if a name is not given to COUNT_DISTINCT by property `@foo`, the resulting name will be `count_distinct(@foo)`. 

* **SORTBY {nargs} {property} {ASC|DESC} [MAX {num}]**: Sort the pipeline up until the point of SORTBY, using a list of properties. By default, sorting is ascending, but `ASC` or `DESC ` can be added for each propery. `nargs` is the number of sorting parameters, including ASC and DESC. for example: `SORTBY 4 @foo ASC @bar DESC`. 

    `MAX` is used to optimized sorting, by sorting only for the n-largest elements. Although it is not connected to `LIMIT`, you usually need just `SORTBY … MAX` for common queries. 

* **APPLY {expr} AS {name}**: Apply a 1-to-1 transformation on one or more properties, and either store the result as a new property down the pipeline, or replace any property using this transforamtion. `expr` is an expression that can be used to perform arithmetic operations on numeric properties, or functions that can be applied on properties depending on their types (see below), or any combination thereof. For example: `APPLY "sqrt(@foo)/log(@bar) + 5" AS baz` will evaluate this expression dynamically for each record in the pipeline and store the result as a new property called baz, that can be referenced by further APPLY / SORTBY / GROUPBY / REDUCE operations down the pipeline. 

* **LIMIT {offset} {num}**. Limit the number of results to return just `num` results starting at index `offset` (zero based). AS mentioned above, it is much more efficient to use `SORTBY … MAX` if you are interested in just limiting the optput of a sort operation.

    However, limit can be used to limit results without sorting, or for paging the n-largest results as determined by `SORTBY MAX`. For example, getting results 50-100 of the top 100 results, is most efficiently expressed as `SORTBY 1 @foo MAX 100 LIMIT 50 50`. Removing the MAX from SORTBY will result in the pipeline sorting _all_ the records and then paging over results 50-100. 

* **FILTER {expr}**. Filter the results using predicate expressions relating to values in each result. They are is applied post-query and relate to the current state of the pipeline. 

### Complexity

Non Deterministic. Depends on the query and aggregations performed, but usually it is linear to the number of results returned. 

### Returns

Array Response. Each row is an array, and represents a single aggregate result.

### Example Output

Here we are counting github events by user (actor), to produce the most active users:

```
127.0.0.1:6379> FT.AGGREGATE gh "*" GROUPBY 1 @actor REDUCE COUNT 0 AS num SORTBY 2 @num DESC MAX 10
 1) (integer) 284784
 2) 1) "actor"
    2) "lombiqbot"
    3) "num"
    4) "22197"
 3) 1) "actor"
    2) "codepipeline-test"
    3) "num"
    4) "17746"
 4) 1) "actor"
    2) "direwolf-github"
    3) "num"
    4) "10683"
 5) 1) "actor"
    2) "ogate"
    3) "num"
    4) "6449"
 6) 1) "actor"
    2) "openlocalizationtest"
    3) "num"
    4) "4759"
 7) 1) "actor"
    2) "digimatic"
    3) "num"
    4) "3809"
 8) 1) "actor"
    2) "gugod"
    3) "num"
    4) "3512"
 9) 1) "actor"
    2) "xdzou"
    3) "num"
    4) "3216"
10) 1) "actor"
    2) "opstest"
    3) "num"
    4) "2863"
11) 1) "actor"
    2) "jikker"
    3) "num"
    4) "2794"
(0.59s)
```
---

## FT.EXPLAIN

### Format

```
FT.EXPLAIN {index} {query}
```

### Description

Return the execution plan for a complex query

Example:

```sh
$ redis-cli --raw

127.0.0.1:6379> FT.EXPLAIN rd "(foo bar)|(hello world) @date:[100 200]|@date:[500 +inf]"
INTERSECT {
  UNION {
    INTERSECT {
      foo
      bar
    }
    INTERSECT {
      hello
      world
    }
  }
  UNION {
    NUMERIC {100.000000 <= x <= 200.000000}
    NUMERIC {500.000000 <= x <= inf}
  }
}
```
### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **query**: The query string, as if sent to FT.SEARCH

### Complexity

O(1)

### Returns

String Response. A string representing the execution plan (see above example). 

**Note**: You should use `redis-cli --raw` to properly read line-breaks in the returned response.

---


## FT.DEL

### Format

```
FT.DEL {index} {doc_id} [DD]
```

### Description

Delete a document from the index. Returns 1 if the document was in the index, or 0 if not. 

After deletion, the document can be re-added to the index. It will get a different internal id and will be a new document from the index's POV.

!!! warning "FT.DEL does not delete the actual document By default!"
        
        Since RediSearch regards documents as separate entities to the index, and allows things like adding existing documents or indexing without saving the document - by default FT.DEL only deletes the reference to the document from the index, not the actual Redis HASH key where the document is stored. 

        Specifying **DD** (Delete Document) after the document ID, will make RediSearch also delete the actual document **if it is in the index**.
        
        Alternatively, you can just send an extra **DEL {doc_id}** to redis and delete the document directly. You can run both of them in a MULTI transaction.

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **doc_id**: the id of the document to be deleted. It does not actually delete the HASH key in which the document is stored. Use DEL to do that manually if needed.


### Complexity

O(1)

### Returns

Integer Reply: 1 if the document was deleted, 0 if not.

--- 

## FT.GET

### Format

```
FT.GET {index} {doc id}
```

### Description

Returns the full contents of a document.

Currently it is equivalent to HGETALL, but this is future proof and will allow us to change the internal representation of documents inside redis in the future. In addition, it allows simpler implementation of fetching documents in clustered mode.

If the document does not exist or is not a HASH object, we reutrn a NULL reply

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **documentId**: The id of the document as inserted to the index

### Returns

Array Reply: Key-value pairs of field names and values of the document

---

## FT.MGET

### Format

```
FT.GET {index} {docId} ...
```

### Description

Returns the full contents of multiple documents. 
Currently it is equivalent to calling multiple HGETALL commands, although faster. 
This command is also future proof, and will allow us to change the internal representation of documents inside redis in the future. 
In addition, it allows simpler implementation of fetching documents in clustered mode.

We return an array with exactly the same number of elements as the number of keys sent to the command. 

Each element in turn is an array of key-value pairs representing the document. 

If a document is not found or is not a valid HASH object, its place in the parent array is filled with a Null reply object.

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **documentIds**: The ids of the requested documents as inserted to the index

### Returns

Array Reply: An array with exactly the same number of elements as the number of keys sent to the command.  Each element in it is either an array representing the document, or Null if it was not found.

---

## FT.DROP

### Format

```
FT.DROP {index} [KEEPDOCS]
```

### Description

Deletes all the keys associated with the index. 

By default DROP deletes the document hashes as well, but adding the KEEPDOCS option keeps the documents in place, ready for re-indexing.

If no other data is on the redis instance, this is equivalent to FLUSHDB, apart from the fact
that the index specification is not deleted.

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **KEEPDOCS**: IF set, the drop operation will not delete the actual document hashes.

### Returns

Status Reply: OK on success.

---

## FT.TAGVALS

### Format

```
FT.TAGVALS {index} {field_name}
```

### Description

Return the distinct tags indexed in a [Tag field](/Tags/). 

This is useful if your tag field indexes things like cities, categories, etc.

!!! warning "Limitations"
  
      There is no paging or sorting, the tags are not alphabetically sorted. 

      This command only operates on [Tag fields](/Tags/).  

      The strings return lower-cased and stripped of whitespaces, but otherwise unchanged.

### Parameters

- **index**: The Fulltext index name. The index must be first created with FT.CREATE
- **filed_name**: The name of a Tag file defined in the schema.

### Returns

Array Reply: All the distinct tags in the tag index.

### Complexity

O(n), n being the cardinality of the tag field.

---

## FT.SUGADD

### Format

```
FT.SUGADD {key} {string} {score} [INCR] [PAYLOAD {payload}]
```

### Description

Add a suggestion string to an auto-complete suggestion dictionary. This is disconnected from the
index definitions, and leaves creating and updating suggestino dictionaries to the user.

### Parameters

- **key**: the suggestion dictionary key.
- **string**: the suggestion string we index
- **score**: a floating point number of the suggestion string's weight
- **INCR**: if set, we increment the existing entry of the suggestion by the given score, instead of replacing the score. This is useful for updating the dictionary based on user queries in real time
- **PAYLOAD {payload}**: If set, we save an extra payload with the suggestion, that can be fetched by adding the `WITHPAYLOADS` argument to `FT.SUGGET`.

### Returns:

Integer Reply: the current size of the suggestion dictionary.

---

## FT.SUGGET

### Format

```
FT.SUGGET {key} {prefix} [FUZZY] [WITHPAYLOADS] [MAX num]
```

### Description

Get completion suggestions for a prefix

### Parameters:

- **key**: the suggestion dictionary key.
- **prefix**: the prefix to complete on
- **FUZZY**: if set, we do a fuzzy prefix search, including prefixes at Levenshtein distance of 1 from the prefix sent
- **MAX num**: If set, we limit the results to a maximum of `num`. (**Note**: The default is 5, and the number cannot be greater than 10).
- **WITHSCORES**: If set, we also return the score of each suggestion. this can be used to merge results from multiple instances
- **WITHPAYLOADS**: If set, we return optional payloads saved along with the suggestions. If no payload is present for an entry, we return a Null Reply.

### Returns:

Array Reply: a list of the top suggestions matching the prefix, optionally with score after each entry

---

## FT.SUGDEL

### Format

```
FT.SUGDEL {key} {string}
```

### Description

Delete a string from a suggestion index. 

### Parameters

- **key**: the suggestion dictionary key.
- **string**: the string to delete

### Returns:

Integer Reply: 1 if the string was found and deleted, 0 otherwise.

----

## FT.SUGLEN

Format

```
FT.SUGLEN {key}
```

### Description

Get the size of an autoc-complete suggestion dictionary

### Parameters

* **key**: the suggestion dictionary key.

### Returns:

Integer Reply: the current size of the suggestion dictionary.

----

## FT.OPTIMIZE ** DEPRECATED **

Format

```
FT.OPTIMIZE {index}
```

Description

This command is deprecated. Index optimizations are done by the internal garbage collector in the background. Client libraries should not implement this command, and remove it if they haven't already. 

---

## FT.SYNADD

Format

```
FT.SYNADD <index name> <term1> <term2> ...
```

Description

The command is used to create a new synonyms group. The command returns the synonym group id which can later be used to add additional terms to that synonym group. Only documents which was indexed after the adding operation will be effected.

---

## FT.SYNUPDATE

Format

```
FT.SYNUPDATE <index name> <synonym group id> <term1> <term2> ...
```

Description

The command is used to update an existing synonym group with additional terms. Only documents which was indexed after the update will be effected.

---

## FT.SYNDUMP

Format

```
FT.SYNDUMP <index name>
```

Description

The Command is used to dump the synonyms data structure. Returns a list of synonym terms and their synonym group ids.

---
