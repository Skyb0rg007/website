---
title: Concurrent ML
draft: true
---

# Concurrent ML

Comparing CML to other parallel paradigms.

## What is Concurrent ML

Concurrent ML is a library designed by John Reppy for writing concurrent
software.
In the time it was designed, applications that would *obviously* use some
form of event loop such as GUIs were written in a naïve blocking fashion.
The CML paper and library was a method to specify how different "threads"
could communicate cooperatively, with the primitive being a synchronous
meeting between these threads.

## How does this compare to today?

The design of concurrent interfaces today is mostly concerned with
*parallelism*: how do I take advantage of all the cores on my machine;
how do I send multiple network requests at once then wait for all of them
to come back no matter the order.
Here the big innovation in terms of language design is `async`/`await`.

### Aside: what is `async`/`await` *really*?

There's a concept in software engineering known as
[inversion of control][wikipedia-ioc].
For our purposes, this is a way to translate between two different kinds of
library APIs.

For a real example, let's look at how `lex` and `yacc` usually interact.
I'm going to use SML syntax since the actual details don't matter.

```sml
type source (* The input source *)
type token  (* The result of lexical analysis *)
type ast    (* The result of parsing *)

val lex : source -> token
val parse : (unit -> token) -> ast

(* Example use:
 *
 * val source = Source.fromFile "input.txt"
 * val ast = parse (fn () => lex source)
 *)

```

Importantly, you can see that the `parse` function relies on you passing
it the lexing function, which it calls itself.

Let's say you are trying to parse some file, but the file actually isn't
available all at once.
You'd like to take the incoming data, process whatever you can,
but then simply put that process on hold until more data comes in.
In a language like C, this API is unsuitable for this task, as the
number of times the lexer is called is controlled by the parser when it
should really be controlled by the file manager.

Bison now has a push-based model which can help solve this.
The API generated with those options set looks more like this:

```sml
type parse_state
datatype step = Done of ast | Waiting of parse_state

val new_parse_state : unit -> step
val step_parser : parse_state * token -> step
```

The speed at which a parser processes tokens is now up to the caller,
not the parser.
When the parser has read enough input to return the result, it does so,
otherwise a result of `Waiting` is a signal that more tokens need to be
supplied.
This API is more general, and an implementation of this second design
can be translated into the first's API.

```sml
fun parse (lex : unit -> token) : ast = let
      fun loop state =
            case step_parser (state, lex ()) of
                Done ast => ast
              | Waiting state' => loop state'
      in
        case new_parse_state () of
            Done ast => ast
          | Waiting state => loop state
      end
```

Having a different option for these two cases is incredibly important for
languages like C which do not have *delimited continuations*.

### Delimited continuations

In the simplest form, there are two functions: `shift` and `reset`:

```sml
type 'a prompt

val reset : ('a prompt -> 'a) -> 'a
val shift : 'a prompt -> (('b -> 'a) -> 'a) -> 'b
```

The `reset` function allocates a *prompt*, then executes its argument.
If no calls to `shift` are made the function's result is returned and nothing
special happens.
You can imagine `reset` as storing a special tag in the call stack to be
utilized in potential future calls to `shift`.
The `shift` function is where the magic happens.
It will in some sense search up through the call stack until the marker that
was put there by the corresponding `reset` is found.
It then takes sets that set of frames from now until then and moves it aside.
Because the corresponding `reset` returns `'a`, and the call to `shift` returns `'b`,
this section of the call stack represents a function whose type is `('b -> 'a)`.
This part of the call stack is now available as its own first-class object.

### Back to the parser example

How does this help unify these two previous ideas?

```sml
(* Converting the first style into the second *)
(* val parse : (unit -> token) -> ast *)

type parse_state = (token -> ast)
datatype step = Done of ast | Waiting of parse_state

fun new_parse_state () : step = reset (fn (p : ast prompt) =>
      fun myLexer () = shift p (fn (k : token -> ast) =>
            Waiting k)
      in
        Done (parse myLexer)
      end)

fun step_parser (s : parse_state, tok : token) =
      s tok
```

### What does this have to do with concurrency?

Promise-based APIs require users to constantly program in callback-passing
style, and extensive use of this pattern has ben termed *callback hell*.
The most commonly used primitive in async programming with promises is
`then : 'a promise * ('a -> 'b promise) -> 'b promise`, and requires you
to nest your function calls every time your function does anything.
`async`/`await` is a syntactical transformation that gives users the
power of continuations without needing to implement the runtime to support it.

### Why am I talking about this?

I think this is important context for Concurrent ML in comparison to modern
considerations on how programming languages implement concurrency.
Nowdays, when a language "adds concurrency", this often means their libraries
utilize promises, and that the surface syntax of the language helps with this.

```sml
(* Using promises *)
infix andThen
fun makeRequest (uri : string) : json promise =
      fetch uri andThen (fn result =>
      respBody result andThen (fn body =>
      parseJSON body))

(* Using a promises library, but in direct style *)
fun makeRequest (uri : string) : json = let
      val result = fetch uri
      val body = respBody result
      in
        parseJSON body
      end
```

With promises, everything is asynchronous.
This is a nice interface for clients, where the interface is based around
you *receiving* data.
One issue you can run into with unbounded asynchronous channels is backpressure,
where data is received faster than it is processed.
If you're only requesting data, you will rarely run into this problem
because you will receive data only when asked.
It falls apart when your application acts like a server.

For example, say your JavaScript frontent opens up a WebSocket to process
logs sent by some server process which you then render into an HTML table.
This works beautifully when logs are coming in 5 per second, but if the
server suddenly starts sending thousands of requests a second the loop you
wrote to process the data is not going to keep up.

<!-- It falls apart when you want to implement server-like operations locally. -->
<!-- For example, let's say you wanted to implement a -->

### How does this differ from CML?



[wikipedia-ioc]: https://en.wikipedia.org/wiki/Inversion_of_control
[backpressure-explained]: https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7
