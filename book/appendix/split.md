# Split

In [the merge sort overview](merge-sort.md), we saw a pretty nice `split` implementation.

There are two alternate `split` implementations that are rather extraordinary. I definitely could not have come up with either of these implementations when I was learning, but I am very glad someone showed them to me!

So when we implemented `split` before, we started with our two step process for writing recursive functions, but gave up when we got to:

```elm
split : List a -> ( List a, List a )
split list =
  case list of
    [] ->
      ( [], [] )

    front :: rest ->
      ...
```

It turns out, we could have made it work! There are two ways we could have done it:

  1. Unroll the pattern.
  2. Be super clever.

Technique one rarely comes up in practice, but it can happen. Technique two... Well, you’ll see!


### Unroll the Pattern

When dealing with lists where the particular order of elements matters, it can be helpful to unroll the pattern match a bit more. So instead of just pattern matching on one element, we can pattern match on two elements:

```elm
split : List a -> ( List a, List a )
split list =
  case list of
    [] ->
      ( [], [] )

    [value] ->
      ...

    v1 :: v2 :: rest ->
      ...
```

That means we take *two* elements at a time. We can put `v1` on the first list and `v2` on the second list. Eventually we will get to either the empty list or a list with only one element. Splitting those cases in half is pretty straight forward.

```elm
split : List a -> ( List a, List a )
split list =
  case list of
    [] ->
      ( [], [] )

    [value] ->
      ( [value], [] )

    v1 :: v2 :: rest ->
      ( v1 :: ... , v2 :: ... )
```

Now we can pretend we are already done implementing `split` to fill in the blanks:

```elm
split : List a -> ( List a, List a )
split list =
  case list of
    [] ->
      ( [], [] )

    [value] ->
      ( [value], [] )

    v1 :: v2 :: rest ->
      let
        (halfOne, halfTwo) =
          split rest
      in
        ( v1 :: halfOne , v2 :: halfTwo )
```

If there is a takeaway here, it is that unrolling the pattern match may be helpful in some situations.


## Be Super Clever

When I was learning about merge sort, my professor had us struggle with `split` implementations on our own for a while. After all that, my professor showed us the following version:

```elm
split : List a -> ( List a, List a )
split list =
  case list of
    [] ->
      ( [], [] )

    front :: rest ->
      let
        (halfOne, halfTwo) =
          split rest
      in
        ( halfTwo, front :: halfOne )
```

It is definitely worth working through this on paper with some example inputs to see how it works.

Now I want to emphasize that **I have no idea how you come up with something like this.** That was true when I first saw this eight years ago, and it is still true after implementing the Elm compiler in Haskell. Fortunately, falling back to the `foo` and `fooHelp` pattern always works well, so we do not actually have to know how people discover stuff like this!

That said, it turns out our trusty two step process actually *could* work on this one. I don’t know what that means. I’m just saying!
