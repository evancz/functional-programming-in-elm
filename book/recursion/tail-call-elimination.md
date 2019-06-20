# Tail-Call Elimination

Before we get into *tail-call elimination*, it is important to understand a bit about how functions work in most programming languages.


## Stack Frames

Say we have a simple recursive implementation of `factorial` like this:

```elm
factorial : Int -> Int
factorial n =
  if n <= 1 then
    1
  else
    n * factorial (n - 1)
```

This is a silly function that I have never actually needed in my programming career. That said, it is particularly helpful for learning about *stack frames*! So the question we care about is: what happens behind the scenes when someone evaluates `factorial 3`? Well, first the computer considers the question:

    factorial 3
    -----------

It figures out the `if` expression and realizes that it needs to figure out `3 * factorial 2`. But what is `factorial 2`? So now it considers that question:

    factorial 2
    -----------
      3 * ???
    -----------

This continues as long as necessary:

    factorial 1
    -----------
      2 * ???
    -----------
      3 * ???
    -----------

At some point, we can stop making `factorial` calls:

         1
    -----------
      2 * ???
    -----------
      3 * ???
    -----------

And start collapsing things down:

      2 * 1
    -----------
      3 * ???
    -----------

And down:

      3 * 2
    -----------

Until we get the answer!

         6
    -----------

It turns out, this is exactly how function calls work in most programming languages. Everything is thrown on this *stack* of questions. And whenever you call a function, you allocate a new *stack frame* to remember any information you will need later. So when we call `factorial 3` we actually allocate some memory to remember that we need to multiply something by `3` later. And when we call `factorial 2` we allocate some memory to remember that we need to multiply something by `2` later. Etc.

So what happens when you call `factorial 10000`? Well, you end up allocating 10000 stack frames! That seems like a huge cost to pay compared with a simple `for` loop. Can we avoid it? Yes! That is exactly what *tail-call elimination* is for.


## Tail-Call Elimination

Let’s start by examining an alternate version of `factorial` that is a bit fancier:

```elm
factorial : Int -> Int
factorial n =
  factorialHelp n 1

factorialHelp : Int -> Int -> Int
factorialHelp n productSoFar
  if n <= 1 then
    productSoFar
  else
    factorialHelp (n - 1) (n * productSoFar)
```

We now have this `factorialHelp` function that keeps track of `n` as well as a `productSoFar` value. So unlike our previous implementation, all the information we need to get the answer is passed along to the next function call. The importance of this becomes clearer when you see the stack:

       factorial 3
    -----------------

This function now immediately calls `factorialHelp` like this:

    factorialHelp 3 1
    -----------------
           ???
    -----------------

Notice that `factorial` does not need to remember anything. It just needs the result of `factorialHelp`. From there, the next step is:

    factorialHelp 2 3
    -----------------
           ???
    -----------------
           ???
    -----------------

And then:

    factorialHelp 1 6
    -----------------
           ???
    -----------------
           ???
    -----------------
           ???
    -----------------

And then:

		    6
    -----------------
           ???
    -----------------
           ???
    -----------------
           ???
    -----------------

Now we just return `6` back through all this stack frames. There is nothing left to do though. **So why keep those intermediate stack frames around if they do not do anything?!** None of them hold any useful information. They are just using up time and space. Getting rid of these useless stack frames is called *tail-call elimination*.

With tail-call elimination, calling `factorial 10000` can use just *one* stack frame.


## How does this work in JavaScript and Elm?

As of this writing, tail-call elimination is in the ES6 spec for JavaScript, but it is not implemented in any browsers yet. It will be great when it is implemented, but in the meantime, it actually does not really matter for Elm. We can do the optimization in the Elm compiler!

The Elm compiler is able to do tail-call elimination on a function when any of the branches are a *direct* self-recursive call. So when the Elm compiler sees the `factorialHelp` function from above, it notices that one branch calls `factorialHelp` directly. From there it can produce a `while` loop like this:

```javascript
function factorialHelp(n, productSoFar)
{
    while (true)
    {
    	if (n <= 1)
    	{
    		return productSoFar;
    	}
    	productSoFar = n * productSoFar;
    	n = n - 1;
    }
}
```

Not exactly like this of course, but that is the idea! So now we just need *one* stack frame to compute `factorial 10000`. Much better!


## Folding over Lists

Comparing `List.foldl` and `List.foldr` is one of the classic ways to build familiarity with tail-call eliminitation. When you write your first `foldl` it looks something like this:

```elm
foldl : (a -> b -> b) -> b -> List a -> b
foldl add answerSoFar list =
  case list of
    [] ->
      answerSoFar

    first :: rest ->
      foldl add (add first answerSoFar) rest
```

Notice that `foldl` has a branch that calls `foldl` directly. Tail-call elimination will kick in on this function. When you write your first `foldr` it looks something like this:

```elm
foldr : (a -> b -> b) -> b -> List a -> b
foldr add defaultAnswer list =
  case list of
    [] ->
      defaultAnswer

    first :: rest ->
      add first (foldr add defaultAnswer rest)
```

Notice that neither of the `foldr` branches call `foldr` directly. There is an indirect call, but we have to come back and call `add` afterwards. So tail-call elimination will not kick in if you choose to write it this way.

> **Note:** There are many tricks out there to make sure functions like this do not run out of stack frames. [This blog post](https://blogs.janestreet.com/optimizing-list-map/) about optimizing `List.map` in OCaml outlines a particularly cool technique. We use similar techniques in Elm’s `List` library.


## Folding over Trees

Tail-call elimination gets a bit more interesting when you have *branching* structures, like the binary trees [described earlier](binary-trees.md).

So say we wanted to get the `sum` of all the values in a binary tree, and we wanted to use tail-call elimination as much as possible.

```elm
type Tree a = Empty | Node a (Tree a) (Tree a)

sum : Tree Int -> Int
sum tree =
  sumHelp tree 0

sumHelp : Tree Int -> Int -> Int
sumHelp tree sumSoFar =
  case tree of
    Empty ->
      sumSoFar

    Node n left right ->
      sumHelp left (sumHelp right (n + sumSoFar))
```

First, notice that we used the `foo` and `fooHelp` pattern again. This pattern is quite common when writing complex recursive functions. Put it in your mental toolbox!

Next, notice that one of the branches of `sumHelp` calls `sumHelp` directly to explore the `left` subtree. That means we can do tail-call elimination on that. But wait! There is another `sumHelp` call to explore the `right` subtree that *cannot* be eliminated. So we will need to allocate stack frames for that. This is quite an odd situation. I encourage you to write out the stack frames by hand and get a feel for how it will work in practice!

We cannot get rid of all recursion in this case, so how big can the tree get before we start having stack problems? Well, if we have a tree with ten levels, we will need ten stack frames. Trees can hold a great deal of information in each level though, so a tree with ten levels can hold `2^10` values, or `1024` values. A tree with 100 levels can hold `2^100` values, which comes out to more than `10^30` values. That is more than one thousand billion billion billion values!

In Elm, `Dict` and `Set` are both balanced binary trees, so they follow these growth patterns. `Array` is actually a tree with a branching factor of 32. That means that an `Array` ten levels deep holds `32^10` values, or more than `10^15` values.


## Summary

First, a common technique for writing recursive functions is to create a function `foo` with the API you want and a helper function `fooHelp` that has a couple additional arguments for accumulating some result. This definitely can come in handy if you are working on some more advanced algorithms!

Second, it turns out that “Is recursion going to crash my program?” is not a practical concern in functional languages like Elm. With linear structures (like `List`) we can do tricks to take advantage of tail-call elimination. With branching structures (like `Dict`, `Set`, and `Array`) you are likely to run out of disk space before you run out of stack space. Those are the only two kinds of structures we have!