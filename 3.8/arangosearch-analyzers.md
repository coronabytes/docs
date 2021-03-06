---
layout: default
description: Analyzers parse input values and transform them into sets of sub-values, for example by breaking up text into words.
title: ArangoSearch Analyzers
redirect_from:
  - views-arango-search-analyzers.html # 3.4 -> 3.5
---
ArangoSearch Analyzers
======================

Analyzers parse input values and transform them into sets of sub-values,
for example by breaking up text into words. If they are used in Views then
the documents' attribute values of the linked collections are used as input
and additional metadata is produced internally. The data can then be used for
searching and sorting to provide the most appropriate match for the specified
conditions, similar to queries to web search engines.

Analyzers can be used on their own to tokenize and normalize strings in AQL
queries with the [`TOKENS()` function](aql/functions-arangosearch.html#tokens).

How Analyzers process values depends on their type and configuration.
The configuration is comprised of type-specific properties and list of features.
The features control the additional metadata to be generated to augment View
indexes, to be able to rank results for instance.

Analyzers can be managed via an [HTTP API](http/analyzers.html) and through
a [JavaScript module](appendix-java-script-modules-analyzers.html).

{% include youtube.html id="tbOTYL26reg" %}

Value Handling
--------------

While most of the Analyzer functionality is geared towards text processing,
there is no restriction to strings as input data type when using them through
Views – your documents could have attributes of any data type after all.

Strings are processed according to the Analyzer, whereas other primitive data
types (`null`, `true`, `false`, numbers) are added to the index unchanged.

The elements of arrays are unpacked, processed and indexed individually,
regardless of the level of nesting. That is, strings are processed by the
configured Analyzer(s) and other primitive values are indexed as-is.

Objects, including any nested objects, are indexed as sub-attributes.
This applies to sub-objects as well as objects in arrays. Only primitive values
are added to the index, arrays and objects can not be searched for.

Also see:
- [SEARCH operation](aql/operations-search.html) on how to query indexed
  values such as numbers and nested values
- [ArangoSearch Views](arangosearch-views.html) for details about how
  compound data types (arrays, objects) get indexed

Analyzer Names
--------------

Each Analyzer has a name for identification with the following
naming conventions:

- The name must only consist of the letters `a` to `z` (both in lower and
  upper case), the numbers `0` to `9`, underscore (`_`) and dash (`-`) symbols.
  This also means that any non-ASCII names are not allowed.
- It must always start with a letter.
- The maximum allowed length of a name is 254 bytes.
- Analyzer names are case-sensitive.

Custom Analyzers are stored per database, in a system collection `_analyzers`.
The names get prefixed with the database name and two colons, e.g.
`myDB::customAnalyzer`.This does not apply to the globally available
[built-in Analyzers](#built-in-analyzers), which are not stored in an
`_analyzers` collection.

Custom Analyzers stored in the `_system` database can be referenced in queries
against other databases by specifying the prefixed name, e.g.
`_system::customGlobalAnalyzer`. Analyzers stored in databases other than
`_system` can not be accessed from within another database however.

Analyzer Types
--------------

The currently implemented Analyzer types are:

- `identity`: treat value as atom (no transformation)
- `delimiter`: split into tokens at user-defined character
- `stem`: apply stemming to the value as a whole
- `norm`: apply normalization to the value as a whole
- `ngram`: create n-grams from value with user-defined lengths
- `text`: tokenize into words, optionally with stemming,
  normalization, stop-word filtering and edge n-gram generation
- `aql`: for running AQL query to prepare tokens for index
- `pipeline`: for chaining multiple Analyzers
- `geojson`: breaks up a GeoJSON object into a set of indexable tokens
- `geopoint`: breaks up a JSON object describing a coordinate into a set of
  indexable tokens

Available normalizations are case conversion and accent removal
(conversion of characters with diacritical marks to the base characters).

Analyzer   /   Feature  | Tokenization | Stemming | Normalization | N-grams
:-----------------------|:------------:|:--------:|:-------------:|:------:
[Identity](#identity)   |      No      |    No    |      No       |   No
[N-gram](#n-gram)       |      No      |    No    |      No       |  Yes
[Delimiter](#delimiter) |    (Yes)     |    No    |      No       |   No
[Stem](#stem)           |      No      |   Yes    |      No       |   No
[Norm](#norm)           |      No      |    No    |     Yes       |   No
[Text](#text)           |     Yes      |   Yes    |     Yes       | (Yes)
[AQL](#aql)             |    (Yes)     |  (Yes)   |    (Yes)      | (Yes)
[Pipeline](#pipeline)   |    (Yes)     |  (Yes)   |    (Yes)      | (Yes)
[GeoJSON](#geojson)     |      –       |    –     |      –        |   –
[GeoPoint](#geopoint)   |      –       |    –     |      –        |   –

Analyzer Properties
-------------------

The valid attributes/values for the *properties* are dependant on what *type*
is used. For example, the `delimiter` type needs to know the desired delimiting
character(s), whereas the `text` type takes a locale, stop-words and more.

### Identity

An Analyzer applying the `identity` transformation, i.e. returning the input
unmodified.

It does not support any *properties* and will ignore them.

### Delimiter

An Analyzer capable of breaking up delimited text into tokens as per
[RFC 4180](https://tools.ietf.org/html/rfc4180)
(without starting new records on newlines).

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `delimiter` (string): the delimiting character(s)

### Stem

An Analyzer capable of stemming the text, treated as a single token,
for supported languages.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format
  `language[_COUNTRY][.encoding][@variant]` (square brackets denote optional
  parts), e.g. `"de.utf-8"` or `"en_US.utf-8"`. Only UTF-8 encoding is
  meaningful in ArangoDB. Also see [Supported Languages](#supported-languages).

###  Norm

An Analyzer capable of normalizing the text, treated as a single
token, i.e. case conversion and accent removal.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format
  `language[_COUNTRY][.encoding][@variant]` (square brackets denote optional
  parts), e.g. `"de.utf-8"` or `"en_US.utf-8"`. Only UTF-8 encoding is
  meaningful in ArangoDB. Also see [Supported Languages](#supported-languages).
- `accent` (boolean, _optional_):
  - `true` to preserve accented characters (default)
  - `false` to convert accented characters to their base characters
- `case` (string, _optional_):
  - `"lower"` to convert to all lower-case characters
  - `"upper"` to convert to all upper-case characters
  - `"none"` to not change character case (default)

### N-gram

An Analyzer capable of producing n-grams from a specified input in a range of
min..max (inclusive). Can optionally preserve the original input.

This Analyzer type can be used to implement substring matching.
Note that it slices the input based on bytes and not characters by default
(*streamType*). The *"binary"* mode supports single-byte characters only;
multi-byte UTF-8 characters raise an *Invalid UTF-8 sequence* query error.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `min` (number): unsigned integer for the minimum n-gram length
- `max` (number): unsigned integer for the maximum n-gram length
- `preserveOriginal` (boolean):
  - `true` to include the original value as well
  - `false` to produce the n-grams based on *min* and *max* only
- `startMarker` (string, _optional_): this value will be prepended to n-grams
  which include the beginning of the input. Can be used for matching prefixes.
  Choose a character or sequence as marker which does not occur in the input.
- `endMarker` (string, _optional_): this value will be appended to n-grams
  which include the end of the input. Can be used for matching suffixes.
  Choose a character or sequence as marker which does not occur in the input.
- `streamType` (string, _optional_): type of the input stream
  - `"binary"`: one byte is considered as one character (default)
  - `"utf8"`: one Unicode codepoint is treated as one character

**Examples**

With *min* = `4` and *max* = `5`, the Analyzer will produce the following
n-grams for the input string `"foobar"`:
- `"foob"`
- `"fooba"`
- `"foobar"` (if *preserveOriginal* is enabled)
- `"ooba"`
- `"oobar"`
- `"obar"`

An input string `"foo"` will not produce any n-gram unless *preserveOriginal*
is enabled, because it is shorter than the *min* length of 4.

Above example but with *startMarker* = `"^"` and *endMarker* = `"$"` would
produce the following:
- `"^foob"`
- `"^fooba"`
- `"^foobar"` (if *preserveOriginal* is enabled)
- `"foobar$"` (if *preserveOriginal* is enabled)
- `"ooba"`
- `"oobar$"`
- `"obar$"`

### Text

An Analyzer capable of breaking up strings into individual words while also
optionally filtering out stop-words, extracting word stems, applying
case conversion and accent removal.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `locale` (string): a locale in the format
  `language[_COUNTRY][.encoding][@variant]` (square brackets denote optional
  parts), e.g. `"de.utf-8"` or `"en_US.utf-8"`. Only UTF-8 encoding is
  meaningful in ArangoDB. Also see [Supported Languages](#supported-languages).
- `accent` (boolean, _optional_):
  - `true` to preserve accented characters
  - `false` to convert accented characters to their base characters (default)
- `case` (string, _optional_):
  - `"lower"` to convert to all lower-case characters (default)
  - `"upper"` to convert to all upper-case characters
  - `"none"` to not change character case
- `stemming` (boolean, _optional_):
  - `true` to apply stemming on returned words (default)
  - `false` to leave the tokenized words as-is
- `edgeNgram` (object, _optional_): if present, then edge n-grams are generated
  for each token (word). That is, the start of the n-gram is anchored to the
  beginning of the token, whereas the `ngram` Analyzer would produce all
  possible substrings from a single input token (within the defined length
  restrictions). Edge n-grams can be used to cover word-based auto-completion
  queries with an index, for which you should set the following other options:
  `accent: false`, `case: "lower"` and most importantly `stemming: false`.
  - `min` (number, _optional_): minimal n-gram length
  - `max` (number, _optional_): maximal n-gram length
  - `preserveOriginal` (boolean, _optional_): whether to include the original
    token even if its length is less than *min* or greater than *max*
- `stopwords` (array, _optional_): an array of strings with words to omit
  from result. Default: load words from `stopwordsPath`. To disable stop-word
  filtering provide an empty array `[]`. If both *stopwords* and
  *stopwordsPath* are provided then both word sources are combined.
- `stopwordsPath` (string, _optional_): path with a *language* sub-directory
  (e.g. `en` for a locale `en_US.utf-8`) containing files with words to omit.
  Each word has to be on a separate line. Everything after the first whitespace
  character on a line will be ignored and can be used for comments. The files
  can be named arbitrarily and have any file extension (or none).

  Default: if no path is provided then the value of the environment variable
  `IRESEARCH_TEXT_STOPWORD_PATH` is used to determine the path, or if it is
  undefined then the current working directory is assumed. If the `stopwords`
  attribute is provided then no stop-words are loaded from files, unless an
  explicit *stopwordsPath* is also provided.

  Note that if the *stopwordsPath* can not be accessed, is missing language
  sub-directories or has no files for a language required by an Analyzer,
  then the creation of a new Analyzer is refused. If such an issue is 
  discovered for an existing Analyzer during startup then the server will
  abort with a fatal error.

**Examples**

The built-in `text_en` Analyzer has stemming enabled (note the word endings):

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerTextStem
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerTextStem}
      db._query(`RETURN TOKENS("Crazy fast NoSQL-database!", "text_en")`).toArray();
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerTextStem
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

You may create a custom Analyzer with the same configuration but with stemming
disabled like this:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerTextNoStem
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerTextNoStem}
      var analyzers = require("@arangodb/analyzers")
    | var a = analyzers.save("text_en_nostem", "text", {
    |   locale: "en.utf-8",
    |   case: "lower",
    |   accent: false,
    |   stemming: false,
    |   stopwords: []
      }, ["frequency","norm","position"])
      db._query(`RETURN TOKENS("Crazy fast NoSQL-database!", "text_en_nostem")`).toArray();
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerTextNoStem
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Custom text Analyzer with the edge n-grams feature and normalization enabled,
stemming disabled and `"the"` defined as stop-word to exclude it:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerTextEdgeNgram
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerTextEdgeNgram}
    ~ var analyzers = require("@arangodb/analyzers")
    | var a = analyzers.save("text_edge_ngrams", "text", {
    |   edgeNgram: { min: 3, max: 8, preserveOriginal: true },
    |   locale: "en.utf-8",
    |   case: "lower",
    |   accent: false,
    |   stemming: false,
    |   stopwords: [ "the" ]
      }, ["frequency","norm","position"])
    | db._query(`RETURN TOKENS(
    |   "The quick brown fox jumps over the dogWithAVeryLongName",
    |   "text_edge_ngrams"
      )`).toArray();
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerTextEdgeNgram
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

### Pipeline

<small>Introduced in: v3.8.0</small>

An Analyzer capable of chaining effects of multiple Analyzers into one.
The pipeline is a list of Analyzers, where the output of an Analyzer is passed
to the next for further processing. The final token value is determined by last
Analyzer in the pipeline.

The Analyzer is designed for cases like the following:
- Normalize text for a case insensitive search and apply ngram tokenization
- Split input with `delimiter` Analyzer, followed by stemming with the `stem`
  Analyzer

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `pipeline` (array): an array of Analyzer definition-like objects with
  `type` and `properties` attributes

**Examples**

Normalize to all uppercase and compute bigrams:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerPipelineUpperNgram
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerPipelineUpperNgram}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("ngram_upper", "pipeline", { pipeline: [
    |   { type: "norm", properties: { locale: "en.utf-8", case: "upper" } },
    |   { type: "ngram", properties: { min: 2, max: 2, preserveOriginal: false, streamType: "utf8" } }
      ] }, ["frequency", "norm", "position"]);
      db._query(`RETURN TOKENS("Quick brown foX", "ngram_upper")`).toArray();
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerPipelineUpperNgram
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Split at delimiting characters `,` and `;`, then stem the tokens:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerPipelineDelimiterStem
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerPipelineDelimiterStem}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("delimiter_stem", "pipeline", { pipeline: [
    |   { type: "delimiter", properties: { delimiter: "," } },
    |   { type: "delimiter", properties: { delimiter: ";" } },
    |   { type: "stem", properties: { locale: "en.utf-8" } }
      ] }, ["frequency", "norm", "position"]);
      db._query(`RETURN TOKENS("delimited,stemmable;words", "delimiter_stem")`).toArray();
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerPipelineDelimiterStem
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

### AQL

<small>Introduced in: v3.8.0</small>

An Analyzer capable of running a restricted AQL query to perform
data manipulation / filtering.

The query must not access the storage engine. This means no `FOR` loops over
collections or Views, no use of the `DOCUMENT()` function, no graph traversals.
AQL functions are allowed as long as they do not involve Analyzers (`TOKENS()`,
`PHRASE()`, `NGRAM_MATCH()`, `ANALYZER()` etc.) or data access, and if they can
be run on DB-Servers in case of a cluster deployment. User-defined functions
are not permitted.

The input data is provided to the query via a bind parameter `@param`.
It is always a string. The query result should be a string, `null` or
an array of strings (and `null`s).

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `queryString` (string): AQL query to be executed
- `collapsePositions` (boolean):
  - `true`: set the position to 0 for all members of the query result array
  - `false` (default): set the position corresponding to the index of the
    result array member
- `keepNull` (boolean):
  - `true` (default): treat `null` like an empty string
  - `false`: discard `null`s from View index. Can be used for index filtering
    (i.e. make your query return null for unwanted data). Note that empty
    results are always discarded.
- `batchSize` (integer): number between 1 and 1000 (default = 1) that
  determines the batch size for reading data from the query. In general, a
  single token is expected to be returned. However, if the query is expected
  to return many results, then increasing `batchSize` trades memory for
  performance.
- `memoryLimit` (integer): memory limit for query execution in bytes.
  (default is 1048576 = 1Mb) Maximum is 33554432U (32Mb)

**Examples**

Soundex Analyzer for a phonetically similar term search:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerAqlSoundex
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerAqlSoundex}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("soundex", "aql", { queryString: "RETURN SOUNDEX(@param)" },
        ["frequency", "norm", "position"]);
      db._query("RETURN TOKENS('ArangoDB', 'soundex')").toArray();
    ~ analyzers.remove(a.name);
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerAqlSoundex
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Concatenating Analyzer for conditionally adding a custom prefix or suffix:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerAqlConcat
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerAqlConcat}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("concat", "aql", { queryString:
    |   "RETURN LOWER(LEFT(@param, 5)) == 'inter' ? CONCAT(@param, 'ism') : CONCAT('inter', @param)"
      }, ["frequency", "norm", "position"]);
      db._query("RETURN TOKENS('state', 'concat')");
      db._query("RETURN TOKENS('international', 'concat')");
    ~ analyzers.remove(a.name);
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerAqlConcat
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Filtering Analyzer that ignores unwanted data based on the prefix `"ir"`,
with `keepNull: false` and explicitly returning `null`:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerAqlFilterNull
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerAqlFilterNull}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("filter", "aql", { keepNull: false, queryString:
    |   "RETURN LOWER(LEFT(@param, 2)) == 'ir' ? null : @param"
      }, ["frequency", "norm", "position"]);
      db._query("RETURN TOKENS('regular', 'filter')");
      db._query("RETURN TOKENS('irregular', 'filter')");
    ~ analyzers.remove(a.name);
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerAqlFilterNull
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Filtering Analyzer that discards unwanted data based on the prefix `"ir"`,
using a filter for an empty result, which is discarded from the View index even
without `keepNull: false`:

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerAqlFilter
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerAqlFilter}
      var analyzers = require("@arangodb/analyzers");
    | var a = analyzers.save("filter", "aql", { queryString:
    |   "FILTER LOWER(LEFT(@param, 2)) != 'ir' RETURN @param"
      }, ["frequency", "norm", "position"]);
      var coll = db._create("coll");
      var doc1 = db.coll.save({ value: "regular" });
      var doc2 = db.coll.save({ value: "irregular" });
    | var view = db._createView("view", "arangosearch",
        { links: { coll: { fields: { value: { analyzers: ["filter"] }}}}})
    ~ db._query("FOR doc IN view OPTIONS { waitForSync: true } LIMIT 1 RETURN true");
      db._query("FOR doc IN view SEARCH ANALYZER(doc.value IN ['regular', 'irregular'], 'filter') RETURN doc");
    ~ db._dropView(view.name())
    ~ analyzers.remove(a.name);
    ~ db._drop(coll.name());
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerAqlFilter
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

Custom tokenization with `collapsePositions` on and off:
The input string `"A-B-C-D"` is split into an array of strings
`["A", "B", "C", "D"]`. The position metadata (as used by the `PHRASE()`
function) is set to 0 for all four strings if `collapsePosition` is enabled.
Otherwise the position is set to the respective array index, 0 for `"A"`,
1 for `"B"` and so on.

| `collapsePosition` | A | B | C | D |
|-------------------:|:-:|:-:|:-:|:-:|
|             `true` | 0 | 0 | 0 | 0 |
|            `false` | 0 | 1 | 2 | 3 |

{% arangoshexample examplevar="examplevar" script="script" result="result" %}
    @startDocuBlockInline analyzerAqlCollapse
    @EXAMPLE_ARANGOSH_OUTPUT{analyzerAqlCollapse}
      var analyzers = require("@arangodb/analyzers");
    | var a1 = analyzers.save("collapsed", "aql", { collapsePositions: true, queryString:
    |   "FOR d IN SPLIT(@param, '-') RETURN d"
      }, ["frequency", "norm", "position"]);
    | var a2 = analyzers.save("uncollapsed", "aql", { collapsePositions: false, queryString:
    |   "FOR d IN SPLIT(@param, '-') RETURN d"
      }, ["frequency", "norm", "position"]);
      var coll = db._create("coll");
    | var view = db._createView("view", "arangosearch",
        { links: { coll: { analyzers: [ "collapsed", "uncollapsed" ], includeAllFields: true }}});
      var doc = db.coll.save({ text: "A-B-C-D" });
    ~ db._query("FOR d IN view OPTIONS { waitForSync: true } LIMIT 1 RETURN true");
      db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, 1, {TERM: 'D'}, 'uncollapsed') RETURN d");
      db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, -1, {TERM: 'D'}, 'uncollapsed') RETURN d");
      db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, 1, {TERM: 'D'}, 'collapsed') RETURN d");
      db._query("FOR d IN view SEARCH PHRASE(d.text, {TERM: 'B'}, -1, {TERM: 'D'}, 'collapsed') RETURN d");
    ~ db._dropView(view.name());
    ~ analyzers.remove(a1.name);
    ~ analyzers.remove(a2.name);
    ~ db._drop(coll.name());
    @END_EXAMPLE_ARANGOSH_OUTPUT
    @endDocuBlock analyzerAqlCollapse
{% endarangoshexample %}
{% include arangoshexample.html id=examplevar script=script result=result %}

The position data is not directly exposed, but we can see its effects through
the `PHRASE()` function. There is one token between `"B"` and `"D"` to skip in
case of uncollapsed positions. With positions collapsed, both are in the same
position, thus there is negative one to skip to match the tokens.

### GeoJSON

<small>Introduced in: v3.8.0</small>

An Analyzer capable of breaking up a GeoJSON object into a set of
indexable tokens for further usage with
[ArangoSearch Geo functions](aql/functions-arangosearch.html#geo-functions).

GeoJSON object example:

```js
{
  "type": "Point",
  "coordinates": [ -73.97, 40.78 ] // [ longitude, latitude ]
}
```

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `type` (string, _optional_):
  - `"shape"` (default): index all GeoJSON geometry types (Point, Polygon etc.)
  - `"centroid"`: compute and only index the centroid of the input geometry
  - `"point"`: only index GeoJSON objects of type Point, ignore all other
    geometry types
- `options` (object, _optional_): options for fine-tuning geo queries.
  These options should generally remain unchanged
  - `maxCells` (number, _optional_): maximum number of S2 cells (default: 20)
  - `minLevel` (number, _optional_): the least precise S2 level (default: 4)
  - `maxLevel` (number, _optional_): the most precise S2 level (default: 23)

### GeoPoint

<small>Introduced in: v3.8.0</small>

An Analyzer capable of breaking up JSON object describing a coordinate into a
set of indexable tokens for further usage with
[ArangoSearch Geo functions](aql/functions-arangosearch.html#geo-functions).

The Analyzer can be used for two different coordinate representations:
- an array with two numbers as elements in the format
  `[<latitude>, <longitude>]`, e.g. `[40.78, -73.97]`.
- two separate number attributes, one for latitude and one for
  longitude, e.g. `{ location: { lat: 40.78, lon: -73.97 } }`.
  The attributes cannot be at the top level of the document, but must be nested
  like in the example, so that the Analyzer can be defined for the field
  `location` with the Analyzer properties
  `{ "latitude": ["lat"], "longitude": ["long"] }`.

The *properties* allowed for this Analyzer are an object with the following
attributes:

- `latitude` (array, _optional_): array of strings that describes the
  attribute path of the latitude value relative to the field for which the
  Analyzer is defined in the View
- `longitude` (array, _optional_): array of strings that describes the
  attribute path of the longitude value relative to the field for which the
  Analyzer is defined in the View
- `options` (object, _optional_): options for fine-tuning geo queries.
  These options should generally remain unchanged
  - `minCells` (number, _optional_): maximum number of S2 cells (default: 20)
  - `minLevel` (number, _optional_): the least precise S2 level (default: 4)
  - `maxLevel` (number, _optional_): the most precise S2 level (default: 23)

Analyzer Features
-----------------

The *features* of an Analyzer determine what term matching capabilities will be
available and as such are only applicable in the context of ArangoSearch Views.

The valid values for the features are dependant on both the capabilities of
the underlying *type* and the query filtering and sorting functions that the
result can be used with. For example the *text* type will produce
`frequency` + `norm` + `position` and the `PHRASE()` AQL function requires
`frequency` + `position` to be available.

Currently the following *features* are supported:

- **frequency**: how often a term is seen, required for `PHRASE()`
- **norm**: the field normalization factor
- **position**: sequentially increasing term position, required for `PHRASE()`.
  If present then the *frequency* feature is also required

Built-in Analyzers
------------------

There is a set of built-in Analyzers which are available by default for
convenience and backward compatibility. They can not be removed.

The `identity` Analyzer has no properties and the features `frequency`
and `norm`. The Analyzers of type `text` all tokenize strings with stemming
enabled, no stopwords configured, accent removal and case conversion to
lowercase turned on and the features `frequency`, `norm` and `position`:

Name       | Type       | Locale (Language)       | Case    | Accent  | Stemming | Stopwords | Features |
-----------|------------|-------------------------|---------|---------|----------|-----------|----------|
`identity` | `identity` |                         |         |         |          |           | `["frequency", "norm"]`
`text_de`  | `text`     | `de.utf-8` (German)     | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_en`  | `text`     | `en.utf-8` (English)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_es`  | `text`     | `es.utf-8` (Spanish)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_fi`  | `text`     | `fi.utf-8` (Finnish)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_fr`  | `text`     | `fr.utf-8` (French)     | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_it`  | `text`     | `it.utf-8` (Italian)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_nl`  | `text`     | `nl.utf-8` (Dutch)      | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_no`  | `text`     | `no.utf-8` (Norwegian)  | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_pt`  | `text`     | `pt.utf-8` (Portuguese) | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_ru`  | `text`     | `ru.utf-8` (Russian)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_sv`  | `text`     | `sv.utf-8` (Swedish)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
`text_zh`  | `text`     | `zh.utf-8` (Chinese)    | `lower` | `false` | `true`   | `[ ]`     | `["frequency", "norm", "position"]`
{:class="table-scroll"}

Note that _locale_, _case_, _accent_, _stemming_ and _stopwords_ are Analyzer
properties. `text_zh` does not have actual stemming support for Chinese despite
what the property value suggests.

Supported Languages
-------------------

Analyzers rely on [ICU](http://site.icu-project.org/){:target="_blank"} for
language-dependent tokenization and normalization. The ICU data file
`icudtl.dat` that ArangoDB ships with contains information for a lot of
languages, which are technically all supported.

{% hint 'warning' %}
The alphabetical order of characters is not taken into account by ArangoSearch,
i.e. range queries in SEARCH operations against Views will not follow the
language rules as per the defined Analyzer locale nor the server language
(startup option `--default-language`)!
Also see [Known Issues](release-notes-known-issues37.html#arangosearch).
{% endhint %}

Stemming support is provided by [Snowball](https://snowballstem.org/){:target="_blank"},
which supports the following languages:

Language     | Code
-------------|-----
Arabic     * | `ar`
Basque     * | `eu`
Catalan    * | `ca`
Danish     * | `da`
Dutch        | `nl`
English      | `en`
Finnish      | `fi`
French       | `fr`
German       | `de`
Greek      * | `el`
Hindi      * | `hi`
Hungarian  * | `hu`
Indonesian * | `id`
Irish      * | `ga`
Italian      | `it`
Lithuanian * | `lt`
Nepali     * | `ne`
Norwegian    | `no`
Portuguese   | `pt`
Romanian   * | `ro`
Russian      | `ru`
Serbian    * | `sr`
Spanish      | `es`
Swedish      | `sv`
Tamil      * | `ta`
Turkish    * | `tr`

\* <small>Introduced in: v3.7.0</small>
