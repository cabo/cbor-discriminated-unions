---
title: >
  CBOR Tags for Discriminated Unions
abbrev: CBOR Tags for Discriminated Unions
docname: draft-bormann-cbor-discriminated-unions-latest

stand_alone: true

ipr: trust200902
keyword: Internet-Draft
cat: info

pi: [toc, sortrefs, symrefs, compact, comments]

author:
  - name: Duncan Coutts
    email: duncan@well-typed.com
  - name: Michael Peyton Jones
    email: me@michaelpj.com
  -
    name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org
    role: editor


normative:
  RFC8949: cbor
  IANA.cbor-tags: tags

informative:
  RFC8610: cddl

--- abstract

The Concise Binary Object Representation (CBOR, RFC 8949) is a data
format whose design goals include the possibility of extremely small
code size, fairly small message size, and extensibility without the
need for version negotiation.

In programming, there is often a need to define types that are
composed of enumerated alternatives.  While CBOR has no problem
representing types that are unions of other types, this may not make
visible which of the enumerated alternatives was chosen, in particular if
multiple alternatives overlap in their CBOR representations.
In other words, CBOR has no direct support for discriminated unions.

In CBOR, one point of extensibility is the definition of CBOR tags.
In protocol development, it is possible to register different tags
for each alternative, but this would be difficult to automate in the
course of using CBOR for normal program development, it might use up
registration resources and tag numbers, and would create an unwelcome
threshold for using CBOR in these activities.

Instead, the alternatives could simply be numbered by a program compiler.
The present document defines tags for such a use in discriminated unions,
i.e., where there is no intention to register application-specific
tags for each of the branches in the union.

--- note_Note_to_Readers

This is an individual submission to the CBOR working group of the
IETF, <https://datatracker.ietf.org/wg/cbor/about/>.
Discussion currently takes places on the github repository
<https://github.com/cabo/cbor-discriminated-unions>.
If the CBOR WG believes this is a useful document, discussion is
likely to move to the CBOR WG mailing list and a github repository at
the CBOR WG github organization, <https://github.com/cbor-wg>.


--- middle

Introduction        {#intro}
============

The Concise Binary Object Representation (CBOR {{-cbor}}) is a data
format whose design goals include the possibility of extremely small
code size, fairly small message size, and extensibility without the
need for version negotiation.

In programming, there is often a need to define types that are
composed of enumerated alternatives.  While CBOR has no problem
representing types that are unions of other types, this may not make
visible which of the enumerated alternatives was chosen, in particular if
multiple alternatives overlap in their CBOR representations.
In other words, CBOR has no direct support for discriminated unions.

In CBOR, one point of extensibility is the definition of CBOR tags.
In protocol development, it is possible to register different tags
for each alternative, but this would be difficult to automate in the
course of using CBOR for normal program development, it might use up
registration resources and tag numbers, and would create an unwelcome
threshold for using CBOR in these activities.

Instead, the alternatives could simply be numbered by a program compiler.
The present document defines tags for such a use in discriminated unions,
i.e., where there is no intention to register application-specific
tags for each of the branches in the union.

Each of the tags specified in this document associates an unsigned
number with the enclosed data item, with the intention that this
number identifies one of a set of enumerated alternatives that could
have been chosen at this position in the CBOR data item.

For example data representing the result of some action might be either
a failure with some failure detail, or a success with some result. In
this example there are two cases, the failure case and the success case,
and they can be enumerated as 0 and 1.

In general the number of alternatives, and what data is expected in each
alternative case, are entirely application dependent.

The tags defined in this specification allow the encoding of any number
of alternatives, but provide compact encoding for the common cases of
low numbers of alternatives:


Alternatives 0..6 can be encoded with two additional bytes:

* Tag number 185..191 (1+1 encoding)
* tag content: the alternative

Alternatives 7..127 can be encoded with three additional bytes:

* Tag number 1927..2047 (1+2 encoding)
* tag content: the alternative

Alternatives 128+ can be encoded with 5-12 additional bytes:

* Tag number 184 (1+1 encoding)
* tag content: array (1+0 encoding), with the two elements:

    * alternative number (`uint`, 1+1 to 1+8 encoding) and
    * data item: the alternative

(The third representation can also be used for alternatives 0 to 127;
see below.)

Semantics
=========

Abstractly speaking, the value to be represented by these tags
consists of a case number and a case body.  The case number is
an unsigned integer that indicates which case out of the set of
alternatives is used. The case body is any CBOR data value.

In a setting where the application uses a schema (formally or
informally), then there will be an appropriate sub-schema for each case
in the set of alternatives. The representation of the case body should
comply with the schema corresponding to the case number used.

To continue the example above about representing failure or success,
suppose that the failure detail consists of an integer code and a
string, and suppose that the successful result is a byte string. A
failure value will use case 0 and the case body will be a list (CBOR array)
containing an integer and a text string. Alternatively, a success value
will use case 1 and the body will be a single CBOR byte string.

Decoders that enforce a schema must check the case number is within the
range of cases allowed, and that the case body follows the schema for
the supplied case number. Generic decoders should allow any case number
and any CBOR data value for the case body.

Canonical CBOR
==============

The tag 184 encoding overlaps with the more compact special encodings
for 0..6 and 7..127.

For applications that need canonical CBOR the following convention must
be used: alternatives in the range 0..127 must be encoded using the
special compact encoding, and must not use the tag 184 encoding. This
follows the general canonical CBOR convention of using the most compact
form.

<aside markdown="1">
Issue:
: Alternatively, tag 184 could encode the alternative number
  minus 128.  This would get some mileage out of the efficient
  encoding of unsigned integers 0..23 for the alternative numbers 128
  to 151; these cases would be represented in 4 additional bytes.
  More importantly, it would also get rid of the deterministic
  encoding considerations.  (I'm not suggesting encoding the
  alternative number minus 152, which would use the efficiency of both
  the negative and unsigned one-byte integers, or, worse, some zigzag
  encoding...)
</aside>

Rationale
=========

CBOR has direct support for *combinations* of multiple values but not
for *alternatives*, i.e. one of identified multiple values. Combinations are
expressed in CBOR using lists (arrays) or maps.

Most programming languages have a notion of data consisting of
combinations of data values, often called records or objects.  Many
programming languages also have a notion of data consisting of
multiple alternative data values.  For example C has unions (the
alternative in use is not identified, though), and other languages
have "tagged" or "discriminated" unions (where it is always clear
which alternative is in use).

Crucially for this specification, the set of alternatives must be closed
and ordered. This allows encoding using a natural number to distinguish
each case.

Note that this does *not* correspond to the notion in some programming
languages of classes and sub-classes since in that context the set of
alternatives is open and unordered. Alternatives of this kind can be
supported by other means, such as tag 27 "Serialised
language-independent object with type name and constructor arguments" {{-tags}}.

In functional programming languages, the primary way of forming new data
types is to enumerate a set of alternatives (each of which may be a
record). Such forms of data are also supported in hybrid functional
languages or languages with functional features.

Thus in some applications, it is very common to have data making use of
alternatives, and it is worth finding a compact encoding, at least for
the common cases. Just as most records are small, most alternatives are
also small.

In this specification we allocate 7 values in the 1+1-byte part of the
available tag encoding space for alternatives 0..6 which are by far
the most common. We allocate a range of 121 values in the 1+2-bytes
tag encoding space. To cover the general case we allocate another
1+1-byte value for a representation using a pair consisting of an
unsigned integer and the case body.

Examples
========

To elaborate on the example from the introduction, we have a "result"
that is a failure or success, where:

-   the failure detail consists of an integer code and a string;
-   the successful result is a byte string;

This corresponds to the following schema, in CDDL {{-cddl}} notation:

~~~ cddl
result = 121(\[int, text\])
         122(bytes)
~~~

Example values:

-   `121([3, "the printer is on fire"])`
-   `122(h'ff00')`

As a second example, here is one based on a data type defined within the
Haskell programming language, representing a simple expression tree.

~~~
-- A data type representing simple arithmetic expressions:
data Expr = Lit Int -- integer literal
          | Add Expr Expr -- addition
          | Sub Expr Expr -- subtraction
          | Neg Expr -- unary negation
          | Mul Expr Expr -- multiplication
          | Div Expr Expr -- integer division
~~~

In CDDL notation, and using the tags in this specification, such data
could be encoded using this schema:

~~~ cddl
; A data type representing simple arithmetic expressions

expr = 121(int) ; integer literal
     / 122(\[expr, expr\]) ; addition
     / 123(\[expr, expr\]) ; subtraction
     / 124(expr) ; unary negation
     / 125(\[expr, expr\]) ; multiplication
     / 126(\[expr, expr\]) ; integer division
~~~

# IANA Considerations

IANA is requested to allocate the following tag numbers in the CBOR
tags registry {{-tags}}:

|        Tag | Data Item                                              | Semantics                                   | Reference |
|        184 | array of one unsigned integer and one item of any type | numbered alternative (numbered by the uint) | [RFCthis] |
|   185..191 | any                                                    | numbered alternative 0..6                   | [RFCthis] |
| 1927..2047 | any                                                    | numbered alternative 7..127                 | [RFCthis] |
{: #tab-tag-values cols='r l l' title="Tags to be registered"}

# Security Considerations

TBD
