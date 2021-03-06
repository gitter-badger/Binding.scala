# Binding.scala

[![Build Status](https://travis-ci.org/ThoughtWorksInc/Binding.scala.svg)](https://travis-ci.org/ThoughtWorksInc/Binding.scala)
[![Maven Central (Core Funtionality)](https://img.shields.io/maven-central/v/com.thoughtworks.binding/core_2.11.svg?label=maven-central%20%28core%29)](https://maven-badges.herokuapp.com/maven-central/com.thoughtworks.binding/core_2.11)
[![Maven Central (DOM Integration)](https://img.shields.io/maven-central/v/com.thoughtworks.binding/dom_sjs0.6_2.11.svg?label=maven-central%20%28dom%29)](https://maven-badges.herokuapp.com/maven-central/com.thoughtworks.binding/dom_sjs0.6_2.11)

**Binding.scala** is a data-binding framework for [Scala](http://www.scala-lang.org/) JVM and [Scala.js](http://www.scala-js.org/).

Binding.scala can be used as a **[reactive](https://en.wikipedia.org/wiki/Reactive_programming) web framework**.
It enables you use native XML literal syntax to create reactive DOM nodes,
which are able to automatically change whenever the data source changes.

## Comparison to other reactive web frameworks

Binding.scala has more features and less concepts than other reactive web frameworks like [Widok](https://widok.github.io/) or [ReactJS](https://facebook.github.io/react/).

<table>
  <thead>
    <tr>
      <th></th>
      <th>Binding.scala</th>
      <th>Widok</th>
      <th>ReactJS</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Support HTML literal?</th>
      <td>Yes</td>
      <td>No</td>
      <td>Yes</td>
    </tr>
    <tr>
      <th>Algorithm to update DOM</th>
      <td>Precise data-binding</td>
      <td>Precise data-binding</td>
      <td>Vitual DOM differentiation, which requires manually managed key attributes for complicated DOM.</td>
    </tr>
    <tr>
      <th>Lifecycle management for data-binding expressions</th>
      <td>Automatically</td>
      <td>Manually</td>
      <td>Manually</td>
    </tr>
    <tr>
      <th>Statically type checking</th>
      <td>Yes, even for HTML tags and attribues</td>
      <td>Yes</td>
      <td>No</td>
    </tr>
    <tr>
      <th>Learning curve</th>
      <td>Always easy</td>
      <td>Unfamiliar DOM creation syntax for newbie. Requires much efforts to understand its corner cases.</td>
      <td>Easy to start. Requires much more efforts to understand its corner cases.</td>
    </tr>
  </tbody>
</table>

See [Desigin](#design) section for more information.

## Getting started

We will build an Binding.scala web page during the following steps.

### Step 0: Setup a Sbt Scala.js project

See http://www.scala-js.org/tutorial/basic/ for information about how to setup such a project.

Note that macros used by Binding.scala require a larger stack size for JVM that runs Scala compiler than default value,
therefore, you should specify a `-Xss5m` JVM argument when launching sbt:

```
sbt -J-Xss5m
```

Otherwise you may see an error message like this:

```
[trace] Stack trace suppressed: run last dom/compile:compileIncremental for the full output.
[error] (dom/compile:compileIncremental) java.lang.StackOverflowError
```

### Step 1: Add Binding.scala dependencies into your `build.sbt`:

``` scala
libraryDependencies += "com.thoughtworks.binding" %%% "dom" % "1.0.5"
```

### Step 2: Create a `data` field, which contains some `Var` and `Vars` as data source for your data-binding expressions

``` scala
case class Contact(name: Var[String], email: Var[String])

val data = Vars.empty[Contact]
```

A `Var` represents a bindable variable,
which also implements `Binding` trait,
hence a `Var` can be seen as a binding expression as well.
If another expression depends on a `Var`, the value of the expression changes whenever value of the `Var` changes.

A `Vars` represents a sequence of bindable variables,
which also implements `BindingSeq` trait,
hence a `Vars` can be seen as a binding expression of a sequence as well.
If another comprehension expression depends on a `Vars`,
the value of the expression changes whenever value of the `Var` changes.

### Step 3: Create a `@dom` method that contains data-binding expressions

``` scala
@dom
def table: Binding[Table] = {
  <table border="1" cellPadding="5">
    <thead>
      <tr>
        <th>Name</th>
        <th>E-mail</th>
      </tr>
    </thead>
    <tbody>
      {
        for (contact <- data) yield {
          <tr>
            <td>
              {contact.name.each}
            </td>
            <td>
              {contact.email.each}
            </td>
          </tr>
        }
      }
    </tbody>
  </table>
}
```

A `@dom` method represents a data-binding expression.

The return type are always wrapped in a `com.thoughtworks.binding.Binding` trait.
For example `@dom def x: Binding[Int] = 1`,  `@dom def message: Binding[String] = "content"`

A `@dom` method supports HTML literal.
Unlike normal XML literal in a normal Scala method,
the type of HTML literal are specific subtypes of `org.scalajs.dom.raw.Node` and `com.thoughtworks.binding.BindingSeq[org.scalajs.dom.raw.Node]`
instead of `scala.xml.Node` and `scala.xml.NodeSeq`.
So we could have `@dom def node: Binding[org.scalajs.dom.raw.HTMLBRElement] = <br/>`
and `@dom def node: Binding[BindingSeq[org.scalajs.dom.raw.HTMLBRElement]] = <br/><br/>`.

A `@dom` method consists with other data-binding expressions in two approach:

 1. You could use `each` method in a `@dom` method to get value of another `Binding`.
 2. You could use `for` / `yield` expression in a `@dom` method to map a `BindingSeq` to another.

You can nest `Node` or `BindingSeq[Node]` in other HTML element literals via `{ ... }` interpolation syntax.

### Step 4: render the data-binding expressions to DOM in the `main` method

``` scala
@JsExport
def main(): Unit = {
  dom.render(document.body, table)
}
```

### Step 5: Invoke the `main` method in a HTML page

``` html
<!DOCTYPE html>
<html>
  <head>
    <script type="text/javascript" src="js-fastopt.js"></script>
  </head>
  <body>
    <script type="text/javascript">
      SampleMain().main()
    </script>
  </body>
</html>
```

Now you will see a table that just contains a header, because `data` is empty at the moment.

### Step 6: Add some `<button>` to fill `data` for the table

``` scala
@dom
def table: Binding[BindingSeq[Node]] = {
  <div>
    <button
      onclick={ event: Event =>
        data.get += Contact(Var("Yang Bo"), Var("yang.bo@rea-group.com"))
      }
    >
      Add a contact
    </button>
  </div>
  <table border="1" cellPadding="5">
    <thead>
      <tr>
        <th>Name</th>
        <th>E-mail</th>
        <th>Operation</th>
      </tr>
    </thead>
    <tbody>
      {
        for (contact <- data) yield {
          <tr>
            <td>
              {contact.name.each}
            </td>
            <td>
              {contact.email.each}
            </td>
            <td>
              <button
                onclick={ event: Event =>
                  contact.name := "Modified Name"
                }
              >
                Modify the name
              </button>
            </td>
          </tr>
        }
      }
    </tbody>
  </table>
}
```

When you click the "Add a contact" button, it appends a new Contact into `data`,
then, Binding.scala knows the relationship between DOM and `data`,
so it decides to append a new `<tr>` corresponding to the newly appended Contact.

And when you click the "Modify the name", the `name` field on `contact` changes,
then, Binding.scala decides to change the content of the corresponding `tr` to new value of `name` field.

See https://git.realestate.com.au/yang-bo/Binding.scala-sample for the complete example.


## Design

### Precise data-binding

ReactJS requires users provide a `render` function for each component.
The `render` function should map `props` and `state` to a ReactJS's virtual DOM,
then ReactJS framework creates a DOM with the same structure as the virtual DOM.

When `state` changes, ReactJS framework invokes `render` function to get a new virtual DOM.
Unfortunately, ReactJS framework does not precisely know what the `state` changing is.
ReactJS framework has to compare the new virtual DOM and the original virtual DOM,
and guesses the changeset between the two virtual DOM,
then apply the guessed changeset to the real DOM as well.

For example, after you prepended a table row, say, `<tr>`, into an existing `<tbody>` in a `<table>`,
ReactJS may think you changed the content of the every existing `<tr>`s of the `<tbody>`,
and appended another `<tr>` at the tail of the `<tbody>`.

The reason is that the `render` function for ReactJS does not describe the relationship between `state` and DOM.
Instead, it describes the process to create a virtual DOM.
As a result, the `render` function does not provide any information about the purpose of the `state` changing,
although a data-binding framework should need the information.

Unlike ReactJS, a Binding.scala `@dom` method is NOT a regular function.
It is a template that describes the relationship between data source and the DOM.
When partial of the data source changed, Binding.scala framework knows the exact partial DOM corresponding to the partial data.
Then, Binding.scala only re-evaluates partial of the `@dom` method to update the partial DOM.

With the help of the ability of precise data-binding provided by Binding.scala,
you can get rid of concepts for hinting ReactJS's guessing algorithm,
like `key` attribute, `shouldComponentUpdate` method, `componentDidUpdate` method or `componentWillUpdate` method.

### Composability

The smallest composable unit in ReactJS is a component.
It is fair to say that a React component is lighter than an AngularJS controller,
while Binding.scala is better than that.

The smallest composable unit in Binding.scala is a `@dom` method.
Every `@dom` method is able to compose other `@dom` methods via `.each`.

``` scala
case class Contact(name: Var[String], email: Var[String])

@dom
def bindingButton(contact: Contact): Binding[Button] = {
  <button
    onclick={ event: Event =>
      contact.name := "Modified Name"
    }
  >
   Modify the name
  </button>
}

@dom
def bindingTr(contact: Contact): Binding[TableRow] = {
  <tr>
    <td>{ contact.name.each }</td>
    <td>{ contact.email.each }</td>
    <td>{ bindingButton(contact).each }</td>
  </tr>
}

@dom
def bindingTable(contacts: BindingSeq[Contact): Binding[Table] = {
  <table>
    {
      for (contact <- contacts) yield {
        bindingTr(contact).each
      }
    }
  </table>
}

@JSExport
def main(): Unit = {
  val data = Vars(Contact(Var("Yang Bo"), Var("yang.bo@rea-group.com")))
  dom.render(document.body, bindingTable(data)
}
```

You may find out this approach is much simpler than ReactJS, as:

 * Instead of passing `props` in ReactJS, you just simply provide parameters for  Binding.scala.
 * Instead of specifying `propTypes` in ReactJS, you just simply define the types parameters in Binding.scala.
 * Instead of raising a run-time error when types of props do not match in ReactJS, you just check the types at compile-time.

### Lifecycle management for data-binding expressions

The ability of precise data-binding in Binding.scala requires listener registrations on the data source.
Others reactive frameworks that has the ability ask users manage the lifecycle of data-binding expressions.

For example, [MetaRx](https://github.com/MetaStack-pl/MetaRx/issues/45) provide a `dispose` method to unregister the listeners created when building data-binding expressions.
The user of MetaRx have the responsibility to call `dispose` method for every `map` and `flatMap` call after the expression changes,
otherwise MetaRx leaks memory. Unfortunately manually `dispose` everything is almost impossible for complicated binding expression.

Another reactive web framework [Widok](https://github.com/widok/widok/issues/29) did not provide any mechanism to manage lifecycle of of data-binding expressions.
As a result, it simply always leaks memory.

Because ReactJS does not have the ability of precise data-binding,
it does not register listeners to data source hence no memory leaks issue for simple cases.

Unfortunately, ReactJS provides `ref` attribute, `getInitialState` method, `componentWillMount` method, and `componentDidMount` method,
encouraging users create operations with side-effects in these methods, which are usually error-prone.

Unlike MetaRx or Widok, all data-binding expressions are pure functional, with no side-effects in Binding.scala.
Binding.scala does not register any listeners when users create individual expressions,
thus users neither need to manually unregister listeners for a single expression like MetaRx,
nor perform additional operations in handlers like ReactJS.

Instead, Binding.scala create all internal listeners together,
when the user calls `dom.render` or `Binding.watch` on root of expressions.
Note that `dom.render` or `Binding.watch` manages listeners on all sub-expressions,
not only the listener on the root expression.

In brief, Binding.scala separates functionality in two kinds:
 * `@dom` methods that produce pure functional expressions with no side-effects.
 * `dom.render` or `Binding.watch` manages all side-effects automatically.

### HTML literal and statically type checking

As you see, you can embed HTML literal in `@dom` methods in Scala source files.
You can also embed Scala expressions in braces in content or attribute values of the HTML literal.

``` scala
@dom
def notificationBox(message: String): Binding[Div] = {
  <div className="notification" title={ s"Tooltip: $message" }>
    {
      message
    }
  </div>
}
```


Regardless the similar syntax of HTML literal between Binding.scala and ReactJS,
Binding.scala create real DOM instead of ReactJS's virtual DOM.


In the above example, `<div>...</div>` create a DOM element with type of `org.scalajs.dom.html.Div`.
And the magic `@dom` let the method wrap the result in a `Binding`.

You can even assign the `Div` to a local variable and invoke native DOM method on the variable:

``` scala
@dom
def notificationBox(message: String): Binding[Div] = {
  val result: Div = <div className="notification" title={ s"Tooltip: $message" }>
    {
      message
    }
  </div>

  result.scrollIntoView()

  result
}
```

`scrollIntoView` method will be invoked when the `Div` is created.
If you invoked a method that does not defined in `Div`,
the Scala compiler will report an compile-time error instead of bringing the failure to run-time,
because Scala is a statically typed language and the Scala compiler understand the type of `Div`.

You may also notice `className` and `title`. They are DOM properties on `Div`.
They are type-checked by Scala compiler as well.

For example, given the following `typo` method:

``` scala
@dom
def typo = {
  val myDiv = <div typoProperty="xx">content</div>
  myDiv.typoMethod()
  myDiv
}
```

The Scala compiler will report errors like this:

```
typo.scala:23: value typoProperty is not a member of org.scalajs.dom.html.Div
        val myDiv = <div typoProperty="xx">content</div>
                     ^
typo.scala:24: value typoMethod is not a member of org.scalajs.dom.html.Div
        myDiv.typoMethod()
              ^
```

With the help of static type system, `@dom` methods can be much robuster than ReactJS components.

## Downloads

Binding.scala has an extremely tiny code base. There is only one source file [Binding.scala](https://git.realestate.com.au/rca/Binding.scala/blob/master/core/src/main/scala/au/com/realcommercial/binding/Binding.scala) for data-binding expressions,
and one source file [dom.scala](https://git.realestate.com.au/rca/Binding.scala/blob/master/dom/src/main/scala/au/com/realcommercial/binding/dom.scala) for HTML DOM integration.

### Core data-binding expressions (Binding.scala)

The former module is available for both JVM and Scala.js. You could add it in your `build.sbt`.

``` scala
// For JVM projects
libraryDependencies += "com.thoughtworks.binding" %% "core" % "1.0.5"
```

``` scala
// For Scala.js projects, or JS/JVM cross projects
libraryDependencies += "com.thoughtworks.binding" %%% "core" % "1.0.5"
```

* [The API documentation of `core` module](https://oss.sonatype.org/service/local/repositories/releases/archive/com/thoughtworks/binding/core_2.11/1.0.5/core_2.11-1.0.5-javadoc.jar/!/index.html)

### HTML DOM integration (dom.scala)

The latter module is only available for Scala.js. You could add it in your `build.sbt`.

``` scala
// For Scala.js projects, or JS/JVM cross projects
libraryDependencies += "com.thoughtworks.binding" %%% "dom" % "1.0.5"
```

* [The API documentation of `dom` module](https://oss.sonatype.org/service/local/repositories/releases/archive/com/thoughtworks/binding/dom_sjs0.6_2.11/1.0.5/dom_sjs0.6_2.11-1.0.5-javadoc.jar/!/index.html)
