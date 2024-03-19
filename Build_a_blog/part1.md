<meta name="daria:article_id" content="build_a_blog_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="0">
<meta name="daria:created_on" content="2022-06-21">
<meta name="daria:tags" content="fsharp,markdown,general">

# Build a blog - Introduction

A series exploring my experiences building a blog "from scratch". 
Although a plethora of blog tools and SSGs exist, I am the the type of person who sometimes likes to reinvent the wheel.

## Why "from scratch"?

Honestly, because I thought it would be a good learning experience and way to test some libraries I wrote.
I really like document generation and have had a personal goal of writing a markdown parser for a while.

Also, as a self taught coder, I hope to give something back.
If it wasn't for various articles, blogs and posts I wouldn't have learn half the stuff I had.

However, as a personal blog I felt like I would be cheating myself if I didn't at least try to write it "from scratch".

Obviously "from scratch" is a relative term, I haven't created my own programming language for it.
But I am happy with the bits I have created and hope something here can inspire someone out there!

## Tools

The main tool for this is a library I created called [FDOM](https://github.com/mc738/FDOM).
It is a documentation library with a (semi-complete) markdown parser and renderer for various formats.

It really helped me learn some of the beauty of F# and I have already from some great uses for it.

Other than that, I have used another library of mine called [Fluff](https://github.com/mc738/Fluff),
a tool for templating based on the [Mustache](https://en.wikipedia.org/wiki/Mustache_%28template_system%29) template system.

One external library I have used is [prism.js](https://prismjs.com/), for code sample syntax highlighting.
It is truly and incredible library and I feel really brings code samples to life!