# 14 - Creating an API with Giraffe

In this chapter, we'll be creating a simple API with Giraffe. We are going to add new functionality to the project we created in the last chapter.

## Getting Started

You will need a tool to run HTTP calls (GET, POST, PUT, and DELETE). I use [Postman](<https://www.postman.com/>) but any tool including those available in VS Code will work.

Open the code from the last chapter in VS Code.

## Our Task

We are going to create a simple API that we can view, create, update and delete Todo items.

## Sample Data

Rather than work against a real data store, we are going to create a simple store with a dictionary and use that in our handlers.

Create a new file above Startup.fs called TodoStore.fs and add the following code to it:

```fsharp
module GiraffeExample.TodoStore

open System
open System.Collections.Concurrent

type TodoId = Guid

type NewTodo = {
    Description: string
}

type Todo = {
    Id: TodoId
    Description: string
    Created: DateTime
    IsCompleted: bool
}

type TodoStore() =
    let data = ConcurrentDictionary<TodoId, Todo>()

    member _.Create todo = data.TryAdd(todo.Id, todo)
    member _.Update todo = data.TryUpdate(todo.Id, todo, data.[todo.Id])
    member _.Delete id = data.TryRemove id
    member _.Get id = data.[id]
    member _.GetAll () = data.ToArray()
```

TodoStore is a simple class type that wraps a concurrent dictionary that we can use to test our API out with. It will not persist between runs. 

Add a reference to the to the import declarations in Startup.fs:

```fsharp
open GiraffeExample.TodoStore
```

To be able to use the TodoStore, we need to make a change to the configureServices function in Startup.fs:

```fsharp
let configureServices (services : IServiceCollection) =
    services
        .AddGiraffe()
        .AddSingleton<TodoStore>(TodoStore()) |> ignore
```

If you're thinking that this looks like dependency injection, you would be correct; We are using the one provided by ASP.NET Core. We add the TodoStore as a singleton as we only want one instance to exist. 

## Routes

We saw in the last chapter that Giraffe uses individual route handlers, so we need to think about how to add our new routes. The routes we need to add are:

```plaintext
GET /api/todo // Get a list of todos
GET /api/todo/id // Get one todo
POST /api/todo // Create a todo
PUT /api/todo/id // Update a todo
DELETE /api/todo/id // Delete a todo
```

Let's create a new handler with the correct HTTP verbs just above the webApp function:

```fsharp
let apiTodoRoutes : HttpHandler =
    choose [
        GET >=> choose [
            routef "/%O" viewTodoHandler
            route "" >=> viewTodosHandler
        ]
        POST >=> route "" >=> createTodoHandler
        PUT >=> routef "/%O" updateTodoHandler
        DELETE >=> routef "/%O" deleteTodoHandler
    ]
```

We will create the missing handlers after we have plugged the new handler into our webApp routing handler as a subroute:

```fsharp
let webApp =
    choose [
        subRoute "/api/todo" Todos.apiTodoRoutes
        GET >=> choose [
            route "/" >=> htmlView indexView
            subRoute "/api" apiRoutes
        ]
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

There are lots of ways of arranging the routes. It's up to you to find an efficient approach. I like this style with all of the route strings in one function. This makes them easy to change.

Next we have to implement the new handlers.

## Handlers

Rather than create the new handlers in Startup.fs, we are going to create them in a new file. We will also move the apiTodoRoutes handler there as well.

Create a new file between Startup.fs and TodoStore.fs called Todos.fs. Copy the following code into the new file:

```fsharp
namespace GiraffeExample

open System
open System.Collections.Generic
open Microsoft.AspNetCore.Http
open Giraffe
open FSharp.Control.Tasks
open GiraffeExample.TodoStore

module Todos =
```

Move (Cut & Paste) the apiTodoRoutes handler function to the new Todos module.

You will need to fix the error in the webApp binding by adding 'Todos.' to apiTodoRoutes:

```fsharp
subRoute "/todo" Todos.apiTodoRoutes
```

Now we can concentrate on adding the handlers for our new routes to the Todos module above the apiTodoRoutes function. We'll start with the two Get requests:

```fsharp
let viewTodosHandler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let store = ctx.GetService<TodoStore>()
            let todos = store.GetAll()
            return! json todos next ctx
        }

let viewTodoHandler (id:Guid) =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let store = ctx.GetService<TodoStore>()
            let todo = store.Get(id)
            return! json todo next ctx
        }
```

We are using the HttpContext (ctx) to gain access to the TodoStore instance we set up earlier.

Let's add the handlers for Put and Post:

```fsharp
let createTodoHandler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let! newTodo = ctx.BindJsonAsync<NewTodo>()
            let store = ctx.GetService<TodoStore>()
            let created = 
                store.Create({ 
                    Id = Guid.NewGuid()
                    Description = newTodo.Description
                    Created = DateTime.UtcNow
                    IsCompleted = false })
            return! json created next ctx
        }

let updateTodoHandler (id:Guid) =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let! todo = ctx.BindJsonAsync<Todo>()
            let store = ctx.GetService<TodoStore>()
            let created = store.Update(todo)
            return! json created next ctx
        }
```

The most interesting thing here is that we use a built-in Giraffe function to gain strongly-typed access to the request body passed into the handler via the HttpContext.

Finally, we handle the Delete route:

```fsharp
let deleteTodoHandler (id:Guid) =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let store = ctx.GetService<TodoStore>()
            let existing = store.Get(id)
            let deleted = store.Delete(KeyValuePair<TodoId, Todo>(id, existing))
            return! json deleted next ctx
        }
```

We should now be able to use the new Todo API.

## Using the API

Run the app and use a tool like Postman to work with the API.

To get a list of all Todos, we call 'GET /api/todo'. This should return an empty json array because we don't have any todos in our store currently.

We create a Todo by calling 'PUT /api/todo' with a json request body like this:

```json
{
    "Description": "Finish blog post"
}
```

You will receive a response of true. If you now call the list again, you will receive a JSON response like this:

```json
[
    {
        "key": "ff5a1d35-4573-463d-b9fa-6402202ab411",
        "value": {
            "id": "ff5a1d35-4573-463d-b9fa-6402202ab411",
            "description": "Finish blog post",
            "created": "2021-03-12T13:47:39.3564455Z",
            "isCompleted": false
        }
    }
]
```

I'll leave the other routes for you to investigate.

## Summary

We have only scratched the surface of what is possible with Giraffe for creating APIs such as content negotiation. I highly recommend reading the [Giraffe documentation](<https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md>) to get a full picture of what is possible.

In the final chapter of the book, we will dive deeper into HTML views with the Giraffe View Engine.