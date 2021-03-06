---
title: >
  Packed CBOR
abbrev: Packed CBOR
docname: draft-ietf-cbor-packed-latest
# date: 2021-02-09

stand_alone: true
kramdown_options:
  auto_id_prefix: sec-

ipr: trust200902
keyword: Internet-Draft
cat: info

pi: [toc, sortrefs, symrefs, compact, comments]

author:
  -
    ins: C. Bormann
    name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org


normative:
  RFC7049: orig
  RFC8949: bis
  IANA.cbor-tags: tags
  IANA.cbor-simple-values: simple
  RFC8610: cddl

informative:
  RFC8742: seq
  RFC6920: ni

--- abstract

The Concise Binary Object Representation (CBOR, RFC 8949) is a data
format whose design goals include the possibility of extremely small
code size, fairly small message size, and extensibility without the
need for version negotiation.

CBOR does not provide any forms of data compression.
CBOR data items, in particular when generated from legacy data models
often allow considerable gains in compactness when applying data
compression.
While traditional data compression techniques such as DEFLATE (RFC 1951) work
well for CBOR, their disadvantage is that the receiver needs to unpack
the compressed form to make use of data.

This specification describes Packed CBOR, a simple transformation of a
CBOR data item into another CBOR data item that is almost as easy to
consume as the original CBOR data item.  A separate decompression
step is therefore often not required at the receiver.


--- note_Note_to_Readers

This is a working-group draft of the CBOR working group of the
IETF, <https://datatracker.ietf.org/wg/cbor/about/>.
Discussion takes places on the github repository
<https://github.com/cbor-wg/cbor-packed> and on the CBOR WG mailing
list, <https://www.ietf.org/mailman/listinfo/cbor>.


--- middle

Introduction        {#intro}
============

(TO DO, expand on text from abstract here; move references here and
neuter them in the abstract as per Section 4.3 of {{?RFC7322}}.)

The specification defines a transformation from a Packed CBOR data
item to the original CBOR data item; it does not define an algorithm
for an actual packer.  Different packers can differ in the amount of
effort they invest in arriving at a minimal packed form.

Packed CBOR can employ two kinds of optimization:

- item sharing: substructures (data items) that occur repeatedly
  in the original CBOR data item can be collapsed to a simple
  reference to a common representation of that data item.
  The processing required during consumption is limited to following
  that reference.
- affix sharing: data items (strings, containers) that share a prefix
  or suffix (affix) can be replaced by a
  reference to a common affix plus the rest of the data item.  For
  strings, the
  processing required during consumption is similar to following the
  affix reference plus that for an indefinite-length string.

A specific application protocol that employs Packed CBOR might allow
both kinds of optimization or limit the representation to item
sharing only.

Packed CBOR is defined in two parts: Referencing packing tables
({{sec-packed-cbor}}) and setting up packing tables
({{sec-table-setup}}).


Terminology         {#terms}
------------

{::boilerplate bcp14}

Packed reference:
: A shared item reference or an affix reference

Shared item reference:
: A reference to a shared item as defined in {{sec-referencing-shared-items}}

Affix reference:
: A reference that combines an affix item as defined in {{sec-referencing-affix-items}}.

Affix:
: Prefix or suffix.

Packing tables:
: The triple of a shared item table, a prefix table, and a suffix table.

Expansion:
: The result of applying a packed reference in the context of given
  Packing tables.

The definitions of {{-bis}} apply.
The term "byte" is used in its now customary sense as a synonym for
"octet".
Where bit arithmetic is explained, this document uses the notation
familiar from the programming language C (including C++14's 0bnnn
binary literals), except that, in the plain text form of this document,
the operator "^" stands for exponentiation.

# Packed CBOR

This section describes the packing tables, their structure, and how
they are referenced.

## Packing Tables

At any point within a data item making use of Packed CBOR, there is a
Current Set of packing tables that applies.

There are three packing tables in a Current Set:

* Shared item table
* Prefix table
* Suffix table

Without any table setup, all these tables are empty arrays.
Table setup can cause these arrays to be non-empty, where the elements are
(potentially themselves packed) data items.
Each of the tables is indexed by an unsigned integer (starting
from 0), which may be computed from information in tags and their
content as well as from CBOR simple values.

## Referencing Shared Items

Shared items are stored in the shared item table of the Current Set.

The shared data items are referenced by using the reference data items
in {{tab-shared}}.  When reconstructing the original data item, such a
reference is replaced by the referenced data item, which is then
recursively unpacked.

| reference                 | table index   |
| Simple value 0-15         | 0-15          |
| Tag 6(unsigned integer N) | 16 + 2\*N     |
| Tag 6(negative integer N) | 16 - 2\*N - 1 |
{: #tab-shared title="Referencing Shared Values"}

As examples in CBOR diagnostic notation ({{Section 8 of RFC8949}}),
the first 22 elements of the shared item table are referenced by
`simple(0)`, `simple(1)`, ... `simple(15)`, `6(0)`, `6(-1)`, `6(1)`,
`6(-2)`, `6(2)`, `6(-3)`.  (The alternation between unsigned and
negative integers for even/odd table index values makes systematic use
of shorter integer encodings first.)

Taking into account the encoding of these referring data items, there
are 16 one-byte references, 48 two-byte references, 512 three-byte
references, 131072 four-byte references, etc.
As integers can grow to very large (or negative) values, there is no
practical limit to how many shared items might be used in a Packed
CBOR item.

Note that the semantics of Tag 6 depend on its content: An integer
turns the tag into a shared item reference, a string or container (map
or array) into a prefix reference (see {{tab-prefix}}).


## Referencing Affix Items

Prefix items are stored in the prefix table of the Current Set;
suffix items are stored in the suffix table of the Current Set.
We collectively call these items affix items; when referencing, which
of the tables is actually used depends on whether a prefix or a suffix
reference was used.

| prefix reference                  |    table index |
|-----------------------------------|----------------|
| Tag 6(suffix)                     |              0 |
| Tag 225-255(suffix)               |           1-31 |
| Tag 28704-32767(suffix)           |        32-4095 |
| Tag 1879052288-2147483647(suffix) | 4096-268435455 |
{: #tab-prefix cols='l r' title="Referencing Prefix Values"}

| suffix reference                  |   table index |
|-----------------------------------|---------------|
| Tag 216-223(prefix)               |           0-7 |
| Tag 27647-28671(prefix)           |        8-1023 |
| Tag 1811940352-1879048191(prefix) | 1024-67108863 |
{: #tab-suffix cols='l r' title="Referencing Suffix Values"}

Affix data items are referenced by using the data items in
{{tab-prefix}} and {{tab-suffix}}.
Each of these implies the table used (prefix or
suffix), a table index (an unsigned integer) and contains a "rump item".
When reconstructing the original data item, such a
reference is replaced by a data item constructed from the referenced
affix data item (affix, which might need to be recursively unpacked
first) "concatenated" with the tag content (rump, again possibly
recursively unpacked).

* For a rump of type array and map, the affix also needs to be an
  array or a map.
  For an array, the elements from the prefix are prepended, and the
  elements from a suffix are appended to the rump array.
  For a map, the entries in the affix are added to those of the rump;
  prefix and suffix references differ in how entries with identical
  keys are combined: for prefix references, an entry in the rump with
  the same key as an entry in the affix overrides the one in the
  affix, while for suffix references, an entry in the affix overrides
  an entry in the rump that has the same key.

<aside markdown="1">
  ISSUE: Not sure that we want to use the efficiencies of overriding,
  but having default values supplied out of a dictionary to be
  overridden by a rump sounds rather handy.
  Note that there is no way to remove a map entry from the table.
</aside>

* For a rump of one of the string types, the affix also needs to be one
  of the string types; the bytes of the strings are concatenated as
  specified (prefix + rump, rump + suffix).
  The result of the concatenation gets the type of the rump; this way
  a single affix can be used to build both byte and text strings,
  depending on what type of rump is being used.

As a contrived (but short) example, if the prefix table is `["foobar",
"foob", "fo"]`, the following prefix references will all unpack to
`"foobart"`: `6("t")`, `224("art")`, `225("obart")` (the last example
isn't really an optimization).

<!-- 2<sup>28</sup>2<sup>12</sup>+2<sup>5</sup>+2<sup>0</sup> -->

Taking into account the encoding, there is one single-byte prefix
reference, 31 (2<sup>5</sup>-2<sup>0</sup>) two-byte references, 4064
(2<sup>12</sup>-2<sup>5</sup>) three-byte references, and 26843160
(2<sup>28</sup>-2<sup>12</sup>) five-byte references for prefixes.
268435455 (2<sup>28</sup>) is an artificial limit, but should be high
enough that there, again, is no practical limit to how many prefix
items might be used in a Packed CBOR item.
The numbers for suffix references are one quarter of those, except
that there is no single-byte reference and 8 two-byte references.

<aside markdown="1">
Rationale: Experience suggests that prefix packing might be more
likely than suffix packing.  Also for this reason, there is no intent
to spend a 1+0 tag value for suffix matching.
</aside>

## Discussion

This specification uses up a large number of Simple Values and Tags,
in particular one of the rare one-byte tags and half of the one-byte
simple values.  Since the objective is compression, this is warranted
if and only if there is consensus that this specific format could be
useful for a wide area of applications, while maintaining reasonable
simplicity in particular at the side of the consumer.

A maliciously crafted Packed CBOR data item might contain a reference
loop.  A consumer/decompressor MUST protect against that.

The current definition does nothing to help with packing CBOR
sequences {{-seq}}; maybe it should.

# Table Setup

The packing references described in {{sec-packed-cbor}} assume that
packing tables have been set up.

By default, all three tables are empty (zero-length arrays).

Table setup can happen in one of two ways:

* By the application environment, e.g., a media type.  These can
  define tables that amount to a static dictionary that can be used in
  a CBOR data item for this application environment.
  Note that, without this information, a data item that uses such a
  static dictionary can be decoded at the CBOR level, but not fully
  unpacked.
  The table setup mechanisms provided by this document are defined in
  such a way that an unpacker can at least recognize if this is the
  case.

* By one or more tags enclosing the packed content.
  These can be defined to add to the packing tables that already apply
  to the tag.  Usually, the semantics of the tag will be to prepend
  items to one of the tables.
  Note that it may be useful to leave a particular efficiency tier
  alone and only prepend to a higher tier; e.g., a tag could insert
  shared items at table index 16 and shift anything that was already
  there further down in the array while leaving index 0 to 15 alone.
  Explicit additions by tag can combine with application-environment
  supplied tables that apply to the entire CBOR data item.

The present specification only defines a single tag for prepending
to the (by default empty) tables.

<aside markdown="1">
We could also define a tag for dictionary referencing (or include that
in the basic packed CBOR), but the details are likely to vary
considerably between applications.  A URI-based reference would be
easy to add, but might be too inefficient when used in the likely
combination with an `ni:` URI {{-ni}}.
</aside>

## Basic Packed CBOR

A predefined tag for packing table setup is defined in CDDL {{-cddl}} as in {{fig-cddl}}:

~~~ cddl
Basic-Packed-CBOR = #6.51([[*shared], [*prefix], [*suffix], rump])
rump = any
prefix = any
suffix = any
shared = any
~~~
{: #fig-cddl title="Packed CBOR in CDDL"}

(This assumes the allocation of tag number 51.)

The arrays given as the first, second, and third element of the
content of the tag 51 are prepended to the tables for shared items,
prefixes, and suffixes that apply to the entire tag (by default empty
tables).

The original CBOR data item can be reconstructed by recursively
replacing shared, prefix, and suffix references encountered in the
rump by their expansions.


IANA Considerations
============

In the registry {{-tags}},
IANA is requested to allocate the tags defined in {{tab-tag-values}}.

|                   Tag | Data Item                                     | Semantics                  | Reference              |
|                     6 | integer, text string, byte string, array, map | Packed CBOR: shared/prefix | draft-ietf-cbor-packed |
|               225-255 | text string, byte string, array, map          | Packed CBOR: prefix        | draft-ietf-cbor-packed |
|           28704-32767 | text string, byte string, array, map          | Packed CBOR: prefix        | draft-ietf-cbor-packed |
| 1879052288-2147483647 | text string, byte string, array, map          | Packed CBOR: prefix        | draft-ietf-cbor-packed |
|               216-223 | text string, byte string, array, map          | Packed CBOR: suffix        | draft-ietf-cbor-packed |
|           27647-28671 | text string, byte string, array, map          | Packed CBOR: suffix        | draft-ietf-cbor-packed |
| 1811940352-1879048191 | text string, byte string, array, map          | Packed CBOR: suffix        | draft-ietf-cbor-packed |
{: #tab-tag-values cols='r l l' title="Values for Tag Numbers"}

In the registry {{-simple}},
IANA is requested to allocate the simple values defined in {{tab-simple-values}}.

| Value | Semantics           | Reference                 |
|  0-15 | Packed CBOR: shared | draft-ietf-cbor-packed |
{: #tab-simple-values cols='r l l' title="Simple Values"}

Security Considerations
============

The security considerations of {{-bis}} apply.

Loops in the Packed CBOR can be used as a denial of service attack,
see {{sec-discussion}}.

As the unpacking is deterministic, packed forms can be used as signing
inputs.  (Note that if external dictionaries are added to cbor-packed,
this requires additional consideration.)


--- back

Examples
========

The (JSON-compatible) CBOR data structure depicted in
{{fig-example-in}}, 400 bytes of binary CBOR, could lead to a packed
CBOR data item depicted in {{fig-example-out}}, ~309 bytes.  Note that
this particular example does not lend itself to prefix compression.

~~~json
{ "store": {
    "book": [
      { "category": "reference",
        "author": "Nigel Rees",
        "title": "Sayings of the Century",
        "price": 8.95
      },
      { "category": "fiction",
        "author": "Evelyn Waugh",
        "title": "Sword of Honour",
        "price": 12.99
      },
      { "category": "fiction",
        "author": "Herman Melville",
        "title": "Moby Dick",
        "isbn": "0-553-21311-3",
        "price": 8.99
      },
      { "category": "fiction",
        "author": "J. R. R. Tolkien",
        "title": "The Lord of the Rings",
        "isbn": "0-395-19395-8",
        "price": 22.99
      }
    ],
    "bicycle": {
      "color": "red",
      "price": 19.95
    }
  }
}
~~~
{: #fig-example-in title="Example original CBOR data item"}

~~~CBORdiag
51(["price", "category", "author", "title", "fiction", 8.95, "isbn"],
   /  0          1         2         3         4       5      6   /
   [], [],
   [{"store": {
      "book": [
        {simple(1): "reference", simple(2): "Nigel Rees",
         simple(3): "Sayings of the Century", simple(0): simple(5)},
        {simple(1): simple(4), simple(2): "Evelyn Waugh",
         simple(3): "Sword of Honour", simple(0): 12.99},
        {simple(1): simple(4), simple(2): "Herman Melville",
         simple(3): "Moby Dick", simple(6): "0-553-21311-3",
         simple(0): simple(5)},
        {simple(1): simple(4), simple(2): "J. R. R. Tolkien",
         simple(3): "The Lord of the Rings",
         simple(6): "0-395-19395-8", simple(0): 22.99}],
      "bicycle": {"color": "red", simple(0): 19.95}}}])
~~~
{: #fig-example-out title="Example packed CBOR data item"}


The (JSON-compatible) CBOR data structure below has been packed with shared
item and (partial) prefix compression only.

~~~
{
  "name": "MyLED",
  "interactions": [
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/rgbValueRed",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "number"
        }
      },
      "name": "rgbValueRed",
      "writable": true,
      "@type": [
        "Property"
      ]
    },
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/rgbValueGreen",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "number"
        }
      },
      "name": "rgbValueGreen",
      "writable": true,
      "@type": [
        "Property"
      ]
    },
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/rgbValueBlue",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "number"
        }
      },
      "name": "rgbValueBlue",
      "writable": true,
      "@type": [
        "Property"
      ]
    },
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/rgbValueWhite",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "number"
        }
      },
      "name": "rgbValueWhite",
      "writable": true,
      "@type": [
        "Property"
      ]
    },
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/ledOnOff",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "boolean"
        }
      },
      "name": "ledOnOff",
      "writable": true,
      "@type": [
        "Property"
      ]
    },
    {
      "links": [
        {
          "href": "http://192.168.1.103:8445/wot/thing/MyLED/colorTemperatureChanged",
          "mediaType": "application/json"
        }
      ],
      "outputData": {
        "valueType": {
          "type": "number"
        }
      },
      "name": "colorTemperatureChanged",
      "@type": [
        "Event"
      ]
    }
  ],
  "@type": "Lamp",
  "id": "0",
  "base": "http://192.168.1.103:8445/wot/thing",
  "@context": "http://192.168.1.102:8444/wot/w3c-wot-td-context.jsonld"
}
~~~
{: #fig-example-in2 title="Example original CBOR data item"}

~~~CBORdiag
51([/shared/["name", "@type", "links", "href", "mediaType",
            /  0       1       2        3         4 /
    "application/json", "outputData", {"valueType": {"type":
         /  5               6               7 /
    "number"}}, ["Property"], "writable", "valueType", "type"],
               /   8            9           10           11 /
   /prefix/ ["http://192.168.1.10", 6("3:8445/wot/thing"),
              / 6                        225 /
   225("/MyLED/"), 226("rgbValue"), "rgbValue",
     / 226             227           228     /
   {simple(6): simple(7), simple(9): true, simple(1): simple(8)}],
     / 229 /
   /suffix/ [],
   /rump/ {simple(0): "MyLED",
           "interactions": [
     229({simple(2): [{simple(3): 227("Red"), simple(4): simple(5)}],
      simple(0): 228("Red")}),
     229({simple(2): [{simple(3): 227("Green"), simple(4): simple(5)}],
      simple(0): 228("Green")}),
     229({simple(2): [{simple(3): 227("Blue"), simple(4): simple(5)}],
      simple(0): 228("Blue")}),
     229({simple(2): [{simple(3): 227("White"), simple(4): simple(5)}],
      simple(0): "rgbValueWhite"}),
     {simple(2): [{simple(3): 226("ledOnOff"), simple(4): simple(5)}],
      simple(6): {simple(10): {simple(11): "boolean"}}, simple(0):
      "ledOnOff", simple(9): true, simple(1): simple(8)},
     {simple(2): [{simple(3): 226("colorTemperatureChanged"),
      simple(4): simple(5)}], simple(6): simple(7), simple(0):
      "colorTemperatureChanged", simple(1): ["Event"]}],
     simple(1): "Lamp", "id": "0", "base": 225(""),
     "@context": 6("2:8444/wot/w3c-wot-td-context.jsonld")}])
~~~
{: #fig-example-out2 title="Example packed CBOR data item"}


Acknowledgements
================
{: numbered="no"}

CBOR packing was originally invented with the rest of CBOR, but did
not make it into {{-orig}}, the predecessor of {{-bis}}.
Various attempts to come up with a specification over the years didn't
proceed.
In 2017, <contact fullname="Sebastian Käbisch"/> proposed
investigating compact representations of W3C Thing Descriptions, which
prompted the author to come up with essentially the present design.

<!--  LocalWords:  CBOR extensibility IANA uint sint IEEE endian
 -->
<!--  LocalWords:  signedness endianness prepended decompressor
 -->
<!--  LocalWords:  prepend
 -->
