<meta name="daria:article_id" content="writing_a_dev_server_in_rust_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="0">
<meta name="daria:created_on" content="2022-07-05">
<meta name="daria:tags" content="rust,html/css,javascript">

# Writing a dev server in rust - Introduction

The series will look at writing a dev server in rust. The server will support hot reloading of pages when changes are made.

## Introduction

One of my most used extensions in VSCode is live server. It is great to be able to update a web page and see then changes when you switch back to the broswer.
I am currently trying to learn vim and thought similar functionality would be a great help.

There are lots of great live servers that exist already, however I thought it would be an interesting challenge to make my own.
I choose rust because it seems like a natural choice for this project and can produce a program that can be used (almost) anywhere.

The following features are what I will be looking at implementing:

* Server files from a specified directory (and sub directories).
* A file watcher to look for montior when files change.
* Hot reloading via websockets.
* Inject the handling script directly into pages.

I should note this server is purely for development and not meant for production.

## Design

The design of the app is relatively simple:

* Start the server and file watcher.
* When a client connects, inject the handler script into any `.html` files.
* The handler script opens a websocket connection to the server.
* When the file watcher sees a file change a message is sent to the client over the websocket connection and the page is reloaded.

There is scope to improve on this but that is the basic functionality I am looking to replicate.

Some nice-to-haves that I might look at in the future are:

* Fully implement websockets.
* SSL support.
* SPA style routing support.
* Mock end points.

For now, scripts will be injected via regex to keep things simple.

## Getting started

The rust book has a great tutorial on making a [http server](https://doc.rust-lang.org/book/ch20-00-final-project-a-web-server.html).
That will serve as the basis of this project. I am going to try to write as much of this myself. I think it is a better learning experience that way.

However a few dependencies I will use are:

* `chrono` - for timestamps in the logs.
* `sha1` - for generating the websockets handshake.
* `base64` - for generating the websockets handshake.
* `notify` - as the basis of the file watcher.
* `regex` - for injecting the handler script.

Other than that I will be writing it from scratch. I am still learning rust so a lot of things could be done better but hopefully it still provides some useful insight.