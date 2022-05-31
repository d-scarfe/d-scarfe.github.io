# Building a REST API with Scala & Play


In this article, we're going to build a simple REST API in Scala using the Play Framework. 

We shall utilise HTTP requests to perform basic CRUD operations on an example dataset. 

We'll demonstrate (in Scala) how we can serve our content back to the user in the JSON format. 

This tutorial shall provide the basis from which we can build efficient, scalable backend systems.

## Introducing the Play Framework

### Why choose Play?
The Play Framework encourages scalability whilst managing to remain simple to use. It remains true to the MVC paradigm, but builds upon reactive principles, meaning that the concept feels familiar but the resulting product is highly performant. For me personally, the biggest advantage of the Play Framework is performance. Play's threading model means that it is both fast & resource efficient. 

This is in contrast to the more traditional approach to MVC, such as Spring's implementation in Java, whereby a single thread is created for each request. Whilst this approach is easy to debug & understand, each thread typically utilises 1MB of heap to service the request. When faced with many concurrent users, this approach can present a number of scaling issues. 

### Who is using Play?
Some key players using the Play Framework in industry include *LinkedIn* and *Walmart*. Both of these large organisations handle millions of requests daily, which demonstrates the power & scalability of the framework. Since its initial release in 2007, Play has become a much loved web framework for the Java & Scala community.

## Our Mission
### What are we making?
In the following example, we'll build a very simple REST API which provides a number of endpoints capable of responding to HTTP requests. 

We will start by implementing a simple GET endpoint, and then we will add support for POST & DELETE requests. 

We shall test the API by performing some basic CRUD operations to manipulate an example in-memory dataset. 

### What do I need?
This article assumes that the reader has already installed a recent version of `sbt`. 

We will peform basic API testing using `curl` on the command line. 

Some readers may prefer to use a graphical testing tool, such as [Postman](https://www.postman.com/). 

## Getting Started
### Initial setup
For convenience, we can start our new Play Framework project using an `sbt` template:

```
$ sbt new playframework/play-scala-seed.g8
```

This creates a new Scala project with an example controller, two HTML files and basic configuration.

Let's go ahead and delete some of these components so that we have a "clean slate" to work from: 

```
$ rm -rf app/controllers/HomeController.scala 
$ rm -rf app/views/index.scala.html app/views/main.scala.html
$ truncate -s 0 conf/routes
```

### Project structure

Now we can take a look at the various directories in a little more detail:

* `app/controllers/` shall contain our Scala classes used to handle the HTTP events.
* `app/models/` shall contain our data objects, in the form of Scala classes.
* `app/views/` shall contain any HTML template files (not used in this tutorial).
* `conf/` shall contain the configuration used to wire everything together.

Note the familiarity of the package names above; *Model*, *View*, *Controller* (MVC).

## Our First GET Endpoint

We begin by creating a simple REST endpoint that responds to *HTTP GET* requests and provides a *NoContent* response. This is arguably the most simple endpoint we can make, but it is useful in helping to demonstrate the classes involved.

### Creating a controller
First, we create an `EmployeeController` class in the `app/controllers` directory:

```scala
package controllers

import javax.inject.Singleton
import com.google.inject.Inject
import play.api.mvc.{Action, AnyContent, BaseController, ControllerComponents}

@Singleton
class EmployeeController @Inject()(val controllerComponents: ControllerComponents)
  extends BaseController {
    // Business logic goes here!
}
```

Our new class must extend the `BaseController` trait, which gives us access to `Action` and `Results` functionality.

We’ve used the `@Inject` annotation to instruct the Play Framework to pass the required dependencies automatically via constructor injection.

Also, we marked the class as a `@Singleton` so that the framework will create only one instance, reusing it for every request.

### Our response function

Now we have a `EmployeeController`, let’s create the method within that will be called when our server receives a REST request. 

First, we define a *getAll* function that returns an aforementioned `Action`:

```scala
def getAll(): Action[AnyContent] = Action {
  NoContent
}
```

The `Action` class gives us access to request parameters and allows us to return a HTTP response. 

For now, we simply return a *NoContent* status, because we have no content to return.

### Enabling the endpoint

Before we create our content, let's add the controller to the routes file:

```
GET    /employees                  controllers.EmployeeController.getAll()
```

We must specify the HTTP method, the path, and the name of the Scala function that handles the request. 

The only separater here is whitespace, but we format it into columns for readability. 

### Running the application

Now we have enabled our endpoint, we can go ahead and start the application:

`$ sbt run 9000`

After a short while, we’ll see the following message:

`[info] p.c.s.AkkaHttpServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9000`

This indicates that the application has started & is ready to handle requests.

Let's make our first HTTP GET call using `curl` like so:

```
$ curl -v localhost:9000/employees
```
You will be greeted with a `HTTP/1.1 204 No Content` response. 

Et voilà! We now have a working RESTful API using Scala & the Play Framework.

Granted, it is a little basic, so let's continue to add some more functionality.

## Returning JSON Content

So far we have returned only a *NoContent* response. That suffices for proving we can build a running Play application, but you aren't going to build an enterpise backend on that. 

At the very least, we need to be able to return some meaningful content. Let's produce some.

Now, we are going to look at returning an `Employee` list. Let’s define the data model, create an in-memory collection of employees, and modify the `EmployeeController` to return JSON objects.
 
### Defining the data model

First, we create a new case class in the `app/models` directory:

```scala
package model

case class Employee(id: Long, name: String)
```

Now, we define the list of tasks in the `EmployeeController` class. 

As this list will be modified, we need Scala's mutable collections package:

```scala
...
import model.Employee
import scala.collection.mutable

class EmployeeController @Inject()(val controllerComponents: ControllerComponents)
  extends BaseController {

    private val employeeList = new mutable.ListBuffer[Employee]()
    employeeList += Employee(1, "Daniel Scarfe")
    employeeList += Employee(2, "Joe Bloggs")
    ...
```

To make testing our application easier, we've hardcoded a number of values here, but in practice you would likely retrieve this information from a data store.

### Formatting into JSON

Next, let’s create the JSON formatter that converts the `Employee` object into JSON. 

We add our required imports, then create the JSON formatter inside the `EmployeeController` class:

```scala
...
import play.api.libs.json.{Json, OFormat}

@Singleton
class EmployeeController @Inject()(val controllerComponents: ControllerComponents)
  extends BaseController {

  implicit val formatter: OFormat[Employee] = Json.format[Employee]
  ...
```

We make it an implicit field to avoid having to pass it to the Json.toJson function on each usage.

### Implementing our response

Now we have some data to return, let’s change our *getAll* function to return the *NoContent* status code only when the list is empty.

Otherwise, it shall return the complete employee list, converted to JSON by the formatter:

```scala
def getAll(): Action[AnyContent] = Action {
  if (employeeList.isEmpty) {
    NoContent
  } else {
    Ok(Json.toJson(employeeList))
  }
}
```

Let’s re-test the endpoint using the same statement as before:

```bash
$ curl localhost:9000/employees

[{"id": 1, "name": "Daniel Scarfe"},{"id": 2, "name": "Joe Bloggs"}]
```

We have successfully returned a list of employees in JSON format.

### Handling parameterised calls

A good REST API shall also allow you to retrieve individual items via a path parameter like so:

```bash
$ curl localhost:9000/employees/1
```

This shall return the item, or *NotFound* if the employee ID is not recognised. 

Let's implement that. First, we define a new endpoint in the routes file:

```
GET    /employees/:id              controllers.EmployeeController.getById(id: Long)
```

The notation */employees/:id* means that the Play Framework shall capture everything after the */employee/* prefix and assign it to the *id* variable. This value is then passed into the *getById* function, which we haven't written yet.

Note that we have specified a type for the parameter (*Long*). The Play Framework will automatically convert text to a number, and return a *BadRequest* response if the provided value is not a number.

We can now add the *getById* method in the `EmployeeController` class:

```scala
def getById(id: Long): Action[AnyContent] = Action {
  val foundEmployee = employeeList.find(_.id == id)
  foundEmployee match {
    case Some(employee) => Ok(Json.toJson(employee))
    case None => NotFound
  }
}
```

Our method retrieves the *id* parameter and tries to find the employee list item associated with it. 

We’re using the find function, which returns an instance of the `Option` class. As such, we can make use of pattern matching to distinguish between an empty `Option` and an `Option` with a value, as seen above.

When the employee is present in the employee list, it’s converted into JSON returned in an *OK* response. Otherwise, *NotFound* is returned, resulting in HTTP 404.

Now, we can try to get an employee that is in our list:

```bash
$ curl localhost:9000/employees/1

{"id":1,"name":"Daniel Scarfe"}
```

Or perhaps try an employee that isn’t:

```bash
$ curl -v localhost:9000/employees/3

HTTP/1.1 404 Not Found
```

## Implementing POST

The Play Framework handles all HTTP methods, so when we want to use PUT / POST / DELETE, we simply configure it in the routes file. 

As an example, let's implement an POST endpoint that allows us to create a new employee. We shall start by adding another entry in our routes file:

```
POST   /employees                  controllers.EmployeeController.addNewEmployee()
```

Then, as before, we simply add the business logic into the `EmployeeController` class:

```scala
def addNewEmployee(): Action[AnyContent] = Action { implicit request => 
  // Extract the JSON object from the request body.
  val body: AnyContent = request.body
  val jsonBody: Option[JsValue] = body.asJson
  val newEmployee: Option[Employee] =
    jsonBody.flatMap(js => Json.fromJson[Employee](js).asOpt)
    
  // If present & valid, add to list and return a Created response.
  newEmployee match {
    case Some(employee) =>
      employeeList.addOne(employee)
      Created(Json.toJson(employee))
    case None => BadRequest
  }
}
```

In our method, *body.asJson* parses the given JSON object and returns an `Option`. We would get a valid object only if the deserialisation were successful.

If the caller has sent us content that cannot be deserialised as an `Employee`, or if the Content-Type is not *application/json*, we end up with a `None` type and a *BadRequest* response is sent. 

Otherwise, our new `Employee` instance is added to our *employeeList*, and a *HTTP 201 Created* response is returned to the client.

We can now try to add a new employee using the API like so:

```bash
$ curl -d '{"id":3,"name":"James Blunt"}' -H 'Content-Type: application/json' localhost:9000/employees

HTTP/1.1 201 Created.
{"id": 3,"name": "James Blunt"}
```

As we can see from our *201 Created* response, the new item has been added and subsequently returned in the response body.

To verify that the item was added to our in-memory data store, we can retrieve all items again:

```bash
$ curl localhost:9000/employees

[
  {"id": 1, "name": "Daniel Scarfe"},
  {"id": 2, "name": "Joe Bloggs"},
  {"id": 3, "name": "James Blunt"}
]
```

This time, the JSON array shall contain three objects, including our new employee instance.

## Implementing DELETE

Finally, let's add support for deleting items from the list using a HTTP DELETE request.

As before, we add our business logic to the `EmployeeController` class:

```scala
def deleteById(id: Long): Action[AnyContent] = Action {
  val foundEmployee = employeeList.find(_.id == id)
  foundEmployee match {
    case Some(employee) =>
      employeeList -= employee
      Ok
    case None => NotFound
  }
}
```

Then we configure the route:

```
DELETE /employees/:id              controllers.EmployeeController.deleteById(id: Long)
```

And we test our endpoint using `curl` like so:

```bash
$ curl -v -X DELETE localhost:9000/employees/3

HTTP/1.1 200 OK.
```

Note here that we have chosen to simply return an *OK* response with no body, for simplicity.

We perform one final validation to make sure that the item has been removed from the list:

```bash
$ curl localhost:9000/employees

[{"id": 1, "name": "Daniel Scarfe"},{"id": 2, "name": "Joe Bloggs"}]
```

A second attempt will result in a *"HTTP 404 Not Found"* response, because the item no longer exists. 

 
## Conclusion

And there we have it, we have successfully implemented a RESTful API in Scala with the Play Framework!

We initialised the project and defined our first route and Scala controller class. Then we demonstrated how to marshal JSON to/from Scala objects.

We showcased support for `GET`, `POST` & `DELETE` requests using the Play Framework to delegate to corresponding Scala functions.

We also looked at how to use `curl` to validate the behaviour of our code.
