<meta name="daria:article_id" content="writing_a_csv_parser_in_fsharp_part_2">
<meta name="daria:title" content="Part 2">
<meta name="daria:title_slug" content="part_2">
<meta name="daria:order" content="1">
<meta name="daria:created_on" content="2022-06-19">
<meta name="daria:tags" content="fsharp,csv">
<meta name="daria:image_id" content="twitter">

# Writing a CSV parser in F# - Basic parsing

In this part covers creating a basic CSV parser to split CSV lines into a collect of strings.

## Getting started

Start off by creating a new F# project. For now, it can be a console project to keep things simple but this can be refactored into a class library later.

Create a module called `Parsing`, in this module add a basic function to read all lines in from a file. For now use `try / with` returning a `Result<string array, string>` as shown below:

```fsharp
namespace CsvParser

module Parsing =
    
    open System.IO
    
    /// A basic function to read all lines in a file,
    /// with catch all error handling.
    let loadFile (path: string) =
        try
            File.ReadLines path |> Ok
        with
        | exn -> Error exn.Message
```

*For now, this error handling will do. However it can be improved later.*

## Parsing a line

Next add 2 helper functions `inBounds` and `getChar`. 
These will be used to check if a index in inbounds and get a character from a string by index respectively.

```fsharp
/// Check if a index is inbounds.
let inBounds (input: string) i = i >= 0 && i < input.Length

/// Get a character by index. If out of bounds None will be return.
let getChar (input: string) i =
    match inBounds input i with
    | true -> Some input.[i]
    | false -> None
```

After that, add another function `readUntilChar`. This will be used to read until a certain character and return a `(string * int) option`.
The string is the accumulated characters that were read, the int is the end index. If the character was not found None will be returns.

This function will accept an input string, the character to look for, a start index and a `StringBuilder` (more on this later).

```fsharp
open System    
open System.Text

// ...

let readUntilChar (input: string) (c: Char) (start: int) (sb: StringBuilder) =
    let rec read i (sb: StringBuilder) =
        match getChar input i with
        // If the current char is ", check if the next one is. If some it is a delimited ".
        // In that case just add a " to the string builder.
        | Some r when r = '"' && getChar input (i + 1) = Some '"' -> read (i + 2) <| sb.Append('"')
        // If the current char is c, return the accumulated string and the current index.
        | Some r when r = c -> Some <| (sb.ToString(), i)
        // If the current index does return a character add it to the string builder
        // and index the index by one.
        | Some r -> read (i + 1) <| sb.Append(r)
        // If the current index does not return a character, return the accumulated string
        // and the next index. This will be out of bounds but will be handled later.
        | None -> Some <| (sb.ToString(), i + 1)
    
    read start sb
```

*This uses an inner recursive function to avoid needing a loop.*

With these functions it is possible to write a function to parse a CSV line. This will accept a `string` and return a `string list`:

```fsharp
let parseLine (input: string) =
    // Use a StringBuilder as a bit of an optimization. 
    let sb = StringBuilder()

    let rec readBlock (i, sb: StringBuilder, acc: string list) =

        match getChar input i with
        // If the block starts with ", it is an double quote enclosed block.
        // Read until an non delimited ".
        | Some c when c = '"' ->
            match readUntilChar input '"' (i + 1) sb with
            | Some (r, i) ->
                // i + 2 to skip end " and ,
                readBlock (i + 2, sb.Clear(), acc @ [ r ])
            | None -> acc
        // If not, simply read until ,
        | Some _ ->
            match readUntilChar input ',' i sb with
            // i + 1 to skip the ,
            | Some (r, i) -> readBlock (i + 1, sb.Clear(), acc @ [ r ])
            | None -> acc
        // If no character return, simply return the accumulated strings.
        // This is where the i + 1 in readUntilChar last branch is effectively handled. 
        | None -> acc

    readBlock (0, sb, [])
```

*Again, this uses an inner recursive function to avoid a loop.*

The full module:

```fsharp
namespace CsvParser

module Parsing =

    open System    
    open System.IO    
    open System.Text
        
    /// A basic function to read all lines in a file,
    /// with catch all error handling.
    let loadFile (path: string) =
        try
            File.ReadLines path |> Ok
        with
        | exn -> Error exn.Message
        
    let inBounds (input: string) i = i >= 0 && i < input.Length

    let getChar (input: string) i =
        match inBounds input i with
        | true -> Some input.[i]
        | false -> None

    let readUntilChar (input: string) (c: Char) (start: int) (sb: StringBuilder) =
        let rec read i (sb: StringBuilder) =
            match getChar input i with
            // If the current char is ", check if the next one is. If some it is a delimited ".
            // In that case just add a " to the string builder.
            | Some r when r = '"' && getChar input (i + 1) = Some '"' -> read (i + 2) <| sb.Append('"')
            // If the current char is c, return the accumulated string and the current index.
            | Some r when r = c -> Some <| (sb.ToString(), i)
            // If the current index does return a character add it to the string builder
            // and index the index by one.
            | Some r -> read (i + 1) <| sb.Append(r)
            // If the current index does not return a character, return the accumulated string
            // and the next index. This will be out of bounds but will be handled later.
            | None -> Some <| (sb.ToString(), i + 1)
        
        read start sb
        
    let parseLine (input: string) =
        // Use a StringBuilder as a bit of an optimization. 
        let sb = StringBuilder()

        let rec readBlock (i, sb: StringBuilder, acc: string list) =

            match getChar input i with
            // If the block starts with ", it is an double quote enclosed block.
            // Read until an non delimited ".
            | Some c when c = '"' ->
                match readUntilChar input '"' (i + 1) sb with
                | Some (r, i) ->
                    // i + 2 to skip end " and ,
                    readBlock (i + 2, sb.Clear(), acc @ [ r ])
                | None -> acc
            // If not, simply read until ,
            | Some _ ->
                match readUntilChar input ',' i sb with
                // i + 1 to skip the ,
                | Some (r, i) -> readBlock (i + 1, sb.Clear(), acc @ [ r ])
                | None -> acc
            // If no character return, simply return the accumulated strings.
            // This is where the i + 1 in readUntilChar last branch is effectively handled. 
            | None -> acc

        readBlock (0, sb, [])
```

This can be tested with the following example:

```fsharp
let line = """Foo,1,Bar,"Hello, World! From ""CSV Parser"".",2"""

let result = Parsing.parseLine line
```

The results should be:

```fsharp
[ "Foo"
  "1"
  "Bar"
  "Hello, World! From "CSV Parser"."
  "2" ]
```

*This would be a good time to add unit tests to ensure everything works correctly.*

## Summary

With this, it is possible to parse a line of CSV. A file could be parsed with the following:

```fsharp
let parseFile (lines: string array) =
    lines |> Array.map Parsing.parseLine
```

However, for this project parsing files will be combined with record building to allow for better error handling/reporting.