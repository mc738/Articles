<meta name="daria:article_id" content="writing_a_shell_in_fsharp_part_6">
<meta name="daria:title" content="Part 6">
<meta name="daria:title_slug" content="part_6">
<meta name="daria:order" content="5">
<meta name="daria:created_on" content="2022-06-23">
<meta name="daria:tags" content="fsharp">
<meta name="daria:image_id" content="pipes">

# Writing a shell in F# - Interpreter

In this part I will go over adding an interpreter to the shell which will translate tokens into steps to be executed.

## Interpreter overview

The interpretation involves taking a list of tokens and converting them to steps to but run.
For now this will involve calling `CoreUtils`, however this will be expanded upon later.

The interpreter needs to split the tokens into blocks (for want of a better name),
containing an operator, a command and arguments.

With this in place the shell is actually usable (though with limited functionality).

## Design

The interpreter takes a list of tokens and transforms them into intermediary blocks.
These consist of a command, it's arguments and an operator. 

These blocks are then converted to steps to be run in a pipeline. 

## Implementation

I added a new file `Intepreter.fs` which will contain the interpreter implementation:

```fsharp
namespace FShell.Core

/// A module containing the interpreter implementation.
module Interpreter =

    open System
    open FShell.Core.Parsing
    open FShell.Core.Pipes
    
    /// A interpreter command.
    type Command =
        { Name: string
          Args: CommandArg list }

        /// Create a blank command.
        static member Blank() = { Name = ""; Args = [] }

        /// Set the command name.
        member cmd.SetName(name) = { cmd with Name = name }

        /// Add an arg to the command.
        member cmd.AddArg(arg) = { cmd with Args = cmd.Args @ [ arg ] }

        /// Try and get the first arg of the command. If not found will return None.
        member cmd.TryGetFirstArg() = cmd.Args |> List.tryHead

        /// Try and get the first arg value of the command. If not found will return None.
        member cmd.TryGetFirstArgValue() =
            cmd.TryGetFirstArg()
            |> Option.map (fun arg -> arg.Value)

        /// Try and get an arg by index. If not found will return None.
        member cmd.TryGetArg(index: int) = cmd.Args |> List.tryItem index

        /// Try and get an arg value by index. If not found will return None.
        member cmd.TryGetArgValue(index: int) =
            cmd.TryGetArg index
            |> Option.map (fun arg -> arg.Value)

        /// Try and get a arg by name. If not found will return None.
        member cmd.TryGetNamedArg(name: string) =
            cmd.Args
            |> List.tryFind (fun arg -> arg.Name = Some name)

        /// Try and get a arg value by name. If not found will return None.
        member cmd.TryGetNamedArgValue(name: string) =
            cmd.TryGetNamedArg name
            |> Option.map (fun arg -> arg.Value)

        /// Check if an arg exists.
        member cmd.HasArg(name: string) =
            match cmd.TryGetNamedArg name with
            | Some _ -> true
            | None -> false

    /// A argument for a command.
    and CommandArg =
        { Name: string option
          Value: string }

        /// Create a blank argument.
        static member Blank() = { Name = None; Value = "" }

        /// Set the argument name.
        member arg.SetName(name) = { arg with Name = Some name }

        /// Set the argument value.
        member arg.SetValue(value) = { arg with Value = value }

        /// Check if an argument is empty.
        member arg.IsEmpty() = arg.Name = None && arg.Value = ""

    /// Represents operator types
    type OperatorType =
        | Start
        | Pipe
        | FSharpPipe
        | FileRedirect
        | StdOutRedirect
        | StdErrRedirect

    /// A block consisting of a command and operator
    type Block =
        { Command: Command
          Operator: OperatorType }

    /// Represents the command builder state.
    type BuildCommandState =
        | SetName
        | BuildArg of CommandArg
    
    /// Build a command from a list of tokens.
    let buildCommand (tokens: Token list) =
        tokens
        |> List.fold
            (fun (cmd: Command, state) t ->
                match state, t.Type with
                | SetName, TokenType.Command
                | SetName, TokenType.DelimitedString -> (cmd.SetName(t.Value.RemoveQuotes()), BuildArg(CommandArg.Blank()))
                | SetName, _ -> failwith $"Incorrect token type: {t.Type}"
                | BuildArg arg, TokenType.ArgName -> (cmd, BuildArg(arg.SetName(t.Value)))
                | BuildArg arg, TokenType.ArgValue
                | BuildArg arg, TokenType.DelimitedString ->
                    (cmd.AddArg(arg.SetValue(t.Value.RemoveQuotes())), BuildArg(CommandArg.Blank()))
                | BuildArg _, _ -> failwith $"Incorrect token type for arg: {t.Type}")
            (Command.Blank(), SetName)
        |> fun (cmd, state) ->
            match state with
            | SetName -> cmd
            | BuildArg arg when arg.IsEmpty() -> cmd
            | BuildArg arg -> cmd.AddArg(arg)

    /// Create an OperatorType from a string.
    let handleOperator (value: string) =
        match value with
        | "|>" -> FSharpPipe
        | "|" -> Pipe
        | ">" -> FileRedirect
        | ">&1" -> StdOutRedirect
        | ">&2" -> StdOutRedirect
        | _ -> failwith $"Unknown operator `{value}`."

    /// Split a list of tokens into blocks.
    let split (tokens: Token list) =
        tokens
        |> List.fold
            (fun (op, acc, cur) t ->
                match t.Type with
                | Operator ->
                    handleOperator t.Value,
                    acc
                    @ [ { Command = buildCommand cur
                          Operator = op } ],
                    []
                | tt when tt.IsTrivial() -> op, acc, cur
                | _ -> op, acc, cur @ [ t ])
            (Start, [], [])
        |> fun (op, acc, cur) ->
            acc
            @ [ { Command = buildCommand cur
                  Operator = op } ]
    
    /// Turn an Option<'T> into a Result<'U, string> with a mapping function and error message. 
    let optionToResult<'T, 'U> (errorMessage: string) (fn: 'T -> 'U) (value: Option<'T>) =
        match value with
        | Some v -> fn v |> Ok
        | None -> Error errorMessage

    /// Attempt to create a step from a block.
    let createStep (block: Block) : Result<Step, string> =

        let binder = optionToResult "Missing arg."

        match block.Operator with
        | OperatorType.Start
        | OperatorType.Pipe ->
            match block.Command.Name with
            | "ls" ->
                block.Command.TryGetFirstArgValue()
                |> Option.orElseWith (fun _ -> Some Environment.CurrentDirectory)
                |> binder CoreUtils.ls
            | "echo" ->
                block.Command.TryGetFirstArgValue()
                |> binder CoreUtils.echo
            | "whoami" -> Ok CoreUtils.whoami
            | "grep" ->
                block.Command.TryGetNamedArgValue("-p")
                |> Option.orElseWith (fun _ -> block.Command.TryGetFirstArgValue())
                |> binder CoreUtils.grep
            | "base64" -> Ok CoreUtils.base64
            | "cat" ->
                block.Command.TryGetNamedArgValue("-p")
                |> Option.orElseWith (fun _ -> block.Command.TryGetFirstArgValue())
                |> binder CoreUtils.cat
            | _ -> Error $"Unknown command `{block.Command.Name}`."
        | OperatorType.FSharpPipe -> Error "TODO - implement fsharp pipes"
        | OperatorType.FileRedirect ->
            Ok(
                CoreUtils.toFile block.Command.Name
                <| block.Command.HasArg("-p")
            )
        | OperatorType.StdErrRedirect -> Ok CoreUtils.toError
        | OperatorType.StdOutRedirect -> Error "TODO - implement stdout redirect"

    /// Attempt to create a list of steps from a list of blocks.
    /// If one step fails to be created an error will be returned.
    let createSteps (blocks: Block list) =
        blocks
        |> List.fold
            (fun (r: Result<Step list, string>) b ->
                r
                |> Result.bind (fun acc ->
                    match createStep b with
                    | Ok s -> acc @ [ s ] |> Ok
                    | Error e -> Error e))
            (Ok [])        
            
    /// Run the interpreter and create a list of steps to be executed.
    let run (input: string) = Parsing.run input |> split |> createSteps
```
Lastly I updated `actionHandler` in the main program to use the interpreter:

```fsharp
let actionHandler (str: string) =
    match Interpreter.run str with
    | Ok steps -> Pipes.run steps
    | Error e ->
         Console.Write(Environment.NewLine)
         Console.ForegroundColor <- ConsoleColor.Red
         printfn $"Error: {e}"
         Console.ResetColor()
         1
```

Example of the shell in action:

![Example output](https://blog.psionic.cloud/img/writing_a_shell_in_fsharp_img2.png "Shell output"){ width:100% }

## Summary

With the interpreter added, the shell is usable. 
Thought it only supports some basic commands at this point and F# script and program support needs to be added it can perform some everyday tasks.

Specific shell commands need to be added like `cd` and `exit` still need to be added and there are a couple of bugs/changes required:

* After a file redirect that does not preserve output (`>` without `-p`), a blank line will be printed.
* Errors/output could wrap around and currently are not handled by the offset which only counts newlines.

But these can be fixed later, for now the most of the basic parts are in place.