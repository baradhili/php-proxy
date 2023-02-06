# PHP Proxy

[![Build Status](http://img.shields.io/travis/jenssegers/php-proxy.svg)](https://travis-ci.org/jenssegers/php-proxy) [![Coverage Status](http://img.shields.io/coveralls/jenssegers/php-proxy.svg)](https://coveralls.io/r/jenssegers/php-proxy?branch=master)

This is a HTTP/HTTPS proxy script that forwards requests to a different server and returns the response. The Proxy class uses PSR7 request/response objects as input/output, and uses Guzzle to do the actual HTTP request.

## Installation

Install using composer:

```
composer require baradhili/proxy
```

## Example

The following example is in Laravel 9, is is a Controller that accepts pouchdb requests on http://host/api/db/ and redirects them to http://localhost:5984/ so that a pouchdb client can avoid CORS errors and the server can use Laravel's auth methods to control access.

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Http\Response;
// use Illuminate\Support\Facades\Http;
use Proxy\Proxy;
use Proxy\Adapter\Guzzle\GuzzleAdapter;
use Proxy\Filter\RemoveEncodingFilter;
use Laminas\Diactoros\ServerRequestFactory;
use Laminas\Diactoros\Uri;
use Psr\Http\Message\RequestInterface;


class CouchProxyController extends Controller
{
    //
    public function __construct()
    {
        //$this->middleware('auth:api', ['except' => ['login','register']]);
    }

    public function proxy()
    {
        //we are going to get the request directly      

        // Create a PSR7 request based on the current browser request.
        $request = ServerRequestFactory::fromGlobals();
        $uri = $request->getUri();
        
        $uri = $uri->withHost('192.168.1.161:5984');
        $path = $uri->__toString();
        //select everything between  '/api/db' stopping at the end or the string '?'
        $pattern = '/\/api\/db\/(.*?)(\?|$)/';
        preg_match($pattern,$path,$match);
        //if no match, don't do anything
        if(count($match)>0){
            $path = $match[1];
            $uri = $uri->withPath($path);
        }
        // change request
        $request = $request->withUri($uri);

        // Create a guzzle client
        $guzzle = new \GuzzleHttp\Client();

        // Create the proxy instance
        $proxy = new Proxy(new GuzzleAdapter($guzzle));

        try {
            // Forward the request and get the response.
            
            $response = $proxy
                ->forward($request)
                ->filter(function ($request, $response, $next) {
                    $response = $next($request, $response);
                    if ($response->hasHeader('Access-Control-Allow-Credentials')) {
                        // Log::debug(print_r($response->headers, true));
                        // $response = $response
                        //     ->withHeader('X-Proxy-Location', '*');
                        $response = $response->withoutHeader('Access-Control-Allow-Credentials');
                    }
                    if ($response->hasHeader('Access-Control-Allow-Origin')) {
                        $response = $response->withoutHeader('Access-Control-Allow-Origin');
                    }
                    if ($response->hasHeader('Server')) {
                        $response = $response->withoutHeader('Server');
                    }

                    return $response;
                })
                ->to('http://localhost:5984/');
            
            $reasonPhrase = $response->getReasonPhrase();
            
            if ($reasonPhrase == 'Bad Request ') {
                $uri = $request->getUri();
                Log::debug('Bad request '.print_r($uri->__toString(), true));
            }
            if ($reasonPhrase == 'Object Not Found') {
                $uri = $request->getUri();
                Log::debug('Object Not Found '.print_r($uri->__toString(), true));
            }
            // Output response to the browser.
            //assemble things for Laravel response
            $rawheaders = $response->getHeaders();
            //clean headers
            $headers=[];
            foreach($rawheaders as $key => $value){
                $headers[$key]=$value[0];
            }
            //extract the response contents
            $body = $response->getBody()->getContents();
            $finalResp =  response($body)
                ->header('X-Couch-Request-ID', $headers['X-Couch-Request-ID'])
                ->header('X-CouchDB-Body-Time', $headers['X-CouchDB-Body-Time'])
                ->header('Date', $headers['Date'])
                ->header('Content-Type', $headers['Content-Type']);
            //ETag doesn't always exist apparently
            if (array_key_exists('ETag', $headers)){
                $finalResp->header('ETag',$headers['ETag']);
            }
            return $finalResp;

        //     (new \Laminas\HttpHandlerRunner\Emitter\SapiEmitter)->emit($response);
        //not the right thing to do, but we'll clean it later TODO  lolz like never
        } catch(\GuzzleHttp\Exception\BadResponseException $e) {
            // Correct way to handle bad responses
            (new \Laminas\HttpHandlerRunner\Emitter\SapiEmitter)->emit($e->getResponse());
        }

    }
    
}

```

## Filters

You can apply filters to the requests and responses using the middleware strategy:

```php
$response = $proxy
	->forward($request)
	->filter(function ($request, $response, $next) {
		// Manipulate the request object.
		$request = $request->withHeader('User-Agent', 'FishBot/1.0');

		// Call the next item in the middleware.
		$response = $next($request, $response);

		// Manipulate the response object.
		$response = $response->withHeader('X-Proxy-Foo', 'Bar');

		return $response;
	})
	->to('http://example.com');
```
