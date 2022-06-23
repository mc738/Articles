# Create a basic file watcher in F#

This article goes over creating a basic configurable file watcher in F#. 
File watchers have many applications and can easily be implemented in .NET.

## Introduction

The `FileSystemWatcher` (in the `System.IO` namespace) offers a simple way to create a file watcher.
The implementation is based on [this example](https://docs.microsoft.com/en-us/dotnet/api/system.io.filesystemwatcher?view=net-6.0).
F# makes it easy to extend the example to be configurable.

## Watcher module

The basic watcher configuration and implementation:

```fsharp
[<RequireQualifiedAccess>]
module Watcher =

    open System.IO

    /// A file watcher configuration.
    type WatcherConfiguration =
        { Filter: string option
          IncludeSubDirectories: bool
          OnCreate: FileSystemEventArgs -> unit
          OnChange: FileSystemEventArgs -> unit
          OnDelete: FileSystemEventArgs -> unit
          OnRename: RenamedEventArgs -> unit
          OnError: ErrorEventArgs -> unit
          OnShutDown: unit -> unit }

    /// Start the watcher on a background thread.
    let startAsync (cfg: WatcherConfiguration) (path: string) =
        async {

            use watcher = new FileSystemWatcher(path)

            // Set the watcher filters to watch for everything.
            watcher.NotifyFilter <-
                NotifyFilters.Attributes
                ||| NotifyFilters.Security
                ||| NotifyFilters.Size
                ||| NotifyFilters.CreationTime
                ||| NotifyFilters.DirectoryName
                ||| NotifyFilters.FileName
                ||| NotifyFilters.LastAccess
                ||| NotifyFilters.LastWrite

            watcher.IncludeSubdirectories <- cfg.IncludeSubDirectories

            match cfg.Filter with
            | Some filter -> watcher.Filter <- filter
            | None -> ()

            // Set watcher event delegates.
            watcher.Created.Add cfg.OnCreate
            watcher.Changed.Add cfg.OnChange
            watcher.Deleted.Add cfg.OnDelete
            watcher.Renamed.Add cfg.OnRename
            watcher.Error.Add cfg.OnError

            watcher.EnableRaisingEvents <- true

            // Set the shut down event.
            use! c = Async.OnCancel cfg.OnShutDown

            // A simple loop to keep the watcher running.
            // Passing a cancellation token will handle shut down.
            while (true) do

                do! Async.Sleep 100
        }
```

## Example module

An example implementation:

```fsharp
module Example =

    open System
    open System.IO
    open System.Threading

    /// A helper function to simulate logging.
    let log msg = printfn $"[{DateTime.UtcNow}] {msg}"

    /// Test function for file creation event.
    let onCreate (e: FileSystemEventArgs) =
        match e.ChangeType with
        | WatcherChangeTypes.Created -> log $"File `{e.FullPath}` created."
        | _ -> ()

    /// Test function for file change event.
    let onChange (e: FileSystemEventArgs) =
        match e.ChangeType with
        | WatcherChangeTypes.Changed -> log $"File `{e.FullPath}` changed."
        | _ -> ()

    /// Test function for file delete event.
    let onDelete (e: FileSystemEventArgs) =
        match e.ChangeType with
        | WatcherChangeTypes.Deleted -> log $"File `{e.FullPath}` deleted."
        | _ -> ()

    /// Test function for file rename event.
    let onRename (e: RenamedEventArgs) =
        match e.ChangeType with
        | WatcherChangeTypes.Renamed -> log $"File `{e.OldFullPath}` renamed to `{e.FullPath}`."
        | _ -> ()

    /// Test function for errors.
    let onError (e: ErrorEventArgs) =
        log $"Error! {e.GetException().Message}"

    /// Test shutdown handler.
    let onShutDown _ = printfn "Watcher - Shutting down"

    // A example file watcher configuration.
    let cfg =
        ({ Filter = None
           IncludeSubDirectories = true
           OnCreate = onCreate
           OnChange = onChange
           OnDelete = onDelete
           OnRename = onRename
           OnError = onError
           OnShutDown = onShutDown }: Watcher.WatcherConfiguration)

    // Run the watcher.
    let start path =

        // Create a CancellationTokenSource for use in the watcher
        use cts = new CancellationTokenSource()
        let ct = cts.Token

        // Start the watcher.
        printfn $"Starting watcher. Path: `{path}`"
        Async.Start(Watcher.startAsync cfg path, cancellationToken = ct)
        
        // Block main thread until user presses any key.
        printfn "Press any key to close"
        Console.ReadLine() |> ignore
        
        // Close the watcher
        cts.Cancel()
```

A test program running this example:

```fsharp
Example.start "[path to directory to watch]"

printfn "Main - Watcher closed. Simulating further work/shutdown."
Async.Sleep 5000 |> Async.RunSynchronously

printfn "Closing"
```

## Wrapping up

This is a basic overview of what can be done. More complex workflows can be implemented via configurations.
A couple of example usages are:

* Monitor/audit file actions
* Rebuild projects on file changes/"hot reload" tools
* Automatic "smart" back ups

Hopefully this gives you some ideas of what is possible.