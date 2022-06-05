# 9 - Single Case Discriminated Union

In this chapter, we are going to see how we can improve the readability of our code by increasing our usage of domain concepts and reducing our use of primitives.

## Setting Up

We are going to improve the code we wrote in Chapter 1. 

Create a new folder for the code in this chapter and open the new folder in VS Code.

Add a new file called 'code.fsx'.

## Solving the Problem

This is where we left the code from the first chapter in this book:

```fsharp
type RegisteredCustomer = {
	Id : string
}

type UnregisteredCustomer = {
    Id : string
}

type Customer =
    | EligibleRegistered of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer

// Customer -> decimal -> decimal
let calculateTotal customer spend = 
    let discount = 
        match customer with
        | EligibleRegistered _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

let john = EligibleRegistered { Id = "John" }
let mary = EligibleRegistered { Id = "Mary" }
let richard = Registered { Id = "Richard" }
let sarah = Guest { Id = "Sarah" }

let assertJohn = calculateTotal john 100.0M = 90.0M
let assertMary = calculateTotal mary 99.0M = 99.0M
let assertRichard = calculateTotal richard 100.0M = 100.0M
let assertSarah = calculateTotal sarah 100.0M = 100.0M
```

Copy the code into code.fsx.

It's nice but we still have some primitives where we should have domain concepts (Spend and Total). I would like the signature of the calculateTotal function to be (Customer -> Spend -> Total). The easiest way to achieve this is to use **Type Abbreviations**. Type abbreviations are simple aliases:

```fsharp
let TrueOrFalse = boolean
```

Anywhere we use a boolean, we could now use TrueOrFalse instead.

Add the following code below the Customer discriminated union:

```fsharp
type Spend = decimal
type Total = decimal
```

We have two approaches we can use with our new type abbreviations and get the type signature we need: Explicitly creating and implementing a function type which we did in Chapter 6 or using explicit types for the input/output parameter. Function type approach first:

```fsharp
type CalculateTotal = Customer -> Spend -> Total

//Customer -> Spend -> Total
let calculateTotal : CalculateTotal =
    fun customer spend ->
        let discount = 
            match customer with
            | EligibleRegistered _ when spend >= 100.0M -> spend * 0.1M
            | _ -> 0.0M
        spend - discount
```

Note the change in the way that input parameters are used with the function type.

The same function using explicit parameters:

```fsharp
// Customer -> Spend -> Total
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let discount = 
        match customer with
        | EligibleRegistered _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

Either approach gives us the signature we want. I have no preference for the style as both are useful to know, so we will use the explicit parameters for the rest of this post.

There is a potential problem with type abbreviations: As long as the underlying type matches, I can use anything for the input, not just Spend. 

There is nothing stopping you supplying either an invalid value or the wrong value to a parameter as shown in this example code:

```fsharp
type Latitude = decimal
type Longitude = decimal

type GpsCoordinate = Latitude * Longitude

// Latitude -90° to 90°
// Longitude -180° to 180°
let badGps : GpsCoordinate = (1000M, -345M)

let latitude = 46M
let longitude = 15M

// Swap latitude and longitude
let badGps2 : GpsCoordinate = (longitude, latitude)
```

We might be able to write tests to prevent this from happening but that is additional work. We want the type system to help us to write safer code. Thankfully, there are a number of ways that we can prevent these scenarios in F#. We will start with the Single Case Discriminated Union.

Let's define one for *Spend* by replacing the existing code with:

```fsharp
type Spend = Spend of decimal
```



You will notice that the calculateTotal function now has errors. We can fix that by deconstructing the Spend parameter value in the function using pattern matching:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let (Spend value) = spend
    let discount = 
        match customer with
        | EligibleRegistered _ when value >= 100.0M -> value * 0.1M
        | _ -> 0.0M
    value - discount
```

If the type had been defined as `type Spend = xSpend of decimal`, the deconstructor would have been `xSpend value`.

We need to change the asserts to use the new type constructor:

```fsharp
let assertJohn = calculateTotal john (Spend 100.0M) = 90.0M
let assertMary = calculateTotal mary (Spend 99.0M) = 99.0M
let assertRichard = calculateTotal richard (Spend 100.0M) = 100.0M
let assertSarah = calculateTotal sarah (Spend 100.0M) = 100.0M
```

The `calculateTotal` function is now safer but less readable. Thankfully, there is a simple fix for this, we can deconstruct the value in the function parameter directly:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer (Spend spend) = 
    let discount = 
        match customer with
        | EligibleRegistered _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

If you replace all of your primitives and type abbreviations with single case discriminated unions, you cannot supply the wrong parameter as the compiler will stop you. Of course it doesn't stop the determined from abusing our code but it is another hurdle they must overcome.

The next improvement is to restrict the range of values that the Spend type can accept since very few domain values will be unbounded. We will restrict Spend to between 0.0M and 1000.0M. To support this, we are going to add a ValidationError type and prevent the direct use of the Spend constructor:

```fsharp
type ValidationError = 
    | InputOutOfRange of string

type Spend = private Spend of decimal
    with
        member this.Value = this |> fun (Spend value) -> value
        static member Create input = 
            if input >= 0.0M && input <= 1000.0M then
                Ok (Spend input)
            else
                Error (InputOutOfRange "You can only spend between 0 and 1000")
```

The use of the private accessor prevents code outside this module from directly accessing the type constructor. This means that to create an instance of Spend, we need to use the Spend.Create factory function defined on the type.

To extract the value, we use the Value property defined on the instance.

We need to make some changes to get the code to compile. Firstly we change the calculateTotal function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let discount = 
        match customer with
        | EligibleRegistered _ when spend.Value >= 100.0M -> spend.Value * 0.1M
        | _ -> 0.0M
    spend.Value - discount
```

We also need to fix the asserts. To do this, we are going to add a new helper function to simplify the code:

```fsharp
let doCalculateTotal name amount =
    match Spend.Create amount with
    | Ok spend -> calculateTotal name spend
    | Error ex -> failwith (sprintf "%A" ex)

let assertJohn = doCalculateTotal john 100.0M = 90.0M
let assertMary = doCalculateTotal mary 99.0M = 99.0M
let assertRichard = doCalculateTotal richard 100.0M = 100.0M
let assertSarah = doCalculateTotal sarah 100.0M = 100.0M
```

We have done some extra work to the Spend type but by doing so we have ensured that an instance of it can never be invalid.

As a piece of homework, think about how you would use the features we have covered in this chapter on the code from the last chapter.

The final change that we can make is to move the discount rate from the calculateTotal function to be with the Customer type definition. The primary reason for doing this would be if we needed to use the discount in another function:

```fsharp
type Customer =
    | EligibleRegistered of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
    with 
        member this.Discount =
            match this with
            | EligibleRegistered _ -> 0.1M
            | _ -> 0.0M
```

This also allows us to simplify the calculateTotal function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let discount = if spend.Value >= 100.0M then spend.Value * customer.Discount else 0.0M
    spend.Value - discount
```

Whilst this looks nice, it has broken the link between the customer type and the spend. Remember, the rule is a 10% discount if an eligible customer spends 100.0 or more. Let's have another go:

```fsharp
type Customer =
    | EligibleRegistered of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
    with 
        member this.CalculateDiscountPercentage(spend:Spend) =
            match this with
            | EligibleRegistered _ -> 
                if spend.Value >= 100.0M then 0.1M else 0.0M
            | _ -> 0.0M
```

**WARNING** You will probably need to move the Spend type to before the Customer type declaration in the file.

We now need to modify our calculateTotal function to use our new function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let discount = spend.Value * customer.CalculateDiscountPercentage spend
    spend.Value - discount
```

To run this code, you will need to load all of it back into FSI using ```ALT+ENTER``` as we have changed quite a lot of code. Your tests should still pass.

A final change would be to simplify the calculateTotal function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    spend.Value * (1.0M - customer.CalculateDiscountPercentage spend)
```

We have covered quite a few options in this chapter and all are valid. Which direction you go in is up to you.

## Final Code

```fsharp
type RegisteredCustomer = {
    Id : string
}

type UnregisteredCustomer = {
    Id : string
}

type ValidationError = 
    | InputOutOfRange of string

type Spend = private Spend of decimal
    with
        member this.Value = this |> fun (Spend value) -> value
        static member Create input = 
            if input >= 0.0M && input <= 1000.0M then
                Ok (Spend input)
            else
                Error (InputOutOfRange "You can only spend between 0 and 1000")

type Total = decimal

type Customer =
    | EligibleRegistered of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
    with 
        member this.CalculateDiscountPercentage(spend:Spend) =
            match this with
            | EligibleRegistered _ -> 
                if spend.Value >= 100.0M then 0.1M else 0.0M
            | _ -> 0.0M

let calculateTotal (customer:Customer) (spend:Spend) : Total =
    spend.Value * (1.0M - customer.CalculateDiscountPercentage spend)

let john = EligibleRegistered { Id = "John" }
let mary = EligibleRegistered { Id = "Mary" }
let richard = Registered { Id = "Richard" }
let sarah = Guest { Id = "Sarah" }

let doCalculateTotal name amount =
    match Spend.create amount with
    | Ok spend -> calculateTotal name spend
    | Error ex -> failwith (sprintf "%A" ex)

let assertJohn = doCalculateTotal john 100.0M = 90.0M
let assertMary = doCalculateTotal mary 99.0M = 99.0M
let assertRichard = doCalculateTotal richard 100.0M = 100.0M
let assertSarah = doCalculateTotal sarah 100.0M = 100.0M
```

## Record Type

You can also use a record type to serve the same purpose as the single case discriminated union:

```fsharp
type Spend = private { Spend : decimal }
    with 
        member this.Value = this.Spend
        static member Create input = 
            if input >= 0.0M && input <= 1000.0M then
                Ok { Spend = input }
            else
                Error (InputOutOfRange "You can only spend between 0 and 1000")
```

## Summary

In this post we have learnt about single case discriminated unions. They allows us to restrict the data that a parameter can accept compared to raw primitives. We have also seen that we can extend them by adding helper functions and properties.

In the next chapter, we will look at object programming in F#.