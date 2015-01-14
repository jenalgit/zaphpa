---
layout: default
title: Zaphpa - Intuitive API Microframework for PHP
---

[![Old Documentation](https://img.shields.io/badge/legacy_pdf_doc-for_ver_1.x-800080.svg?style=flat)](assets/zaphpa-doc-v1.0.pdf)
[![Link to Github](https://img.shields.io/badge/github-sources-blue.svg?style=flat)](https://github.com/zaphpa/zaphpa)

## Installation

To start serving RESTful HTTP requests, you need to go through three simple steps:

1. Configure virtualhost in your web-server so that all HTTP requests to "non-existant" "files" are sent to your 
router configuration PHP file, say: api.php (see: [Appendix A](/doc.html#Appendix_A_Setting_Up_Zaphpa_Library) )
1. Create api.php where you instantiate and configure a router object
1. Write controller callbacks.
1. Install Zaphpa through composer:
    1. In your composer.json, add:
    
    	```json
    	{
 		  "require": {
	        "zaphpa/zaphpa": "^2.0.0"
		  }
        }
    	```
    	
    1. Install Composer (if you don't already have it) and then install Zaphpa via Composer:
     
        ```bash
        > curl -sS https://getcomposer.org/installer | php
        > php composer.phar install
        ```

## A Simple Router

For a very simple case of getting specific user object, the code of api.php would 
look something like the following:

```php
<?php

require_once('./vendor/autoload.php');
$router = new \Zaphpa\Router();

$router->addRoute(array(
  'path'     => '/users/{id}',
  'handlers' => array(
    'id'         => \Zaphpa\Constants::PATTERN_DIGIT, //enforced to be numeric
  ),
  'get'      => array('\MyApp\MyController', 'getPage'),
));

try {
  $router->route();
} catch (\Zaphpa\InvalidPathException $ex) {      
  header("Content-Type: application/json;", TRUE, 404);
  $out = array("error" => "not found");        
  die(json_encode($out));
}     
```

In this example, {id} is a URI parameter of the type "digit", so `MyController->getPage()` function will get control to serve URLs like:

* http://example.com/users/32424
* http://example.com/users/23

However, we asked the library to ascertain that the {id} parameter is a number by attaching a validating handler: "\Zaphpa\Constants::PATTERN_DIGIT" to it. As such, following URLs will not be handed over to the `MyController->getPage()` callback:

* http://example.com/users/ertla
* http://example.com/users/asda32424
* http://example.com/users/32424sfsd
* http://example.com/users/324sdf24

## Simple Callbacks

A callback can be a simple PHP function. In most cases, however, it will probably be a method on a class. Callbacks are passed two arguments:

1. `$req` is an object created and populated by Zaphpa from current HTTP request. 
1. `$res` is a response object. It is used by your callback code to incrementally assemble a response, including both the response 
headers, as well as: the response body. 

We will look into the details of $req and $res objects further in the documentation. Following are some example callback implementations:

```php
<?php

class MyController {

  public function getPage($req, $res) {
    $res->setFormat("json");
    $res->add(json_encode($req->params));
    $res->add(json_encode($req->data));
    $res->send(301);    
  }

  public function postPage($req, $res) {
    $res->add(json_encode($req->params));
    $res->add(json_encode($req->data));
    $res->send(201, 'json');    
  }

}	
```

## Request Object

`$req (request)` object contains data parsed from the request, and can include properties like:

1. `$params` - which contains all the placeholders matched in the URL (e.g. the value of the "id" argument)
1. `$data`  - an array that contains HTTP data. In case of HTTP GET it is: parsed request parameters, for HTTP POST, PUT and DELETE requests: data variables contained in the HTTP Body of the request.
1. `$version` - version of the API if one is versioned (not yet implemented)
1. `$format` - data format that was requested (e.g. XML, JSON etc.)
	
Following is an example request object:

```php
<?php

Zaphpa\Request Object
(
  [params] => Array
    (
      [id] => 234234
    )
  [data] => Array
    (
      [api] => 46546456456
    )
  [formats] => Array
    (
      [0] => text/html
      [1] => application/xhtml+xml
      [2] => application/xml
    )
  [encodings] => Array
    (
      [0] => gzip
      [1] => deflate
      [2] => sdch
    )
  [charsets] => Array
    (
      [0] => ISO-8859-1
      [1] => utf-8
    )
  [languages] => Array
    (
      [0] => en-US
      [1] => en
    )
  [version] => 
  [method] => GET
  [clientIP] => 172.30.25.142
  [userAgent] => Mozilla/5.0 (Macintosh; Intel Mac OS X 10_6_8) AppleWebKit/535.2...
  [protocol] => HTTP/1.1
)
```

### Request parsing

Zaphpa provides multiple shortcuts to make request-parsing easier and less error-prone. One such shortcut is `$req->get_var(<varname>)` function.

1. `$req->get_var('varname')` - since $req object populates `$data` object, you can access request variables 
(query variables for GET or HTTP Body data for PUT/POST/DELETE) through the $data array directly. However am HTTP client may not set a variable 
that your callback expects, causing PHP to throw a warning. Instead of having your callback code check each call to `$req->data['varname']` on 
being empty Zaphpa provides a convenience method: $req->get_var('varanme'). get_var() returns value of the HTTP variable if it is set, 
or null, otherwise.

## Response Object

`$res (response)` object is used to incrementally create content. You can add chunks of text to the output buffer 
by calling: `$res->add (String)` and once you are done you can send entire buffer to the HTTP client by issuing: 
`$res->send(<HTTP_RESPONSE_CODE>)`. HTTP_RESPONSE_CODE is an optional parameter which defaults to (you guessed it:) 200.

Response object is basically an output buffer that you can keep adding output chunks to, while you are building a response. 
Following methods are available on the response class:

1. `$res->add($string)` - adds a string to output buffer
1. `$res->flush($code, $format)` - sends current output buffer to client. Can take optional output code (defaults to 200) and output format 
   (defaults to request format) arguments. 
   **Caution**: obviously you can/should only indicate $code or $format, the first time
   you invoke the method, since these values can not be set once output is sent to the client.
1. `$res->send($code, $format)` - sends current output buffer to the client and terminates response.

## Middleware

One of the powerful features of Zaphpa is the ability to intercept request and perform centralized pre- and post- processing of a request. To hook into request processing flow, register your middleware
implementation with the router:

    $router->attach('MyMiddlewareImpl');
    
Where `MyMiddlewareImpl` is the class name of an implementation of a Zaphpa_Middleware abstract class. 

You can implement following methods, in your middleware:

* `->preprocess(&$route)`- hooks very early into the request servicing process and allows adding custom route mappings.
* `preflight` - hooks after routes are finalized but before `preroute` processors kick-in. It is reserved for CORS implementation and unless you are implementing an alternative CORS support.
* `->preroute(&$req, &$res)` -   hooks into the process once request has been analyzed, a route handler has been identified, but before route handler fires. 
* `->prerender(&$buffer)` - gets a chance to alter output buffer right before it is assembled for output. Please note that this method may be called multiple times. To be more precise: every time your callback function invokes `->flush` and tries to output a chunk of the response buffer to HTTP, `->prerender` will get a chance
to alter the buffer.

Middleware is an excellent place to implement central authentication, authorization, versioning and other infrastructural features.

An example implementation (however meaningless) of a middleware can be found in Zaphpa tests:

```php
<?php

class ZaphpaTestMiddleware extends Zaphpa\BaseMiddleware {
  function preprocess(&$router) {
    $router->addRoute(array(
          'path'     => '/middlewaretest/{mid}',
          'get'      => array('TestController', 'getTestJsonResponse'),
    ));
  }
  
  function preroute(&$req, &$res) {
    // you get to customize behavior depending on the pattern being matched in the current request
    if (self::$context['pattern'] == '/middlewaretest/{mid}') {
      $req->params['bogus'] = "foo";
    }
  }
  
  function prerender(&$buffer) {
      if (self::$context['pattern'] == '/middlewaretest/{mid}') {
        $dc = json_decode($buffer[0]);
        $dc->version = "2.0";
        $buffer[0] = json_encode($dc);
      }
  }  
}
```

### Middleware Context

Please note the usage of `self::$context['pattern']` variable in the `->preroute` method. Often `preroute` needs to modify behavior based on the current URL Route being matched. The variable `self::$context['pattern']` carries 
that pattern. Please make sure to match it with the exact definition(s) in your routes configurations.

Full list of variables exposed through context:

* `pattern` - URI pattern being matched (in the format it was defined in the routes configuration, includes placeholders)
* `http_method` - current HTTP Method being processed.
* `callback` - callback in PHP format i.e.: name of the function or an array containing classname and method name.
    
### Middleware Route Restrictions

As we saw in the example above, frequently you may want to only enable your middleware for certain routes. 

Instead of hard-coding that logic as part of the middleware implementation, Zaphpa makes it easy to declaratively set the scope of Middleware activity ("restrict" middleware execution to only certain routes):

```php
<?php

$router->attach('\Myapp\MyMiddleWare')
       ->restrict('GET', '/users')
       ->restrict(array('POST', 'GET'), '/tags')
       ->restrict('*', '/groups');
```

**Caution:** the restrictions do not apply to the `preprocess` hook of a middleware class. If middleware has `preprocess` declared it will fire for all routes, because this events is raised before routing destination is identified and is generally used to alter routing table itself.

### Prebuilt Middleware 

#### HTTP Method Overrides

In REST you typically operate with following common HTTP Methods ("verbs" for CRUD): GET, PUT, POST, DELETE (and sometimes: PATCH). Using these methods can be problematic, in certain cases however. Some HTTP
Proxies often block any methods but GET AND POST, as well as: making cross-domain Ajax calls with custom
verbs can be hard.

Zaphpa implements common HTTP Method override trick to allow an effective solution. You can still implement
proper HTTP Methods and if clients have problem making a particular HTTP Method-based request, they can make
HTTP POST instead, and indicate the method they "meant" in request headers with the "X-HTTP-Method-Override"
header.

Method override is a middleware plugin that is disabled by default. To enable it, add following line to your router initialization code:

    $router->attach('\Zaphpa\Middleware\MethodOverride');

#### Auto-Documentator

On to more useful middleware implementation, Zaphpa comes with an auto-documentation middleware plugin. This
plugin creates an endpoint that can list all available endpoints, even parse PHPDoc comments off your code
to provide additional documentation. To enable the middleware:
 
    $router->attach('\Zaphpa\Middleware\AutoDocumentator');
    
which will create documentation endpoint at: '/docs'. If you would rather create endpoint at another URI:

    $router->attach('\Zaphpa\Middleware\AutoDocumentator', '/apidocs');
    

If you want documentation to also show filename, class and callback method for each endpoint:

    $router->attach('\Zaphpa\Middleware\AutoDocumentator', '/apidocs', $details = true);     
    
If you don't want some endpoints to be exposed in the documentation (say, for security reasons) you can
easily hide those by adding `@hidden` attribute to the PHP docs of the callback for the endpoint. To build
a more elaborate authorization schema, you would need to implement a custom middleware, but it's certainly
possible.

**Beware**: problem has been <a href="http://us.php.net/manual/en/reflectionfunctionabstract.getdoccomment.php#108061">reported</a>
when trying to use doc comment parsing in PHP with eAccelerator. There seems to be no such problem with APC.

#### Ajax-friendly Endpoints

As you probably know, Ajax calls cannot normally access API URLs on another domain (or even another port 
of the same domain, actually). This is a problem sometimes solved using 
[JSONP](http://en.wikipedia.org/wiki/JSONP). We think a better solution is: 
[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS).
Zaphpa comes with a simple pre-built middleware plugin to enable CORS for all endpoints. To enable CORS for any domain:

    $router->attach('CORS');
    
or if you want to enable CORS only for specific domain(s):

    $router->attach('CORS', 'http://example.com http://foo.example.com');
    
If you want to enable CORS only for specific routes:

```php
<?php

$router->attach('CORS', '*')
       ->restrict('GET', '/users')
       ->restrict(array('POST', 'GET'), '/tags')
       ->restrict('*', '/groups');
```    
        
## Output format aliases

The $format argument of the send() and flush() should be passed as a standard mime-type string. However, for convenience and brevity Zaphpa
allows indicating some simple aliases for common mime types:

    'html' => 'text/html',
    'txt' => 'text/plain',
    'css' => 'text/css',
    'js' => 'application/x-javascript',
    'xml' => 'application/xml', 
    'rss' => 'application/rss+xml',
    'atom' => 'application/atom+xml',
    'json' => 'application/json',

## A More Advanced Router Example

```php
<?php
   
$router = new Zaphpa_Router();

$router->addRoute(array(
  'path'     => '/pages/{id}/{categories}/{name}/{year}',
  'handlers' => array(
    'id'         => \Zaphpa\Constants::PATTERN_DIGIT, //regex
    'categories' => \Zaphpa\Constants::PATTERN_ARGS,  //regex
    'name'       => \Zaphpa\Constants::PATTERN_ANY,   //regex
    'year'       => 'handle_year',       //callback function
  ),
  'get'      => array('MyController', 'getPage'),
  'post'     => array('MyController', 'postPage'),
  'file'     => 'controllers/mycontroller.php'
));

// Add default 404 handler.
try {
  $router->route();
} catch (\Zaphpa\InvalidPathException $ex) {
  header("Content-Type: application/json;", TRUE, 404);
  $out = array("error" => "not found");        
  die(json_encode($out));
}

function handle_year($param) {
  return preg_match('~^\d{4}$~', $param) ? array(
    'ohyesdd' => $param,
    'ba' => 'booooo',
  ) : null;
}
```

Please note the "file" parameter to the `->addRoute()` call. This parameter indicates file where MyController class should be loaded from,
if you do not already have the corresponding class loaded (through an auto-loader or explicit require() call).

## Routing to Entities

So far we have discussed routing individual URI patterns. However, when building a RESTful API, you often need to create 
full Resources or Endpoints - API lingo for objects that can be managed in a full: Create, Read, Update, Delete (CRUD) lifecycle.

One way you can do this is to fully declare all four routes. But that would mean a lot of duplicated configuration. 
We hate code duplication, so here's a nifty shortcut you can use:

```php
<?php

$router->addRoute(array(
  'path'     => '/books/{id}',
  'handlers' => array(
    'id'         => \Zaphpa\Constants::PATTERN_DIGIT, 
  ),
  'get'      => array('BookController', 'getBook'),
  'post'     => array('BookController', 'createBook'),
  'put'      => array('BookController', 'updateBook'),
  'delete'     => array('BookController', 'deleteBook'),
  'file'     => 'controllers/bookcontroller.php'
));
```

## Pre-defined Validator Types

Zaphpa allows indicating completely custom function callbacks as validating handlers, but for convenience it also 
provides number of pre-defined, common validators:

```php
<?php

const PATTERN_NUM        = '(?P<%s>\d+)';
const PATTERN_DIGIT      = '(?P<%s>\d+)';
const PATTERN_MD5        = '(?P<%s>[a-z0-9]{32})';
const PATTERN_ALPHA      = '(?P<%s>(?:/?[-\w]+))';
const PATTERN_ARGS       = '?(?P<%s>(?:/.+)+)';
const PATTERN_ARGS_ALPHA = '?(?P<%s>(?:/[-\w]+)+)';
const PATTERN_ANY        = '(?P<%s>(?:/?[^/]*))';
const PATTERN_WILD_CARD  = '(?P<%s>.*)'; 
const PATTERN_YEAR       = '(?P<%s>\d{4})';
const PATTERN_MONTH      = '(?P<%s>\d{1,2})';
const PATTERN_DAY        = '(?P<%s>\d{1,2})';
```

You may be able to guess the functionality from the regexp patterns associated with each pre-defined validator, but let's 
go through the expected behavior of each one of them:

* `PATTERN_NUM` - ensures a path element to be numeric
* `PATTERN_DIGIT` - alias to `PATTERN_NUM` 
* `PATTERN_MD5` - ensures a path element to be valid MD5 hash
* `PATTERN_ALPHA` - ensures a path element to be valid alpha-numeric string (i.e. latin characters and numbers, as defined 
by \w pattern of regular expression syntax).
* `PATTERN_ARGS` - is a more sophisticated case that takes some explanation. It tries to match multiple path elements and 
could be useful in URLs like: 
    * `/news/212424/**us/politics/elections**/some-title-goes-here/2012` 
where "us/politics/elections" is a part with variable number of "categories". To parse such URL you could define a validator 
like: 
    
    ```php
    <?php
    
    'path'     => '/news/{id}/{categories}/{title}/{year}',  
    'handlers' => array(
      'id'          => \Zaphpa\Constants::PATTERN\_NUM, 
      'categories'  => \Zaphpa\Constants::PATTERN\_ARGS, 
      'title'       => \Zaphpa\Constants::PATTERN\_ALPHA,
      'year'       => \Zaphpa\Constants::PATTERN\_YEAR, 
     ),
    ```
and you would get the function arguments in the callback as: 

    ```php
    <?php
    
    [params] => Array
    (
      [id] => 212424
      [categories] => Array
          (
              [0] => us
              [1] => politics
              [1] => elections
          )
      [title] => some-title-goes-here
      [year] => 2012
    )
    ```
    
* `PATTERN_ARGS_ALPHA` - acts the exact same way as `PATTERN_ARGS` but limits character set to alpha-numeric ones.
* `PATTERN_ANY` (default) - matches any one argument
* `PATTERN_WILD_CARD` - "greedy" version of `PATTERN_ANY` that can match multiple arguments
* `PATTERN_YEAR` - matches a 4-digit representation of a year.
* `PATTERN_MONTH` - matches 1 or 2 digit representation of a month
* `PATTERN_DAY` - matches 1 or 2 digit representation of a numeric day.

For more custom cases, you can use a custom regex:

    'handlers' => array(
      'id'   => \Zaphpa\Constants::PATTERN_DIGIT, //numeric
      'full_date' => \Zaphpa\Template::regex('\d{4}-\d{2}-\d{2}'); // custom regex
    ),

or attach a validator (you can also think of it as: URL parameter parser) callback function where you can get almost unlimited
flexibility: 

    'handlers' => array(
      'id'         => \Zaphpa\Constants::PATTERN_DIGIT, //numeric
      'uuid'       => 'handle_uuid',       //callback function
    ),

The output of the custom validator callback should match that of a PHP regex call i.e.: should return a parsed array of matches or a null value.

## Appendix A: Setting Up Zaphpa Library

You need to register a PHP script to handle all HTTP requests. For Apache it would look something like the following: 

    RewriteEngine On
    RewriteRule "(^|/)\." - [F]
    RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-f
    RewriteCond %{DOCUMENT_ROOT}%{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} !=/favicon.ico
    RewriteRule ^ /your_www_root/api.php [NC,NS,L]

Please note that this configuration is for a httpd.conf, if you are putting it into an .htaccess file, you may want to remove 
the leading %{DOCUMENT_ROOT} in the corresponding RewriteConds.

The very first RewriteRule is a security-hardening feature, ensuring that system files (the ones typically starting with dot) 
do not accidentally get exposed.

For Nginx, you need to make sure that Nginx is properly configured with PHP-FPM as CGI and the actual configuration in the 
virtualhost may look something like:

    location / {
      # the main router script
      if (!-e $request_filename) {
        rewrite ^/(.*)$ /api.php?q=$1 last;
      }
    }

That's it, for now.