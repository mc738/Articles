<meta name="daria:article_id" content="writing_a_database_in_rust_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="0">
<meta name="daria:created_on" content="2024-03-19">
<meta name="daria:tags" content="rust">
<meta name="daria:image_id" content="mountains">

# Writing a database in rust

This series will explore my experinces with writing a database in rust. This is defintely a experiment and learning process rather then the next big thing in database technology.

## Introduction

I want to preface this by saying I am still learning `rust` and know a lot of things can be done better and more effectively.
I am also no expert on database design and theory. However I have always found on the best ways to learn something is by attempting it.

The goal is to is to learn some `rust`, how to use vim (which I am trying out properly for the first time) and a bit about how databases work under the hood.

I am going to look at a document/object based database (I think, but this could change/evolve). However, hopefully the core components offer some insight to creating other types.
This isn't mean to be the next step in database technology, but (fingers crossed) there should be some semiusable solution at the end.

## Core design

The storage design of the database will be blocked based. Blocks are 4kb (4096 bytes) with a reserved space of header space of 256 bytes.
This leaves 3840 bytes of actual storage space per block.

## Why blocks?

As the data grows it should make it easier to maintain, edit and index. It also adds some robustness because it corruption in one block doesn't have to affect the whole database.

The cost is some extra overheads to maintain blocks and unused space within blocks. However, it seems a reasonable trade off.

## Data types

The database will support value base datatypes:

* Byte
* Short
* Int
* Long
* UByte
* UShrot
* UInt
* ULong
* ShortText
* Text
* LongText
* VarTest (TODO)
* ShortBinary
* Binary
* LongBinary
* VarBinary (TODO)
* DateTime (stored as timestamps)
* Uuid

Each data type will have a byte type code and a length. This is to facilitate serialization and deserialization.

## Storing and reading data

A block will store a tip pointer, this is a reference to how much of the data in a block is used. it starts at 256. Everytime data is written to a block this pointer will be increased by the data types size.

If a block doesn't have enough room for the value to be stored, a new block will be created and it stored in that. At the moment there effectively a 2.14gb limit on a value (`int32::MAX`).
However this should be more than enoung for now. Though it is important to be able to store values over multiple blocks.

When data is read the type is provided, a specifc number of bytes are read then deserialzed into the corresponding base type.

## Objects

TODO - being fleshed out.


