<meta name="daria:article_id" content="writing_a_shell_in_fsharp_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="0">
<meta name="daria:created_on" content="2022-06-23">
<meta name="daria:tags" content="fsharp">

# Writing a shell in F# - Introduction

This series will look at writing a shell in F#. 
It is not meant to create a tool to replace bash or powershell, 
but is meant as a project to help me learn more about how shells work under the hood.

## Why?

Honestly, it seemed like a fun project. I don't know how far it will go but a few features I'd like to implement are:

* Basic shell functionality
* Syntax highlighting
* Integration of `.fsx` scripts and F# expressions
* Basic `CoreUtils`

## Design

The shell will be based on pipelines of bytes. Pipelines will be broken down into steps.

A step will either be successful or fail. Internally it will have 3 streams: `stdIn`, `stdOut` and `stdErr`.

For more complex tasks and flexibility there will be a step that runs user defined `.fsx` scripts.

There will be 2 main stages:

* Parsing - parse the user input into a set of steps.
* Execution - run a set of steps and output the result.

I am going to focus on the execution stage first.

## Notes

* This is not mean as a replacement for preexisting shell, which will likely be more efficient and tested more vigorously.
* I am no expert on the inter workings of shells, so don't claim any of this is the best way to do anything.

With that said, hopefully this is a fun project and a good learning experience.