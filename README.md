# Koa 2.x Extensive Introductory Guide

#### Foreword
[Koa](http://koajs.com/) is a lightweight, elegant web application framework for NodeJS leaning heavily on the ES7 asyc/await function. I'm writing this guide as most writings on Koa gloss over its admittedly simple mechanisms that to some, might be self-evident, but to less seasoned coders might feel a little too magical. Like a lot of user friendly JS modules, it's very easy to jump into with very little idea of what you're really doing, leading to convoluted patterns, unexpected behaviour & generally bad practices. Koa's internal code however, is actually quite easy to reason about and with a thorough examination, can teach a beginner quite a lot about ES6 & 7 features, the HTTP protocol, NodeJS, developing web applications & dealing with asynchronous I/O.

#### Contents

1. Prerequisites
2. Changes from Koa 1.x

Part 1: How Koa Works

3. Getting started
4. Node's HTTP interface
5. Adding some sugar
6. What is middleware?
7. Orchestrating middleware
8. Working asynchronously
9. The context object
10. Examing real Koa code

Part 2: Using Koa practically

11. Content negotiation & headers
12. Getting the body of an HTTP request
13. Routing
14. Templating
15. Cookies & sessions
17. Testing
18. Debugging
19. Afterword

#### Prerequisites

I will try to add information useful for relative beginners to JavaScript.  However, a basic understanding of asynchronous JavaScript including the event loop, Promises, Generator functions & Async/Await will go a long way. [YDKJS Book 5](https://github.com/getify/You-Dont-Know-JS/tree/master/async%20&%20performance) provides a fantastic & substantial primer for JS asynchrony. Honestly if you're at all interested in understanding JS and don't have a good handle on asynchrony, there's really no excuse to not read this book. If you are very familiar with Generator functions but don't know Async/Await, the latter will be very easy to pick up. A slightly reductive but practical way to think of them is as self-iterating generator functions. You will also need an understanding of recursive programming. [EloquentJS Ch 05](http://eloquentjavascript.net/05_higher_order.html) has a good introduction to recursion that may take some time for beginners to grasp.

#### Changes from Koa 1.x

I won't cover much of Koa 1.x and it's not necessary in understanding Koa 2.x. Instead of relying on the co-routine module [co](https://github.com/tj/co) to resolve promises with a function call to the next middleware, Koa relegates this to the engine (Node), specifically utilising the ES7 feature Async/Await, available without flags from NodeJS 7.6.0 onwards. This almost certainly entails superior performance thanks to the possibility of engine  optimisations. It also likely means a smaller app footprint. In general this also means less obscure and more portable middleware thanks to adhering to ECMA standardised methodologies. One potential loss of dropping co-routines is the ability to recognise & resolve thunks.

## Getting Started

Let's build an incredibly simple Koa app. Koa will take any request & spit out a message to the console.

1. Download NodeJS 7.6.0 or later as Koa relies on Async/Await. Earlier versions of Node (7.0.x - 7.5.x) let you use the `--harmony` flag in the command line to use this feature experimentally. A version manager like NVM will help you install multiple versions of Node.
2. Create a new directory and initialise a new NPM project inside that folder using the command line command `npm init -yes` or `yarn init -yes` (See [Yarn vs NPM](https://blog.risingstack.com/yarn-vs-npm-node-js-package-managers/)). The 'yes' flag will produce a `package.json` file with all the default answers. You may have to add the Node installation to your PATH variable if npm isn't available in the terminal.
3. Install Koa with `npm install koa`, `npm i koa` (shorthand syntax) or `yarn add koa`
3. Make a new file `server.js`.
4. Write the app:
```
const http = require('http'); // Require Node's HTTP module
const koa = require('Koa'); // Require Koa

/* Koa 2 is defined with the ES2015 class syntax so we need to instantiate it with the following line */
app = new Koa();

/* Let's write our first piece of middleware */
app.use(function() {
    console.log('Request made.');
});

/* And finally, let's use Koa to listen for requests over the HTTP protocol on port 8000 */
http.createServer(app.callback()).listen(8000);
```

5. Run the app with `node server`.

6. Hit [localhost:8080](http://localhost:8080) or [127.0.0.1:8080](http://127.0.0.1:8080) with your web browser

If all has gone well, you should see "Not found" in your browser & "A request was made" in your terminal window. If you see a second "A request was made", it might have been your browser requesting a favicon for the domain. You're seeing "Not found" because we haven't actually sent anything back to the browser. Koa has run through the function it received in `app.use`

###### Notes on `require('koa')`

When we `const Koa = require('koa')`, we are telling Node to create a new [constant](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Statements/const) named Koa and assigning it to the module that require('koa') exports. Because there is no `./`, `/` or `../` preceeding `koa` in the require call and Koa isn't a core Node module, Node's module loader will look in `node_modules/koa/package.json` and seek out the `main` key (read [Node's Module API](https://nodejs.org/api/modules.html)). In the case of Koa, this is `node_modules/koa/lib/application.js`. The `node_modules/koa/lib/application.js` is the Koa package's public interface. It exports an ES2015 class named `Application` - the internal name the Koa's developers have chosen for it.

When we import it, we are assigning it the name `Koa` with a capital "K" by convention, to signal to developers that this is a class or object to be instantiated, rather than a plain object. We use "Koa" rather than "Application" in our code because "Application" is confusing within our context.

###### Notes on Es6 classes and Koa's main `Application` class

By calling `const app = new Koa();`, we are instantiating a single, self-contained instance of Koa in our main local module scope i.e. `server.js`. app becomes a new object with very few of its own properties we don't need to know about, and importantly a [[Prototype]] (the double brackets syntax means this is an internal JS implementation detail, we technically don't have this available to us as a public API - although there are ways of finding this) link to `Koa.prototype`. This means that our `app` object can use all the methods defined in the `Application` class mentioned earlier.

Here is a brief look at the salient code of Koa's Application class:

```
class Application {
    constructor() { ... } // Instantiation details that don't interest us yet
    listen() {
    /* sugar for listening too callback with HTTP
    }
    use(fn) {
    /* code that adds middleware to the stack */
    }
    callback() {
    /* function passed to a Node HTTP server to route requests through registered middleware and deliver responses */
    }
}
```

When an ES6 class is instantiated, its methods are added to `ClassName.prototype.method` and a `[[Prototype]]` link is established from the new object to `ClassName.prototype`. Most implementations of JS let us view an object's `[[Prototype]]` directly with `__proto__`, test it with `isPrototypeOf()` or set it with `setPrototypeOf()`. The purpose of the [[Prototype]] link is that we can use all of the parent's methods on the child object directly. For those with a background in Class-based programming, it is important to remember that JS does not really classes but dynamic links to other objects.

In our case, `app` has a `[[Prototype]]` of `Koa.prototype`, thus we can use all the methods attached to `Koa.prototype` from `app` directly. This is why in general it is not a good idea to add new methods or properties to `app` - they may shadow some of Koa's inbuilt methods, preventing you from calling them from `app`.

The two methods we have used so far `Koa.prototype.use()` and `Koa.prototype.callback()`. As explained, we can use them as `app.use()` and `app.callback()`. I will explain this methods shortly.

If the language of class, objects, constructors, prototypes, `__proto__` and `[[Prototype]]` are deeply confusing to you - [YDKJS this & Object Prototypes](https://github.com/getify/You-Dont-Know-JS/tree/master/this%20%26%20object%20prototypes) was written just for you. [Mozilla Developer docs](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/proto) also provide a good refresher.

## Node's HTTP interface

Before we get any deeper into Koa, let's take a brief look at [Node's HTTP module](https://nodejs.org/api/http.html#http_class_http_server). This is how Koa is actually able to interact with the internet. The HTTP core module provides us with a convenient interface for reading HTTP requests, manipulating their contents and sending back appropriate responses. To use it, we first require the module, then we create a new server using its method `createServer()` and the argument that actually handles the request (the request listener) and finally we tell that server to listen on a particular port. Together, that looks like this:

```
const http = require('http');

const server = http.createServer(function (req, res) {
    console.log('Request made.');
    res.writeHead(200, {'Content-Type': 'text/plan' });
    res.write('Not found', 'utf8');
	res.end();
});

server.listen(8000);
```

The details here don't matter too much. The point is that ostensibly, this provides us with basically the exact same app as written above (there are some differences re. HTTP headers and methodology as we will see later). At this stage, we probably don't need Koa's overhead even at the benefit of hiding implementation details. As the app grows however, a callback based workflow and defining all behaviour [imperatively](http://stackoverflow.com/questions/1784664/what-is-the-difference-between-declarative-and-imperative-programming) won't cut it.

The second important thing to realise is that in our Koa app, Koa.prototype.callback() is our HTTP request listener. It is the callable object (function) that handles requests to our newly created HTTP server.

## Adding some sugar

```
const koa = require('Koa'); // Require Koa
app = new Koa();

app.use(() => console.log('Request made.'));

app.listen(8000);
```

The first area we can cut down on some typing while maintaining legibility is using ES6's arrow functions. We'll be using these from hereon out. When using an arrow function, we can optionally omit the curly braces if we are returning a single expression. The ommission of the `return` keyword is mandatory if ommitting curly braces.

The second think we can do is use Koa's `Koa.prototype.listen()` sugar. As the most frequent use of Koa will be through Node's `HTTP#createServer` object (as opposed to the `HTTPS#createServer`), listen is provided as a convenient method to listen on a new server using Koa. Here is the actual code for listen:

```
  listen() {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen.apply(server, arguments);
  }
  ```
  
  Very simple. It creates a server with the app.callback() app as its argument, as `this` references the object from which `listen` was called from, then calls the instantiated server's `listen` method with the arguments given to `app.listen()` e.g. `8000`. Because http is already required into `node_modules/koa/lib/application.js` we no longer need to include it.
  
```
const koa = require('Koa');
const app = new koa;
app.use(() => console.log('Request made.')).listen(8000);
```

We're also able to chain `app.use` calls as `Koa.prototype.use` returns the `this` - which is of course the `app` object in this case.

## What is middleware?

You've probably have used a bunch of middleware if you've come from Express. It's easy to consume middleware and have very little idea of what it's doing or even why you're using it. Middleware is quite a broad term depending on its context but it can be vaguely described as a chunk or chunks of code that sit between a request and a response.

In Koa, middleware can take information, transform that information and generate a response . Our simple piece of middleware in our app is `app.use(() => console.log('Request made.'))`. You've probably noticed that isn't hugely useful. We can't read the contents of the request, we can't send anything back to the request. In fact Koa defaults to the response `Not found` with the status code `404` (more on that in chapters x & y #TODO). 

Let's build a *slightly* more interesting piece of middleware using some Koa magic. This one simply returns 'Heyyyyyyyyyyyy.' to the user if and only if the requested path is '/greet' e.g. [localhost:8080/greet](http://localhost:8080/greet) is accessed in a web browser.

```
const greetMiddleware = (ctx, next) => {
    if (ctx.path = '/greet') ctx.body = 'Heyyyyyyyyyyyy.'
};
app.use(greetMiddleware);
```

Each time Koa uses a piece of middleware, it injects two arguments into it:
1. `ctx` (short for context), is an object that lets you interface with Node's  request AND the response object, made possible thanks to and JavaScript's setters and getters. We will explore this in detail in The Context Object. #TODO
2. `next` is actually a reference to the next function in the middleware stack. If zeri ir one piece of middleware remains, Koa injects an empty function in its place.

## Orchestrating middleware

Technically Koa can run on regular functions, however unless you only stick to `next()` calls as opposed to `await next()` calls this will lead to extremely confusing behaviour. To ensure maximum reusability, compatibility, consistency & predictability use async functions.  Let's take a look at using 2 pieces of middleware with regular functions:

```
const koa = require('koa');
const app = new koa();

// middleware #1
app.use(async (ctx, next) => {
    console.log('greetings');
    next();
    console.log('lit af, fam');
});

// middleware #2
app.use(async (ctx, next) => {
    console.log('how are things?');
    next();
});

app.listen(3000);
```

To give you a description of what is happening behind the scenes before revealing what .
1. When the Koa `app` is instantiated with `new Koa()`, a property named `middleware` is attached to the app object itself with an empty array.
**Note:** Koa v2 will attempt to convert legacy generator based middleware but support will be completely removed in v3.
2. Each `app.use` call pushes the piece of middleware called to it, synchronously, in the order they are called to `app.middleware`. Thus app.middleware = [middleware1, middleware2];
3. When a request is received, Node calls the `app.callback()` function. This argument calls each piece of middleware, starting at `app.middleware[0]` with the succeeding middleware as its argument until no more middleware remain. If next() is called on the final function, a blank function is returned, so nothing happens. Thus it is the `next()` call within the function that calls the following function. A simplified  version of `app.callback` might look like the following: (in reality, callback defers this behaviour to another function).

```
callback (ctx, index = 0) {
    const isLast = index === middleware.length - 1;
    let next = isLast ? () => {} : this.callback.bind(this, ctx, index + 1);
    return this.middleware[index](ctx, next);
}
```

So the next middleware function isn't injected directly, instead it injects a call to this iterator function which in turn, returns the next middleware function injected with a call to the next iterator function, and so on and so forth until there's no more middleware left to call. Recursion! We'll examine the actual code that is Async/Await friendly in The Koa engine #TODO.

Anyway, the output, hopefully you guessed it, is:

```
greetings
how are things?
lit af, fam
```

Now let's introduce a 3rd piece just to make sure that you're following:

```
// middleware #3
app.use(async (ctx, next) => {
    console.log('not great tbh');
    next();
});
```

The output is:

```
greetings
how are things?
not great tbh
lit af, fam
```

To expand the entire middleware stack out as it's called, in pseudocode:

```
INDEX = 0;
START:MIDDLEWARE[INDEX]
    LOG('greetings');
    START:MIDDLEWARE[INDEX+1]
        LOG('how are things?');
        START:MIDDLEWARE[INDEX+2]
            LOG('not great, tbh');
        END:MIDDLEWARE[INDEX+2]
    END:MIDDLEWARE[INDEX+1]
    LOG('lit af, fam');
END:MIDDLEWARE[INDEX]
});
```

I have uploaded a complete working model of this [MiniKoa](https://gist.github.com/danielgormly/ef05053d73c163e528ce151d3f3284bf). It might help to play around with it. Try switching around the order of the `console.log` calls and the `next()` calls. The core iterator, which we will look at later, works on the same principles and really isn't much more complicated than that.

Note: The real Koa protects us against calling next() twice from the same function.

## Working Asynchronously

We've been using Async functions for every piece of middleware so far but we haven't exploited any of their features that differentiate them from regular functions. For all the applications we've written so far we can omit the `async` keyword and the output will remain identical. Actually th

Async functions start to show their benefits when we want to do things like make database calls. They allow us to write incredibly legible and easy to reason about asynchronous code.

As mentioned, Async functions are the standard way to do things in Koa. If we start mixing `await next()` with unadorned `next()` calls, we begin getting confusing results. Let's not shy away from it though and see what happens.

This is pretty wild and isn’t easily discerned without a little bit of knowledge
of JS’s event loop. What is happening here, is that each call to  is being
evaluated synchronously. Once we reach the last piece of middleware, we then 
once more. We are not just executing synchronous code anymore, we have
implicitly converted the expression returned by await to a promise. Just like
using a , the await keyword means we will not execute proceeding code until the
current thread has completed, i.e. at this point we defer code after our await
statements to the job queue, which will execute after our first piece of
middleware finishes.

## Koa's engine

## The context object

So far, our examples have been ignoring the fact that we have clients who we want to get information from and serve information to. This is where the context object comes into play.

Http request and response headers are largely isomorphic.

## Content headers, negotiation & CORs

## Parsing

## Routing

## Templating

## Error handling

## Cookies & Sessions

## Testing

## Debugging

## Afterword

Thanks for reading!

## Sources

kick-off-koa
koa.js