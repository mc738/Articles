# Writing a shell in F# - Pipes and CoreUtils

This part will looking at setting up a basic pipe mechanism and created some basic CoreUtils.

## Project

I have named the shell `FShell` (for want of a better name). The solution initial consists of 2 projects:

* `FShell.Core` - the base library.
* `FShell.App` - A console application to run the shell.

## Extensions

To start will, I added an created a module `Extensions` in `FShell.Core`. For now this is used to add some helpful functionality to the `MemoryStream` class.

```fsharp
namespace FShell.Core

[<AutoOpen>]
module Extensions =

    open System
    open System.IO
    open System.Text

    type MemoryStream with

        /// Try and read to then end the stream.
        /// If the stream length is longer than Int32.MaxValue this will fail.
        member ms.TryReadToEnd() =
            match ms.Length < (int64 Int32.MaxValue) with
            | true ->
                let buffer =
                    Array.zeroCreate (int ms.Length)

                ms.Read(Span buffer) |> ignore
                Ok buffer
            | false -> Error "Stream is bigger that 2.14748365 gb (2147483647 bytes). Can not read to end."

        /// Try and read a string from the stream.
        member ms.TryReadToString() =
            ms.TryReadToEnd()
            |> Result.map Encoding.UTF8.GetString

        /// Write a string to the stream.
        member ms.WriteString(value: string) =
            let b = value |> Encoding.UTF8.GetBytes
            ms.Write(ReadOnlySpan b)

        /// Reset the position of the stream.
        member ms.Reset() = ms.Position <- 0L

        /// Clear the stream.
        member ms.Clear() =
            ms.Reset()
            let len = int ms.Length
            ms.Write(Array.zeroCreate len, 0, len)
            ms.Reset()
```

This will make working with memory streams easier later on.

## Pipes

Next I added another module to `FShell.Core`: `Pipes`. This will handle pipelines.

```fsharp
namespace FShell.Core

/// Module containing the basic pipe implementation.
module Pipes =

    open System
    open System.IO
    
    /// A step in a pipeline. Takes a byte array (stdIn) and returns a byte array,
    /// either for stdOut or stdErr. 
    type Step = byte array -> Result<byte array, byte array>

    /// Run a individual step by reading from stdIn and either outputting to stdOut or stdErr.
    /// Returns a result indicating if the step was successful. 
    let runStep (step: Step) (stdIn: MemoryStream) (stdOut: MemoryStream) (stdErr: MemoryStream) =
        match stdIn.TryReadToEnd() with
        | Ok b ->
            match step b with
            | Ok r ->
                stdOut.Write r
                Ok()
            | Error e ->
                stdErr.Write e
                Error()
        | Error e ->
            stdErr.WriteString e
            Error()

    /// Run a pipeline (a collection of steps).
    let run (steps: Step list) =

        // Create memory streams for stdIn, stdOut and stdErr
        use stdIn = new MemoryStream()
        use stdOut = new MemoryStream()
        use stdErr = new MemoryStream()

        // On success, copy the contents of stdOut to stdIn,
        // for use in the next step.
        let onSuccess () =
            stdOut.Reset()
            stdIn.Clear()
            stdOut.CopyTo(stdIn)
            stdOut.Clear()
            stdIn.Reset()

        // Get a result from a memory stream,
        // for use with stdOut and stdErr.
        let getResult (ms: MemoryStream) =
            ms.Reset()
            match ms.TryReadToString() with
            | Ok s -> s
            | Error _ -> String.Empty
        
        // Output a string to the console if not null or empty.
        let printIfNotEmpty (str: string) =
            match String.IsNullOrEmpty str with
            | true -> ()
            | false -> printfn $"{str}"
        
        // Run the steps and get the result.
        // If a step fails no steps afterward will be run.
        let result =
            steps
            // This is basically a foldi folder - i just tracks the current items index.
            |> List.fold
                (fun (r, i) s ->
                    match r with
                    | Ok _ ->
                        match runStep s stdIn stdOut stdErr with
                        | Ok _ ->
                            // If not the last step run onSuccess to copy stdOut to stdIn.
                            // This is mainly done so the streams are not disposed.
                            if i <> steps.Length - 1 then onSuccess ()
                            Ok(), i + 1
                        | Error _ ->
                            // If the result is an error do nothing.
                            // runStep will already of handled writing to stdErr.
                            Error(), i + 1
                    | Error _ ->
                        // Pipeline failed at an earlier step,
                        // do nothing.
                        r, i + 1)
                (Ok(), 0)
            // Discard i, this is not needed.
            |> fun (r, _) -> r
        
        // Handle the result.
        match result with
        | Ok _ ->
            getResult stdOut |> printIfNotEmpty
        | Error _ ->
            Console.ForegroundColor <- ConsoleColor.Red
            getResult stdErr |> printIfNotEmpty
            Console.ResetColor()
```

Basically a pipe will consist of a steps and have 3 internal memory streams: `stdIn`, `stdOut` and `stdErr`.

This could be more efficient, currently it copies the result of `stdOut` back into `stdIn`. 
The main reason for this is so all three streams can be defined in to top level `run` and won't get disposed until the pipeline is complete.

## CoreUtils

With a basic pipe mechanism set up I added one more module to `FShell.Core`: `CoreUtils`.
For now this will have some basic utilities including `cat`, `ls`, `grep` and `echo`.
This can be expanded over time. Each on is with a `Step` or can be partially applied to be one.

```fsharp
namespace FShell.Core

/// Module containing CoreUtils. 
module CoreUtils =

    open System
    open System.IO
    open System.Text
    open System.Text.RegularExpressions    
    
    /// Helper functions for internal use.
    [<AutoOpen>]
    module private Helpers =

        /// Get a string from a byte array.
        let getString (bytes: Byte array) = Encoding.UTF8.GetString bytes

        /// Get a string array from a byte array split on a separator.
        let getStringCollection (separator: String) (bytes: Byte array) =
            getString bytes |> fun s -> s.Split separator

        /// Get bytes from a string.
        let getBytes (str: string) = Encoding.UTF8.GetBytes str

    /// A basic implementation of cat.
    let cat (path: string) _ =
        //let args = getString data

        try
            File.ReadAllBytes path |> Ok
        with
        | exn -> getBytes exn.Message |> Error

    
    /// A basic implementation of grep.
    let grep (pattern: string) (data: byte array) =

        getStringCollection Environment.NewLine data
        |> Array.choose (fun l ->
            match Regex.IsMatch(l, pattern) with
            | true -> Some l
            | false -> None)
        |> String.concat Environment.NewLine
        |> getBytes
        |> Ok

    
    /// A basic implementation of echo.
    let echo str _ =
        getBytes str |> Ok
        
    
    /// A basic implementation of ls.
    let ls (path: string) _ =
        [ Directory.EnumerateDirectories(path) |> List.ofSeq
          Directory.EnumerateFiles(path) |> List.ofSeq ]
        |> List.concat
        |> fun s -> String.Join(Environment.NewLine, s)
        |> getBytes
        |> Ok
        
    /// Redirect a result to an error.
    let toError (bytes: byte array) = Error bytes
        
    /// A basic implementation of whoami.
    let whoami _ = getBytes Environment.UserName |> Ok
    
    /// A basic implementation of base64
    let base64 (bytes: byte array) =
        Convert.ToBase64String bytes |> getBytes |> Ok
```

## Testing

With a basic shell set it, I could test it out. In `FShell.Core` I added some examples:

```fsharp
open FShell.Core

printfn "Example 1"
[ CoreUtils.cat "C:\\ProjectData\\Test\\test.txt"
  CoreUtils.grep "^Hello" ]
|> Pipes.run

printfn "---------------------------------"
printfn "Example 2"
[ CoreUtils.ls "C:\\ProjectData\\Test"
  CoreUtils.grep ".txt$" ]
|> Pipes.run

printfn "---------------------------------"
printfn "Example 3"
[ CoreUtils.echo "Test error"
  CoreUtils.toError ]
|> Pipes.run

printfn "---------------------------------"
printfn "Example 4"
[ CoreUtils.whoami ] |> Pipes.run

printfn "---------------------------------"
printfn "Example 5"
[ CoreUtils.cat "C:\\Users\\44748\\Downloads\\lighthouse_preview.jpg"
  CoreUtils.base64 ]
|> Pipes.run
```

## Summary

This should give a decent basis for going forwards. 
The basic pipe mechanism and some utilities make it possible to actually run some (simple) workflows.

So far it has taught me some bits about memory stream and the basic architecture of a shell, 
I look forward to expanding upon it further, adding some more complex functionality and reading user input.