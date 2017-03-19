# Koa 2.x In-depth introductory guide

#### Foreword
[Koa](http://koajs.com/) is a lightweight, elegant web application framework for NodeJS based on the. I'm writing this guide as most writings on Koa gloss over its admittedly simple mechanisms that to some, might be self-evident, but to less seasoned coders might feel a little too magical. Like a lot of user friendly JS interfaces, it's very easy to jump into with very little idea of what you're really doing, leading to convoluted patterns, unexpected behaviour & generally bad practices. Koa's internal code however, is actually very easy to reason about and with a thorough examination, can teach a beginner quite a lot about the HTTP protocol, NodeJS, developing web applications & dealing with asynchronous I/O.

#### Contents

1. Prerequisites

2. Forward

3. Changes from Koa 1.x

4. Getting Started

#### Prerequisites
A basic understanding of asynchronous JavaScript including the event loop, Promises, Generator functions & Async/Await will go a long way. [YDKJS Book 5](https://github.com/getify/You-Dont-Know-JS/tree/master/async%20&%20performance) provides a fantastic & substantial primer for JS asynchrony. Honestly if you're at all interested in understanding JS and don't have a good handle on asynchrony, there's really no excuse to not read this book. If you are very familiar with Generator functions but don't know Async/Await, the latter will be very easy to pick up. A slightly reductive but practical way to think of them is as self-iterating generator functions. You will also need an understanding of recursive programming. [EloquentJS Ch 05](http://eloquentjavascript.net/05_higher_order.html) has a good introduction to recursion that may take some time for beginners to grasp.

#### Changes from Koa 1.x

I won't cover much of Koa 1.x and it's not necessary in understanding Koa 2.x. Instead of relying on the co-routine module [co](https://github.com/tj/co) to resolve promises with a function call to the next middleware, Koa relegates this to the engine (Node), specifically utilising the ES7 feature Async/Await, available without flags from NodeJS 7.6.0 onwards. This almost certainly entails superior performance thanks to the possibility of engine  optimisations. It also likely means a smaller app footprint. In general this also means less obscure and more portable middleware thanks to adhering to ECMA standardised methodologies. One potential loss of dropping co-routines is the ability to recognise & resolve thunks.

## Getting Started

Let's build an incredibly simple Koa app. Koa will take any request & spit out a message to the console.

1. Download NodeJS 7.6.0 or later as Koa relies on Async/Await. Earlier versions of Node (7.0.x - 7.5.x) let you use the `--harmony` flag in the command line to use this feature experimentally. A version manager like NVM will help you install multiple versions of Node.
2. Create a new directory and initialise a new NPM project inside that folder using the command line command `npm init -yes` or `yarn init -yes` (See [Yarn vs NPM](https://blog.risingstack.com/yarn-vs-npm-node-js-package-managers/)). The 'yes' flag will produce a `package.json` file with all the default answers.
3. Install Koa with `npm install koa`, `npm i koa` (shorthand syntax) or `yarn add koa`
3. Make a new file `server.js`.
4. Write the app:
```
const http = require('http'); // Require Node's HTTP module
const koa = require('Koa'); // Require Koa

/* Koa 2 is defined with the ES6 class syntax so we need to instantiate it with the following line */
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

When we `const Koa = require('koa')`, we are telling Node to create a new [constant](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Statements/const) named Koa and assigning it to the module that require('koa') exports. Because there is no `./`, `/` or `../` preceeding `koa` in the require call and Koa isn't a core Node module, Node's module loader will look in `node_modules/koa/package.json` and seek out the `main` key (read [Node's Module API](https://nodejs.org/api/modules.html)). In the case of Koa, this is `node_modules/koa/lib/application.js`. The `node_modules/koa/lib/application.js` is the Koa package's public interface. It exports an ES6 class named `Application` - the internal name the Koa's developers have chosen for it.

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

Before we get any deeper into Koa, let's take a brief look at [Node's HTTP module](https://nodejs.org/api/http.html#http_class_http_server). This is how Koa is actually able to interact with the internet. The HTTP core module provides us with a convenient interface for reading HTTP requests, manipulating their contents and sending back appropriate responses. To use it, we first require the module, then we create a new server using its method `createServer()` and the argument that actually handles the request and finally we tell that server to listen on a particular port. Together, that looks like this:

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

The details here don't matter too much. The point is that ostensibly, this provides us with basically the exact same app as written above (there are some differences as we will see later). At this stage, we probably don't need Koa's overhead even at the benefit of hiding implementation details. As the app grows however, a callback based workflow and defining all behaviour [imperatively](http://stackoverflow.com/questions/1784664/what-is-the-difference-between-declarative-and-imperative-programming) won't cut it.

## Adding some sugar

```
const http = require('http'); // Require Node's HTTP module
const koa = require('Koa'); // Require Koa

/* Koa 2 is defined with the ES6 class syntax so we need to instantiate it with the following line */
app = new Koa();

/* Let's write our first piece of middleware */
    app.use(function() {
    console.log('Request made.');
});

/* And finally, let's use Koa to listen for requests over the HTTP protocol on port 8000 */
http.createServer(app.callback()).listen(8000);
```

The first area we can cut down on some typing while maintaining legibility is using ES6's arrow functions. We'll be using these from hereon out.


## Adding asynchrony




## Quick refresh on using Koa 2 as a consumer

Let’s make a basic Koa 2 app. Obviously we need to  (or  (pleb)) first.

```
koa = require('koa'); // We need Koa

app = new koa(); // Koa 2 is defined with ES6 class syntax
app.use(async (ctx, next) => {
    console.log('1. This');
    // Await all successive middleware before continuing:
    await next();
    console.log('6. app.');
});

app.use(async (ctx, next) => {
    console.log('2. is');
    // Await all successive middleware before continuing:
    await next();
console.log('5. 2');
});

app.use(async (ctx, next) => {
    console.log('3. a');
    // No more middleware left, we don't need a next() here but Koa will automatically fill it with null anyway:
    await next();
    console.log('4. Koa');
});

http.createServer(app.callback()).listen(3000);
/* Same as the short form sugar app.listen(3000) */
```

So when we run this and send *any* HTTP request to this app, we get the output:

```
1. This
2. is
3. a
4. Koa
5. 2
6. app
```

#### What is happening here?

As they are defined in your code, each piece of middleware you register with 
gets added to the middleware stack (a simple array) in the order you register
them in.

The stack is consumed by our app.callback() handler defined above. Firstly, each
time the server receives a request, Koa will first create a very neatly
organised (context) object with:

1.  Getters that allow you to access HTTP headers & body (though you’ll need
middleware to do anything useful with the body)
1.  Setters that allow you to set parameters for responding to the requester.

Then Koa will call the Middleware stack. Koa calls each middleware function one
by one, recursively from the previous function. This means that each next() call
inside any middleware will be the initiator for every successive middleware.
Thus if we call  on our first middleware (as we have), we are telling Koa to
wait for *all* the middleware that follows to complete before we continue
executing code.

What if we took out the  keyword and just ran  solo? Let’s try it on the first
middleware. The output we get back is:

```
1. This
2. is
3. a
4. app.
5. Koa
6. 2
```

This is pretty wild and isn’t easily discerned without a little bit of knowledge
of JS’s event loop. What is happening here, is that each call to  is being
evaluated synchronously. Once we reach the last piece of middleware, we then 
once more. We are not just executing synchronous code anymore, we have
implicitly converted the expression returned by await to a promise. Just like
using a , the await keyword means we will not execute proceeding code until the
current thread has completed, i.e. at this point we defer code after our await
statements to the job queue, which will execute after our first piece of
middleware finishes.