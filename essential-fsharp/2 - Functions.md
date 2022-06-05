# 2 - Functions

In this chapter, we are going to concentrate on functions. We will introduce the practice of composing pipelines of functions into functions that can perform more complicated tasks, commonly referred to as function composition. We will also see why understanding function signatures is very important in F# programming.

## Getting Started

Functions in F# are fundamentally very simple:

> Functions have one rule: They take one input and return one output.

In the previous chapter, we wrote a function that had two input parameters. Later in this chapter, we will discover why that fact and the one input parameter rule for functions are not actually conflicting. 

In this chapter, we are going to concentrate on a special subset of functions: Pure functions.

> A **Pure Function** has the following rules:
> - They are deterministic. Given the same input, they will always return the same output.
> - They generate no side effects.
> 
>**Side Effects** are activities such as talking to a database, sending email, handling user input, and random number generation or using the current date and time. It is highly unlikely that you will ever write a program that is free of side effects.

Functions that satisfy these rules have many benefits: They are easy to test, are cacheable, and are parallelizable.

You can do a lot in a single function but you can have better code reusability by combining smaller functions together: We call this Function Composition.

## Theory

The composition of two functions relies on the output of the first one matching the input of the next.

If we have two functions (f1 and f2) that look like this pseudocode:

```fsharp
// The ' implies a generic (any) type
f1 : 'a -> 'b
f2 : 'b -> 'c
```

As the output of f1 matches the input of f2, we can combine them together to create a new function (f3):

```fsharp
f3 : f1 >> f2 // 'a -> 'c
```

Treat the composition operator ```>>``` as a general-purpose operator for composing any two functions.

What happens if the output of f1 does not match the input of f2?:

```fsharp
f1 : 'a -> 'b
f2 : 'c -> 'd
where 'b is not the same type as 'c
```

To resolve this, we would create an adaptor function, or use an existing one, that we can plug in between f1 and f2:

```fsharp
f3 : 'b -> 'c
```

After plugging f3 in, we get a new function f4:

```fsharp
f4 : f1 >> f3 >> f2 // 'a -> 'd
```

Any number of functions can be composed together in this way.  

Let's look at a concrete example.

## In Practice

I've taken and slightly simplified some of the code from Jorge Fioranelli's excellent [F# Workshop](<http://www.fsharpworkshop.com/>). Once you've finished this chapter, I suggest that you download the workshop (it's free!) and complete it.

This example has a simple record type and three functions that we can compose together because the function signatures match up.

```fsharp
type Customer = {
    Id : int
    IsVip : bool
    Credit : decimal
}

// Customer -> (Customer * decimal)
let getPurchases customer = 
    let purchases = if customer.Id % 2 = 0 then 120M else 80M
    (customer, purchases) // Parentheses are optional

// (Customer * decimal) -> Customer 
let tryPromoteToVip purchases = 
    let (customer, amount) = purchases
    if amount > 100M then { customer with IsVip = true }
    else customer

// Customer -> Customer
let increaseCreditIfVip customer = 
    let increase = if customer.IsVip then 100M else 50M
    { customer with Credit = customer.Credit + increase }
```

There are a few things in this sample code that we haven't seen before. The *getPurchases* function returns a tuple. Tuples can be used for transferring small chunks of data around, generally as the output from a function. Notice the difference between the definition of the tuple ```(Customer * decimal)``` and the usage ```(customer, amount)``` when we decompose it into its constituent parts.

The other new feature is the copy-and-update record expression. This allows you to create a new record instance based on an existing record, usually with some modified data. Records and their properties are immutable by default.

These are some of the ways that F# supports to compose functions together:

```fsharp
// Composition operator
let upgradeCustomerComposed = // Customer -> Customer
    getPurchases >> tryPromoteToVip >> increaseCreditIfVip

// Nested
let upgradeCustomerNested customer = // Customer -> Customer
    increaseCreditIfVip(tryPromoteToVip(getPurchases customer))

// Procedural
let upgradeCustomerProcedural customer = // Customer -> Customer
    let customerWithPurchases = getPurchases customer
    let promotedCustomer = tryPromoteToVip customerWithPurchases
    let increasedCreditCustomer = increaseCreditIfVip promotedCustomer
    increasedCreditCustomer

// Forward pipe operator
let upgradeCustomerPiped customer = // Customer -> Customer
    customer 
    |> getPurchases 
    |> tryPromoteToVip 
    |> increaseCreditIfVip
```

They all have the same function signature and will produce the same result with the same input.

The *upgradeCustomerPiped* function uses the forward pipe operator (|>). It is equivalent to the *upgradeCustomerProcedural* function but without having to specify the intermediate values. The value from the line above gets passed down as the last input argument of the next function. The difference between the function composition operator ```>>``` and the forward pipe operator ```|>``` is that the function composition operator sits between two functions, whereas the forward pipe sits between a value and a function.

> Use the forward pipe operator (|>) as your default style.

It is quite easy to verify the output of the upgrade functions using FSI.

```fsharp
let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }

let assertVIP = 
    upgradeCustomerComposed customerVIP = {Id = 1; IsVip = true; Credit = 100.0M }
let assertSTDtoVIP = 
    upgradeCustomerComposed customerSTD = {Id = 2; IsVip = true; Credit = 200.0M }
let assertSTD = 
    upgradeCustomerComposed { customerSTD with Id = 3; Credit = 50.0M } = {Id = 3; IsVip = false; Credit = 100.0M }
```

Record types use **structural equality** which means that if the records look the same, they are equal.

> If two records contain the same data, they have structural equality through the equality operator (=).

Try replacing the *upgradeCustomerComposed* function with any of the other three functions in the asserts to confirm that they produce the same results in FSI.

## Unit

All functions must have one input and one output. To solve the problem of a function that doesn't require an input value or produce any output, F# has a special type called unit.

> Any function taking unit as the only input or returning it as the output is probably causing side effects

```fsharp
open System

// unit -> System.DateTime
let now () = DateTime.UtcNow 

// 'a -> unit
let log msg = 
    // Log message or similar task that doesn't return a value
    ()
```

Unit appears in the function signature as unit but in code, you will use ().

If you call the *now* function without the unit () parameter you will get a result similar to this:

```fsharp
val it : (unit -> DateTime) = <fun:it@30-4>
```

This looks really strange but what you actually get is a function as the output that takes unit and returns a DateTime value as shown by the signature. To execute the function, you need to supply the input parameter. This is an important F# feature called Partial Application, which we will explain later in this chapter.

If you forget to add the unit parameter '()' when you define the binding, you get a fixed value from the first time the line is executed:

```fsharp
let fixedNow = DateTime.UtcNow

let theTimeIs = fixedNow
```

Wait a few seconds and run the *theTimeIs* binding again and you'll notice that the date and time haven't changed from when the binding was first used.

> A function binding always has at least one parameter, even if it is just unit, otherwise, it is a value binding. You can tell if it's a function by checking whether the signature has an arrow (->) or not.

## Anonymous Functions

So far, we have only dealt with named functions but we can also create functions without names, commonly called anonymous functions.

```fsharp
// int -> int -> int
let add x y = x + y

// int -> int -> int
let add x = fun y -> x + y
```

Anytime you see the fun keyword, you are looking at an anonymous function.

They are useful in many ways. If we create a function that returns a random number between 0 and 99:

```fsharp
open System

// unit -> int
let rnd () =
    let rand = Random()
    rand.Next(100)

List.init 50 (fun _ -> rnd())
```

This will create a new instance of the Random class every time that it is called when the list is created.

The version below uses scoping rules to re-use the same instance of the Random class each time you call the *rnd* function:

```fsharp
open System

// unit -> int
let rnd =
    let rand = Random()
    fun () -> rand.Next(100)

List.init 50 (fun _ -> rnd())
```

## Multiple Parameters

At the start of the chapter, I stated that it is a rule that all functions must have one input and one output but in the last chapter, we created a function with multiple input parameters:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend = ... 
```

Let's write the function signature using an anonymous function:

```fsharp
Customer -> (decimal -> decimal)
let calculateTotal customer =
	fun spend -> (...)
```

The *calculateTotal* function takes a Customer as input and returns a function with the signature (decimal -> decimal) as output; That function takes a decimal as input and returns a decimal as output. The ability to automatically chain single input/single output functions together like this is called **Currying** after Haskell Curry, a US Mathematician. It allows you to write functions that appear to have multiple input arguments but are actually a chain of one input/one output functions. This seemingly odd behaviour that F# performs under the hood for us, opens the way to another very powerful functional concept called **Partial Application**.

## Partial Application (Part 1)

We will use the *calculateTotal* function to illustrate how partial application works. We can ignore the actual implementation of the function as we only care about the input parameters:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend = ...
```

If I only provide the first parameter, I get a new function as output with signature (Decimal -> Decimal):

```fsharp
// Decimal -> Decimal
let partial = calculateTotal john
```

If we then supply the final parameter, the original function completes and returns the expected output:

```fsharp
// Decimal
let complete = partial 100.0M
```

Run the code in FSI and look at the outputs from both lines of code.

You must add the input parameters in strict left to right order, in ones or multiples. Trying to add a parameter out of order will not compile if the types don't match:

```fsharp
let doesNotWork = calculateTotal 100M // Does not compile
```

You may wonder why you would want to do this but the partial application of function parameters has some very important uses such as allowing the forward pipe operator ```|>``` we used earlier in this chapter.

## The Forward Pipe Operator

As the forward pipe operator is something that we will be using a lot, it's worth spending some time investigating what is is and how it works under the covers. Hint: It relies on partial application.

The simple asserts that we wrote in the last chapter can be made more readable than this example:

```fsharp
let assertJohn = calculateTotal john 100.0M = 90.0M
```

We added a simple helper function and used that:

```fsharp
// 'a -> 'a -> bool (' means generic in F#)
let areEqual expected actual =
	expected = actual

let assertJohn = areEqual 90.0M (calculateTotal john 100.0M)
```

By using the forward pipe operator, we can improve readability:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = calculateTotal john 100.0M |> isEqualTo 90.0M
```

The name of the helper function has changed but the signature and result have not:

```fsharp
// 'a -> 'a -> bool
let isEqualTo expected actual =
    expected = actual
```

So what is the forward pipe operator? It is defined as part of FSharp.Core but it looks something like this:

```fsharp
// 'a -> ('a -> 'b) -> 'b
let (|>) v f = f v
```

Despite this being very short, there is a lot to unpack here. 

This looks like a function and it is but it's a special category called a custom operator. The name of the function is ```|>``` but the parentheses surrounding it are required because it can be used in two different ways which we will see shortly.

The operator takes a value of type *'a* as the first parameter and a partially applied function that takes an *'a* and returns a *'b* as the second parameter. This operator is a higher order function because it takes a function as a parameter. 

>  **Higher Order Functions** take one or more functions as parameters and/or return a function as its output.

When the value is applied to the partially applied function, the partially applied function has all of its parameters and can execute to return a value of type *'b*.

Let's use the operator in our assert in the defined prefix form:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = (|>) (calculateTotal john 100.0M) (isEqualTo 90.0M)
```

Normally we would skip this step as it looks worse that our original version but we can convert our operator from the current prefix form into the infix form like this:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = (calculateTotal john 100.0M) |> (isEqualTo 90.0M)
```

Notice that as an infix operator, you don't use the parentheses.

The last part is to remove the unnecessary parentheses:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = calculateTotal john 100.0M |> isEqualTo 90.0M
```

Understanding how the forward pipe operator works is important as we will be using it throughout the book.

## Partial Application (Part 2)

Now that we have a better understanding of partial application, let's have a look at a simple example. Create a function that takes the log level as a discriminated union and a string for the message:

```fsharp
type LogLevel = 
    | Error
    | Warning
    | Info

// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    printfn "[%A]: %s" level message
    ()
```

Every function in F# must return an output. In this case, we added an extra line to output *unit*. However, *printfn* returns *unit*, so we can remove the additional row:

```fsharp
// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    // or string interpolation printfn $"[%A{level}]: %s{message}"
    printfn "[%A]: %s" level message 
```

Both *printfn* and string interpolation have support for format specifiers. If the types supplied don't match the format specifier, it will result in a compile error rather than a runtime error. You don't need to supply format specifiers for string interpolation but then you don't get type safety from the format specifiers.

```fsharp
// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    printfn "[%A]: %s" level message
    
// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    printfn $"[%A{level}]: %s{message}"
    
// LogLevel -> 'a -> unit
let log (level:LogLevel) message = 
    printfn $"[{level}]: {message}"
```

To partially apply the *log* function, I'm going to define a new function that only takes the *log* function and its level argument but not the message:

```fsharp
let logError = log Error // string -> unit
```

The name *logError* is bound to a function that takes a *string* and returns *unit*. So now, we can use the *logError* function instead:

```fsharp
let m1 = log Error "Curried function" 

let m2 = logError "Partially Applied function"
```

As the return type is *unit*, you don't have to let bind the function to a value:

```fsharp
log Error "Curried function" 

logError "Partially Applied function"
```

Partial application is a very powerful concept that is only made possible because of curried input parameters. It is not possible with a single, tupled parameter as you need to supply the whole parameter at once:

```fsharp
type LogLevel = 
    | Error
    | Warning
    | Info

// (LogLevel * string) -> unit
let log (level:LogLevel, message:string) = 
    printfn "[%A]: %s" level message
```

There's so much more that we could cover here but we've already covered a lot in this chapter.

## Summary

In this chapter, we have covered:

- Pure Functions
- Anonymous Functions
- Function Composition
- Tuples
- Copy-and-update record expression
- Curried and Tupled parameters
- Currying and Partial Application

We have now covered the fundamental building blocks of programming in F#: Composition of types and functions, expressions, and immutability.

In the next chapter, we will investigate the handling of null and exceptions in F#.