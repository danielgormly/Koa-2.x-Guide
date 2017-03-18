# Koa 2.x In-depth (WIP)

#### Contents

1. Prerequisites

2. Forward

3. Changes from Koa 1.x

4. Getting Started

#### Prerequisites
A basic understanding of asynchronous JavaScript including the event loop, Promises, Generator functions & Async/Await. [YDKJS Book
5](https://github.com/getify/You-Dont-Know-JS/tree/master/async%20&%20performance) provides a fantastic & substantial primer for JS asynchrony. Honestly if you're at all interested in understanding JS and don't have a good handle on asynchrony, there's really no excuse to not read this book. If you are very familiar with Generator functions but don't know Async/Await, the latter will be very easy to pick up. A slightly reductive but practical way to think of them is as self-iterating generator functions.

#### Forward
[Koa](http://koajs.com/) is a lightweight, elegant web application framework for NodeJS. Its major change from Koa 1.x is its utilisation of the ES7 Async/Await feature available in Node 7.6 without flags, meaning that it works right out of the box with a default installation.

I'm writing this guide as most writings on Koa gloss over its admittedly simple mechanisms that to some, might be self-evident, but to less seasoned coders might feel a little too magical. Like most things in JS, it's very easy to jump 

#### Changes from Koa 1.x

I won't cover much of Koa 1.x and it's not necessary in understanding Koa 2.x. Instead of relying on the co-routine module [co](https://github.com/tj/co)
to resolve promises with a function call
to the next middleware, Koa relegates this to the engine (Node). This almost
certainly entails superior performance thanks to the possibility of engine
optimisations. It also likely means a smaller app footprint. In general this
also means less obscure and more portable middleware thanks to adhering to ECMA
standardised methodologies. There maybe a couple of downsides too including co-routines ability to use recognise & resolve thunks.

## Getting Started

Let's build an incredibly simple Koa app.

1. Download NodeJS 7.6.0 or later. Earlier versions of Node (7.0.x - 7.5.x) let you use the `--harmony` flag in the command line to use this feature experimentally. A version manager like NVM will help you install multiple versions of Node.

2. Create a new directory and initialise a new NPM project using `npm init` or `yarn init`.

3. Make a new file `server.js`.

4. Write the app:

    const http = require('http'); // Require Node's HTTP module
    const koa = require('koa'); // Require Koa
    
    /* Koa 2 is defined with ES6 class syntax. Here we are defining a new instance of Koa. */
    app = new koa();
    
    /* Let's write our first piece of middleware */
    app.use(async (ctx, next) => {
      console.log('A request was made');
    });
    
    /* Listen for requests over the HTTP protocol on port 8000 */
    http.createServer(app.callback()).listen(8000);

#### Quick refresh on using Koa 2 as a consumer

Let’s make a basic Koa 2 app. Obviously we need to  (or  (pleb)) first.

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

So when we run this and send *any* HTTP request to this app, we get the output:

    1. This
    2. is
    3. a
    4. Koa
    5. 2
    6. app

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

    1. This
    2. is
    3. a
    4. app.
    5. Koa
    6. 2

This is pretty wild and isn’t easily discerned without a little bit of knowledge
of JS’s event loop. What is happening here, is that each call to  is being
evaluated synchronously. Once we reach the last piece of middleware, we then 
once more. We are not just executing synchronous code anymore, we have
implicitly converted the expression returned by await to a promise. Just like
using a , the await keyword means we will not execute proceeding code until the
current thread has completed, i.e. at this point we defer code after our await
statements to the job queue, which will execute after our first piece of
middleware finishes.