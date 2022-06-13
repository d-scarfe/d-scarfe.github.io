# Introduction to Functions in Scala


Scala is both an object-oriented (OO) and functional programming (FP) language. In practice, many businesses choose Scala for its functional approach which allows us to write complex code that is concise, performant and concurrent. 

Functional programming pushes the more basic programming tasks (such as list comprehension & caching) down into the compiler. This reduces overhead & results in fewer bugs. It also means that our code is easier to read and therefore easier to maintain.

In this article, we demonstrate several functional programming techniques, including the ability to define a function and pass it around as an instance. We'll showcase several examples of functional programming, and assess the advantages of this approach. We will also demonstrate several Scala programming shortcuts.

## Anonymous Functions
We use anonymous functions, or *function literals*, when we need to pass a function into a method or assign it to a variable.

Let us imagine we wish to filter a list of numbers, extracting only the even values. We can do this with an anonymous function:

```scala
val rawList = List.range(1, 20)
val evenList = rawList.filter(i => i % 2 == 0)
```

Here we can see that `i` is *transformed*, and the values are filtered only if `i % 2` is equal to zero (and so the number is even).

The above example demonstrates a more traditional coding style. However, because `i` is used in only one statement inside our function literal, we can make use of Scala's `_` wildcard to simplify things further:

```scala
val evenList = rawList.filter(_ % 2 == 0)
```

The two statements are equivalent in operation, but the latter is easier to read & requires less code. The `_` shortcut is used throughout Scala when dealing with a functional programming model.

### Functions as variables

A function is really no different to any other object, so we can treat it much like we might treat a `String` or `Int`.

We can simply assign a function literal to a variable, and then invoke it as required:

```scala
val double = (i: Int) => i * 2
double(5)  // Returns 10
double(10) // Returns 20
```

In the above, we define a function capable of doubling a given `Int` value, and then we store it in a variable.

As we now have an instance of a function, we consider this a *function value*. 

We can now pass this into a `.map()` call on our *rawList*, resulting in the doubling of all values within:

```scala
val doubledList = rawList.map(i => double(i))
```

Again, we have demonstrated a traditional coding style here and readers may notice that we could utilise a `_` shortcut in place of `i`.

However, we can make use of another Scala shortcut to simplify things even further than that. Because our function literal consists only of a single statement that takes a single argument, we do not need to pass in the value at all:

```scala
val doubledList = rawList.map(double)
```

In just two short lines of code, we have defined a function that can transform an argument, and executed our function on a collection.

### Declaring return types

It is not necessary to declare a return type for a Scala function, and in many cases it may not feel appropriate. For example, let us consider our function from earlier:

```scala
val isEven = (i: Int) => i % 2 == 0
```

Most programmers can immediately infer that the return type here is a `Boolean`, and the Scala compiler is clever enough to work it out too. 

However, we've picked a wonderfully simple function to showcase. When things become more complicated, declaring the return type can make the code more readable and therefore more maintainble. An example of the two styles in practice:

```scala
// Implicit approach - no return declaration.
val multiply = (a: Int, b: Int) => a * b

// Explicit approach - declaring Int return type.
val multiply = (Int, Int) => Int = (a, b) => a * b
```

For the most part, you will find that Scala programmers tend to prefer the implicit approach. It's concise and once you are familiar with functional programming, it tends to feel more natural.

However, there are plenty of times that a function becomes complicated enough to justify declaring the return type, so it is a case of deciding what feels appropriate given the circumstances.

## Executing Functions

We've seen how powerful an anonymous function can be, and in the section that follows we will explore how to define a method that takes a simple function as a parameter & executes it. 

### Defining our method

The general syntax for defining a method which accepts a function in its signature is as follows:

```scala
def methodName(functionName:(parameters) => returnType) { }
```

Therefore, the most simple functional method might look like so:

```scala
def execute(f:() => Unit) {
    f()
}
```

In the above example, we create a `.execute()` method that takes a function `f` with no parameters and returns a `Unit`. In Scala, a `Unit` is similar to that of Java's `Void`, so we are indicating that we are effectively going to return nothing here.

In reality though, you're probably going to need some parameters, so let's adapt our method to expect a function with a `String`:

```scala
def print(f:() => String) {
    println(f)
}
```

Now, we've effectively aliased the "println" method with our own implementation that expects a function returning a String. Of course, this is not something you'd likely do in the real-world, but it is useful for the purpose of this tutorial.

### Triggering the behaviour

We now define a function that matches our method's signature, and pass it to our method:

```scala
val helloWorld = () => "Hello" + " " + "World!"
print(helloWorld)
```

Later, if we decide we want to change our behaviour, we needn't change the method but merely the function:

```scala
case class Person(name: String)
val openDoors = () => "Open the pod bay doors" + Person("Hal")
val sorryDave = () => "I'm sorry" + Person("Dave") + ", I can't do that."
print(openDoors)
print(sorryDave)
```

One method definition, with two separate calls using different functions, results in completely different behaviour.

## Increasing Complexity

So far we have only looked at functions that perform basic mathematics or the printing of String values to the console.

However, the main advantage of the functional approach only becomes obvious when we start to increase the complexity.

In the section that follows, we are going to look at some more complicated function examples with real-world usages.

### Multiple parameters

Eagle-eyed readers may have noticed a multiple parameter example earlier, when assessing return type declarations. Let's go ahead and reuse our `multiply` function, and create a new `sum` function too:

```scala
val multiply = (a: Int, b: Int) => a * b
val sum = (a: Int, b: Int) => a + b
```

In both examples above, our function expects two `Int` parameters and then performs an operation on those values. We don't specify the return type, but we can infer from the behaviour that it is going to be an `Int` value. 

We can now adapt our print method from earlier, to expect a function & two parameters:

```scala
def print(f:(Int, Int), a: Int, b: Int) {
    val result = f(a, b)
    println(result)
}
```

The `print` method will now execute our function using the parameters provided, and then print the result to the console:

```scala
print(multiply, 2, 10) // Prints "20"
print(sum, 2, 10)      // Prints "12"
```

What's clever is that the method does not know which function is ran. It takes the parameters `a` and `b` and it passes them to the function. 

It doesn't care what our function is doing, all it is concerned with is printing the result. The logic for the function is defined elsewhere.

## Final Thoughts

### Comparison with Methods

In Scala, it is entirely possible to define a method on a class & pass it around as we did with our anonymous function. 

Let's create an isEven method like so:

```scala
def isEven(i: Int) = i % 2 == 0
```

Once again we have not specified the return type of `Boolean` because it is not compulsory in Scala. It can be inferred from looking at the method statement, but usually I would advise also naming the method appropriately too. Here, `isEven()` implies a true or false response, and hence we don't need to specify our return type.

Now we are able to pass our method instance into any function that takes an `Int` and expects a `Boolean` response. One such example is the `filter()` function on a collection. Rather than defining a new predicate, we can simply pass our method instance: 

```scala
val evenList = rawList.filter(isEven)
```

So what's the difference between defining a method & defining a function? Well the answer is, not a lot. One is a method attributed to a class, the other is a function assigned to a variable. They are both valid solutions to a single problem. 

There are some differences "under the hood". Specifically, the function implements one of the several *Function* traits whereas the method does not. Scala doesn't really care, and you shouldn't either. Pick whichever style works for you & your current code base. 


### Conclusion

In this article, we reviewed how to create & use functions within Scala. We built some basic example functions, and then we wired them into a method call and demonstrated how to pass parameters into the function.

We showcased how we can use functions to abstract away complicated behaviour into small, well-scoped units of code. This will form the cornerstone of our work in Scala, and promotes a better programming style with reduced complexity.
