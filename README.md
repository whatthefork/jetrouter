# JetRouter - WordPress Routing, Fast & Easy

### THIS IS A FORK OF THE ORIGINAL [JETROUTER](https://github.com/sformisano/jetrouter) WITH ADDED CAPABILITIES TO LOAD IT WITHOUT WORDPRESS BEING LOADED FIRST, AND TO ALLOW ADDING FILTER CALLBACKS THAT CAN RUN BEFORE ROUTES ARE CALLED

This WordPress plugin adds a regular expression based router on top of the WordPress standard routing system. Any route declared through this router will take priority over standard WordPress URLs.

- [Installation](#installation)
- [Usage](#usage)
  - [Initialization & Configuration](#initialization--configuration)
    - [Configuration properties](#available-router-configuration-properties)
  - [Adding Routes](#adding-routes)
    - [The basics](#the-basics)
    - [Dynamic routes](#dynamic-routes)
    - [Parameters regular expressions](#parameters-regular-expressions)
    - [Optional parameters](#optional-parameters)
  - [Reverse Routing](#reverse-routing)
    - [Dynamic routes reverse routing](#dynamic-routes-reverse-routing)
  - [Handler's respond_to Feature](#handlers-respond_to-feature)
- [Why I built it](#why-i-built-the-jetrouter)
- [Credits](#credits)


## Installation

#### Download zip via GitHub repo release page

You can also grab the latest release from this repository's [releases page](https://github.com/whatthefork/jetrouter).

#### Composer

If your setup uses [Composer](https://getcomposer.org/) for proper dependency management (check out the awesome [Bedrock project](https://roots.io/bedrock/) if you are interested in running WordPress as a 12 factor app) installing the JetRouter is quite easy.

##### Composer via GitHub releases

Add this package to the "repositories" property of your composer json definition (make sure the release version is the one you want to install):

```json
{
  "repositories": [
    {
      "type": "package",
      "package": {
        "name": "jetrouter",
        "type": "wordpress-plugin",
        "version": "1.0",
        "dist": {
          "type": "zip",
          "url": "https://github.com/whatthefork/jetrouter/archive/v1.0.zip",
          "reference": "v1.0"
        },
        "autoload": {
          "classmap": ["."]
        }
      }
    }
  ]
}
```

Once that's done, add the JetRouter in the composer require property (the version has to match the package version):

```json
{
  "require":{
    "jetrouter": "1.0"
  }
}
```

Finally, make sure to specify the correct path for the "wordpress-plugin" type we assigned
to the JetRouter package in the definition above (note: if you are using
[Bedrock](https://roots.io/bedrock/) this is already taken care of).

```json
{
  "extra": {
    "installer-paths": {
      "path/to/your/wordpress/plugins-directory/{$name}/": ["type:wordpress-plugin"],
    }
  }
}
```
Run `composer update` from your composer directory and you're good to go!

## Usage

### Initialization & Configuration
In your theme’s `functions.php` file or somewhere in your plugin:

```php
// Import the JetRouter
use JetRouter\Router;
  
// Router config
$config = [];
  
// Create the router instance
$r = Router::create($config);
```
  
#### Available router configuration properties:

* `namespace`: if defined, the namespace prefixes all routes, e.g. if you set the namespace to “my-api” and then define a route as “comments/published”, the actual endpoint will be “/my-api/comments/published/“.
* `outputFormat`: determines the routes handlers output format. Available values: `auto`, `json`, `html`.
* `filters_before`: an array of callbacks that are called before every route is dispatched.

The router automatically hooks itself to WordPress, so once you’ve created the router instance with the configuration that suits your needs, all that is left to do is adding routes. After that, the router is ready to dispatch requests.

##### Pro tip on the namespace

If you use the JetRouter to build a json/data api, or if you simply don’t mind having a namespace before your urls, having one will speed up standard WordPress requests, especially if you have many dynamic routes.

This is because, if the router has a namespace, but the http request’s path does not begin with that namespace (like it would be the case with all WordPress urls), the router won’t even try to dispatch the request.
 
Without a namespace, on the other hand, the router will try to dispatch all requests, building the dynamic routes array every time. *(Note: caching the generated dynamic routes is on my todo list.)*

### Adding Routes

#### The basics
  
##### Make sure that you create your route callback code before you define a route
 
The basic method to add routes is `addMethod`. Example:

```php
$r->addRoute('GET', 'some/resource/path', 'the_route_name', function(){
  // the callback fired when a request matches this route
});
```
You can also add a route with one or more route-specific filter callbacks (these are run after any global filter callbacks defined in the router config. Example:

```php
$r->addRoute('GET', 'some/resource/path', 'the_route_name', function(){
  // the callback fired when a request matches this route
}, array( 'my_auth_function', 'MYCLASS::this_custom_method' ) );
```


The `addRoute` method has aliases for each http method (get, post, put, patch, delete, head, options). Example:

```php
$r->get(‘users/new’, ‘route_name’, function(){
  // new user form view
});

$r->post('users', 'create_user', function(){
  // create user in database
});
```

#### Dynamic routes

Use curly braces to outline parameters, then declare them as arguments of the callback function and the router will take care of everything else. Example:

```php
$r->get('users/{username}', 'get_user_by_username', function($username){
  echo "Welcome back, $username!";
});

$->get('schools/{school}/students/{year}', 'get_school_students_by_year', function($school, $year){
  echo "Welcome, $school students of $year!";
});
```

#### Parameters regular expressions
By default, all parameters are matched against a very permissive regular expression:

```php
/**
  * One or more characters that is not a '/'
  */
  const DEFAULT_PARAM_VALUE_REGEX = '[^/]+';
```

You can use your own regular expressions by adding a colon after the parameter name and the regex right after it. Example:

```php
$r->get('schools/{school}/students/{year:[0-9]+}', 'get_school_students_by_year', function($school, $year){
  // If $year is not an integer an exception will be thrown
});
```

You can also use one of these regex shortcuts for convenience:

```php
private $regexShortcuts = [
  ':i}' => ':[0-9]+}',           // integer
  ':a}' => ':[a-zA-Z0-9]+}',     // alphanumeric
  ':s}' => ':[a-zA-Z0-9_\-\.]+}' // alphanumeric, "_", "-" and ".""
];
```

If, for example, we wanted to write the same `get_school_students_by_year` route by using the `:i` shortcut, this is how we would do it:

```php
$r->get('schools/{school}/students/{year:i}', 'get_school_students_by_year', function($school, $year){
  // If $year is not an integer an exception will be thrown
});
```

#### Optional parameters

You can make a parameter optional by adding a question mark after the closing curly brace. Example:

```php
$r->get('schools/{school}/students/{year}?’, 'get_school_students_by_year', function($school, $year){
  // $year can be empty!
});
```

You can make all parameters optionals, even when they are not the last segment in a route. Example:

```php
$r->get('files/{filter_name}?/{filter_value}?/latest', 'get_latest_files', function($filter_name, $filter_value){
  // get files and work with filters if they have a value
});
```

All the example requests below would match the route defined above:

```php
'/files/latest'
'/files/type/pdf/latest'
'/files/author/sformisano/latest'
```

This is not necessarily the best use of optional parameters. The examples here are just to show that, if you have a valid use case, the JetRouter will allow you to setup your routes this way.

### Reverse Routing

Reverse routing is done through the same router object for simplicity’s sake. 

Given this simple example route:

```php
$r->addRoute('GET', 'posts/popular', 'popular_posts', function(){});
```

To print out the full path you use the `thePath` method:

```php
$r->thePath('popular_posts'); // prints /posts/popular/
```

You can also get the value returned, rather than printed out, by using the `getThePath` method (trying to follow some WordPress conventions with this):

```php
$r->getThePath('popular_posts'); // returns /posts/popular/
```

#### Dynamic routes reverse routing

Dynamic routes work in the exact same way, you just pass the values of the parameters after the route name in the same order they are defined in when you created the route.

For example, given this route:

```php
$r->get('schools/{school}/students/{year}?’, 'get_school_students_by_year', function($school, $year){
});
```

This is how you would print out a full path to this route:

```php
$r->thePath('get_school_students_by_year', 'caltech', 2005); // prints  /schools/caltech/students/2005/
```

As the last parameter is optional you can simply omit it:

```php
$r->thePath('get_school_students_by_year', 'mit'); // prints  /schools/mit/students/
```

If the optional parameter you need to omit is not the last parameter, you can simply pass `null` as a value and it will be ignored. Example:

```php
$r->get('{school}?/students/{year}’, 'get_students', function($school, $year){
});

$r->thePath('get_students', null, 1999); // prints /students/1999/
```

I prefer this approach to the more common associative array for arguments as it makes everything simpler (less typing in the vast majority of scenarios) and more explicit (a missing optional parameter is still defined as null, so there’s a 1:1 match between formal parameters and arguments).

###  Handler's respond_to Feature

If you ever played around with Ruby on Rails, you're probably familiar with the [respond_to and respond_with blocks](http://www.davidwparker.com/2010/03/09/api-in-rails-respond-to-and-respond-with/). This respond_to feature is a basic, simplistic imitation of that clever way to handle different request types with different behaviours.

If you never tried out Rails, let me make this simple with an example:

You created a public signup form that posts to a `create_user` endpoint created with the JetRouter. You definitely want that form to work with ajax, but you're one of those few developers left who still cares about graceful degradation, so you don't just want to set the `outputFormat` config parameter to `json` and do a `return $data;` of sorts. You need to be able to receive json data with ajax requests, while doing something else (redirects, loading views in error mode etc.) for standard, synchronous requests.

Technically, you *could* manually check for `$_SERVER['HTTP_X_REQUESTED_WITH']` and write some conditional code that works with that, but who wants to do that kind of manual labour for every single endpoint?

JetRouter's respond_to to the rescue! All you have to do is leave the `outputFormat` config parameter to `auto` and have your route handler return an array following this format:

```php

// endpoint doing stuff up here

return [ 'respond_to' => [
  'json' => $output // this is whatever came out of this endpoint, to be returned as json!,
  'html' => function(){
    // this callback will run if this the route handler is dispatching a standard, synchronous request
    // do something here, then maybe redirect with wp_redirect or whatever else!
  }
] ];
```

**Pro tip:** appending ?json to a request path will force the json output.

## Why I built the JetRouter

* I wanted something fast, easy to setup and with a minimal API. The JetRouter is setup with one line of code (`$r = Router::create();`) and all functionality is readily available through that object.

* I needed extra features not offered by any other small footprint php router I am familiar with, including the ones I credited below.

* The routers I am aware of, even the ones I credited below, are conceived to be used as the core routing system of a website/webapp. This does not make them the best fit for the use case of a WordPress extra routing layer, e.g. they will throw an exception if a route is not found or if there’s a path match but the http method of the request is different. Because this router is an extra layer on top of WordPress, it needs to fail silently in most of these scenarios and give control back to WordPress. WordPress can then dispatch those requests matching, return a 404 or do whatever else it’s supposed to do.

* Finally, building this looked like a fun way to revisit PCRE, which are something I have not worked with in many years.

## Credits

* This router implements [Nikita Popov](https://github.com/nikic)’s  group position based, non-chunked approach to regex matching for lightning fast routing. Learn more about it by reading [this great post](http://nikic.github.io/2014/02/18/Fast-request-routing-using-regular-expressions.html) by Nikita himself.  

* [Joe Green](https://github.com/mrjgreen) and his [phroute](https://github.com/mrjgreen/phroute) project (also loosely based on Nikita’s work) are to be thanked for:
  *  the regex used to capture dynamic routes parameters
  *  the regex shortcuts (a simple yet very elegant idea)
  *  the basics of the optional parameters and reverse routing implementation

Go check both these authors and their projects, they are awesome.
