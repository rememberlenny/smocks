smocks
==============

Stateful HTTP mocking service built on top of [HAPI](http://hapijs.com/).  Easily add routes and different scenarios for each route including the ability to maintain state with an admin interface to control everything.

With smocks you can

* create route definitions (with dynamic tokens)
* define multiple route handlers (variants) for for any route (selectable through an admin panel)
* add input configuration components for routes and variants (accessable through an admin panel)
* define actions which can manipulate the current state and be executed from the admin pnael
* use route request handlers that can keep a state for true dynamic mocking capabilities
* define global request handlers (variants) which can be selected for any route
* use plugins which can intercept all requests to perform actions
* use a RESTful API to make configuration changes programatically
* Replay exports from HAR files
* be able to use as a direct proxy to real service endpoints



Sections
--------------
### Starting the server
After you define your fixtures, you will want to start the server (or you can actually refer to your fixtures with an API if you want to use with unit tests).

There are 2 ways to start the server:

Starting the server directly
```
var smocksInstance = require('smocks')(_id_)...
smocksInstance.start({
  // options provided to Hapi.Server.start
  port: 8080,
  host: 'localhost'
}, {
  // smocks init options
}, optionalCallback);
```

Or, you can just export a Hapi plugin to be included elsewhere
```
var smocksInstance = require('smocks')(_id_)...
var plugin = smocksInstance.toPlugin({
  // hapi plugin options
  onRegister: function (server, options, next) {
    // this is optional but "next" must be called if used
  }
}, {
  // smocks init options
});

plugin.attributes = {
  pkg: require('/path/to/package.json')
};

module.exports = plugin;
```

The different smocks core options are discussed in detail below.


### Smocks init options

| Attribute | value |
|-----------|-------|
| initialState | (optional) the initial application state that should be applied when first loaded (or reset)
| har | see ***HTTP Archive Replay***
| proxy | see ***Proxying***


### Concepts
#### Admin Panel
Whenever the mock server is running, you can view an admin panel at ```{host}:{port}```.  Using this, you can

* Select what type of response your routes should have (see [Variants](#section/Concepts/Variants))
* Execute route specific actions (see Actions)
* View all available routes by label or path
* Set route specific or global input fields (see [Route / Variant Input](#section/Concepts/Route%2520%252F%2520variant%2520input))
* Save all current settings as a profile (selected route variant and input values) to be applied at a later time (see [Profiles](#section/Concepts/Profiles))

Throughout these docs, you will see different screenshots to understand how different route configurations are represented in the admin panel.



#### Routes
A route by itself is really nothing more than a [HAPI route](http://hapijs.com/api#serverrouteoptions).  The route definition has a ```handler``` attribute which is called when the endpoint is hit.  This handler is nothing more than a [HAPI route handler](http://hapijs.com/api#route-handler).

Routes are defined using the ```route``` method on the ```smocks``` object.  An object parameter is provided with the following attributes

* _id_: (optional) the route id - used for the RESTful admin API and profile settings
* _label_: (optional) the route label - used for display on the admin panel
* _path_: the route path
* _method_: (optional) the route method - defaults to ```GET```
* _group_: (optional) arbitrary group name to organize routes in the admin panel
* _handler_: (optional) the [HAPI route handler](http://hapijs.com/api#route-handler) which provides the route response.  This is optional because you could use multiple vairants to handle the response.  (see Variants).
* _input_: An object contain input values which should be shown in the admin panel (see Input)
* _display_: A function which can return [markdown]([markdown](http://daringfireball.net/projects/markdown/)) where the contents are exposed when viewing the route information on the amin panel
* _actions_: An object containing a set of actions associated with this route (see Actions)
* _config_: [HAPI route options](http://hapijs.com/api#route-options)
* _connection_: optional connection label for route to be added to. allows
  for smocks endpoint to run on port of choice, for instance.

In more detail, you can...

Organize your routes into logical groups
```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      id: 'group_1_1',
      group: 'group 1',
      path: '/api/group/1/1',

      handler: function (req, reply) {
        reply('I\'m in group 1');
      }
    })
    .route({
      id: 'group_1_2',
      group: 'group 1',
      path: '/api/group/1/2',

      handler: function (req, reply) {
        reply('I\'m also in group 1');
      }
    })
    .route({
      id: 'group_2_1',
      group: 'group 2',
      path: '/api/group/2/1',

      handler: function (req, reply) {
        reply('I\'m in group 2');
      }
    })
    .route({
      id: 'group_2_2',
      group: 'group 2',
      path: '/api/group/2/2',

      handler: function (req, reply) {
        reply('I\'m also in group 2');
      }
    });
```

![groups](http://jhudson8.github.io/smocks/images/groups.png)


Add input parameters that are exposed through the admin panel

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      // labels aren't required but make things easier to view in the admin panel for non-technical people
      id: 'my_route',
      label: 'My Route',
      path: '/api/foo',
      method: 'GET',

      input: {
        // more about config later
        agreementAccepted: {
          label: 'agreement accepted?',
          type: 'boolean',
          defaultValue: false
        }
      },

      handler: function (request, reply) {
        // you can control this value through the admin panel
        var agreementAccepted = this.input('agreementAccepted');
        reply({accepted: agreementAccepted});
      }
    })
```

![route ex1](http://jhudson8.github.io/smocks/images/route-ex1.png)


Provide multiple response types for each route (called Variants).  With the variants below, you can select which type of response the ```/api/foo``` route should respond with in the admin panel.  More about variants later...

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      id: 'my_route',
      path: '/api/foo',
      handler: function (request, reply) {
        // this is essentially the same as the "default" variant
        reply({firstName: 'John'});
      }
    })
    .variant({
      // in this case the label really isn't necesary since it would be the same as the id
      id: 'Billy',
      handler: function (request, reply) {
        reply({firstName: 'Billy'});
      }
    })
    .variant({
      // in this case the label really isn't necesary since it would be the same as the id
      id: 'Clark',
      handler: function (request, reply) {
        reply({firstName: 'Billy'});
      }
    })
```

![route ex2](http://jhudson8.github.io/smocks/images/route-ex2.png)


You can provide a display value which will be used when viewing the route details in the admin panel.  We haven't discussed state yet but this is meaningful to represent the current state of things for quick glance in the admin panel.  The admin panel supports [markdown](http://daringfireball.net/projects/markdown/) for your display response.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      // ...
      display: function () {
        return '* this will show up as a unordered list';
      }
    })
```

![route ex3](http://jhudson8.github.io/smocks/images/route-ex3.png)


You can expose "actions" which are represented as buttons.  These are meaningful to quickly make changes to the state of things.  Actions, like routes and variants, can accept config parameters which will allow you to input data required to perform the action.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      // ...
      input: {
        yourPhoneNumber: {
          label: 'What is your phone number?',
          type: 'text',
          defaultValue: '999.867.5309'
        }
      },

      // now define our action for the previous route
      actions: {
        'the_action_id': {
          label: 'the button label',
          input: {
            yourName: {
              label: 'What is your name?',
              type: 'text',
              defaultValue: 'John Doe'
            }
          },
          handler: function (input) {
            // this is how you access action specific user input
            var yourName = input.yourName;
            // this is how you access user input created for the route
            var phoneNumber = this.input('yourPhoneNumber');
            // now I would perform whatever action needs to be taken
            // I would make changes to "state" most likely (more about state later)
          }
        }
      }
    })
```

![route ex4](http://jhudson8.github.io/smocks/images/route-ex4.png)

You can use dynamic parameters in the route path, get access to query parameters and the body payload.  See [path parameters](http://hapijs.com/api#path-parameters) for more details.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      path: '/api/customer/{id}'
      handler: function (config) {
        // would be "123" if the endpiont hit was "/api/customer/123"
        var id = request.params.id;

        // would be "bar" if the endpoint hit was "/api/customer/123?foo=bar"
        var foo = request.query.foo;

        // would be "bar" if the posted body content (as JSON) was {"foo": "bar"}
        var foo = request.payload.foo;
      }
    })
```


#### Variants
We briefly touch on variants when discussing routes but variants are route handlers that can be selected by you in the admin panel (or with a RESTful API) to determine what type of response a route should have.

Routes are defined using the ```variant``` method on the ```Route``` object (returned by calling the ```route``` method.  An object parameter is provided with the following attributes

* _id_: (optional) the route id - used for the RESTful admin API and profile settings
* _label_: (optional) the route label - used for display on the admin panel
* _input_: An object contain input configuration data to be shown in the admin panel (see Input)
* _handler_: (optional) the [HAPI route handler](http://hapijs.com/api#route-handler) which provides the route response

Variants are useful because they allow yout test multiple scenarios that can happen with your route.  Say, for example, you have a route exposing the ability to update a password.  You might have several exceptional scenarios that you would want to test out (each could be a vairant that you simply select to tell the route handler to use the appropriate response)

* the password was reset successfully
* the password didn't pass validation
* the old password wasn't entered correctly
* the username doesn't exist
* and so on...

In more detail, you can...

Have multiple variants associated with a single route

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({...})

      .variant(...)

      .variant(...)

      .variant(...)
```


Add variant specific config parameters (only visible if the variant is selected as the active variant) that are exposed through the admin panel

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({...})

      .variant({
        id: 'invalid_password',
        label: 'invalid password',
        handler: function (request, reply) {
          // the input value is retrieved using this.input('varName');
          var typeOfValidationError = this.input('typeOfValidationError')
          reply({error: 'field', message: typeOfValidationError}).code(400);
        },
        input: {
          // the key is the identifier used to retrieve the value
          typeOfValidationError: {
            // the input field label
            label: 'What type of validation error?',
            // type can be "boolean", "text", "select", "multiselect"
            type: 'text',
            // the value shown if no selection has been made
            defaultValue: 'password too short'
          }
        }
      })
```


#### Route / variant input

Different types of input fields can be defined for routes or variants including ```boolean```, ```text```, ```select```, ```multiselect```.  Through the admin panel, you can modify these input values and they can be accessed in your route handler.

Routes and Variants when defined can provide an optional ```input``` attribute which defines any input fields that should be shown in the admin panel.  This input attribute is an object with each attribute relating to a single input field and the associated key as the input field id.

The attributes for each input field setting are

* _label_: the input field label
* _type_: the input field type (```boolean|text|select|multiselect```)
* _defaultValue_: the value for the input field when the user has not yet made a selection in the admin panel
* _options_: (specific to ```select``` and ```multiselect```) an array of options for the input field.  Each element in the array is an object with a ```value``` and ```label``` attribute.

Input values are referenced using ```this.input('varName')``` where ```varName``` is the specific input attribute key.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smock.route({
      ...
      input: {
        aBooleanField: {
          label: 'Is this a checkbox?',
          type: 'boolean',
          defaultValue: true
        },
        someTextField: {
          label: 'A text field',
          type: 'text',
          defaultValue: 'Hello'
        },
        someSelectBox: {
          label: 'A select box',
          type: 'select',
          options: [{label: 'One', value: 1}, {label: 'Two', value: 2}, {label: 'Three', value: 3}],
          defaultValue: 2
        },
        someMultiSelect: {
          label: 'A check list',
          type: 'multiselect',
          options: [{label: 'One', value: 1}, {label: 'Two', value: 2}, {label: 'Three', value: 3}],
          defaultValue: [2, 3]
        }
      },
      handler: function (request, response) {
        var aBooleanField = this.input('aBooleanField'); // boolean
        var someTextField = this.input('someTextField'); // string
        var someSelectBox = this.input('someSelectBox'); // integer (because the values are integers)
        var someMultiSelect = this.input('someMultiSelect'); // array of integer (because the values are integers)
        // ...
      });
```

![input example](http://jhudson8.github.io/smocks/images/config-ex1.png)

The same ```config``` values can be applied at the variant level as well.


#### Route meta data

This is similar to Route / variant input except that these values are not exposed within the admin console.  They are accessable within the route handlers though.

This is mostly useful for global plugins (see Plugins).

Within the route handler, the options values can be accessed by using ```this.meta('varName')```.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      ...
      meta: {
        requiresLogin: true
      },
      handler: function (request, reply) {
        // this doesn't make a lot of sense because the value is defined here
        // but would actually make more sense as a plugin... this is just showing
        // how you would (and could) get the meta value in a route handler
        var value = this.meta('requiresLogin');
        ...
      }
    );
```


#### State

The real benefit to using ```smocks``` is that state can be maintained.  Within any response handler, use ```this.state('varName')``` to access an object stored in the state and ```this.state('varName', 'varValue')``` where ```varValue``` can be any type of object you want.  There is a button on the admin panel which allows you to reset the state and start over.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      route: '/api/login',
      method: 'POST',
      handler: function (request, reply) {
        // now you can use this.state('loggedIn') in any route handler to see if the user has logged in
        this.state('loggedIn', true);
        reply().code(204);
      }
    });
```


#### Route / variant / plugin handlers

This has been discussed previously, but to have all handler context methods in a single place (things referenced using `this.`) in your handlers

| Attribute | Description |
|-----------|-------------|
| state | see ***State***
| meta | see ***Meta***
| input | see ***Input***
| route | the current `route object`
| variant | the currently selected `variant object`


#### Input, state and meta, oh my

There are 3 different ways of introducting dynamic behavior into your responses but each serve a different purpose.

_input_

Input values are exposed as input fields in the admin panel so you have the ability to change the value at runtime.

These are accessed using ```this.input('varName')``` in any route handler.

_state_

State is used to, obviously, maintain state.  For example, if you expose a route that adds a new piece of data, you should store it in state.  The user can reset the state with a button on the admin panel.

State values can be accessed using ```this.state('varName')```.
State values can be set using ```this.state('varName', 'value')```.

_meta_

Meta settings are like config but are not exposed in the admin panel.  These are most useful to expose metadata for a global pugin.  For example, you could have a plugin that examined all requests and, if the user hasn't signed in yet, respond with a 401 error.  The routes could expose a meta value that indicated whether they were routes that required authentication.

These are accessed using ```this.meta('varName')```.


#### Plugins

Plugins can be used to perform an action on all requests or just to encapsulate a set of route handlers.  Plugins can have config values just like Routes or Variants.

Plugins are just simple objects that have the following attributes

* _plugin_: (optional) if exists, will simply be called with a single parameter (the smocks object) so you can add new routes.
* _input_: (optional) input definitions to allow the user with different types of input fields in the admin panel.  See the next section (Input types) for more details
* _onRequest_: Called before the route handler (variant) executes for every request.  It is similar to the ([request, reply](http://hapijs.com/api#route-handler)) of the route handlers (Variants) but has an additional callback method that should be executed when the plugin has finished doing what it needs to do.

* _onResponse_: Called after the route handler (variant) executes for every request.  Parameters are similar to ```onRequest``` except the 2nd parameter is the return value from the [reply method (see response object)](http://hapijs.com/api#reply-interface).

The following plugin will add simulated latency (which can be controlled by the user) to all requests.

```
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.plugin({
      // define the input field for the admin panel allowing the user to adjust the delay
        input: {
          delay: {
            label: 'Add delay to all responses',
            type: 'select',
            options: [{label: 'no delay', value: 0}, {label: '1 sec', value: 1000}, {label: '5 sec', value: 5000}],
            defaultValue: 0
          }
        },

        // call "next" after a timeout if the user requested a delay
        onRequest: function (request, reply, next) {
          // get the delay value from config
          var delay = this.input('delay');
          if (delay > 0) {
            // if there is a delay, call next after a timeout
            setTimeout(next, delay);
          } else {
            next();
          }
        },

        onResponse: function (request, response) {
          // I can do things to the response here
          response.code(404);
        }
      })
```

Any route request handler can access any plugin input using `this.pluginInput(_pluginId_, _inputId_)`

![plugin example](http://jhudson8.github.io/smocks/images/plugin-ex1.png)

Or, check to see if the use has logged in (assuming the route exposed a ```requiresLogin``` option; see Route Options).  We are using state (see State) to track if the login endpoing has been hit prior to the current route.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.plugin({
      onRequest: function (request, reply, next) {
        // only do this check if the route exposed a "requiresLogin" option
        if (this.meta('requiresLogin')) {
          // now, see if we have previously logged in (the login route would have set this state value)
          if (!this.state('loggedIn')) {
            // if we haven't logged respond with a 401 and bypass calling "next"
            return reply({error: 'auth', message: 'Not logged in'}).code(401);
          }
        }

        next();
      }
    });
```

#### Addons
Some pre-defined plugins can be used with `smocks.addOn`;

##### reset-variants
Reset any non-default variants after a request is made.

Options include:

* ***defaultValue***: boolean indicating if it should be enabled by default (false if undefined)

```javascript
var smocksInstance = require('smocks')(_id_);
smocksInstance.addOn('reset-variants', { defaultValue: true });
```

#### response-delay
Add a modifiable delay to all responses

Options include:

* ***defaultValue***: milis (0 if undefined)

```javascript
var smocksInstance = require('smocks')(_id_);
smocksInstance.addOn('response-delay');
```

#### response-error
Add an error variant to all routes

Options include:

* ***code***: response http status code (500 if undefined)
* ***label***: variant label ("_code_ error response" if undefined)
* ***input***: any input values if necessary (useful if `response` is a function)
* ***response***: object or function returning the response content

```javascript
var smocksInstance = require('smocks')(_id_);
smocksInstance.addOn('response-error', {
  code: 503,
  label: 'Service Unavailable',
  response: {
    message: 'Service Unavailable'
  }
});
```


#### Connections

Smocks can use one or more connections for where to set endpoints. The main
endpoints are set using `smocksInstance.connection('label')` where label is the name
of the connection.

This is useful for where smocks is setup as a plugin.

Example:
``` javascript

var hapi = require('hapi');
var server = new hapi.Server();
server.connection({ port: 8080, labels: 'main' });
server.connection({ port: 8088, labels: 'smocks' });
server.connection({ port: 8089, labels: 'mockhost1' });

var smocksInstance = require('smocks')(_id_)...;
smocksInstance.connection('smocks');

var plugin = smocksInstance.toPlugin({
    onRegister: (server, options, next) => {

         smocksInstance.route({
            id: 'counter',
            connection: 'mockhost1',
            method: 'GET',
            path: '/',
            label: 'Simple get',
            handler: function (request, reply) {
                return reply('you got!');
            }
         });
        return next();
    }
});

server.register({ register: plugin, options: {} }, function (err) {
    server.start(function () {
        // ..
    });
});

```

This way the main server is not affected by the smocks routes, and there is no
risk of endpoint collision. Here all the admin connections are on the admin
connection, and the mocked host is on it's own. This is useful when mocking
external hosts that the code may rely on.

When a connection is not explicitly defined, the smocks routes will show up on
_ALL_ connections.


#### Profiles

Using the Admin Panel, you can save all route, variant and input settings as a "profile".  Profiles can either be saved locally (using localStorage) or remotely by providing the code to update in your project.

The profiles can also be changed using an admin endpoint (for example, to use this with an automated testing solution).  To do so, simply POST to {host}:{port}/api/profile/{profile name}.

Global profiles can be set applied to the ```smocks``` object.  The easiest way to do this is to make your changes in the admin panel, enter the ```Profile Name``` in the settings header, and click the ```for everyone``` button.  You will be provided with the code that is necessary for the provide to be loaded globally.

![profile example](http://jhudson8.github.io/smocks/images/profile-ex1.png)



### RESTful admin API
_reset the state_
POST to ```{host}:{port}/api/state/reset```

_set an active route variant_
POST to ```{host}:{port}/api/route/{routeId}``` with body content ```{ variant: _variant id_ }```

_select a profile_
POST to ```/api/profile/{profile name}```


### HTTP Archive Replay

Using [Google Chrome](http://www.google.com/chrome/) and other browsers, you can create a [HAR](https://en.wikipedia.org/wiki/.har) file which is basically a recording of the network activity during your session.

The smocks server has the ability to upload a [HAR](https://en.wikipedia.org/wiki/.har) file and reply the exact payloads as seen in the browsing session.  This is intended to only be used with a RESTful API and has the following caviats

* Only the `.json` responses will be recorded
* The HAR response will be rendered as long as the path and method match (query parameters and request payload are not evaluated)
* Network calls defined in the HAR but are not defined with the Smocks server will return a 404

Click the ***Har Replay*** tab to upload a HAR file

![profile example](http://jhudson8.github.io/smocks/images/har-empty.png)

Drag and drop your HAR file into the `HTTP Archive Upload` box

![profile example](http://jhudson8.github.io/smocks/images/har-uploaded.png)

There may be cases where you need to alter the URL of what is in the `.har` file to match what the
mock server will respond to.  These can be set as `pathMapper` in the `har` options when starting the smocks server
```
smocksInstance.start({
  // hapi options
}, {
  // smocks options
  har: {
    pathMapper: function (path) {
      if (pathFromHarShouldBeReturned) {
        return convertPathInSomeWay(path); // or leave as-is
      }
      // if nothing is returned, the mock server will just respond
      // normally to what would represent this path
    }
  },
```


### Proxying
You can use smocks to be a straight proxy to another server.  To do so, you must provide the proxy details in the hapi start options.
n
```
smocksInstance.start({
  // hapi options
  port: 8000,
  host: 'localhost',
}, {
  // smocks options
  proxy: {
    '{key shown in admin panel}': '{fully qualified endpoint prefix before the request path}',
    // example
    'google': 'http://www.google.com'
    // or, using a function
    'google': function (request) {
      return ...
    }
  }
});
```

View the `Config` tab on the admin panel to make any proxy setting changes.

### Pluggable State Handler

The user application state handler is pluggable.  To assign a new state impl to your smocks server, use the `state` smocks option.

```
smocksInstance.start({
    // hapi options
  }, {
   // smocks options
   state: myStateImpl
  });
```

The state object must have the following methods (the can be instance prototype methods or simple object methods).

| attribute | parameters | comments |
|-----------|------------|----------|
| initialize | (request, callback) | called at the beginning of each request; callback must be called with (err, value);  value = true if initialization needs to happen, false if the session has already been initialized
| userState | (request, options) | return the user application state object.  `options` object contains `route` and `variant` attributes
| resetUserState | {request, initialState) | reset the complete application state to be the initial state (no need to clone object);  this is also called if `initialize` callback is executed with `true`.
| routeState | (request) | return the state object used to contain all admin panel route config changes
| resetRouteState | (request) | reset the state object


API:Route Config Objects
--------------
### Smocks

This refers to functions which can be executed from the `smocks` object.

#### route(options)
* ***options***: object containing the following values
* ***path***: the route path (must start with ```/```).  Ex: ```/api/customer/{id}```
* ***label***: An optional human readable label that can be seen in the admin panel for this route
* ***method***: optional method (GET is default)
* ***handler***: optional default route handler (same as the first variant applied to this route)

Refer to the [route docs](#project/jhudson8/smocks/section/Concepts/Routes) for more details.


#### plugin(plugin)
* ***plugin***: the plugin object

Refer to the [plugin docs](#project/jhudson8/smocks/section/Concepts/Plugins) for more details.


#### start(options)
* ***options***: either object {host, port} or a HAPI server instance

Start a HAPI server with the defined mock configuration.

Both the host and port are optional and will default to ```localhost``` and ```8080```.

If a HAPI server instance is provided, the routes will be bound to the HAPI server provided but the ```start``` method will not be called.


### Route
#### route(options)

Object returned when calling [smocksInstance.route](#project/tmp/smocks/method/global/route)

Refer to the [route docs](#project/jhudson8/smocks/section/Concepts/Routes) for more details.

#### input(attributes)
* ***attributes***: The input attributes

Set any input attributes that will be available for modification on the admin panel.

See [config example](#project/jhudson8/smocks/section/Examples/Route%20%2F%20variant%20input) for details.

Return the same route object for chaining.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({
      ...
      input: {
        myVar: {
          label: 'Input label',
          type: 'boolean|text|select|multiselect',
          defaultValue: ...
        }
      })
      .respondWith(...)
```


#### variant(id)
* ***id***: the variant id

Set up a new route variant with the provided id.  The id is meaningful when selecting the active variant from the admin panel.

A variant is basically a single request handler for a defined route.  This is useful to test out different scenarios for a single route definition.

Return the [Variant object](#project/jhudson8/smocks/snippet/package/Variant) for chaining.

Refer to the [variant docs](#project/jhudson8/smocks/section/Concepts/Variants) for more details.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route(...)
    .variant('respond like this').respondWith(...)
    .variant('respond like that').respondWith(...)
```


#### plugin(plugin)
Refer to [global:plugin](#project/jhudson8/smocks/snippet/method/global/plugin)


#### respondWith(requestHandler)
* ***requestHandler***: The [RequestHandler](#project/jhudson8/smocks/section/Object%20Types/RequestHandler)

Convienance method for creating a default variant (id of "default") and then calling [Variant:respondWith](#project/jhudson8/smocks/snippet/method/Variant/respondWith) on the variant.


#### respondWithFile(filePath)
Convienance method for creating a default variant (id of "default") and then calling [Variant:respondWithFile](#project/jhudson8/smocks/snippet/method/Variant/respondWithFile) on the variant.


### Variant

Object returned when calling [Route.variant](#project/tmp/smocks/method/smocksInstance.route/variant)

Refer to the [route docs](#project/jhudson8/smocks/section/Concepts/Variants) for more details.

#### route(options)
Refer to [global:route](#project/jhudson8/smocks/snippet/method/global/route)


#### input(attributes)
* ***attributes***: The input attributes

Set any variant-scoped input attributes that will be available for modification on the admin panel.

See [input example](#project/jhudson8/smocks/section/Examples/Route%20%2F%20variant%input) for details.

Return the same Variant object for chaining.


#### variant(id)
Refer to [Route:variant](l#project/jhudson8/smocks/snippet/method/Route/variant)


#### plugin(plugin)
Refer to [global:plugin](#project/jhudson8/smocks/snippet/method/global/plugin)


#### respondWith(requestHandler)
* ***requestHandler***: The [RequestHandler](#project/jhudson8/smocks/section/Object%20Types/RequestHandler)

Associate a request handler with the current route/method/variant combination.

Return the same Variant object for chaining.

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route(...).respondWith(function (request, reply) {
      var theMessage = request.params.message;
      var aQueryStringValue = request.query.theQueryStringKey;
      reply({message: theMessage}); // reply with a JSON payload
      // or reply with something else
      reply({error: {message: 'Some unknown error'}}).code(500);
    });
```


#### respondWithFile(options)
* ***options***: options or the path name as a string

Option can include
* ***code***: http status code
* ***path***: the file path which can include route token references

Remember that using ```./``` will refer to the top level module directory (the directory where ```node_modules``` exists regardless of the location of the file that is referring to a file location with ```./```);

```javascript
    var smocksInstance = require('smocks')(_id_);
    smocksInstance.route({path: '/customer/{id}'}).respondWithFile('./customer-{id}.json')
    .start(...)
```

This would cause a request to ```/customer/1``` to return the file ```./customer-1.json```

If the `path` option is not provided a custom file handler provided as a smocks option as `respondWithFileHandler` will be used.  This is a function which takes a single object parameter which contains
* ***request***: the HAPI request object
* ***response***: the HAPI response object
* ***route***: the route object
* ***variant***: the variant object
* ***options***: the options object passed to the `respondWithFile` function
* ***smocksOptions: the smocks options object

```
// smocks start code
smocksInstance.start({
  port: 8000,
  host: 'localhost',
}, {
  respondWithFileHandler: function (data) {
    // example file handler
    var reply = data.reply;
    var code = data.options.code || 200;
    var route = data.route;
    var path = route.path().replace(/\{\}/g, '');
    var variant = data.variant;
    var filePath = // create file path from all this data
    fs.readFile(function (err, stream) {
      if (err) {
        if (err.code === 'ENOENT') {
     return reply().code(404);
        } else {
     return reply(err);
        }
      }
      reply(stream).type('application/json').code(code);
    });
  }
});
```

API:Pseudo Database
-------------------

### smocksInstance.db
Smocks has a database-like object which can be used to store data in state in a why that makes data querying and normalization easy.
You can access this object at any time during the event loop of a current request using ```require('smocks').db```.
By default the smocks `state` object will be used to persist the data but that can be changed at the domain (AKA database table) level.

#### insert(domain, object)
* ***domain***: the top level store attribute that will contain this object
* ***object***: the object to insert.  If the object does not have an `id` attribute, a uuid will be created.  (the attribute can be changed using the ***init*** function)

Insert an object into the "table".  The object will be returned.

```
var db = smocksInstance.db;

// of `object` doesn't contain an id a uuid will be set
var insertedObject = db.insert('foo', object);
```

#### get(domain, id)
* ***domain***: the top level store attribute that will contain this object
* ***id***: the object id

Return an object from the "table" identified by the provided id.

```
var db = require('smocks').db;

var object = db.get('foo', 'some_id');
```

#### delete(domain, id)
* ***domain***: the top level store attribute that will contain this object
* ***id***: the object id

Delete and an object from the "table" identified by the provided id.  The deleted object will be returned.

```
var db = require('smocks').db;

var deletedObject = db.delete('foo', 'some_id');
```

#### update(domain, idOrObject[, object, overwrite])
* ***domain***: the top level store attribute that will contain this object
* ***idOrObject***: the object id or the object contents to update
* ***object***: (if `idOrObject` is an id) the object contents to be updated
* ***overwrite***: if false, merge attributes provide into object (default to true)

Update an existing object in the "table" identified by the provided id.

if the id is provided as the `idOrObject` with a separate object provided as the `object` parameter, the existing contents will only have individual fields updated identified by the provided object.

if the object is provided as the `idOrObject` (so only 2 function parameters) then the entire object will be replaced.

```
var db = smocksInstance.db;

// this will only overwrite the attributes provided as the 3rd parameter
var updatedObject = db.update('foo', 'some_id', data, false);

// this will completely overwrite the stored object.  `newObject` must have an id
var updatedObject = db.update('foo', newObject);
```

#### insertOrUpdate(domain, object)
* ***domain***: the top level store attribute that will contain this object
* ***object***: the object to insert or update.  If the object does not have an `id` attribute, a uuid will be created.  (the attribute can be changed using the ***init*** function)

Either inserts or updates an object depending on whether it already exists.  Using ***update*** alone will throw an Error if no object can be found matching the provided id.

```
var db = smocksInstance.db;

var insertedOrUpdatedObject = db.insertOrUpdate('foo', object);
```

#### meta(domain, id, object)
* ***domain***: the top level store attribute that will contain this object
* ***id***: the object id
* ***object***: meta contents.  If `false`, delete existing metadata.  If `undefined`, return the existing metadata.  If an object, replace the existing metadata.

Store arbitrary object data which is kept separately from the object itself.

```
var db = require('smocks').db;

var insertedObject = db.insert('foo', object);

// this data will not be accessable from `insertedObject` directly
db.meta('foo', insertedObject.id, { abc: 'def' });

// meta will be { abc: 'def' }
var meta = db.meta('foo', insertedObject.id);
```

#### list(domain)
* ***domain***: the top level store attribute

Return an object which will return the contents of the "table" as an array using the `result` function

```
var fooItems = db.list('foo').result();
```

The returned object has several additional helper methods which can be chained
##### filter(criteria)
* ***criteria***: the filter criteria

Filter the results using the provided criteria.  The criteria can be
* a function which accepts a domain object and returns a truthy if the item should be included
* an object with key/values that should match domain object values (key an value paths can use "." for nesting)
* the previous object but with functions for values.  The function parameter is the value from the object identified by the key

```
// the following statements do the same thing
var filteredArray = db.list('foo').filter(function (object) { return object.nested.abc === 'def'; }).result();
var filteredArray = db.list('foo').filter({ 'nested.abc': function (value) { return value === 'def'; }}).result();
var filteredArray = db.list('foo').filter({ 'nested.abc': 'def' }).result();
```

##### sort(comparator, ascending)
* ***comparator***: a string representing the attribute name or a `Array.sort` comparator function
* ***ascending***: if `comparator` is a strong, true for ascending and false for descending (default is true)

```
// the following statements do the same thing
var sortedArray = db.list('foo').sort('nested.abc', false).result();
var sortedArray = db.list('foo').sort(function (a, b) { return a.nested.abc > b.nested.abc ? -1 : 1; }).result();
```

##### limit(offset, size)
* ***offset***: 0-based starting point offset
* ***size***: maximum number of results to return

Return a limited set of results from the main set
```
// the following will only return the first 5 entries
var limitedArray = db.list('foo').limit(0, 5).result();
```
When using limit, you may want to know the original size of the array.  You can use the `data` function for that (see below).

##### data()

Return an object containing the results as `result` attribute with additional attributes.

* ***filtered***: `true` if the results have been filtered
* ***sorted***: `true` if the results have been sorted
* ***totalCount***: if the results were limited using the `limit` function, the total itme count ***before** limit was called
* ***offset***: the `offset` value provided to `limit` if applicable
* ***size***: the `size` value provided to `limit` if applicable
* ***result***: the same value as if you had called `result()`
