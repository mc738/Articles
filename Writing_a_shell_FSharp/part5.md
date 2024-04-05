<meta name="daria:article_id" content="writing_a_shell_in_fsharp_part_5">
<meta name="daria:title" content="Part 5">
<meta name="daria:title_slug" content="part_5">
<meta name="daria:order" content="4">
<meta name="daria:created_on" content="2022-06-23">
<meta name="daria:tags" content="fsharp">

# Writing a shell in F# - Changes and fix ups

In this part I will go over a list of changes and fix ups I made required for the interpreter and to fix some bugs.

## Reason

While writing the interpreter I notices some issues, there were a few changes and fix ups needed so it seemed a 
smart idea to split this up into a separate part to keep things clear.

## Changes and fix ups

Firstly the signature of `ActionHandler` in `Configuration` need to change to return a `int`.
This is so the `InputController` knows how many lines to offset by (multiple lines could be printed):

```fsharp
// Line 13 of InputController.fs
type Configuration =
    private
        { PromptHandler: unit -> string
          // Old code:
          // ActionHandler: string -> unit
          ActionHandler: string -> int }
```

The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-d45abb98d275fb16bdbc2130cd4fbe2614b9dbf08a6fe0cf665e82fa720f54e0R16).

The `Create` method also needs to be updated to reflect this:

```fsharp
// Line 19 of InputController.fs
// Old Code:
// static member Create(promptFn: unit -> string, actionFn: string -> unit) =
static member Create(promptFn: unit -> string, actionFn: string -> int) =
    { PromptHandler = promptFn
      ActionHandler = actionFn }
```

The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-d45abb98d275fb16bdbc2130cd4fbe2614b9dbf08a6fe0cf665e82fa720f54e0R19).

Next, `handleLine` in `handleInput` needs to be updated to handle the offset correctly:

```fsharp
// Line 336 of Inputcontroller.fs
// Old code:
// cfg.ExecuteAction v
// state.NextLine(2, Some v)
let offset = cfg.ExecuteAction v
state.NextLine(1 + offset, Some v)
```

The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-d45abb98d275fb16bdbc2130cd4fbe2614b9dbf08a6fe0cf665e82fa720f54e0R336).

After that I updated `printIfNotEmpty` in `Pipes.fx` to return the number of lines printed:

```fsharp
// Line 58 of Pipes.fx
// Old code:
// let printIfNotEmpty (str: string) =
//  match String.IsNullOrEmpty str with
//  | true -> ()
//  | false -> printfn $"{str}" 
let printIfNotEmpty (str: string) =
    match  String.IsNullOrWhiteSpace str with
    | true -> 0
    | false ->
        Console.Write(Environment.NewLine)
        let lines = str.Split(Environment.NewLine)
        lines |> Array.iter (fun l -> printfn $"{l}")
        lines.Length
```
The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-a743015189b9d56821af438eb32a0e29046eaa47d5fdd00218566ea5e23ae107R58).

Lastly I needed to change `run` function in `Pipes.fs` to return an offset on error:

```fsharp
// Line 94 of Pipes.fsx
// Old code:
// match result with
// | Ok _ ->
//     getResult stdOut |> printIfNotEmpty
// | Error _ ->
//     Console.ForegroundColor <- ConsoleColor.Red
//     getResult stdErr |> printIfNotEmpty
//     Console.ResetColor()
match result with
| Ok _ ->
    getResult stdOut |> printIfNotEmpty
| Error _ ->
    Console.ForegroundColor <- ConsoleColor.Red
    let offset = getResult stdErr |> printIfNotEmpty
    Console.ResetColor()
    offset
```

The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-a743015189b9d56821af438eb32a0e29046eaa47d5fdd00218566ea5e23ae107R99).

For fix ups, I added some basic error handling to the `ls` function in `CoreUtils.fs`:

```fsharp
// Line 55 of CoreUtils.fx
// Old code:
// let ls (path: string) _ =
//    [ Directory.EnumerateDirectories(path) |> List.ofSeq
//      Directory.EnumerateFiles(path) |> List.ofSeq ]
//    |> List.concat
//    |> fun s -> String.Join(Environment.NewLine, s)
//    |> getBytes
//    |> Ok

let ls (path: string) _ =
    try
        [ Directory.EnumerateDirectories(path) |> List.ofSeq
          Directory.EnumerateFiles(path) |> List.ofSeq ]
        |> List.concat
        |> fun s -> String.Join(Environment.NewLine, s)
        |> getBytes
        |> Ok
    with
    | exn -> Error (Encoding.UTF8.GetBytes exn.Message)
```

The change is [here](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676#diff-7d74fecc89aea72e721211636efa96b3622e1f0203029d158c2bc1a67d5e88ddR55).

## Additions

I added a new function to `CoreUtils.fs` - `toFile` which will handle file redirects:

```fsharp
let toFile (path: string) (preserve: bool) (bytes: byte array) =
    try
        File.WriteAllBytes(path, bytes)
        match preserve with
        | true -> Ok bytes
        | false -> Ok [||]
    with
    | exn -> Error (Encoding.UTF8.GetBytes exn.Message)
```

I also added a new extension to strings in `Extenions.fx`:

```fsharp
member s.RemoveQuotes() =
    match (s.StartsWith('"') || s.StartsWith(''')), (s.EndsWith('"') || s.EndsWith(''')) with
    | true, true -> s.[1..(s.Length - 2)]
    | true, false -> s.[1..]
    | false, true -> s.[0..(s.Length - 2)]
    | false, false -> s
```

This simple attempts to remove `"` or `'` from the start and end of a string.

## Summary

All the changes can be found in [this commit](https://github.com/mc738/FShell/commit/a4f8b0987fcb97880958bd906e2a7088fbf94676).

With these changes in place it is time to move on to the interpreter, 
which will handle turns a list of tokens into a list of steps to be executed.