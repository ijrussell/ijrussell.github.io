# Introduction

This book is aimed at software developers with experience in other languages such as C# or Python and will give you a solid grounding in the essentials of functional-first programming in F#. 

This isn't a book primarily about Functional Programming. You are not going to learn about Category Theory, Lambda Calculus or even Monads. You will learn things about functional programming but only as a by-product of how we solve problems with F#. What you will learn is how F#'s functional-first approach to programming [empowers everyone to write succinct, robust, and performant code](<https://fsharp.org/>). You will learn a lot of new terminology as you read this book but only enough to understand the features of F# that allow us to solve real-world problems with clearly expressed and concise code. What you will take away most from this book is an understanding of how F# supports us by making it easier for us to write code the F# way than it is to be a functional programming purist.

## Why Choose F#?

For many historical reasons, functional programming has been viewed as difficult to learn and not very practical by most .NET developers. Functional programming is too often regarded as being too steeped in mathematics and academia to be of any use for writing the everyday line of business applications that most of us work on. This book is designed to help dispel those impressions. So the obvious question to ask is:

> What is F# good for?

Whilst it's great for many things like web programming, cloud programming, Machine Learning, AI, and Data Science, the most accurate answer comes from a long-standing F# Community member [Dave Thomas](<https://twitter.com/7sharp9_>):

>  "F# is good for programming"

F# is a general-purpose language for the .NET platform along with C# and VB.NET. F# has been bundled with Visual Studio, and now the .NET SDK, since 2010. The language has been quite stable since then. Code written in 2010 is not only still valid today but is also likely to be stylistically similar to current standards.

Why would you choose F# over another language? I noticed a question from [cancelerx](<https://twitter.com/ndy40>) on Twitter:

> "I would like to do a lightning talk on F#. My team is primarily PHP and Python devs. What should I showcase?  I want to light a spark."

My response was:

> The expressive type system, composition with the forward pipe operator, pattern matching, collections, and the REPL.

I've been thinking about my answer and whilst I still think that the list is a good one, I think that I should have said that it is how the various features work together that make F# so special, not the individual features themselves.

F# supports many paradigms, such as imperative and object-oriented programming but we will concentrate on how it encourages functional-first programming. My view of the choices F# has made in its language design can be summed up quite easily:

> I enjoy functional programming but I like programming in F# even more.

I'm not alone in feeling like this about F#. I hope that this book is a useful step on a similar journey to the one that I've been on.

## What does this book cover?

This book covers the core features and practices that a developer needs to know to work effectively on the types of F# Line of Business (LOB) applications we work on at Trustbit. It starts with the basics of type and function composition, pattern matching, and testing, and ends with a worked example of a simple website and API built using the wonderful [Giraffe](<https://github.com/giraffe-fsharp/Giraffe>) library.

The original source for this book is two series of blog posts that I wrote on the [Trustbit blog](<https://trustbit.tech/blog>) in 2020/21. They have been significantly re-written to clarify and expand the explanations, to ensure that the code works on VS Code plus the ionide F# extension, and to take advantage of some new F# features introduced in versions 5 and 6. 

During the course of reading this book, you are going to be introduced to a wide range of features that are the essentials of functional-first F#:

- F# Interactive (FSI)
- Algebraic Type System
  - Records
  - Discriminated Unions
  - Tuples
- Pattern Matching
  - Active Patterns
  - Guard Clauses
- Let bindings
- Immutability
- Functions
  - Pure
  - Higher Order
  - Currying
  - Partial Application
  - Pipelining
  - Signatures
  - Composition
- Unit
- Effects
  - Option
  - Result
  - Task/Async
- Computation Expressions
- Collections
  - Lists
  - Sequences
  - Arrays
  - Sets
- Recursion
- Object Programming
  - Class types
  - Interfaces
  - Object Expressions
- Exceptions
- Unit Tests
- Namespaces and Modules

You will also start the journey towards writing APIs and websites in Giraffe.

## Setting up your environment

1. Install F#. The best and simplest way to do this is to install the latest [.NET SDK](<https://dotnet.microsoft.com/download>) (6.0.x at time of publishing).
1. Install Visual Studio Code (VS Code).
1. Install the ionide F# extension for VS Code. You may need to reload the environment. The installation page for the extension will tell you.
1. Create a folder for the book and then folders for each book chapter to hold the code you will write.

### F# Interactive (FSI)

If you are coming to F# from C#, the F# Interactive window is similar to the C# Interactive Window. The primary difference is that rather than relying on line-by-line debugging, F# developers will use the FSI instead to validate code. I never debug any code with breakpoints when doing F# development. 

FSI appears in the VS Code Terminal window. In addition to the .NET Terminal, you will also have access to an F# Interactive window. You can type directly into FSI but we will send code from our files directly to FSI to compile and run. 

> We run code in FSI by selecting the code that we want to run and pressing ALT+ENTER. The code will be copied into FSI, compiled, and run. If you don't change that code, you can reuse it later as FSI keeps your compiled code in memory.

If you type directly into FSI, you will need to end your input with two semi-colons ```;;``` to tell FSI to execute your code.

### Script Files vs Code Files

F# supports two types of files: Script (.fsx) and Code (.fs). We will use both types throughout this book. The primary differences in VS Code are that only .fs files are compiled into the output .exe or .dll and .fsx files are primarily used to try things out with F# Interactive. 

F# code from any type of file can be run in F# Interactive by sending it using ALT + ENTER.

There is a command-line version of FSI that can be used to run script files but that is outside the scope of this book.

### Getting to know VS Code and Ionide

Whilst the VS Code and ionide environment is not as fully featured as the full IDEs (Visual Studio and Jet Brains Rider), they are not devoid of features. You don't actually need that many features to easily work on F# codebases, even large ones.

[Compositional-IT](<https://www.compositional-it.com/>) have released a number of videos on YouTube that give an introduction to [VS Code + F# shortcuts](<https://www.youtube.com/playlist?list=PLlzAi3ycg2x27lPUwp3Z-M2m44n681cED>).

[Ionide](<https://ionide.io/index.html>) is much more than just the F# extension for VS Code. Take some time to have a look at what they offer to help your F# experience.

## F# Software Foundation

You should join the [F# Software Foundation](<https://foundation.fsharp.org/join>); It's free. By joining, you will be able to access the dedicated Slack channel where there are excellent tracks for beginners and beyond.

## One Last Thing

This book contains lots of code. It will be tempting to copy and paste the code from the book but you may learn more by actually typing the code out as you go. I definitely learn more by doing it this way. 

The time for talking has finished. Now it is time to dive into some F# code. In chapter 1 we will work through a common business use case using F#.
