<meta name="daria:article_id" content="writing_a_csv_parser_in_fsharp_part_3">
<meta name="daria:title" content="Part 3">
<meta name="daria:title_slug" content="part_3">
<meta name="daria:order" content="2">
<meta name="daria:created_on" content="2022-06-19">
<meta name="daria:tags" content="fsharp,csv">
<meta name="daria:image" content="twitter">

# Writing a CSV parser in F# - Record building

This part covers adding functionality to create a generic record from a collection of string values,
as well as adding the ability to specify a format for certain types.

## Format attribute

Start by creating a new module called `Common`. This will hold common functionality.

For now it will be used to define an attribute that can specify a format for `DateTime` and `Guid` fields.
Other types do not request this.

The module looks like this, it uses an `AutoOpen` attribute to make using it easier:

```fsharp
[<AutoOpen>]
module Common =

    open System
    
    /// An attribute to specify a specific format for DateTime and Guid fields in CSV.
    type CsvValueFormatAttribute(format: string) =

        inherit Attribute()

        member att.Format = format

```

*It is important that this module comes above all over modules in your solution, or else it will cause errors.*

## Record building

Next, add another module called `RecordBuilder`, this will handle building records. Inside this module add a private sub module called `TypeHelpers`.
This will be used to help parse values to the correct type:

```fsharp
module RecordBuilder =

    //These will be used later
    open System
    open System.Globalization
    open Microsoft.FSharp.Reflection

    /// A module to get base .NET type names for comparison.
    module private TypeHelpers =
        let getName<'T> = typeof<'T>.FullName

        let typeName (t: Type) = t.FullName

        let boolName = getName<bool>

        let uByteName = getName<uint8>

        let uShortName = getName<uint16>

        let uIntName = getName<uint32>

        let uLongName = getName<uint64>

        let byteName = getName<byte>

        let shortName = getName<int16>

        let intName = getName<int>

        let longName = getName<int64>

        let floatName = getName<float>

        let doubleName = getName<double>

        let decimalName = getName<decimal>

        let charName = getName<char>

        let timestampName = getName<DateTime>

        let uuidName = getName<Guid>

        let stringName = getName<string>
```

*Again, it is important this module comes before the `Parsing` module, or else it will cause errors.*

This simply gets the full name of base .NET types, which can be used in comparisons.

After that add a discriminated union called `SupportedType` to the `RecordBuilder` module. This will be used to control what types are supported.
It will have one method called `FromName` which will accept a type name string and return the supported type:

```fsharp
[<RequireQualifiedAccess>]
type SupportedType =
    | Boolean
    | Byte
    | Char
    | Decimal
    | Double
    | Float
    | Int
    | Short
    | Long
    | String
    | DateTime
    | Guid
    
    static member FromName(name: String) =
        match name with
        | t when t = TypeHelpers.boolName -> SupportedType.Boolean
        | t when t = TypeHelpers.byteName -> SupportedType.Byte
        | t when t = TypeHelpers.charName -> SupportedType.Char
        | t when t = TypeHelpers.decimalName -> SupportedType.Decimal
        | t when t = TypeHelpers.doubleName -> SupportedType.Double
        | t when t = TypeHelpers.floatName -> SupportedType.Float
        | t when t = TypeHelpers.intName -> SupportedType.Int
        | t when t = TypeHelpers.shortName -> SupportedType.Short
        | t when t = TypeHelpers.longName -> SupportedType.Long
        | t when t = TypeHelpers.stringName -> SupportedType.String
        | t when t = TypeHelpers.timestampName -> SupportedType.DateTime
        | t when t = TypeHelpers.uuidName -> SupportedType.Guid
        | _ -> failwith $"Type `{name}` not supported."
```

*The `failwith` will do for now. However this could be refactored to return a `Result<SupportedType,string>` later*

Now add to functions to the `RecordBuilder` module - `tryGetAtIndex` and `createRecord<'T>`:

```fsharp
/// A helper function to try and get field value by index.
let tryGetAtIndex (values: string array) (i: int) =
        match i >= 0 && i < values.Length with
        | true -> Some values.[i]
        | false -> None

/// Create a record of type 'T from a list of strings
let createRecord<'T> (values: string list) =
        
        // A helper function to get a field value by index.
        // Converts to an array for easier access by index,
        // however this could be skipped
        let getValue = values |> Array.ofList |> tryGetAtIndex

        // Get the generic type.
        let t = typeof<'T>

        // Create the values and box them.
        let values =
            // Get the properties from the type
            t.GetProperties()
            |> List.ofSeq
            // Use List.mapi to have access to the field index - to match up with the values list.
            |> List.mapi
                (fun i pi ->
                    // Check if the property has a format attribute.
                    let format =
                        match Attribute.GetCustomAttribute(pi, typeof<CsvValueFormatAttribute>) with
                        | att when att <> null -> Some <| (att :?> CsvValueFormatAttribute).Format
                        | _ -> None

                    // Get the supported type.
                    let t = SupportedType.FromName(pi.PropertyType.FullName)

                    // Attempt to get the value,
                    // Then match on the supported type, parse and box.
                    match getValue i, t with
                    | Some v, SupportedType.Boolean -> bool.Parse v :> obj
                    | Some v, SupportedType.Byte -> Byte.Parse v :> obj
                    | Some v, SupportedType.Char -> v.[0] :> obj
                    | Some v, SupportedType.Decimal -> Decimal.Parse v :> obj
                    | Some v, SupportedType.Double -> Double.Parse v :> obj
                    | Some v, SupportedType.DateTime ->
                        match format with
                        | Some f -> DateTime.ParseExact(v, f, CultureInfo.InvariantCulture)
                        | None -> DateTime.Parse(v)
                        :> obj
                    | Some v, SupportedType.Float -> Double.Parse v :> obj
                    | Some v, SupportedType.Guid ->
                        match format with
                        | Some f -> Guid.ParseExact(v, f)
                        | None -> Guid.Parse(v)
                        :> obj
                    | Some v, SupportedType.Int -> Int32.Parse v :> obj
                    | Some v, SupportedType.Long -> Int64.Parse v :> obj
                    | Some v, SupportedType.Short -> Int16.Parse v :> obj
                    | Some v, SupportedType.String -> v :> obj
                    | None, _ -> failwith "Could not get value")

        // Create the record.
        // This will return an object.
        let o =
            FSharpValue.MakeRecord(t, values |> Array.ofList)

        // Downcast the newly created object back to type 'T
        o :?> 'T
```

*Values are boxed because `FSharpValue.MakeRecord` accepts a array of objects*

The whole `RecordBuilder` module should look like this:

```fsharp
module RecordBuilder =

    open System
    open System.Globalization
    open Microsoft.FSharp.Reflection

    module private TypeHelpers =
        let getName<'T> = typeof<'T>.FullName

        let typeName (t: Type) = t.FullName

        let boolName = getName<bool>

        let uByteName = getName<uint8>

        let uShortName = getName<uint16>

        let uIntName = getName<uint32>

        let uLongName = getName<uint64>

        let byteName = getName<byte>

        let shortName = getName<int16>

        let intName = getName<int>

        let longName = getName<int64>

        let floatName = getName<float>

        let doubleName = getName<double>

        let decimalName = getName<decimal>

        let charName = getName<char>

        let timestampName = getName<DateTime>

        let uuidName = getName<Guid>

        let stringName = getName<string>
            
    [<RequireQualifiedAccess>]
    type SupportedType =
        | Boolean
        | Byte
        | Char
        | Decimal
        | Double
        | Float
        | Int
        | Short
        | Long
        | String
        | DateTime
        | Guid
        
        static member FromName(name: String) =
            match name with
            | t when t = TypeHelpers.boolName -> SupportedType.Boolean
            | t when t = TypeHelpers.byteName -> SupportedType.Byte
            | t when t = TypeHelpers.charName -> SupportedType.Char
            | t when t = TypeHelpers.decimalName -> SupportedType.Decimal
            | t when t = TypeHelpers.doubleName -> SupportedType.Double
            | t when t = TypeHelpers.floatName -> SupportedType.Float
            | t when t = TypeHelpers.intName -> SupportedType.Int
            | t when t = TypeHelpers.shortName -> SupportedType.Short
            | t when t = TypeHelpers.longName -> SupportedType.Long
            | t when t = TypeHelpers.stringName -> SupportedType.String
            | t when t = TypeHelpers.timestampName -> SupportedType.DateTime
            | t when t = TypeHelpers.uuidName -> SupportedType.Guid
            | _ -> failwith $"Type `{name}` not supported."
    
    /// A helper function to try and get field value by index.
    let tryGetAtIndex (values: string array) (i: int) =
            match i >= 0 && i < values.Length with
            | true -> Some values.[i]
            | false -> None
    
    /// Create a record of type 'T from a list of strings
    let createRecord<'T> (values: string list) =
            
            // A helper function to get a field value by index.
            // Converts to an array for easier access by index,
            // however this could be skipped
            let getValue = values |> Array.ofList |> tryGetAtIndex

            // Get the generic type.
            let t = typeof<'T>

            // Create the values and box them.
            let values =
                // Get the properties from the type
                t.GetProperties()
                |> List.ofSeq
                // Use List.mapi to have access to the field index - to match up with the values list.
                |> List.mapi
                    (fun i pi ->
                        // Check if the property has a format attribute.
                        let format =
                            match Attribute.GetCustomAttribute(pi, typeof<CsvValueFormatAttribute>) with
                            | att when att <> null -> Some <| (att :?> CsvValueFormatAttribute).Format
                            | _ -> None

                        // Get the supported type.
                        let t = SupportedType.FromName(pi.PropertyType.FullName)

                        // Attempt to get the value,
                        // Then match on the supported type, parse and box.
                        match getValue i, t with
                        | Some v, SupportedType.Boolean -> bool.Parse v :> obj
                        | Some v, SupportedType.Byte -> Byte.Parse v :> obj
                        | Some v, SupportedType.Char -> v.[0] :> obj
                        | Some v, SupportedType.Decimal -> Decimal.Parse v :> obj
                        | Some v, SupportedType.Double -> Double.Parse v :> obj
                        | Some v, SupportedType.DateTime ->
                            match format with
                            | Some f -> DateTime.ParseExact(v, f, CultureInfo.InvariantCulture)
                            | None -> DateTime.Parse(v)
                            :> obj
                        | Some v, SupportedType.Float -> Double.Parse v :> obj
                        | Some v, SupportedType.Guid ->
                            match format with
                            | Some f -> Guid.ParseExact(v, f)
                            | None -> Guid.Parse(v)
                            :> obj
                        | Some v, SupportedType.Int -> Int32.Parse v :> obj
                        | Some v, SupportedType.Long -> Int64.Parse v :> obj
                        | Some v, SupportedType.Short -> Int16.Parse v :> obj
                        | Some v, SupportedType.String -> v :> obj
                        | None, _ -> failwith "Could not get value")

            // Create the record.
            // This will return an object.
            let o =
                FSharpValue.MakeRecord(t, values |> Array.ofList)

            // Downcast the newly created object back to type 'T
            o :?> 'T
```

*This could handle errors better, however it can be easily refactored to do so.*

## Summary

With this, it is now possible to create a generic F# record with one call:


```fsharp
type Foo = { Bar: string; Baz: int }

let record = RecordBuilder.createRecord<Foo> [ "Hello, World!"; "42" ]
```

In the next part we will looking at bring it all together to actually parse a CSV file to a collection of records.