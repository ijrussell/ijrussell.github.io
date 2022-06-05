# 15 - Creating Web Pages with Giraffe

In the last chapter, we created a simple API for managing a Todo list. In this post, we are going to start our journey into HTML views with the Giraffe View Engine.

If you haven't already done so, read the previous two chapters on Giraffe.

## Getting Started

We are going to continue to make changes to the project we updated in the last chapter.

## Our Task

We are going to create a simple HTML view of a Todo list and populate it with dummy data from the server.

Rather than rely on my HTML/CSS skills, we are going to start with a pre-built sample: The ToDo list example from w3schools:

<https://www.w3schools.com/howto/howto_js_todolist.asp>

## Configuration

Add a new folder to the project called WebRoot and add a couple of files to the folder: main.js and main.css.

Open the '.fsproj' file and add the following snippet:

```xml
<ItemGroup>
  <Content Include="WebRoot\**\*">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

The pattern used allows you to add subfolders but if you do that, you must remember to fix your paths in the views.

We have to tell the app to serve static files if requested. We add some new configuration to the Configure function in Startup.fs:

```fsharp
member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
    if env.IsDevelopment() then
        app.UseDeveloperExceptionPage() |> ignore
    app.UseStaticFiles() |> ignore
    app.UseGiraffe Handlers.webApp
```

The final step to enabling these files to be used is to edit the createHostBuilder function in Program.fs to tell ASP.NET Core to use the new folder to serve static files like JavaScript, CSS, and images:

```fsharp
let createHostBuilder args =
    let contentRoot = Directory.GetCurrentDirectory()
    let webRoot = Path.Combine(contentRoot, "WebRoot")
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(fun webBuilder ->
            webBuilder
                .UseStartup<Startup>()
                .UseWebRoot(webRoot) |> ignore
        )
```

We have added a couple of lines to tell the code where the WebRoot folder is and have passed that path to a built-in helper function called UseWebRoot.

That's the end of the configuration but we need to copy the code from the w3schools site for the CSS and JavaScript into our files.

First the CSS:

```css
/* Include the padding and border in an element's total width and height */
* {
  box-sizing: border-box;
}

/* Remove margins and padding from the list */
ul {
  margin: 0;
  padding: 0;
}

/* Style the list items */
ul li {
  cursor: pointer;
  position: relative;
  padding: 12px 8px 12px 40px;
  background: #eee;
  font-size: 18px;
  transition: 0.2s;

  /* make the list items unselectable */
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

/* Set all odd list items to a different color (zebra-stripes) */
ul li:nth-child(odd) {
  background: #f9f9f9;
}

/* Darker background-color on hover */
ul li:hover {
  background: #ddd;
}

/* When clicked on, add a background color and strike out text */
ul li.checked {
  background: #888;
  color: #fff;
  text-decoration: line-through;
}

/* Add a "checked" mark when clicked on */
ul li.checked::before {
  content: '';
  position: absolute;
  border-color: #fff;
  border-style: solid;
  border-width: 0 2px 2px 0;
  top: 10px;
  left: 16px;
  transform: rotate(45deg);
  height: 15px;
  width: 7px;
}

/* Style the close button */
.close {
  position: absolute;
  right: 0;
  top: 0;
  padding: 12px 16px 12px 16px;
}

.close:hover {
  background-color: #f44336;
  color: white;
}

/* Style the header */
.header {
  background-color: #f44336;
  padding: 30px 40px;
  color: white;
  text-align: center;
}

/* Clear floats after the header */
.header:after {
  content: "";
  display: table;
  clear: both;
}

/* Style the input */
input {
  margin: 0;
  border: none;
  border-radius: 0;
  width: 75%;
  padding: 10px;
  float: left;
  font-size: 16px;
}

/* Style the "Add" button */
.addBtn {
  padding: 10px;
  width: 25%;
  background: #d9d9d9;
  color: #555;
  float: left;
  text-align: center;
  font-size: 16px;
  cursor: pointer;
  transition: 0.3s;
  border-radius: 0;
}

.addBtn:hover {
  background-color: #bbb;
}
```

Now the JavaScript:

```javascript
// Create a "close" button and append it to each list item
var myNodelist = document.getElementsByTagName("LI");
var i;
for (i = 0; i < myNodelist.length; i++) {
  var span = document.createElement("SPAN");
  var txt = document.createTextNode("\u00D7");
  span.className = "close";
  span.appendChild(txt);
  myNodelist[i].appendChild(span);
}

// Click on a close button to hide the current list item
var close = document.getElementsByClassName("close");
var i;
for (i = 0; i < close.length; i++) {
  close[i].onclick = function() {
    var div = this.parentElement;
    div.style.display = "none";
  }
}

// Add a "checked" symbol when clicking on a list item
var list = document.querySelector('ul');
list.addEventListener('click', function(ev) {
  if (ev.target.tagName === 'LI') {
    ev.target.classList.toggle('checked');
  }
}, false);

// Create a new list item when clicking on the "Add" button
function newElement() {
  var li = document.createElement("li");
  var inputValue = document.getElementById("myInput").value;
  var t = document.createTextNode(inputValue);
  li.appendChild(t);
  if (inputValue === '') {
    alert("You must write something!");
  } else {
    document.getElementById("myUL").appendChild(li);
  }
  document.getElementById("myInput").value = "";

  var span = document.createElement("SPAN");
  var txt = document.createTextNode("\u00D7");
  span.className = "close";
  span.appendChild(txt);
  li.appendChild(span);

  for (i = 0; i < close.length; i++) {
    close[i].onclick = function() {
      var div = this.parentElement;
      div.style.display = "none";
    }
  }
}
```

## Converting HTML To Giraffe View Engine

We are going to create a master page and pass the title and content into it.

Let's start with a basic HTML page:

```html
<html>
    <head>
        <title>My Title</title>
        <link rel="stylesheet" href="main.css" />
    </head>
    <body />
</html>
```

We will create the function above indexView in the Handlers module in Startup.fs. If we convert the HTML to Giraffe View Engine format, we get:

```fsharp
// string -> XmlNode list -> XmlNode
let createMasterPage msg content =
    html [] [
        head [] [
            title [] [ str msg ]
            link [ _rel "stylesheet"; _href "main.css" ]
        ]
        body [] content
    ]
```

Most tags have two lists, one for styling and one for content. Some tags, like *input*, only take the style list.

To create the original HTML view using our new master page, we would change indexView to:

```fsharp
let indexView =
    [
        h1 [] [ str "I |> F#" ]
        p [ _class "some-css-class"; _id "someId" ] [
            str "Hello World from the Giraffe View Engine"
        ]
    ]
    |> createMasterPage "Giraffe Example"
```

This approach is very different to most view engines which rely on search and replace but are primarily still HTML. The primary advantage of the Giraffe View Engine approach is that you get full type safety when creating and generating views and can use the full power of the F# language.

Run the website to prove that you haven't broken anything.

The Todo HTML from the w3schools source below has to be converted into Giraffe View Engine format:

```html
<div id="myDIV" class="header">
  <h2>My To Do List</h2>
  <input type="text" id="myInput" placeholder="Title...">
  <span onclick="newElement()" class="addBtn">Add</span>
</div>

<ul id="myUL">
  <li>Hit the gym</li>
  <li class="checked">Pay bills</li>
  <li>Meet George</li>
  <li>Buy eggs</li>
  <li>Read a book</li>
  <li>Organise office</li>
</ul>
```

Create a new function below the createMasterPage function to generate our todoView value binding:

```fsharp
let todoView =
    [
        div [ _id "myDIV"; _class "header" ] [
            h2 [] [ str "My To Do List" ]
            input [ _type "text"; _id "myInput"; _placeholder "Title..." ]
            span [ _class "addBtn"; _onclick "newElement()" ] [ str "Add" ]
        ]
        ul [ _id "myUL" ] [
            li [] [ str "Hit the gym" ]
            li [ _class "checked" ] [ str "Pay bills" ]
            li [] [ str "Meet George" ]
            li [] [ str "Buy eggs" ]
            li [] [ str "Read a book" ]
            li [ _class "checked" ] [ str "Organise office" ]
        ]
        script [ _src "main.js"; _type "text/javascript" ] []
    ]
    |> createMasterPage "My ToDo App" 
```

We are nearly ready to run; We only need to tell the router about our new view handler. Change the root '/' route in webApp to:

```fsharp
GET >=> route "/" >=> htmlView todoView
```

If you now run the app using dotnet run in the terminal. Point your browser at <https://localhost:5001> and you will see the Todo app running.

## Next Stage

Rather than hard code the list of Todos into the view, we can load it in on the fly by creating a list of items. Create the following above the todoView:

```fsharp
let todoList = [
    { Id = Guid.NewGuid(); Description = "Hit the gym"; Created = DateTime.UtcNow; IsCompleted = false }
    { Id = Guid.NewGuid(); Description = "Pay bills"; Created = DateTime.UtcNow; IsCompleted = true }
    { Id = Guid.NewGuid(); Description = "Meet George"; Created = DateTime.UtcNow; IsCompleted = false }
    { Id = Guid.NewGuid(); Description = "Buy eggs"; Created = DateTime.UtcNow; IsCompleted = false }
    { Id = Guid.NewGuid(); Description = "Read a book"; Created = DateTime.UtcNow; IsCompleted = true }
    { Id = Guid.NewGuid(); Description = "Read Essential Functional-First F#"; Created = DateTime.UtcNow; IsCompleted = false }
]
```

There's a lot of duplicate code, so let's create a helper function to simplify things:

```fsharp
let create description isCompleted =
    { 
        Id = Guid.NewGuid()
        Description = description
        Created = DateTime.UtcNow
        IsCompleted = isCompleted 
    }

let todoList = 
    [
        create "Hit the gym" false
        create "Pay bills" true
        create "Meet George" false
        create "Buy eggs" false
        create "Read a book" true
        create "Read Essential Functional-First F#" false
    ]
```

We need to create a partial view, which is simple helper function to style each item in the Todo list. You should write this function above the todoView code:

```fsharp
// Todo -> XmlNode
let showListItem (todo:Todo) =
    let style = if todo.IsCompleted then [ _class "checked" ] else []
    li style [ str todo.Description ]
```

Now we can use the showListItem partial view function in the todoView and we can pass the Todo items in as an input parameter:

```fsharp
let todoView items =
    [
        div [ _id "myDIV"; _class "header" ] [
            h2 [] [ str "My ToDo List" ]
            input [ _type "text"; _id "myInput"; _placeholder "Title..." ]
            span [ _class "addBtn"; _onclick "newElement()" ] [ str "Add" ]
        ]
        ul [ _id "myUL" ] [
            for todo in items do 
                showListItem todo
        ]
        script [ _src "main.js"; _type "text/javascript" ] []
    ]
    |> createMasterPage "My ToDo App"
```

We need to change the root route as we are now passing the list of Todos into the todoView function:

```fsharp
GET >=> route "/" >=> htmlView (todoView todoList)
```

Run the app to see the items displayed on the HTML page.

What we also have is view code where it probably shouldn't be, so let's have a go at tidying up a little.

## Tidying Up

We have a master page and a view of Todos with supporting code that need moving. Let's move things around and see what we've broken by doing so.

Let's start by creating a new file called Shared.fs above the existing files and add the createMasterPage function to it. Fixing this code means adding an import declaration for the Giraffe ViewEngine as well:

```fsharp
namespace GiraffeExample

module SharedViews =

    open Giraffe.ViewEngine

    let createMasterPage msg content =
        html [] [
            head [] [
                title [] [ str msg ]
                link [ _rel "stylesheet"; _href "css/main.css" ]
            ]
            body [] content
        ]
```

It is recommended that you limit the scope of the import declaration to the module containing the view functions.

We can delete the indexView from Startup.fs as it is no longer used. 

We need to fix the todoView function as it can't find the master page that we have just moved by adding the name of the new module:

```fsharp
|> SharedViews.createMasterPage "My ToDo App"
```

Create a new module inside the Todos module in Todos.fs called Views, add an import declaration for the Giraffe View Engine and move the two todo view functions from Startup.fs into it:

```fsharp
module Todos =

    module Views =

        open Giraffe.ViewEngine

        let showListItem (todo:Todo) =
            let style = if todo.IsCompleted then [ _class "checked" ] else []
            li style [ str todo.Description ]

        let todoView items =
            [
                div [ _id "myDIV"; _class "header" ] [
                    h2 [] [ str "My ToDo List" ]
                    input [ _type "text"; _id "myInput"; _placeholder "Title..." ]
                    span [ _class "addBtn"; _onclick "newElement()" ] [ str "Add" ]
                ]
                ul [ _id "myUL" ] [
                    for todo in items do 
                        showListItem todo
                ]
                script [ _src "main.js"; _type "text/javascript" ] []
            ]
            |> SharedViews.createMasterPage "My ToDo App"
```

The final fix is to let the webApp function know where to find the todoView function:

```fsharp
let webApp = 
    choose [
        subRoute "/api/todo" Todos.apiTodoRoutes
        GET >=> choose [
            route "/" >=> htmlView (Todos.Views.todoView todoList)
            subRoute "/api" apiRoutes
        ]
        RequestErrors.NOT_FOUND "Not Found"
    ]
```

If you build the website now, it should be ready to run.

Run the website and check that the Todos page displays correctly.

I highly recommend reading the [Giraffe View Engine documentation](<https://github.com/giraffe-fsharp/Giraffe.ViewEngine>) to find out about some of the other features available.

If you like the Giraffe View Engine but don't like writing JavaScript, you should investigate the [SAFE Stack](<https://safe-stack.github.io/>) where **everything** is written in F#.

## Summary

We have only scratched the surface of what is possible with Giraffe and the Giraffe View Engine for creating web pages and APIs.

That concludes our journey into functional-first F# and Giraffe. I hope that I have given you enough of a taste of what F# can offer to encourage you to want to take it further.