# 10 - Object Programming

In this chapter, we are going to see how we can utilise some of the object programming features that F# offers. F# is a functional-first language but sometimes it's beneficial to use objects, particularly when interacting with code written in other, less functional .NET languages or when you want to encapsulate some internal data structures and/or mutable state.

F# can do most of the things that C#/VB.Net can do with objects. We are going to concentrate on the core object programming features; class types, interfaces, encapsulation, and equality.

## Setting Up

Create a new folder for the chapter code and open it in VS Code.

Add three new files, *FizzBuzz.fsx*, *RecentlyUsedList.fsx* and *Coordinate.fsx*.

## Class Types

We will start in *FizzBuzz.fsx* where we will be implementing FizzBuzz using object programming.

We are going to create our first class type and use the code from the `fizzBuzz` function we created in Chapter 7:

```fsharp
type FizzBuzz() =
    member _.Calculate(value) =
        [(3, "Fizz");(5, "Buzz")]
        |> List.map (fun (v, s) -> if value % v = 0 then s else "")
        |> List.reduce (+)
        |> fun s -> if s <> "" then s else string value
```

Points of interest:

- The brackets () after the type name are required. They can contain arguments as we will see later in the chapter.
- The member keyword defines the accessible members of the type.
- The \_ is just a placeholder - it can be anything. It is the convention to use one of \_, \_\_ or *this*.

Now that we have created our class type, we need to instantiate it to use it.

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz()
    [1..15]
    |> List.map (fun n -> fizzBuzz.Calculate n)
```

We don't use *new* when creating an instance of a class type in F# except when it implements IDisposible<'T> and then we would use *use* instead of *let* to create a scope block.

The code can be simplified to:

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz()
    [1..15]
    |> List.map fizzBuzz.Calculate
```

At the moment, we can only use [(3, "Fizz");(5, "Buzz")] as the mapping but it is easy to pass the mapping in through the default constructor:

```fsharp
type FizzBuzz(mapping) =
    member _.Calculate(value) = 
        mapping
        |> List.map (fun (v, s) -> if n % v = 0 then s else "")
        |> List.reduce (+)
        |> fun s -> if s <> "" then s else string n
```

Notice that we don't need to assign the constructor argument with a let binding to use it.

Now we need to pass the mapping in as the argument to the constructor in the doFizzBuzz function:

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz([(3, "Fizz");(5, "Buzz")])
    [1..15]
    |> List.map fizzBuzz.Calculate
```

We can move the function code from the member into the body of the class type as a new inner function:

```fsharp
type FizzBuzz(mapping) =
    let calculate n =
        mapping
        |> List.map (fun (v, s) -> if n % v = 0 then s else "")
        |> List.reduce (+)
        |> fun s -> if s <> "" then s else string n
    
    member _.Calculate(value) = calculate value
```

You cannot access the new calculate function from outside the class type. You don't have to do this but I find that it makes the code easier to read, especially as the number of class members increases.

## Interfaces

Interfaces are very important in object programming as they define a contract that an implementation must handle. Let's create an interface to use in our fizzbuzz example:

```fsharp
// The 'I' prefix to the name is not required but is used by convention in .NET 
type IFizzBuzz =
    abstract member Calculate : int -> string
```

> #### Abstract Classes
> To convert IFizzBuzz into an abstract class, decorate it with the [<AbstractClass>] attribute.

Now we need to implement the interface in our FizzBuzz class type:

```fsharp
type FizzBuzz(mapping) =
    let calculate n =
        mapping
        |> List.map (fun (v, s) -> if n % v = 0 then s else "")
        |> List.reduce (+)
        |> fun s -> if s <> "" then s else string n
    
    interface IFizzBuzz with
        member _.Calculate(value) = calculate value
```

Nice and easy but you will see that we have a problem; The compiler has highlighted the fizzBuzz.Calculate function call in our doFizzBuzz function.

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz([(3, "Fizz");(5, "Buzz")])
    [1..15]
    |> List.map (fun n -> fizzBuzz.Calculate(n)) // Problem
```

There is an error because F# does not support implicit casting, so we have to upcast the instance to the IFizzBuzz type ourselves:

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz([(3, "Fizz");(5, "Buzz")]) :> IFizzBuzz //Upcast  
    [1..15]
    |> List.map (fun n -> fizzBuzz.Calculate(n)) // Fixed
```

An alternative would be to upcast as you use the interface function:

```fsharp
let doFizzBuzz =
    let fizzBuzz = FizzBuzz([(3, "Fizz");(5, "Buzz")])
    [1..15]
    |> List.map (fun n -> (fizzBuzz :> IFizzBuzz).Calculate(n)) 
```

If you have come from a language like C# where it supports implicit casting, it may seem odd to have to explicitly cast in F# but it gives you the safety of being certain about what your code is doing as you are not relying on compiler magic.

The code above is designed to show how to construct class types and use interfaces. If you find yourself constructing interfaces with one function, ask yourself if you really, really need the extra code and complexity or whether a simple function is enough.

There is another feature that interfaces offer that we haven't covered so far: Object Expressions.

## Object Expressions

Create a new script file called expression.fsx and add the following import declaration:

```fsharp
open System
```

Now we are going to create a new interface:

```fsharp
type ILogger =
    abstract member Info : string -> unit
    abstract member Error : string -> unit
```

We would normally create a class type:

```fsharp
type Logger() =
    interface ILogger with
        member _.Info(msg) = printfn "Info: %s" msg
        member _.Error(msg) = printfn "Error: %s" msg
```

To use this, we need to create an instance and cast it to an ILogger.

Instead, we are going to create an object expression which simplifies the usage:

```fsharp
let logger = {
    new ILogger with
        member _.Info(msg) = printfn "Info: %s" msg
        member _.Error(msg) = printfn "Error: %s" msg
}
```

We have actually created an anonymous type, so we are restricted in what we can do with it. Let's create a class type and pass the ILogger in as a constructor argument:

```fsharp
type MyClass(logger:ILogger) =
    let mutable count = 0

    member _.DoSomething input = 
        logger.Info $"Processing {input} at {DateTime.UtcNow.ToString()}"
        count <- count + 1
        ()

    member _.Count = count
```

Note that we can create members of the class type that are not associated with an interface.

We can now use the class type with the object expression logger:

```fsharp
let myClass = MyClass(logger)
[1..10] |> List.iter myClass.DoSomething
printfn "%i" myClass.Count
```

Run all of these blocks of code in FSI.

Create a function that takes in an ILogger as a parameter:

```fsharp
let doSomethingElse (logger:ILogger) input =
    logger.Info $"Processing {input} at {DateTime.UtcNow.ToString()}"
    ()
```

We could use our existing object expression logger but we can also create an object expression directly in the function call:

```fsharp
doSomethingElse {
    new ILogger with
        member _.Info(msg) = printfn "Info: %s" msg
        member _.Error(msg) = printfn "Error: %s" msg
    } "MyData"
```

This is a really useful feature if you have one-off services that you need to run.

Next we move on to a more complex example of using interfaces: A recently used list.

## Encapsulation

We are going to create a recently used list as a class type. We will encapsulate a mutable collection within the class type and provide an interface for how we can interact with it. The recently used list is an ordered list with the most recent item first but it is also a set, so each value can only appear once in the list.

First we need to create an interface in RecentlyUsedList.fsx:

```fsharp
type IRecentlyUsedList =
    abstract member IsEmpty : bool
    abstract member Size : int
    abstract member Clear : unit -> unit
    abstract member Add : string -> unit
    abstract member Get : int -> string option
```

By looking at the signatures, we can see that IsEmpty and Size are read-only properties and Clear, Add and Get are functions. Now we can create our class type and implement the IRecentlyUsedList interface:  

```fsharp
type RecentlyUsedList() =
    let items = ResizeArray<string>()

    let add item =
        items.Remove item |> ignore
        items.Add item 
    
    let get index =
        if index >= 0 && index < items.Count 
        then Some items.[items.Count - index - 1]
        else None

    interface IRecentlyUsedList with
        member _.IsEmpty = items.Count = 0
        member _.Size = items.Count
        member _.Clear() = items.Clear()
        member _.Add(item) = add item
        member _.Get(index) = get index
```

The ResizeArray<'T> is the F# synonym for the standard .NET mutable (List<'T>). Encapsulation ensures that you cannot access it directly, only via the public interface.

Let's test our code in FSI. Run each of the following lines separately:

```fsharp
let mrul = RecentlyUsedList() :> IRecentlyUsedList

mrul.Add "Test"

mrul.IsEmpty = false // Should return true

mrul.Add "Test2"
mrul.Add "Test3"
mrul.Add "Test"

mrul.Get(0) = Some "Test" // Should return true
```

Now let's add a maximum size (capacity) to our IRecentlyUsedList interface:

```fsharp
type IRecentlyUsedList =
    abstract member IsEmpty : bool
    abstract member Size : int
    abstract member Capacity : int
    abstract member Clear : unit -> unit
    abstract member Add : string -> unit
    abstract member Get : int -> string option
```

You will notice that the compiler is complaining that we haven't implemented all of the interface items, so let's fix it. Add the capacity as a constructor argument and implement the Add function to ensure the oldest item is removed if we are at capacity when adding a new item:

```fsharp
type RecentlyUsedList(capacity:int) =
    let items = ResizeArray<string>(capacity)

    let add item =
        items.Remove item |> ignore
        if items.Count = items.Capacity then items.RemoveAt 0
        items.Add item
    
    let get index =
        if index >= 0 && index < items.Count 
        then Some items.[items.Count - index - 1]
        else None

    interface IRecentlyUsedList with
        member _.IsEmpty = items.Count = 0
        member _.Size = items.Count
        member _.Capacity = items.Capacity
        member _.Clear() = items.Clear()
        member _.Add(item) = add item
        member _.Get(index) = get index
```

Let's test our recently used list with capacity of 5 using FSI:

```fsharp
let mrul = RecentlyUsedList(5) :> IRecentlyUsedList

mrul.Capacity // Should be 5

mrul.Add "Test"
mrul.Size // Should be 1
mrul.Capacity // Should be 5

mrul.Add "Test2"
mrul.Add "Test3"
mrul.Add "Test4"
mrul.Add "Test"
mrul.Add "Test6"
mrul.Add "Test7"
mrul.Add "Test"

mrul.Size // Should be 5
mrul.Capacity // Should be 5
mrul.Get(0) = Some "Test" // Should return true
mrul.Get(4) = Some "Test3" // Should return true
```

Encapsulation inside class types works really nicely, even for mutable data.

Now we move on to the issue of equality. 

## Equality

Most of the types in F# support structural equality but class types do not: They rely on reference equality like most things in .NET.

Create a simple class type to store GPS coordinates in Coordinate.fsx:

```fsharp
type Coordinate(latitude: float, longitude: float) =
    member _.Latitude = latitude
    member _.Longitude = longitude
```

To test equality, we can write some simple checks we can run in FSI:

```fsharp
let c1 = Coordinate(25.0, 11.98)
let c2 = Coordinate(25.0, 11.98)
let c3 = c1
c1 = c2 // false
c1 = c3 // true - reference the same instance
```

To support something like structural equality, we need to override the GetHashCode and Equals functions, implement IEquatable<'T> and if we are going to use it with other .NET languages, we need to handle the '=' operator using op_Equality and set the Allow Null Literal attribute:

```fsharp
open System

[<AllowNullLiteral>]
type GpsCoordinate(latitude: float, longitude: float) =
    let equals (other: GpsCoordinate) =
        if isNull other then
            false
        else
            latitude = other.Latitude
            && longitude = other.Longitude
    
    member _.Latitude = latitude
    member _.Longitude = longitude
    
    override this.GetHashCode() =
        hash (this.Latitude, this.Longitude)
    
    override _.Equals(obj) =
        match obj with
        | :? GpsCoordinate as other -> equals other
        | _ -> false
    
    interface IEquatable<GpsCoordinate> with
        member _.Equals(other: GpsCoordinate) =
            equals other
    
    static member op_Equality(this: GpsCoordinate, other: GpsCoordinate) =
        this.Equals(other)
```

We have used two built-in functions: *hash* and *isNull*. We make use of pattern matching in the Equals function by using a match expression to see if the object passed in can be cast as a GpsCoordinate and if it can, it is checked for equality.

If we test this, we get the expected structural equality:

```fsharp
let c1 = GpsCoordinate(25.0, 11.98)
let c2 = GpsCoordinate(25.0, 11.98)
c1 = c2 // true
```

We have only scratched the surface of what is possible with F# object programming. If you want to find out more about this topic (plus many other useful things), I highly recommend that you read [Stylish F#](<https://www.apress.com/us/book/9781484239995>) by [Kit Eason](<https://twitter.com/kitlovesfsharp>).

## Summary

In this chapter, we have had an introduction to object programming in F#. In particular, we have looked at scoping/visibility, encapsulation, interfaces/casting, and equality. The more you interact with the rest of the .NET ecosystem, the more you will potentially need to use object programming.

In the next chapter we will discover how F# handles recursion.