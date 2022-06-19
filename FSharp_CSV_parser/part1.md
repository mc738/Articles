# Writing a CSV parser in F# - Introduction

How to create a (good enough) CSV parser in F#

## Introduction

One of the big advantages of CSV is it's simplicity. It a lot of cares you can simply join a collection of strings with a `,` to create valid CSV.

For example:

```fsharp
let data = [ "Foo"; string 1; "Bar" ]

let csv = data |> String.concat ","
```

However, there is not standardization for CSV like other formats. The closes is `RFC 4180` but files can vary.

The series will be based around `RFC 4180`, which includes:

* Lines are delimited by `CRLF`
* There is an optional header line at the start
* Each line should contain the same number of fields
* The last field in a line should not be followed by a `,`
* Each field may or may not be enclosed in `"`. If a field is not enclosed in double quotes, double quotes can not appear in the field.
* Fields containing `,` should be enclosed in `"`
* If `"` are used to enclose fields, then a `"` appearing in the field must be escaped by a preceding `"`

## "Good enough"?

The major part not implemented in the initial version is `CRLF` enclosed in in `"` fields not being handled. This is mainly because of the extra level of complexity for a (relatively) minor use case. According to `RFC 4180` these should delimited and not end the line.

Other than they the goal is to creae a library/app to parse CSV files into F# records via generics.

## The process

The process can be split into the following parts:

* Split the CSV file into lines
* Split each line into fields
* Coerce each field into to relevant .net base type
* Create records from the results

## Features

To handle different formats (for example for `DateTime` types), an attribute will be added so record fields can define their expected format.

## Notes

The records should consist of base types fields. This makes sense for CSV files in general anyway. 