# Writing a shell in F# - Parsing and syntax highlighting

In this part I will discuss writing a parser tokenizing user input and adding real time syntax highlighting.

## Parsing overview

There are many ways to write a parser, 
however with a relatively low number of token types and short input I decided to create a one-pass parser.

Initially there will be 7 token types but this might be expanded later:

* `Command` - A command like `ls` or `cat`. This comes at the start of a section.
* `ArgName` - An argument name like `-o`. This start with `-`.
* `ArgValue` - An argument value.
* `Operator` - An operator such as `|` or `>`. These might be split into pipes and redirects later
* `DelimitedString` - A delimited string such as `"Hello, World!"` or `'Hello, World!'`, both single and double quotes are supported.
* `Text` - A general catch-all token, this is currently not used.
* `Whitespace` - A whitespace character.

There are 5 supported operators for now:

* `|` - a pipe operator.
* `|>` - a F# pipe operator.
* `>` - a redirect operator.
* `>&1` - redirect to `StdOut`.
* `>&2` - redirect to `StdErr`

Delimited strings can contain any character except for the delimit. For example the can contain `|` or whitespace.
In the future I might look to add a next character delimited (`\`).

The parse iterates over a string fun index `i` until it reaches the next split character. 
Then it creates a token with help of a look ahead (if require) and any proceeding substring if there is one.
The split characters are:

* Whitespace
* `|`
* `>`
* `"`
* `'`

## Parser

I started by adding 4 extension methods to `String` to act as helpers: 

* `InBounds` - Checks if an index is in bounds.
* `TryGetChar` - Tries to get an char at an index. Returns `None` if index is out of bounds. 
* `ReadUntilChar` - Reads until one of a selection of characters. Supports delimiting.
* `GetSubString` - Gets a substring between a start and end index.

```fsharp
type String with

    /// Check if an index in a string is in bounds.
    member s.InBounds(i: int) = i >= 0 && i < s.Length
    
    /// Try and get a character from a string by index (if in bounds)
    member s.TryGetChar(i) =
        match s.InBounds i with
        | true -> s.[i] |> Some
        | false -> None
        
    /// Read from the string until one of a selection of characters are found.
    /// Supports an optional delimiter character.
    /// If present, will only return the next applicable character index that is not delimited.
    /// For example in single or double quotes.
    member s.ReadUntilChar(start: int, chars: char list, delimiter: char option) =
        let rec read(i: int, delimited: bool) =
            match s.TryGetChar i, delimiter with
            | Some c, Some delimiter ->
                match c = delimiter, delimited, chars |> List.contains c with
                | true, true, _ -> read (i + 1, false)
                | true, false, _ -> read (i + 1, true)
                | false, _, true -> (i, Some c)
                | false, _, false -> read (i + 1, delimited)
            | Some c, None ->
                match chars |> List.contains c with
                | true -> (i, Some c)
                | false -> read (i + 1, false)
            | None, _ ->
                // Read to end and out of bounds, so return the last value.
                (i, None)
        
        read(start, false)
        
    /// Get a substring starting at startIndex and ending at endIndex.
    member s.GetSubString(startIndex: int, endIndex: int) =
        s.[startIndex..endIndex]
```

With this in place I added a `Parsing` module.
It is import this comes above `InputController` in the project, as it will be used in there.

The `Parser` module defines token types and handles creating them from a string:

```fsharp
namespace FShell.Core

module Parsing =

    /// Supported operators.
    let operators =
        [ "|"; "|>"; ">"; ">&1"; ">&2" ]

    /// Token types.
    type TokenType =
        | Command
        | ArgName
        | ArgValue
        | Operator
        | DelimitedString
        | Text
        | Whitespace
        
        /// Checks if a token type is trivial and can be ignored.
        member tt.IsTrivial() = match tt with Whitespace -> true | _ -> false

    /// A record representing a tokenized input value.
    type Token =
        { Type: TokenType
          Position: int
          Value: string }

        /// A catch-all method to create a token. This will handle getting the token type.
        static member Create(value: string, position: int, isCommand: bool) =
            let tt =
                match operators |> List.contains value, isCommand with
                | true, _ -> TokenType.Operator
                | false, true -> TokenType.Command
                | false, false when value.StartsWith('-') -> TokenType.ArgName
                | false, false when value.StartsWith('"') || value.StartsWith(''') -> TokenType.DelimitedString
                | false, false when value = " " -> TokenType.Whitespace
                | false, false -> TokenType.ArgValue

            { Type = tt
              Position = position
              Value = value }

        /// Create a `Command` token.
        static member CreateCommand(value: string, position: int) =
            { Type = TokenType.Command
              Position = position
              Value = value }

        /// Create an `ArgName` token.
        static member CreateArgName(value: string, position: int) =
            { Type = TokenType.ArgName
              Position = position
              Value = value }

        /// Create an `ArgValue` token.
        static member CreateArgValue(value: string, position: int) =
            { Type = TokenType.ArgValue
              Position = position
              Value = value }

        /// Create an `Operator` token.
        static member CreateOperator(value: string, position: int) =
            { Type = TokenType.Operator
              Position = position
              Value = value }

        /// Create a `DelimitedString` token.
        static member CreateDelimitedString(value: string, position: int) =
            { Type = TokenType.DelimitedString
              Position = position
              Value = value }

        /// Create a `Text` token. Not currently used.
        static member CreateText(value: string, position: int) =
            { Type = TokenType.Text
              Position = position
              Value = value }

        /// Create a `Whitespace` token.
        static member CreateWhitespace(value: string, position: int) =
            { Type = TokenType.Whitespace
              Position = position
              Value = value }

    /// Run the parser and return a collection of tokens.
    let run (value: string) =

        // character to split the input on.
        let chars = [ ' '; '|'; '>'; '"'; ''' ]

        /// Create a new accumulator by checking if there is a previous substring before the current position.
        /// If so add that and the current token. If not just add the current token.
        let createNewAcc (startIndex: int) (endIndex: int) (token: Token) (isCommand: bool) (acc: Token list) =
            acc
            @ [ if endIndex > startIndex then
                    match value.GetSubString(startIndex, endIndex - 1) with
                    | s when isCommand -> Token.CreateCommand(s, startIndex)
                    | s when s.StartsWith('-') -> Token.CreateArgName(s, startIndex)
                    | s -> Token.CreateArgValue(s, startIndex)
                //Token.Create(value.GetSubString(startIndex, endIndex - 1), startIndex, isCommand)
                token ]

        /// A recursive function to handle parsing.
        let rec handle (acc: Token list, i: int, delimiter: char option, isCommand: bool) =
            let (endIndex, c) =
                value.ReadUntilChar(i, chars, delimiter)

            match c with
            // Handle whitespace.
            | Some c when c = ' ' ->

                // A bit convoluted. Essentially, is there is a prior substring (i.e. command, arg etc) and
                // isCommand is not true, pass false. If there is no prior substring then don't change is command.
                // This is ensure white spaces dont break the output.
                handle (
                    acc
                    |> createNewAcc i endIndex (Token.CreateWhitespace(" ", endIndex)) isCommand,
                    endIndex + 1,
                    None,
                    (not (endIndex > i) || not isCommand)
                    && isCommand <> false
                )
            // Handle pipe operators (`|` or `|>`)
            | Some c when c = '|' ->

                let token, newEnd =
                    match value.TryGetChar(endIndex + 1) with
                    | Some c when c = '>' -> Token.CreateOperator("|>", endIndex), endIndex + 2
                    | Some _
                    | None -> Token.CreateOperator("|", endIndex), endIndex + 1

                handle (acc |> createNewAcc i endIndex token isCommand, newEnd, None, true)
            // Handle redirect operators (`>`, `>&1` or `>&2`).
            | Some c when c = '>' ->
                let token, newEnd =
                    match value.TryGetChar(endIndex + 1), value.TryGetChar(endIndex + 2) with
                    | Some c1, Some c2 when c1 = '&' && c2 = '1' -> Token.CreateOperator(">&1", endIndex), endIndex + 3
                    | Some c1, Some c2 when c1 = '&' && c2 = '2' -> Token.CreateOperator(">&2", endIndex), endIndex + 3
                    | Some _, _
                    | None, _ -> Token.CreateOperator(">", endIndex), endIndex + 1

                handle (acc |> createNewAcc i endIndex token isCommand, newEnd, None, true)
            // Handle delimited strings (`"Some value"` or `'Some value'`).
            | Some c when c = '"' || c = ''' ->
                let (newEnd, newC) =
                    value.ReadUntilChar(endIndex + 1, [ c ], None)

                let token =
                    match newC with
                    | Some _ -> Token.CreateDelimitedString(value.GetSubString(endIndex, newEnd), i)
                    | None -> Token.CreateDelimitedString(value.GetSubString(endIndex, value.Length - 1), endIndex)

                handle (acc |> createNewAcc i endIndex token isCommand, newEnd + 1, None, false)
            // This should not be hit. Throw and exception for now just to report on anytime it might be.
            | Some c ->
                failwith $"Error - unknown character `{c}`"
            // Handle end of string
            | None ->
                acc
                @ [ Token.Create(value.GetSubString(i, value.Length - 1), i, isCommand) ]

        // Parse the input into tokens.
        handle ([], 0, None, true)
```

There might be some edge cases I missed but so far in testing it seems to work pretty well.

## Syntax highlighting

With the parser module in place I added another module `DisplayHandler` to handle syntax highlighting.
It is important this comes after `Parsing` and before `InputController`. 

The module was pretty simple in the end:

```fsharp
namespace FShell.Core

[<RequireQualifiedAccess>]
module DisplayHandler =

    open System
    open Parsing

    /// Print a collection of tokens to the console with syntax highlighting.
    let print (values: Token list) =
      values
      |> List.iter (fun t ->
          match t.Type with
          | TokenType.Command -> Console.ForegroundColor <- ConsoleColor.Blue
          | TokenType.ArgName -> Console.ForegroundColor <- ConsoleColor.Yellow
          | TokenType.ArgValue -> Console.ForegroundColor <- ConsoleColor.White
          | TokenType.Operator -> Console.ForegroundColor <- ConsoleColor.Magenta
          | TokenType.DelimitedString -> Console.ForegroundColor <- ConsoleColor.Green
          | TokenType.Text -> Console.ForegroundColor <- ConsoleColor.DarkGray
          | TokenType.Whitespace -> ()
          
          printf $"{t.Value}")
      
      Console.ResetColor()
```

## Plugging it all up

With the parser and syntax highlighting add the last task was to set the input controller to use it.
The was done simply by changing `Actions.updateLine` (in `InputController`) to use it:

```fsharp
/// Update a line based on the state.
let updateLine (prompt: string) (state: State) =
    Console.CursorVisible <- false
    Console.CursorLeft <- prompt.Length
    clearCurrentLine prompt ()
    setStartOfLine prompt state

    // Old code
    //Console.Write(state.GetLeftBufferString())
    //Console.Write(state.GetRightBufferString())

    // New code
    state.GetString()
    |> Parsing.run
    |> DisplayHandler.print
    
    Console.CursorVisible <- true
```

An example output:

![Example output](https://blog.psionic.cloud/img/writing_a_shell_in_fsharp-img.png "Syntax"){ width:100% }

## Summary

With that in place, basic parsing and syntax highlight are taken care of and things really are coming together.
The can be expanded upon later and made configurable but for now it works and I am quite happy with the results.