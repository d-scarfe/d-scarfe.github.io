# Building a REST API with Spring WebFlux


In this article, we're going to build a *reactive* REST API in Java using Spring's WebFlux framework.

We'll demonstrate how to make use of `Flux` and `Mono` publishers to serve JSON content to the client.

We shall showcase both annotation-based & functional coding styles, so that you can decide which approach is best for you.

This tutorial shall provide the basis from which we can build a reactive, scalable web backend in Java.

## Introducing Spring WebFlux

### Why choose WebFlux?
WebFlux is Spring's answer to the need for a non-blocking web stack that handles concurrency with fewer threads, and scales with less reliance upon physical hardware. WebFlux builds upon [Project Reactor](https://projectreactor.io/), a well-loved enterprise framework that provides non-blocking message parsing with a focus on low memory usage & a high throughput.

Let us first recap how Spring MVC functions, which many of us are familiar with. In this implementation, a single thread is created to service each request. Whilst this approach is easy to debug & understand, each thread typically utilises 1MB of heap to fulfil the request. When faced with many concurrent users, this approach can present a number of scaling issues.

In contrast, WebFlux's reactive model keeps the thread pool small. The non-blocking approach brings about a significant leap in performance, whilst the annotation-based processing feels familiar & comfortable for those who have worked with Spring before. if you are looking to produce a reactive Java backend, in an environment where efficiency and/or resource utilisation are of utmost importance, WebFlux is the perfect tool.

But allow me to be clear, whilst reactive frameworks are powerful & scale well, they can be more difficult to implement & debug. If you have a Spring MVC application that is sufficient for your needs, there is no need to convert it to WebFlux. The old mantra of "if it ain't broke, don't fix it" applies!

### Who is using WebFlux?
WebFlux facilitates the backend for a number of large organisations working in Java. One notable example includes *JP Morgan*, whose financial platform is used by hundreds of thousands of clients around the world. Their application therefore needs to service a large number of concurrent requests in a trading environment where high throughput is vital. The reactive model is cornerstone to this solution.

## Our Mission
### What are we making?
In the following example, we'll build a reactive REST API which provides a number of endpoints capable of responding to HTTP requests. 

We will start by implementing some simple GET endpoints, and then we will add support for POST requests.

We shall test the API by performing some basic CRUD operations to manipulate an example in-memory dataset. 

### What do I need?
This article assumes that the reader has already installed JDK 8 onwards, alongside a recent version of Apache Maven.

There are no other requirements for this project. All testing will be done within Java.

## Getting Started
### Initial setup

Let's create a new Maven project, and add the following WebFlux dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <version>2.7.0</version>
</dependency>
```

This will pull in a number of required dependencies, including:

* `spring-boot-starter` to provide us with a Spring Boot base.
* `spring-webflux` to provide us with our WebFlux framework.
* `reactor-core` to provide us with the reactive streams API.
* `reactor-netty` to provide us with an embedded web server.

The version `2.7.0` was correct at the time of writing, but it is always worth validating against [Maven Central](https://search.maven.org/search?q=g:org.springframework.boot%20AND%20a:spring-boot-starter-webflux).

### Project structure

Since we have a blank project, let's go ahead and create the following directories:

* `src/main/java/uk/scarfe/webflux/` which will be our base package for the project.
* `.../webflux/model` which shall contain our example data model.
* `.../webflux/annotation` which will demonstrate annotation-based WebFlux deployment.
* `.../webflux/functional` which will demonstrate functional style WebFlux deployment.

Later we will add some automated tests, but we needn't worry about those for now.

## Introducing Flux & Mono

Before we get started with our Webflux application, we must familiarise ourselves with its two publisher implementations, `Flux` and `Mono`.

`Mono` is a Publisher that emits *at most* 1 element:

```java
Mono<String> mono = Mono.just("Daniel");
Mono<Object> monoEmpty = Mono.empty();
Mono<Object> monoError = Mono.error(new RuntimeException());
```

In contrast, `Flux` is a publisher that emits *0* to *n* number of elements:

```java
Flux<Integer> flux = Flux.just(1, 2, 3, 4, 5);
Flux<String> fluxString = Flux.fromArray(new String[]{"A", "B", "C"});
Flux<String> fluxIterable = Flux.fromIterable(Arrays.asList("A", "B", "C"));
Flux<Integer> fluxRange = Flux.range(0, 10);
Flux<Long> fluxLong = Flux.interval(Duration.ofSeconds(60));
```

The `Flux` instance returns a sequence of elements and notifies upon completion.

Note that despite creating the `Flux` instance, the data will not flow until `.subscribe()` is called:

```java
List<String> dataStream = new ArrayList<>();
Flux.just("Daniel", "David", "Dory", "Diana")
    .log()
    .subscribe(dataStream::add);
```

Project Reactor also provides us with a number of stream-based operators, including `.map()`/`.flatMap()`/`.merge()`/`.concat()` and more.

Readers should take a moment to familiarise themselves with [the API](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Flow.Publisher.html), as it is an incredibly powerful toolset when mastered.

## Annotation Approach

### Defining the data model

Let's go ahead and create our application. We start by creating a POJO that will store some example data.

In this instance, we're going to create an `Employee` class under the `model` namespace:

```java
package uk.scarfe.webflux.model;

import java.util.Objects;

public class Employee {

    private final long id;
    private final String name;

    public Employee() {
        this.id = -1;
        this.name = "";
    }

    public Employee(long id, String name) {
        this.id = id;
        this.name = name;
    }

    public long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Employee employee = (Employee) o;
        return id == employee.id && Objects.equals(name, employee.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

}
```

In the above example, we have created a basic immutable class that stores an employee ID and a name.

It should be noted that we need the default no-args constructor to faciliate serialisation of the data.

We've implemented an `.equals()` and `.hashCode()` so we can identify two equal Employee instances.

### Creating a controller

Now we need to create a controller who is responsible for handling our request and creating the response.

Let's add an `EmployeeController` class under the `controllers` package:

```java
package uk.scarfe.webflux.annotation;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;
import uk.scarfe.webflux.model.Employee;

import java.util.Arrays;
import java.util.List;

@RestController
@RequestMapping("/employees")
public class EmployeeController {

    public static final List<Employee> EMPLOYEES =
            Arrays.asList(new Employee(1, "Daniel Scarfe"), new Employee(2, "Joe Bloggs"));

    @GetMapping
    public Flux<Employee> getAll() {
        return Flux.fromIterable(EMPLOYEES);
    }

}
```

This simple reactive class provides a GET mapping for `/employees` that will return a `Flux` object containing two hard-coded `Employee` instances. 

In practice, you would likely retrieve your content from a data store, but we are merely demonstrating the reactive layer here.


### Creating a client

Now we have our server-side implementation, we can make use of a Spring's `WebClient` to provide client-side access.

The `WebClient` is Spring's non-blocking approach to performing HTTP requests. 

Let's implement an `EmployeeClient` class which connects to the default port of 8080:

```java
package uk.scarfe.webflux.annotation;

import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;

import org.springframework.http.MediaType;
import org.springframework.web.reactive.function.client.WebClient;
import uk.scarfe.webflux.model.Employee;

@Component
public class EmployeeClient {

    private static final String BASE_URL = "http://localhost:8080";
    private final WebClient client;

    public EmployeeClient(WebClient.Builder builder) {
        this.client = builder.baseUrl(BASE_URL).build();
    }

    public Flux<String> getEmployeeNames() {
        return this.client.get().uri("/employees").accept(MediaType.APPLICATION_JSON)
                .retrieve()
                .bodyToFlux(Employee.class)
                .map(Employee::getName);
    }

}
```

In the above code, we've implented some basic logic to get all employee names as a `Flux<String>`.

Any caller subscribing to this flux will receive the full list of names upon completion.

### Running the application

Let's make our application executable, by creating an `Application` class that utilises `@SpringBootApplication` annotation:

```java
package uk.scarfe;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;
import uk.scarfe.webflux.annotation.EmployeeClient;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
        EmployeeClient client = context.getBean(EmployeeClient.class);
        client.getEmployeeNames().subscribe(System.out::println);
    }
}
```

Thanks to Spring Boot's auto-configuration, this is all we need to do. Spring Boot will automatically start our Netty server on port 8080 and perform injection of the relevant dependencies.

By retrieving the `EmployeeClient` from the application context and subscribing to the publisher instance, we can retrieve the employee data in a non-blocking manner. 

If you go ahead and run the application you will see our two employee names printed to the console:

```bash
uk.scarfe.Application                    : Started Application in 1.978 seconds (JVM running for 2.435)
Daniel Scarfe
Joe Bloggs
```

There you have it. We have successfully demonstrated how to utilise Spring's annotation-based processing (similar to that of Spring MVC) to produce a reactive web backend in Java. 

Implementing additional PUT / POST / DELETE functionality is as simple as using the corresponding annotations:

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public Mono<Employee> create(@RequestBody Employee employee){
  // As we have not created an 'employeeService' yet, this is merely an example.
  return employeeService.createEmployee(employee);
}
```

Annotation-based processing isn't for everybody. In the next section, we will evaluate the functional approach also supported by WebFlux.

## Functional Approach

The "Spring Functional Web Framework" was designed for Spring WebFlux, but has now also been introduced to Spring MVC. 

Instead of using the annotation-based approach, we can utilise functions for routing & handling our HTTP requests. 

### Implementing a service

To avoid mixing the two approaches, we're going to create a very basic `EmployeeService` like so:

```java
package uk.scarfe.webflux.functional;

import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;
import uk.scarfe.webflux.model.Employee;

import java.util.Optional;

import static uk.scarfe.webflux.annotation.EmployeeController.EMPLOYEES;

@Component
public class EmployeeService {

    public Mono<Employee> getById(long id) {
        final Optional<Employee> foundEmployee = EMPLOYEES.stream().filter(e -> e.getId() == id).findAny();
        return foundEmployee.map(Mono::just).orElseGet(Mono::empty);
    }

}
```

The only public facing method, `getById()` shall return a `Mono` instance containing an employee corresponding to the given ID value, or `Mono.empty()` if one is not present. 

### Creating the handler

Nowe we create an `EmployeeHandler` that can accept a `ServerRequest` and returns a response containing our employee object:

```java
package uk.scarfe.webflux.functional;

import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.server.ServerRequest;
import org.springframework.web.reactive.function.server.ServerResponse;
import reactor.core.publisher.Mono;
import uk.scarfe.webflux.model.Employee;

@Component
public class EmployeeHandler {

    private final EmployeeService employeeService;

    public EmployeeHandler(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    public Mono<ServerResponse> getById(ServerRequest request) {
        final long id = Long.parseLong(request.pathVariable("id"));
        return employeeService
                .getById(id)
                .flatMap(e -> ServerResponse.ok()
                        .contentType(MediaType.APPLICATION_JSON)
                        .body(Mono.just(e), Employee.class)
                )
                .switchIfEmpty(ServerResponse.notFound().build());
    }

}
```

Note the functional style. The "id" path variable is passed into the `.getById()` call, and the result is flat-mapped into a `ServerResponse`.

The `Employee` instance that matches our ID value is stored in the body of the response, and a *HTTP 200 OK* is sent.

In the event an employee instance corresponding to our ID cannot be verified, the `.switchIfEmpty()` statement ensures that a *HTTP 404 Not Found* response is sent instead.


### Defining our routes

Finally, we create an `EmployeeRouter` which provides the route configuration for our endpoint mapping:

```java
package uk.scarfe.webflux.functional;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.ServerResponse;

import static org.springframework.web.reactive.function.server.RequestPredicates.GET;
import static org.springframework.web.reactive.function.server.RouterFunctions.route;

@Configuration
public class EmployeeRouter {

    @Bean
    public RouterFunction<ServerResponse> employees(EmployeeHandler employeeHandler) {
        return route(GET("/employees/{id}"), employeeHandler::getById);
    }

}
```

Here we can see that we have mapped a GET endpoint at *"/employees/{id}"* and delegated to the `.getById()` function in `EmployeeHandler`.

The class is annotated with `@Configuration` so Spring will construct our `RouterFunction` bean & automatically configure our Netty server accordingly.


## Automated Testing

Whilst manual testing has it's place in software development, all good projects will have proper automated testing, and this is no exception. 

Fortunately, Spring makes the testing of REST APIs very easy. Let's go ahead and add the following dependencies to our `pom.xml`:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>RELEASE</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <version>2.7.0</version>
    <scope>test</scope>
</dependency>
```

### Validating a Flux response

Now we can test our endpoints by creating an `EmployeeHandlerTest` with JUnit 5.

Let's start by testing the `.getAll()` method:

```java
package uk.scarfe.webflux;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.web.reactive.server.WebTestClient;
import uk.scarfe.webflux.model.Employee;

import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static uk.scarfe.webflux.annotation.EmployeeController.EMPLOYEES;

@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class EmployeeHandlerTest {

    // Spring Boot will create a pre-configured `WebTestClient` for you.
    @Autowired
    private WebTestClient webTestClient;

    @Test
    public void testGetAll() {
        webTestClient.get().uri("/employees")
                .accept(MediaType.APPLICATION_JSON)
                .exchange()
                .expectStatus().isOk()
                .expectBodyList(Employee.class)
                .isEqualTo(EMPLOYEES);
    }

}
```

Thanks to Spring Boot, the web environment is started on a random available port, and the `WebTestClient` is automatically configured to test against it.

We simply inject this into our test suite and then use the functional API to assert that our responses match our desired behaviour.

### Validating a Mono response

We can extend our test suite further, by validating calls to `.getById()` like so:

```java
@Test
public void testGetById() {
    webTestClient.get().uri("/employees/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectBody(Employee.class).value(employee -> {
                assertEquals("Daniel Scarfe", employee.getName());
            });
}
```

Here we make use of `.expectBody(...)` to validate that the body of the JSON response contains our desired `Employee` instance.

### Validating error scenarios

Finally, it's always worth testing the various error scenarios, such as when an item cannot be found:

```java
@Test
public void testNotFound() {
    webTestClient.get().uri("/employees/123")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isNotFound();
}
```

In this example, we tell the test client to expect a *"Not Found"* response because no employee exists with ID #123.

Of course, the implementation has both functional & annotation styles, but the testing methodology remains the same. 

This represents a very powerful way of testing APIs; the overall suite takes milliseconds to run, yet can provide complete coverage with minimal effort.

## Conclusion

In this article, we explored how to create & work with the reactive web components supplied by the Spring WebFlux framework. 

As an example, we built a small reactive REST API for retrieving employee data in a non-blocking manner. 

We introduced both the `Flux` and `Mono` publisher implementations, available through Project Reactor / JDK 9 onwards.

Then we learned how to use `RestController` and `WebClient` to publish & consume our reactive streams in the JVM.

We also reviewed the functional approach to creating WebFlux applications, and demonstrated how to test the application in JUnit 5.
