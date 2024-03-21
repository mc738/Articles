<meta name="daria:article_id" content="generating_svgs_fsharp_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="9">
<meta name="daria:created_on" content="2024-03-19">
<meta name="daria:tags" content="fsharp,svg">

# Generating SVGs in F# - Introduction

SVGs have many applications, from charts and diagrams to logos.
They are scalable and because they are based on a text file can easily be generated in code.
This series will look at creating an F# library to do just that.

## Overview

SVGs are incredibly flexible and have many uses.
Because they are text based they are are easy generate via code without needing specific graphics library.

By combining the basic elements you can create all types of visualizations and images.
This means the possibilities are endless (and results can be incredibly satisfying).

The first parts of these series will go over creating the basic building blocks,
then later I will explore real life uses like generating charts.

## Basic elements

SVGs contain the following basic elements:

* `rect`
* `circle`
* `ellipse`
* `line`
* `polygon`
* `polyline`
* `path`
* `text`

A description of these can be found [here](https://www.w3schools.com/graphics/svg_intro.asp).

# FSVG

This series will follow the development of a library I am writing: `FSVG`.
The first iteration will be reasonable basic but I plan to add to it over time. 