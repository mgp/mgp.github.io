---
layout: post
title: "WSGI, uWSGI, and Nginx"
tags: ["wsgi", "uwsgi", "nginx"]
---

## WSGI

WSGI is specified in [PEP 333](http://www.python.org/dev/peps/pep-0333/) and standardizes an interface between web applications or frameworks and web servers. It presents a very simple interface to the framework author; it is not creating a new web framework. It has two sides: a server side and an application side. For each HTTP request it receives directed at the application, the server side invokes a callable that is provided by the application side. Note that this design allows stacking of WSGI "middleware," or components that act as both a server to some application, and an application to some server. 

### Specification details

The application callable accepts two positional arguments:

* The first is a dictionary object containing CGI-style environment variables. This includes the request method, path, query string, content type and length.
* The second is a callable accepting an HTTP status code and collection of response headers to return to the client.

The application callable must return an iterable yielding zero or more strings, which the server transmits to the client in an unbuffered fashion as the response body. Before it yields the first body string, the application callable must invoke the callable passed as the second argument, allowing the server to send the headers before any body content. If the application omits a header required by HTTP, the server must add it. The server does not transmit the headers until the iterable returned from the application callable returns a value or is exhausted; this allows changing the response status from "200 OK" to "500 Internal Error" if an error occurs, for example.

## uWSGI

[uWSGI](http://projects.unbit.it/uwsgi/) is an application container. It can run Python applications that support the WSGI interface, Perl applications that support the PSGI interface (Perl Superglue Interface, inspired by WSGI), or Ruby applications that support the Rack interface (accepts a CGI environment, returns a status code, and header and body generators).

You can start uWSGI with the `--http` argument to act as a web server for some WSGI application. But this web server is minimal, and you'll want to use a full web server like Nginx or Apache in production. To that end, uWSGI implements a fast, binary protocol called uwsgi (all lowercase) to accept requests instead of HTTP. (The third protocol uWSGI supports is FastCGI.) As of version 0.8.40, Nginx supports the uwsgi protocol out of the box, allowing it to handle parsing all HTTP requests and configuration options for the backing uWSGI.

## Nginx

[Nginx](http://nginx.org/) is the modern alternative to Apache: an open source Web server and a reverse proxy server that uses an asynchronous event-driven approach to handling requests instead of threads or processes. When used with uWSGI and a WSGI application framework like Django or Flask, Nginx parses the HTTP request from the client, forwards it to uWSGI over the uwsgi protocol, which invokes the WSGI callable exposed by the the application framework. The generated response is returned to Nginx over the uwsgi protocol, which Nginx delivers as an HTTP response to the client.

