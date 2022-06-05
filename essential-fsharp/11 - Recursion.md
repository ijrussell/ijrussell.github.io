# 11 - Recursion

In this chapter, we are going to look at recursive functions, that is functions that call themselves in a loop. We will start with a naive implementation and then make it more efficient using an accumulator. As a special treat, I will show you a way of writing FizzBuzz and an elegant quicksort algorithm using this technique.

## Setting Up

Create a new folder for the code in this chapter.

All of the code in this chapter can be run using FSI from a .fsx file that you create.

## Solving The Problem

We are going to start with a naive implementation of the factorial function (!):

```plaintext
5! = 5 * 4 * 3 * 2 * 1 = 120
```

To create a recursive function, we use the *rec* keyword and we would create a function like this:

```fsharp
// int -> int
let rec fact n = 
    match n with
    | 1 -> 1
    | n -> n * fact (n-1)
```

You'll notice that we have two cases: 1 and greater than 1. When n is 1, we return 1 and the recursion terminates but if it is greater than 1, we return the number multiplied by the factorial of the number minus 1 and continue the recursion. If we were to write out what happens, it would look like this:

```text
fact 5 = 5 * fact 4
       = 5 * (4 * fact 3)
       = 5 * (4 * (3 * fact 2))
       = 5 * (4 * (3 * (2 * fact 1)))
       = 5 * (4 * (3 * (2 * 1)))
       = 5 * (4 * (3 * 2))
       = 5 * (4 * 6)
       = 5 * 24
       = 120
```

This is a problem because you can't perform any calculations until you've completed all of the iterations but you also need to store all of the parts as well. This means that the larger n gets, the more memory you need to perform the calculation and it can also lead to stack overflows. We can solve this problem with **Tail Call Optimisation**.

## Tail Call Optimisation

There are a few possible approaches available but we are going to use an accumulator. The accumulator is passed around the recursive function on each iteration:

```fsharp
let fact n =
    let rec loop n acc =
        match n with
        | 1 -> acc
        | _ -> loop (n-1) (acc * n)
    loop n 1
```

We leave the public interface of the function intact but create a function enclosed in the outer one to do the recursion. We also need to add a line at the end of the function to start the recursion and return the result. You'll notice that we've added an additional parameter (acc) which holds the accumulated value. As we are multiplying, we need to initialise the accumulator to 1. If the accumulator used addition, we would set it to 0 initially.

Lets write out what happens when we run this function as we did the previous version in pseudocode:

```plaintext
fact 5 = loop 5 1
       = loop 4 5
       = loop 3 20
       = loop 2 60
       = loop 1 120
       = 120
```

This is much simpler than the previous example, requires very little memory and is extremely efficient.

We can also solve factorial using the List.reduce and List.fold functions from the List module. The difference is that fold has an initial value and reduce uses only the values in the list:

```fsharp
let fact n = [1..n] |> List.fold (fun acc n -> acc * n) 1
let fact n = [1..n] |> List.reduce (fun acc n -> acc * n)
```

The F# tooling in VS Code will inform you that the accumulator functions can be shortened to:

```fsharp
let fact n = [1..n] |> List.fold (*) 1 
let fact n = [1..n] |> List.reduce (*)
```

Now that we've learnt about tail call optimisation using an accumulator, let's look at a harder example, the Fibonacci Sequence.

## Expanding the Accumulator

The Fibonacci Sequence is a simple list of numbers where each value is the sum of the previous two:

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, ...
```

Let's start with the naive example to calculate the nth item in the sequence:

```fsharp
let rec fib (n:int64) =
    match n with
    | 0L -> 0L
    | 1L -> 1L
    | s -> fib (s-1L) + fib (s-2L)
```

If you want to see just how inefficient this is, try running fib 50L. It will take almost a minute on a fast machine! Let's have a go at writing a more efficient version that uses tail call optimisation:

```fsharp
let fibL (n:int64) =
    let rec loop n (a,b) =
        match n with
        | 0L -> a
        | 1L -> b
        | n -> loop (n-1L) (b, a+b)
    loop n (0L,1L)
```

Let's write out what happens (ignoring the type annotation):

```plaintext
fibL 5L = loop 5L (0L, 1L)
        = loop 4L (1L, 1L+0L)
        = loop 3L (1L, 1L+1L)
        = loop 2L (2L, 1L+2L)
        = loop 1L (3L, 2L+3L)
        = 3L
```

The 5th item in the sequence is indeed 3 as the list starts at index 0. Try running fibL 50L - It should return almost instantaneously.

Next, we'll now continue on our journey to find as many functional ways of solving FizzBuzz as possible. :)

## FizzBuzz Using Recursion

We start with a list of rules that we are going to recurse over:

```fsharp
let fizzRules = 
    [ (3, "Fizz"); (5, "Buzz") ]
```

Our fizzbuzz function using tail call optimisation and has an accumulator that will use string concatenation and an initial value of "".

```fsharp
let fizzBuzz initialRules n =
    let rec loop remainingRules acc =
        match remainingRules, acc with
        | [], "" -> string n
        | [], _ -> acc
        | head::tail, _ -> 
            let value = 
                head |> (fun (i, v) -> if n % i = 0 then v else "") 
            loop tail (acc + value)
    loop initialRules ""
```

The pattern match is asking:

1. If there are no rules left and the accumulator is "", return the original input as a string
1. If there are no rules left, return the accumulator.
1. If there are rules left, loop with the tail as the rules and add the result of the function to the accumulator.

Finally we can run the function and print out the results to the terminal:

```fsharp
[ 1 .. 105 ]
|> List.map (fizzBuzz fizzRules)
|> List.iter (printfn "%s")
```

This is quite a nice extensible approach to the FizzBuzz problem as adding 7-Bazz is as easy as adding another rule.

```fsharp
let fizzRules = 
    [ (3, "Fizz"); (5, "Buzz"); (7, "Bazz") ]
```

Having produced a nice solution to the FizzBuzz problem using recursion, can we use the List.fold function to solve it? Of course we can!

```fsharp
let fizzBuzz n =
    [ (3, "Fizz"); (5, "Buzz") ]
    |> List.fold (fun acc (value,name) -> 
        if n % value = 0 then acc + name else acc) ""
    |> fun s -> if s = "" then string n else s

[1..105] 
|> List.iter (fizzBuzz >> printfn "%s")
```

We can modify the code to do all of the mapping in the fold function rather than passing the value on to another function:

```fsharp
let fizzBuzz n =
    [ (3, "Fizz"); (5, "Buzz") ]
    |> List.fold (fun acc (value,name) -> 
        match (if n % value = 0 then name else "") with
        | "" -> acc
        | s -> if acc = string n then s else acc + s) (string n)
```

The List.fold function is a very useful one to know well.

Lets have a look at another favourite algorithm, quicksort.

## Quicksort using recursion

[Quicksort](<https://www.tutorialspoint.com/data_structures_algorithms/quick_sort_algorithm.htm>) is a nice algorithm to create in F# because of the availability of some very useful collection functions in the List module:

```fsharp
let rec qsort input =
    match input with
    | [] -> []
    | head::tail ->
        let smaller, larger = List.partition (fun n -> head >= n) tail
        List.concat [qsort smaller; [head]; qsort larger]

[5;9;5;2;7;9;1;1;3;5] |> qsort |> printfn "%A"
```

The List.partition function splits a list into two based on a predicate function, in this case, items smaller than or equal to the head value and those larger than it. List.concat converts a deeply nested sequence of lists into a single list.

## Recursion with Hierarchical Data



```fsharp
open System.IO

type Tree<'T> =
    | Branch of 'T * Tree<'T> seq
    | Leaf of 'T

type Waypoint = { Location:string; Route:string list; TotalDistance:int }

type Connection = { Start:string; Finish:string; Distance:int }

let loadData path =
    path
    |> File.ReadAllText
    |> fun text -> text.Split(System.Environment.NewLine)
    |> Array.skip 1
    |> fun rows -> [
        for row in rows do
            match row.Split(",") with
            | [|start;finish;distance|] -> 
                { Start = start; Finish = finish; Distance = int distance }
                { Start = finish; Finish = start; Distance = int distance }
            | _ -> failwith "Row is badly formed"
    ]
    |> List.groupBy (fun cn -> cn.Start)
    |> Map.ofList

let getUnvisited connections current =
    connections
    |> List.filter (fun cn -> current.Route |> List.exists (fun loc -> loc = cn.Finish) |> not)
    |> List.map (fun cn -> { 
        Location = cn.Finish
        Route = cn.Start :: current.Route
        TotalDistance = cn.Distance + current.TotalDistance })

let rec treeToList tree =
    match tree with 
    | Leaf x -> [x]
    | Branch (_, xs) -> List.collect treeToList (xs |> Seq.toList)

let findPossibleRoutes start finish (routeMap:Map<string, Connection list>) =
    let rec loop current =
        let nextRoutes = getUnvisited routeMap[current.Location] current
        if nextRoutes |> List.isEmpty |> not && current.Location <> finish then
            Branch (current, seq { for next in nextRoutes do loop next })
        else 
            Leaf current
    loop { Location = start; Route = []; TotalDistance = 0 }
    |> treeToList
    |> List.filter (fun wp -> wp.Location = finish)

let selectShortestRoute routes =
    routes 
    |> List.minBy (fun wp -> wp.TotalDistance)
    |> fun wp -> wp.Location :: wp.Route |> List.rev, wp.TotalDistance

let run start finish = 
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "data.csv") 
    |> loadData
    |> findPossibleRoutes start finish 
    |> selectShortestRoute
    |> printfn "%A"

let result = run "Cogburg" "Leverstorm"
```







## Other Uses Of Recursion

The examples we've used are necessarily simple to concentrate on tail call optimisation using an accumulator but what would we use recursion for in a real-world application? Recursion is great for handling hierarchical data like the file system and XML, converting between flat data and hierarchies and for event loops or loops where there is no defined finish.

## Further Reading

As always, Scott Wlaschin's excellent [site](<https://fsharpforfunandprofit.com/posts/recursive-types-and-folds-3b/#series-toc>) has many posts on the topic .

## Summary

In this chapter, we have looked at recursion in F#. In particular, we have looked at the basics and how to use accumulators with tail call optimisation to make recursion more efficient.
