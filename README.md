# Ravel
[![npm version](https://badge.fury.io/js/ravel.svg)](http://badge.fury.io/js/ravel) [![Build Status](https://travis-ci.org/raveljs/ravel.svg?branch=master)](https://travis-ci.org/raveljs/ravel) [![Coverage Status](https://coveralls.io/repos/raveljs/ravel/badge.svg?branch=master)](https://coveralls.io/r/raveljs/ravel?branch=master) [![Dependency Status](https://david-dm.org/raveljs/ravel.svg)](https://david-dm.org/raveljs/ravel)

Forge past a tangle of node.js modules. Make a cool app.

## Introduction

Ravel is a tiny, sometimes-opinionated foundation for rapidly creating complex, highly-scalable [node](https://github.com/joyent/node) applications.

Layered on top of such fantastic technologies as [Express](https://github.com/strongloop/express), [Primus](https://github.com/primus/primus), [Passport](https://github.com/jaredhanson/passport), [Intel](https://github.com/seanmonstar/intel) and [Redis](https://github.com/antirez/redis), Ravel aims to provide a pre-baked, well-tested and highly modular solution for constructing enterprise web applications by providing:

 - Dependency injection
 - A set of well-defined architectural components
 - Automatic database transaction management
 - Authentication and authorization with transparent handling of mobile (i.e. non-web) clients
 - Standards-compliant REST API definition
 - Websocket-based front-end model synchronization
 - Easy security, via an enforced, reference configuration of [Express](https://github.com/strongloop/express)
 - Horizontal scalability

## Installation

    $ npm install ravel

Ravel also needs [Redis](https://github.com/antirez/redis). As part of the 1.0 release, a reference project including a [Vagrant](https://www.vagrantup.com/) development VM will be provided as a [Yeoman](http://yeoman.io/) generator, but for now you'll need to install Redis somewhere yourself.

## Ravel Architecture

Ravel applications consist of four basic parts:

### Modules

Plain old node.js modules encapsulating business logic, supporting dependency injection of core Ravel services, other modules and npm dependencies.

### Resources

What might be referred to as a *controller* in other frameworks, a Resource module defines HTTP methods on an endpoint, supporting the session-per-request transaction pattern via fancy middleware. Also supports dependency injection, allowing for the easy creation of RESTful interfaces to your module-based business logic, as well as front-end model synchronization via websockets.

### Routes

Only supporting GET requests, routes are used to serve up content such as EJS/Jade templates. Everything else should be a Resource.

### Rooms

Websocket 'rooms' which users may subscribe to, supporting authorization functions. Designed from the start to work in a clustered setting, as long as you're careful to use a reverse proxy supporting sticky sessions!

## Building a Simple Ravel Application

### Make a Module

Business logic sits in plain old node.js modules, which are generally not network-aware. Ravel modules are most powerful when they are factory functions which support **Dependency Injection**, though plain object modules are supported as well. Cyclical dependencies are automatically detected and prevented between modules using dependency injection.

*modules/cities.js*

    // Ravel error and logging services $E and $L can
    // be injected alongside your own modules and
    // npm dependencies. No require statements or
    // relative paths!
    module.exports = function($E, $L, async) {
      var Cities = {};
      var c = ['Toronto', 'New York', 'Chicago'];

      Cities.getAllCities = function(callback) {
        // pretend we used async for something here
        // since we magically injected it above
        callback(null, c);
      };

      Cities.getCity = function(name, callback) {
        var index = c.indexOf(name);
        if (index) {
          callback(null, c[index]);
        } else {
          $L.warn('User requested unknown city ' + name);
          // callback with an error from $E, and Resources will
          // be able to respond with appropriate HTTP status codes
          // automatically via $Rest (see below)
          callback(new $E.NotFound('City ' + name + ' does not exist.'), null);
        }
      };

      return Cities;
    };

To register and name your module, we need a top-level *app.js* file:

*app.js*

    var Ravel = new require('ravel')();
    //...we'll initialize Ravel with some important parameters later

    // supply the path to this module. you'll be able to inject
    // it into other modules using the name 'cities'
    Ravel.module('./modules/cities');

Note: Modules with filenames including hyphens are made available for injection via camel case  (e.g. './modules/my-module.js' would be available as 'myModule').

Another note: Modules also support array notation for injecting npm dependencies which have invalid js variables names (such as 'bad.module'). To use this notation, define your module like this:

    module.exports = ['bad.module', function(badModule) {
      ...
    }];

Yet another note: You can recursively import a directory of modules via:
```js
Ravel.modules('/modules/directory/path');
```

### Then, define a Resource

Resources are a special kind of module which help you build REST endpoints to expose your business logic. They support Express middleware and are designed to make it easy to adhere to the proper REST semantics.

*resources/city.js*

    // Resources support dependency injection too!
    // $Resource is unique to resources, and
    // notice that we have injected our cities
    // module by name.
    module.exports = function($E, $L, $Resource, $Rest, cities) {

      // Register this Resource with the base path /cities
      $Resource.bind('/cities');

      // will become GET /cities
      $Resource.getAll(function(req, res) {
        // $Rest makes it easy to build RESTful responses with
        // proper status codes, headers, etc. More on this later.
        cities.getCities($Rest.respond(req, res));
      });

      // will become GET /cities/:id
      $Resource.get(
        function(req, res, next) {
          //some middleware
          next();
        },
        function(req, res) {
          cities.getCity(req.params['id'], $Rest.respond(req, res));
        }
      );

      // post, put, putAll, delete and deleteAll are
      // also supported. Not specifying them for
      // this resource will result in calls using
      // those verbs returning HTTP 501 NOT IMPLEMENTED

      // postAll is not supported, because that's stupid.
    };

Like before, we need to register our resource:

*app.js*

    var Ravel = new require('ravel')();
    //...we're still getting to this part

    Ravel.module('./modules/cities');
    // Specify the base endpoint (/cities), and the location of the resource module
    Ravel.resource('./resources/city');

Note: You can recursively import a directory of resources via:
```js
Ravel.resources('/resources/directory/path');
```

### Add a Route for good measure

Routes are used to serve content such as EJS/Jade templates. Assuming you have a template index file at views/index.ejs:

*views/index.ejs*

    <!DOCTYPE html/>
    <html>
      <!-- Put whatever content or EJS stuff you want here -->
    </html>

*routes/index_r.js*

    module.exports = function($E, $L, $RouteBuilder) {
      $RouteBuilder.add('/', $PrivateRedirect, function(req, res) {
        res.render('index', {});
      });
    };

Once again, register the routes:

*app.js*

    var Ravel = new require('ravel')();
    //Since we're using EJS, we need to tell Ravel some things
    Ravel.set('express view directory', 'views');
    Ravel.set('express view engine', 'ejs');
    //...we're still getting to this part

    Ravel.module('./modules/cities');
    Ravel.resource('./resources/city');
    //Just specify the location of the routes and Ravel will load them
    Ravel.routes('./routes/index_r.js');

### Make Room for more

Websocket Rooms are topic *patterns* which represent a collection of topics to which clients can subscribe. They support path parameters and custom authorization functions. Furthermore, if a room is specified with a path which matches the path of a Resource, model updates will automatically be published to that room. Don't worry, we'll cover this later - for now let's just make a simple room in *app.js*:

*app.js*

    var Ravel = new require('ravel')();
    Ravel.set('express view directory', 'views');
    Ravel.set('express view engine', 'ejs');
    //...we're still getting to this part

    Ravel.module('./modules/cities');
    Ravel.resource('./resources/city');
    Ravel.routes('./routes/index_r.js');

    //define a chat room pattern representing
    //rooms such as /chatroom/1, /chatroom/30
    Ravel.room('/chatroom/:chatroomId', function(userId, params, callback) {
      //auth function
      //should the user be allowed to join the given room?
      //for /chatroom/30 params will contain: {chatroomId:30}
      callback(null, true);
    });

### Ravel.init() and Ravel.listen()

After defining all the basic components of a Ravel application, we need to initialize it with Ravel.init(). This step configures Express, Primus and other base technologies, and instantiates your defined Modules, Resources, Routes and Rooms.

After Ravel.init(), a *post init* event is emitted which can be used as a hook (via Ravel.on()) to perform other startup logic before spinning up the actual web server with Ravel.listen().

We've been avoiding some mandatory Ravel.set() parameters up until now, including Redis connection parameters and a session secret, so let's add them in now.

*app.js*

    var Ravel = new require('ravel')();
    Ravel.set('express view directory', 'views');
    Ravel.set('express view engine', 'ejs');
    //Here are those extra parameters we mentioned before
    Ravel.set('redis host', 'localhost');
    Ravel.set('redis port', 5432);
    Ravel.set('express session secret', 'a very random string');

    Ravel.module('./modules/cities');
    Ravel.resource('./resources/city');
    Ravel.routes('./routes/index_r.js');

    Ravel.room('/chatroom/:chatroomId', function(userId, params, callback) {
      callback(null, true);
    });

    //Instantiate and configure everything
    Ravel.init();
    //Anything listening to post init via Ravel.on('post init') will fire here

    //Now, we actually start the web server
    Ravel.listen();

That's it! Assuming redis is running at the given host and port, your server should start and you should be able to access the /cities endpoint at [localhost:8080/cities](http://localhost:8080/cities)


## A more complex example

*"You mentioned transactions! Authentication and authorization! Mobile-ready APIs! Front-end model synchronization! Get on with the show, already!" --Some Impatient Guy*

TODO

## API Reference

TODO

## Deploying and Scaling Ravel Applications

TODO
