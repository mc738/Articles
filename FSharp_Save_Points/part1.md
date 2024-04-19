<meta name="daria:article_id" content="save_points_in_fsharp_part_1">
<meta name="daria:title" content="Part 1">
<meta name="daria:title_slug" content="part_1">
<meta name="daria:order" content="0">
<meta name="daria:created_on" content="2022-06-19">
<meta name="daria:tags" content="fsharp">
<meta name="daria:image_id" content="pine-watt-2Hzmz15wGik-unsplash">

# Save points in F# - Introduction

This series is going to explore the concept of save points in F#, their benefits and when you might want to use them.

## What is a save point?

For the purposes of these series, a save point is a deduplication method to ensure a part of a function only needs to be run once.

The general idea is to allow a consumer to only have to call a certain function once in a set context and reuse the results.

For example, imagine a situation where you need to call a computational expensive function but following calls to other functions could fail:

```fsharp
let myFunction _ =

    let expensiveComputationResult = expensiveComputation ()

    // If this call fails, or myFunction needs to be called again in the same context.
    // You might want to skip recalling expensiveComputation and use the result already returned.    
    anotherFunction expensiveComputationResult
```

The idea is inspired by save points in `sql`, where you can roll back a transaction to a certain point if something fails later.

## Why?

There are a few good reasons why:

* A function might be computational expensive and/or take time.
* A (non pure) function might have side effects, like updating a file or database and you don't want to do this twice or more.
* A function might be time sensitive and running it again later could cause data issues.
* You might want to use the same result else where or at a later date.
* Potentially anytime when idempotence matters.

## Downsides

The major downside is the added overhead. The result needs to be saved and possibly loaded, 
for a simple function call this overhead is probably not worth it.

Also, it adds complexity which might not be needed. It is important to weight up the pros and cons to decide if it is worth it.

## Alternatives

The best alternative is to either refactor code so either failure are handled gracefully (and retries are possible),
or to use an approach like the `saga design pattern`.

However, I think it is an interesting concept and worth exploring. 
Even if it ultimately proves to be the wrong approach it is a good way to think more about robust system design.

## Idempotence

There as much better explains of idempotence than I can give available, but that is the key concept here.
Basically it means any function that can be executed several times without changing the final result beyond its first iteration.
There are (nearly) always times where this matters, especially in distributed systems.
In these cases the extra overhead and complexity can be an acceptable trade off to make sure everything works as expected.

This is not meant to be a one-stop solution (or "best" way) to create idempotent functions, but one possible way.