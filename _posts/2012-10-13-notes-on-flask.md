---
layout: post
title: "Notes on Flask"
tags: ["flask", "python"]
---

Notes taken from the [Flask documentation](http://flask.pocoo.org/docs/) when the latest version was 0.9:

## Foreword for experienced programmers

* Flask uses thread-local objects internally so that you don’t have to pass objects around from function to function within a request in order to stay threadsafe.

## Quickstart

* The `Flask` constructor takes the module or package name to know where to look for templates, static files, and so on.
* If `debug` is disabled, pass `host='0.0.0.0'` to `run()` to listen on all public IPs and make the server publicly available.
* Setting `debug` to `True` allows the execution of arbitrary code, and so should never be run in production environments.
* A variable part of an URL with a `path` converter is like the default string converter, but also accepts slashes.
* If a URL has a trailing slash, accessing it without one will automatically redirect; if a URL doesn't have a trailing slash, accessing it with one will generate a 404.
* The `url_for` method maps keyword arguments to variable parts; unknown parts are appended as query parameters.
* Method `PUT` is similar to `POST`, but the server might trigger the store procedure multiple times by overwriting the old values more than once.
* Form data in a `PUT` or `POST` request is accessed through the `form` attribute; query parameters are accessed through `args`.
* Passing an error code to `abort` calls the corresponding method with an `errorhandler()` decorator, which must return the error code after its `render_template()` call.
* The `session` object is separate from the `request` object, and is implemented using cryptographically signed cookies.
* To generate a good secret key for sessions, use the output from a call to `os.urandom(24)`.

## Templates

* In templates you have access to the `config`, `request`, `session`, and `g` objects, and the `url_for()` and `get_flashed_messages()` functions.
* If using the `tojson()` filter, chain it with `safe()` if the output is displayed inside a `<script>` tag.
* If you want to inject secure HTML into a template, wrap it in a `Markup` object before passing it to the template.
* The keys and values returned by a context processor are merged with the template context, for all templates in the app.

## Testing Flask applications

* Flask allows testing your application by exposing the Werkzeug test `Client` and handling the context locals for you.
* The `TESTING` configuration flag disables the error catching during request handling so that you get better error reports when issuing requests.
* Using `test_request_context()` in a `with` statement activates a request context temporarily, giving access to the `request`, `g` and `session` objects like in view functions.
* The `before_request()` and `after_request()` functions aren't called with a test request context; for the former, call `preprocess_request()` yourself.
* The `session_transaction()` function allows creating or modifying a session before a request is issued in the context of the test client.

## Logging application errors

* To override the human-readable time returned by `%(asctime)s`, subclass the formatter and override the `formatTime()` method.
* To configure loggers for third-party modules like `sqlalchemy`, retrieve them using `getLogger`.

## Configuration handling

* The `config` attribute is where Flask itself puts certain configuration values, extensions put their configuration values, and you can put configuration values.
* For debugging, set `TRAP_HTTP_EXCEPTIONS` to `True` to not execute error handlers of HTTP exceptions, and `TRAP_BAD_REQUEST_ERRORS` to `True` to return tracebacks for Werkzeug and `BadRequest` errors.
* The `from_envvar` function specifies an environment variable that is a pathname; its contents then override values in the configuration.
* Best practice is to keep a default configuration, use an environment variable to switch between configurations, and use a tool like fabric to push code and configurations separately.
* Instance folders are suitable for values that change at runtime or configuration files, and can specify resources outside the application’s folder, or `Flask.root_path`.

## Signals

* The `connected_to()` helper method allows temporarily subscribing a function to a signal with a context manager on its own.
* Signals, or events, are singletons; to emit a signal, you must pass the sender to its `send()` method.
* If emitting a signal from a random function, you can pass `current_app._get_current_object()` as the sender.

## Pluggable views

* By representing views as methods of classes, you can use inheritance and pass differing arguments to the constructor to promote reuse and modularity.
* To specify decorators for the view function of a pluggable class, create a static, list variable named `decorators`.

## The application context

* The application context contains functionality that may be needed during a request, but is derived from the application, and not the request.
* The application context never moves between threads or is shared between requests, and is therefore suitable to store database connection information, etc.

## The request context

* The request context is internally maintained as a stack; pushing and popping multiple times is useful to implement things like internal redirects.
* If a `before_request()` function executed before the view returns a response, the other functions are no longer called.
* With an unhandled exception, the `after_request()` callbacks are not called, but the `teardown_request()` function is always executed.
* Note that when using `with app.test_client()` in tests, `teardown_request()` callbacks aren't executed until the block has been exited.
* Method `_get_current_object()` returns the underlying proxied object for inheritance checks or when the real object reference is needed, like for sending signals.

## Blueprints

* Blueprints provide separation at the Flask level, share application configuration, and can change an application object as necessary.
* Whether a blueprint can be mounted more than once depends on how it is implemented.
* A blueprint's template folder is added to the searchpath of templates but with a lower priority than that of the application, allowing overriding of templates.
* When using `url_for()` outside a blueprint, you must qualify endpoints with the blueprint name; inside the blueprint, prefix the endpoint with `.`.

## Patterns: Larger applications

* An `__init__.py` file should create the Flask application object, and then import all the view functions.

## Patterns: Deploying with Distribute

* distribute, formerly setuptools, extends distutils to add dependency support, a package registry, and `easy_install`, soon to be replaced by `pip`.
* Distributing resources with standard modules is not supported by distribute; your application must be a package.
* For distribute to lookup subpackages automatically instead of listing them explicitly, use `find_packages()`.
* To include the `templates` and `static` subdirectories, add them to your `MANIFEST.in` file and set `include_package_data` to `True`.
* The `install` command copies into the site-packages folder; `develop` creates a symlink so changes are seen instantly.

## Patterns: Deploying with Fabric

* To use Fabric, the application already has to be a package and requires a working `setup.py` file.
* All functions defined in `fabfile.py` are fab subcommands, and execute on hosts defined either in the file or on the command line.
* The WSGI file must import the application and also set an environment variable so that the application knows where to find the configuration file.
* To handle differing configuration files between servers, copy them to all servers, and then symlink each file for a server to its expected location, like in `/var/www`.

## Patterns: SQLAlchemy in Flask

* If not using the Flask-SQLAlchemy extension, you must use `scoped_session`, and have a `teardown_request` decorator that calls `db_session.remove()`.

## Patterns: View decorators

* When defining a decorator, use `functools.wraps()` to update the `__name__`, `__module__`, and some other attributes of a function.
* Common decorator uses include redirecting logged out users to a login page when needed, and returning values from and populating a cache.

## Patterns: Form validation with WTForms

* The Flask-WTF extension adds a handful of helper functions that make working with forms easier in Flask.
* Rendered fields in WTForms accept HTML attributes as keyword arguments, and because they are already HTML escaped, should be passed through the `|safe` filter.

## Patterns: Template inheritance

* The `extends` tag must be the first tag in a child template.
* To render the contents of a block defined in the parent template, use `{{ super() }}`.

## Patterns: Message flashing

* Flashing allows recording a message at the end of a request and accessing it on the next request, and only that request.
* Categories can be used to style messages differently, or to display only a subset of messages to the user.

## Patterns: AJAX with jQuery

* The `url_for()` method handles your application moving to a different path; in a `script` tag, set the root as `request.script_root|tojson|safe`.
* The `script` tag is declared CDATA and so there must not be `</` inside; the `tojson` filter will escape slashes for you in this context.

## Patterns: Custom error pages

* If you have some kind of access control on your website, send a 403 (forbidden) code for disallowed resources.
* The 410 (gone) code is for resources that previously existed but were deleted.

## Patterns: Deferred request callbacks

* If you must modify the response at a point where the response doesn't exist yet, attach callbacks to the `g` object and call them in an `after_request` callback.

## API: Application object

* If using a single module, pass `__name__` to the `Flask` constructor; if using a package, hardcode the name, or pass `__name__.split('.')[0]`.
* `app_context()` binds the application only; as long it is bound to the current context, `flask.current_app` points to that application.
* When `debug` is `True`, the debugger is run when an unhandled exception occurs, and the integrated server automatically recognizes changes to the code.
* The `errorhandler` decorator can be registered not just with status codes, but with arbitrary exception types.
* By default the logger name is the name passed to the `Flask` constructor.
* `make_response()` creates a `response_class` instance; a passed string is used as the response body, or it accepts a `(response, status, headers)` tuple.
* Instead of overriding `open_session`, replace the `session_interface`.
* `permanent_session_lifetime` specifies the expiration date of a permanent session; by default it is 31 days.
* To run the application in debug mode, but disable the code execution on the interactive debugger, pass `use_evalex=False` to `run()`.
* Teardown functions registered with `teardown_request()` must take every necessary step to avoid failure, like catching all exceptions.
* `update_template_context` injects `request`, `session`, `config`, and `g`, and everything template context processors want to inject into the given template context.

## API: Incoming request data

* The `values` attribute combines the contents of both `form` and `args`.
* If the mimetype is application/json then the JSON is stored in the `json` attribute; otherwise it is in the `data` attribute.

## API: Response objects

* Typically you don't have to create a response object because `make_response()` will do this for you.

## API: Sessions

* Modifications of mutable structures in a sessions are not picked up automatically; you must set `modified` to `True` yourself.
* If `permanent` is `False`, then the session will be deleted when the user closes the browser.

## API: Session interface

* To aid user experience, a ready-only null session is used if real session support could not be loaded due to a configuration error.

## API: Application globals

* `flask.g` allows sharing data between functions for the lifetime of a request in a thread-safe manner; useful for storing the user currently logged in.

## API: Useful functions and classes

* Function `make_response()` delegates to `flask.Flask.make_response()`, and can accept the same parameters, like the status code.
* The `after_this_request` decorator is useful when not in a view function and the response object is not yet created.
* Function `safe_join()` safely joins a directory and filename, raising `NotFound` if the resulting path is outside the directory.
* Wrapping a value in `Markup` marks it as without need for escaping; to actually escape a value before wrapping, call `Markup.escape()`.
* `Markup` extends the unicode (string) class, and so it supports operations like `%` and `+` while escaping the arguments.

## API: Returning JSON

* The `dumps()` function of the JSON module is available as the `tojson` filter; inside a `<script>` tag, pass its output through `safe` to disable escaping.

## API: Configuration

* When loading a configuration from a file or module, only uppercase keys are added, so lowercase values can be used as temporary ones.
* When loading configurations, `from_pyfile()` behaves as if the file were imported with `from_object()`, and `from_envvar()` as if read using `from_pyfile()`.
