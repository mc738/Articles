# Save points in F# - A basic implementation

This part will cover a basic save point implement. 
This will involve serializing result to JSON, saving them to a file and loading them on repeated calls.

# Basic process

The basic process will involve a function that accepts a save point name, a key and a function the returns type `'T`.

The save point name allows different save points to be defined and the key allows for deduplication.

Start by adding a `Common` module, this will hold common types and functions.

For now this will contain a discriminated union `SavePointType`, with one case identifier `File` that has field called `BasePath`:

```fsharp
[<AutoOpen>]
module Common =

    /// Types of save points
    type SavePointType = File of BasePath: string


```

*In the future this can be expanded for difference save point types/strategies*

Next, add a module called `Dedupe` with an inner module called `File`, 
with a `RequireQualifiedAccess` attribute to prevent naming conflicts later. 
Inside this module add another inner module called `Internal` (I have made this private), this will contain 2 helper functions to allow easy piping later.
The functions are `saveToFile` and `loadFromFile`. 
These functions will both accept a string `path`, a string `savePoint` and a string `key`.
The function `saveToFile` will also accept a string `value` - the value to be saved.
The job of these functions is to combine the base path and a file name `[savePoint]-[key].json` so the consumer code is simpler:

```fsharp
module Dedupe =

    open System.IO
    open System.Text.Json   
    
    [<RequireQualifiedAccess>]
    module File =

        module private Internal =

            /// A helper function to save a string to a file.
            /// Saves the file as [savePoint]-[key].json in the base path.
            let saveToFile (path: string) (savePoint: string) (key: string) (value: string) =
                File.WriteAllText(Path.Combine(path, $"{savePoint}-{key}.json"), value)

            /// A helper function to load a string from a file.
            /// Attempts to load the file called [savePoint]-[key].json from the base path.
            /// If the file doesn't exist None is return.
            let loadFromFile (path: string) (savePoint: string) (key: string) =
                let path =
                    Path.Combine(Path.Combine(path, $"{savePoint}-{key}.json"))

                match File.Exists path with
                | true -> File.ReadAllText path |> Some
                | false -> None
```

After that add 2 more functions, `saveResult<'T>` and `loadResult<'T>` to the `File` module.
These will serialize/deserialize a object to JSON and either save or attempt to load it from disk:

```fsharp
let saveResult<'T> (path: string) (savePoint: string) (key: string) (value: 'T) =
    JsonSerializer.Serialize value
    |> Internal.saveToFile path savePoint key

let loadResult<'T> (path: string) (savePoint: string) (key: string) =
    Internal.loadFromFile path savePoint key
    |> Option.map JsonSerializer.Deserialize<'T>
```

Now add one function to the `Dedupe` module called `runInSavePoint<'T>`. 
This will accept a `SavePointType`, a `savePoint` string, a `key` string and the function to be executed:

```fsharp
/// Run a function within a save point.
let runInSavePoint<'T> (savePointType: SavePointType) (savePoint: string) (key: string) (fn: unit -> 'T) =
    // Only one type at the moment, but this could expanded.
    match savePointType with
    | File basePath ->
        // Attempt to love the result. If successful, return it.
        // If not run fn, save the result and return it.
        match File.loadResult<'T> basePath savePoint key with
        | Some result -> result
        | None ->
            let result = fn ()
            File.saveResult basePath savePoint key result
            result
```

The full `Dedupe` module should look like:

```fsharp

module Dedupe =

    open System.IO
    open System.Text.Json
    
    [<RequireQualifiedAccess>]
    module File =

        module private Internal =

            /// A helper function to save a string to a file.
            /// Saves the file as [savePoint]-[key].json in the base path.
            let saveToFile (path: string) (savePoint: string) (key: string) (value: string) =
                File.WriteAllText(Path.Combine(path, $"{savePoint}-{key}.json"), value)


            /// A helper function to load a string from a file.
            /// Attempts to load the file called [savePoint]-[key].json from the base path.
            /// If the file doesn't exist None is return.
            let loadFromFile (path: string) (savePoint: string) (key: string) =
                let path =
                    Path.Combine(Path.Combine(path, $"{savePoint}-{key}.json"))

                match File.Exists path with
                | true -> File.ReadAllText path |> Some
                | false -> None

        let saveResult<'T> (path: string) (savePoint: string) (key: string) (value: 'T) =
            JsonSerializer.Serialize value
            |> Internal.saveToFile path savePoint key

        let loadResult<'T> (path: string) (savePoint: string) (key: string) =
            Internal.loadFromFile path savePoint key
            |> Option.map JsonSerializer.Deserialize<'T>
    
    /// Run a function within a save point.
    let runInSavePoint<'T> (savePointType: SavePointType) (savePoint: string) (key: string) (fn: unit -> 'T) =
        // Only one type at the moment, but this could expanded.
        match savePointType with
        | File basePath ->
            // Attempt to love the result. If successful, return it.
            // If not run fn, save the result and return it.
            match File.loadResult<'T> basePath savePoint key with
            | Some result -> result
            | None ->
                let result = fn ()
                File.saveResult basePath savePoint key result
                result
```

This can now be tested to make sure everything works:

```fsharp
type Foo =
    { Bar: string
      Baz: int
      Timestamp: DateTime }

let testFn _ =
    let timestamp = DateTime.UtcNow
    printfn "Running a expensive computation"
    Async.Sleep 1000 |> Async.RunSynchronously

    { Bar = "Hello, World!"
      Baz = 42
      Timestamp = timestamp }

// Change this for a base path. If not it will be [Project]\bin\Debug\net6.0.
let spt = SavePointType.File ""

let test1 =
    Dedupe.runInSavePoint<Foo> spt "test_sp" "test_key" testFn

let test2 =
    Dedupe.runInSavePoint<Foo> spt "test_sp" "test_key" testFn

printfn $"Are equal: {test1 = test2}"
```

## Summary

This is a basic implement, it has a few issues. Firstly it relies on `'T` being serializable as JSON.
Also a file per call might be a bit wasteful. 

However, it does does achieve the goal of making sure the function is only run once. 
Also, it does allow the result to be used else where if needed. 