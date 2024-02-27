# Generating SVGs in F# - Basic elements

In this part, I will go over creating the basic the basic elements and generating a SVG.

## Overview

This part will feature a long block of code. 
The main reason for this is because it is dealing with the basic building blocks.
It should be reasonably simple and hopefully pretty self explanatory, 
but is quite long.

## Basic building blocks

I started by adding a `Common.fs` file, this will hold the basic definitions of SVG elements.

```fsharp
namespace FSVG

[<AutoOpen>]
module Common =

    /// A SVG point
    type SvgPoint =
        { X: float
          Y: float }

        static member Create(x, y) = { X = x; Y = y }

        member p.Render() = $"{p.X},{p.Y}"

    /// A collection of SVG points
    type SvgPoints =
        { Values: SvgPoint array
          CurrentIndex: int }

        /// Create a collection of SVG points.
        static member Create(points: SvgPoint list) =
            { Values = points |> Array.ofList
              CurrentIndex = 0 }

        /// Check if an index in inbounds.
        member p.IsInBounds(i) = i >= 0 && i < p.Values.Length

        /// Get the current point.
        member p.Current() = p.Values.[p.CurrentIndex]

        /// Get a point at a specific index.
        member p.Get(i) =
            match p.IsInBounds i with
            | true -> p.Values.[i] |> Some
            | false -> None

        /// Get the next point.
        member p.Next() =
            match p.IsInBounds(p.CurrentIndex + 1) with
            | true -> p.Values.[p.CurrentIndex + 1] |> Some
            | false -> None

        /// Get the previous point.
        member p.Prev() =
            match p.IsInBounds(p.CurrentIndex - 1) with
            | true -> p.Values.[p.CurrentIndex - 1] |> Some
            | false -> None

        /// Advance the index by one. If it would be out of bounds None is returned.
        member p.Advance() =
            match p.IsInBounds(p.CurrentIndex + 1) with
            | true ->
                { p with CurrentIndex = p.CurrentIndex + 1 }
                |> Some
            | false -> None

    /// An SVG style.
    type Style =
        { Fill: string option
          Stroke: string option
          StrokeWidth: float option
          Opacity: float option
          GenericValues: Map<string, string> }

        static member Default() =
            { Fill = None
              Stroke = None
              StrokeWidth = None
              Opacity = Some 1.
              GenericValues = Map.empty }

        member s.Render(leadingSpace: bool) =
            let start =
                match leadingSpace with
                | true -> " "
                | false -> ""

            let e = ""

            let fill =
                s.Fill |> Option.defaultValue "none"

            let stroke =
                s.Stroke |> Option.defaultValue "none"

            let opacity =
                s.Opacity |> Option.defaultValue 0.

            let strokeWidth =
                s.StrokeWidth |> Option.defaultValue 0.

            $"""{start}fill="{fill}" stroke="{stroke}" strokeWidth="{strokeWidth}" opacity="{opacity}"{e}"""

    /// A rectangle element.
    type RectElement =
        { Height: float
          Width: float
          X: float
          Y: float
          RX: float
          RY: float
          Style: Style }

        static member Default() =
            { Height = 0.
              Width = 0.
              X = 0.
              Y = 0.
              RX = 0.
              RY = 0.
              Style = Style.Default() }

        member r.Render() =
            $"""<rect height="{r.Height}" width="{r.Width}" x="{r.X}" y="{r.Y}"{r.Style.Render(true)} />"""

    /// A circle element.
    type CircleElement =
        { CX: float
          CY: float
          R: float
          Style: Style }

        static member Default() =
            { CX = 0.
              CY = 0.
              R = 0.
              Style = Style.Default() }

        member c.Render() =
            $"""<circle cx="{c.CX}" cy="{c.CY}" r="{c.R}"{c.Style.Render(true)} />"""

    /// An ellipse element.
    type EllipseElement =
        { Height: float
          Width: float
          CX: float
          CY: float
          RX: float
          RY: float
          Style: Style }

        static member Default() =
            { Height = 0.
              Width = 0.
              CX = 0.
              CY = 0.
              RX = 0.
              RY = 0.
              Style = Style.Default() }

        member e.Render() =
            $"""<ellipse cx="{e.CX}" cy="{e.CY}" rx="{e.RX}" ry="{e.RY}"{e.Style.Render(true)} />"""

    /// A line element.
    type LineElement =
        { X1: float
          X2: float
          Y1: float
          Y2: float
          Style: Style }

        static member Default() =
            { X1 = 0.
              X2 = 0.
              Y1 = 0.
              Y2 = 0.
              Style = Style.Default() }

        member l.Render() =
            $"""<line x1="{l.X1}" y1="{l.Y1}" x2="{l.X2}" y2="{l.Y2}"{l.Style.Render(true)} />"""

    /// A polygon element.
    type PolygonElement =
        { Points: SvgPoint list
          Style: Style }

        static member Default() =
            { Points = []; Style = Style.Default() }

        member p.Render() =
            let points =
                p.Points
                |> List.map (fun p -> p.Render())
                |> String.concat " "

            $"""<polygon points="{points}"{p.Style.Render(true)} />"""

    /// A polyline element.
    type PolylineElement =
        { Points: SvgPoint list
          Style: Style }

        static member Default() =
            { Points = []; Style = Style.Default() }

        member p.Render() =
            let points =
                p.Points
                |> List.map (fun p -> p.Render())
                |> String.concat " "

            $"""<polyline points="{points}"{p.Style.Render(true)} />"""

    /// A path element.
    type PathElement =
        { Commands: PathCommand list
          Style: Style }

        static member Default() =
            { Commands = []
              Style = Style.Default() }

        member p.Render() =
            let d =
                p.Commands
                |> List.map (fun cmd -> cmd.Render())
                |> String.concat " "

            $"""<path d="{d}"{p.Style.Render(true)} />"""

    /// A path command.
    and PathCommand =
        | MoveTo of MoveToCommand
        | LineTo of LineToCommand
        | HorizontalLineTo of HorizontalLineToCommand
        | VerticalLineTo of VerticalLineToCommand
        | CurveTo of CurveToCommand
        | SmoothCurveTo of SmoothCurveToCommand
        | QuadraticBezierCurveTo of QuadraticBezierCurveToCommand
        | SmoothQuadraticBezierCurveTo of SmoothQuadraticBezierCurveToCommand
        | EllipticalArc of EllipticalArcCommand
        | ClosePath

        member p.Render() =
            match p with
            | MoveTo mt -> mt.Render()
            | LineTo lt -> lt.Render()
            | HorizontalLineTo hlt -> hlt.Render()
            | VerticalLineTo vlt -> vlt.Render()
            | CurveTo ct -> ct.Render()
            | SmoothCurveTo sct -> sct.Render()
            | QuadraticBezierCurveTo qbct -> qbct.Render()
            | SmoothQuadraticBezierCurveTo sqbct -> sqbct.Render()
            | EllipticalArc ea -> ea.Render()
            | ClosePath -> "Z"

    /// Move to path command (M/m).
    and MoveToCommand =
        { Point: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "m"
                | false -> "M"

            $"{c}{cmd.Point.Render()}"

    /// Line to path command (L/l).
    and LineToCommand =
        { Point: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "l"
                | false -> "L"

            $"{c}{cmd.Point.Render()}"

    /// Horizontal line to command (H/h).
    and HorizontalLineToCommand =
        { X: float
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "h"
                | false -> "H"

            $"{c}{cmd.X}"

    /// Vertical line to path command (V/v).
    and VerticalLineToCommand =
        { Y: float
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "v"
                | false -> "V"

            $"{c}{cmd.Y}"

    /// Curve to path command (C/c).
    and CurveToCommand =
        { ControlPoint1: SvgPoint
          ControlPoint2: SvgPoint
          FinalPoint: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "c"
                | false -> "C"

            $"{c}{cmd.ControlPoint1.Render()} {cmd.ControlPoint2.Render()} {cmd.FinalPoint.Render()}"

    /// Smooth curve to path command (S/s).
    and SmoothCurveToCommand =
        { ControlPoint2: SvgPoint
          FinalPoint: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "s"
                | false -> "S"

            $"{c}{cmd.ControlPoint2.Render()} {cmd.FinalPoint.Render()}"

    /// Quadratic bezier curve to path command (Q/q)
    and QuadraticBezierCurveToCommand =
        { ControlPoint1: SvgPoint
          FinalPoint: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "q"
                | false -> "Q"

            $"{c}{cmd.ControlPoint1.Render()} {cmd.FinalPoint.Render()}"

    /// Smooth quadratic bezier curve path command (T/t).
    and SmoothQuadraticBezierCurveToCommand =
        { Point: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "t"
                | false -> "T"

            $"{c}{cmd.Point.Render()}"

    /// Elliptical arch path command (A/a).
    and EllipticalArcCommand =
        { RadiiPoint: SvgPoint
          XRotation: float
          LargeArcFlag: bool
          SweepFlag: bool
          FinalPoint: SvgPoint
          IsRelative: bool }

        member cmd.Render() =
            let c =
                match cmd.IsRelative with
                | true -> "a"
                | false -> "A"

            let laf =
                match cmd.LargeArcFlag with
                | true -> 1
                | false -> 0

            let sf =
                match cmd.SweepFlag with
                | true -> 1
                | false -> 0

            $"{c}{cmd.RadiiPoint.Render()} {cmd.XRotation} {laf},{sf} {cmd.FinalPoint.Render()}"

    /// A text element.
    type TextElement =
        { X: float
          Y: float
          Value: TextType list
          Style: Style }

        static member Default() =
            { X = 0.
              Y = 0.
              Value = []
              Style = Style.Default() }

        member t.Render() =
            let contents =
                t.Value
                |> List.map (fun tt -> tt.Render())
                |> String.concat " "

            $"""<text x="{t.X}" y="{t.Y}"{t.Style.Render(true)}>{contents}</text>"""

    /// Text types.
    and TextType =
        | Literal of string
        | TextSpan of TextSpan

        member tt.Render() =
            match tt with
            | Literal v -> v
            | TextSpan ts -> ts.Render()

    /// A text span.
    and TextSpan =
        { X: float
          Y: float
          Value: string
          Style: Style }

        member ts.Render() =
            $"""<tspan x="{ts.X}" y="{ts.Y}"{ts.Style.Render(true)}>{ts.Value}</tspan>"""

    /// An SVG element.
    type Element =
        | Rect of RectElement
        | Circle of CircleElement
        | Ellipse of EllipseElement
        | Line of LineElement
        | Polygon of PolygonElement
        | Polyline of PolylineElement
        | Path of PathElement
        | Text of TextElement

        member e.Render() =
            match e with
            | Rect r -> r.Render()
            | Circle c -> c.Render()
            | Ellipse e -> e.Render()
            | Line l -> l.Render()
            | Polygon p -> p.Render()
            | Polyline p -> p.Render()
            | Path p -> p.Render()
            | Text t -> t.Render()

    /// A SVG document.
    type SvgDocument =
        { Elements: Element list }

        member sd.Render() =
            let elements =
                sd.Elements
                |> List.map (fun el -> el.Render())
                |> String.concat " "
            // TODO add defs.
            // TODO add view box
            $"""<svg viewBox="0 0 100 100" version="1.1" xmlns="http://www.w3.org/2000/svg" class="svg">
                    {elements}
                </svg>"""
```