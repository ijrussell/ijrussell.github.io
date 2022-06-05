# 13 - Introduction to Web Programming with Giraffe

In this chapter, we will start investigating web programming in F#.

We are going to use [Giraffe](<https://github.com/giraffe-fsharp/Giraffe>). Giraffe is "an F# micro web framework for building rich web applications" and was created by [Dustin Moris Gorski](<https://twitter.com/dustinmoris>). It is a thin functional wrapper around ASP.NET Core. Both APIs and web pages are supported by Giraffe and we will cover both in the last three chapters of this book.

Giraffe comes with excellent [documentation](<https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md>). I highly recommend that you spend some time looking through it to see what features Giraffe offers.

If you want something more opinionated or want to use F# everywhere including to create JavaScript, have a look at [Saturn](<https://saturnframework.org/>) and the [Safe Stack](<https://safe-stack.github.io/>).

## Getting Started

Create a new folder called *GiraffeExample* and open it in VS Code.

Using the Terminal in VS Code, type in the following command to create an empty ASP.NET Core web app project:

```plaintext
dotnet new web -lang F#
```

Add the following NuGet packages from the terminal:

```plaintext
dotnet add package Giraffe
dotnet add package Giraffe.ViewEngine
```

Open *Startup.fs*. If you've done any ASP.NET Core development before, it will look very familiar, even though it is in F#:

```fsharp
type Startup() =

    // This method gets called by the runtime. Use this method to add services to the container.
    // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
    member _.ConfigureServices(services: IServiceCollection) =
        ()

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
        if env.IsDevelopment() then
            app.UseDeveloperExceptionPage() |> ignore

        app.UseRouting()
           .UseEndpoints(fun endpoints ->
                endpoints.MapGet("/", fun context ->
                    context.Response.WriteAsync("Hello World!")) |> ignore
            ) |> ignore
```

We need to make some changes to plug in Giraffe. Add the following import declarations below the ones already there:

```fsharp
open Giraffe
open Giraffe.ViewEngine
open FSharp.Control.Tasks
```

We need to register the default Giraffe dependencies in the *ConfigureServices* function:

```fsharp
member _.ConfigureServices(services: IServiceCollection) =
	services.AddGiraffe() |> ignore
```

To add Giraffe to the ASP.NET Core pipeline, we need to replace the ```app.UseRouting``` code in the *Configure* function:

```fsharp
member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
	if env.IsDevelopment() then
		app.UseDeveloperExceptionPage() |> ignore
	app.UseGiraffe (route "/" >=> text "Hello World!")
```

## Running the Sample Code

In the Terminal, type the following to run the project:

```plaintext
dotnet run
```

Go to your browser and type in the following Url:

<https://localhost:5001>

You should see "Hello World!" in your browser window.

To shut the running website down, press ```CTRL+C``` in the Terminal window where the code is running. You should be back to the command prompt.

Before we add new functionality, we need to understand what is going on in ```(route "/" >=> text "Hello World!")```.

## HttpHandlers and Combinators

Let's take a look at the Giraffe sourcecode for *route* and *text*. They are both functions that return HttpHandler. The route function takes the path as input and the text function takes the message as input:

```fsharp
let route (path : string) : HttpHandler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        if (SubRouting.getNextPartOfPath ctx).Equals path
        then next ctx
        else skipPipeline

let text (str : string) : HttpHandler =
    let bytes = Encoding.UTF8.GetBytes str
    fun (_ : HttpFunc) (ctx : HttpContext) ->
        ctx.SetContentType "text/plain; charset=utf-8"
        ctx.WriteBytesAsync bytes
```

The *route* HttpHandler checks the input path against the one from the route parameter and if they match, it passes control to the next handler in that pipeline otherwise it exits this pipeline and hands control back to routing to try the next route.

The *text* HttpHandler simply writes to the response and sets the response content type.

Pretty much everything you are going to see in Giraffe for routing, middleware etc is a HttpHandler.

The ```>=>``` operator represents a combinator. It looks and sounds scary but, like most things in F#, it looks far worse than it is! It is a custom operator to use instead of the compose function. It is defined like this:

```fsharp
let (>=>) = compose
```

I'm not going to show the code for the compose function but it takes two HttpHandlers as input and returns a HttpHandler as output so that the result can also be composed with other HttpHandlers in a pipeline.

The compose operator ```(>=>)``` is a custom operator that is used as a type abbreviation for the compose function.

We can use the compose function in one of three ways:

```fsharp
// Using the compose function directly
let webApp = compose (route "/") (text "Hello World!")

// Using the custom operator directly
let webApp = (>=>) (route "/") (text "Hello World!")

// Using the infix version of the custom operator
let webApp = route "/" >=> text "Hello World!"
```

I don't normally like custom operators because they can make it harder to quickly grasp what the code is doing but in this case, the infix version of the custom operator makes the code easier to read and use.

## Next Steps

It's a bit limiting only having one route in your app which we do at the moment, so we'll start to rectify that by moving the current route out to a new module. Create a new module called *Handlers* between the import declarations and the *Startup* class type definition and move the ```webApp``` binding into it:

```fsharp
module Handlers =

    let webApp = route "/" >=> text "Hello World!"
```

Replace the route used for ```app.UseGiraffe``` with:

```fsharp
member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
	if env.IsDevelopment() then
		app.UseDeveloperExceptionPage() |> ignore
	app.UseGiraffe Handlers.webApp
```

Run the app again using dotnet run to ensure it is still working as expected.

There are many other HttpHandlers built into Giraffe along with another combinator called *choose* that allows us to define a list of potential routes:

```fsharp
    let webApp = 
        choose [
            route "/" >=> text "Hello World!"
            setStatusCode 404 >=> text "Not Found"
        ]
```

The routing will choose the first route that the request can complete. 

We should also restrict the HTTP verb to GET requests only. HTTP verbs in Giraffe are HttpHandlers, so they can be composed in the pipeline. Anything other than 'GET /' is going to result in a 404 error:

```fsharp
    let webApp = 
        choose [
            GET >=> route "/" >=> text "Hello World!"
            RequestErrors.NOT_FOUND "Not Found"
        ]
```

The setStatusCode 404 can be replaced by a built-in helper to simplify it.

## Creating an API

We are going to add a new route '/api' and respond with some JSON using another built-in Giraffe HttpHandler, *json*:

```fsharp
let webApp = 
    choose [
        GET >=> route "/" >=> text "Hello World!"
        GET >=> route "/api" >=> json {| Response = "Hello world!!" |}
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

The payload for the JSON response on the '/api' route is an anonymous record, designated by the \{|..|} structure. You define the structure and data of the anonymous record in place rather than defining a type and creating an instance.

Now run the code and try the following Url in the browser. You should see a JSON response:

<https://localhost:5001/api>

## Creating a Custom HttpHandler

There are lots of built-in HttpHandlers but it is simple to create your own because they are just functions. Let's add a new route for the API that takes a string value in the request querystring that is handled by a custom handler that we haven't written yet:

```fsharp
let webApp = 
    choose [
        GET >=> route "/" >=> text "Hello World!"
        GET >=> route "/api" >=> json {| Response = "Hello world!!" |}
        GET >=> routef "/api/%s" sayHelloNameHandler
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

Note that the querystring item is automatically bound to the handler function if it has a matching input parameter of the same type in the handler, so the sayHelloNameHandler needs to take a string input parameter. Create the custom handler in the Handlers module above the webApp binding:

```fsharp
let sayHelloNameHandler (name:string) : HttpHandler =
    fun (next:HttpFunc) (ctx:HttpContext) ->
        task {
            let msg = $"Hello {name}, how are you?"
            return! json {| Response = msg |} next ctx
        }
```

This function could be rewritten as the following but it is convention to use the style above:

```fsharp
let sayHelloNameHandler (name:string) (next:HttpFunc) (ctx:HttpContext) : HttpHandler =
    task {
        let msg = $"Hello {name}, how are you?"
        return! json {| Response = msg |} next ctx
    }
```

The task \{...} is a computation Expression of type System.Threading.Tasks.Task<'T> and is the equivalent to async/await in C#. F# has another asynchronous feature called async but we won't use that as Giraffe has chosen to work directly with Task instead.

You will see a lot more of the task computation expression in the next chapter as we expand the API side of our website.

Now run the code and try the following Url in the browser. If you replace the placeholder '{yourname}' with your name, you should see a JSON response asking how you are:

<https://localhost:5001/api/{yourname}>

Not only is Giraffe excellent as an API, it is equally suited to server-side rendered web pages using the Giraffe View Engine.

## Creating a View

The Giraffe View Engine is a DSL that generates HTML. This is what a simple page looks like:

```fsharp
let indexView =
    html [] [
        head [] [
            title [] [ str "Giraffe Example" ]
        ]
        body [] [
            h1 [] [ str "I |> F#" ]
            p [ _class "some-css-class"; _id "someId" ] [
                str "Hello World from the Giraffe View Engine"
            ]
        ]
    ]
```

Each element of html has the following structure:

```fsharp
// element [attributes] [sub-elements/data]
html [] [] 
```

Each element has two supporting lists: The first is for attributes and the second for sub-elements or data. You may wonder why we need a DSL to generate HTML but it makes sense as it helps prevent badly formed structures. All of the features you need like master pages, partial views and model binding are included. We will have a deeper look at views in the final chapter of this book.

Add the indexView value code above the webApp handler and replace the root route with the following:

```fsharp
GET >=> route "/" >=> htmlView indexView
```

Run the web app and you will see the text from the indexView in your browser.

## Tidying up Routing

One last thing to do and that is to simplify the routing as we have quite a lot of duplicate code:

```fsharp
let webApp = 
    choose [
        GET >=> route "/" >=> htmlView indexView
        GET >=> route "/api" >=> json {| Response = "Hello world!!" |}
        GET >=> routef "/api/%s" sayHelloNameHandler
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

The first thing we do is extract the GET HttpHandler and add a choose combinator:

```fsharp
let webApp = 
    choose [
        GET >=> choose [
            route "/" >=> htmlView indexView
            route "/api" >=> json {| Response = "Hello world!!" |}
            routef "/api/%s" sayHelloNameHandler
        ]
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

We can extract routes to another HttpHandler like we do with the '/api' route here:

```fsharp
let apiRoutes : HttpHandler = 
    choose [
        route "" >=> json {| Response = "Hello world!!" |}
        routef "/%s" sayHelloNameHandler
    ]

let webApp =
    choose [
        GET >=> choose [
            route "/" >=> htmlView indexView
            subRoute "/api" apiRoutes
        ]
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

Note the change from 'route' to 'subRoute' if we extract the '/api' route to its own handler.

Run the app and try the routes out in your browser.

## Reviewing the Code

The finished code for this chapter is:

```fsharp
namespace GiraffeExample

open System
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Hosting
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Giraffe
open Giraffe.ViewEngine
open FSharp.Control.Tasks

module Handlers =

    let indexView =
        html [] [
            head [] [
                title [] [ str "Giraffe Example" ]
            ]
            body [] [
                h1 [] [ str "I |> F#" ]
                p [ _class "some-css-class"; _id "someId" ] [
                    str "Hello World from the Giraffe View Engine"
                ]
            ]
        ]

    let sayHelloNameHandler (name:string) : HttpHandler =
        fun (next:HttpFunc) (ctx:HttpContext) ->
            task {
                let msg = $"Hello {name}, how are you?"
                return! json {| Response = msg |} next ctx
            }
    
    let apiRoutes = 
        choose [
            route "" >=> json {| Response = "Hello world!!" |}
            routef "/%s" sayHelloNameHandler
        ]

    let webApp =
        choose [
            GET >=> choose [
                route "/" >=> htmlView indexView
                subRoute "/api" apiRoutes
            ]
            RequestErrors.NOT_FOUND "Not Found"
        ]

type Startup() =

    // This method gets called by the runtime. Use this method to add services to the container.
    // For more information on how to configure your application, visit https://go.microsoft.com/fwlink/?LinkID=398940
    member _.ConfigureServices(services: IServiceCollection) =
        services.AddGiraffe() |> ignore

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
        if env.IsDevelopment() then
            app.UseDeveloperExceptionPage() |> ignore
        app.UseGiraffe Handlers.webApp
```

## Summary

We have only scratched the surface of what is possible with Giraffe. In this chapter we had an introduction to using Giraffe to build an API and how to use the Giraffe View Engine to create HTML pages. We also learnt about the importance of HttpHandlers in Giraffe routing and we're introduced to a number of the built-in ones Giraffe offers and we have seen that we can create our own handlers.

In the next chapter we will expand the API side of the application.