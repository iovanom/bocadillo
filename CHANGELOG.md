# Changelog

All notable changes to Bocadillo are documented here.

The format of this document is based on [Keep a Changelog](https://keepachangelog.com).

**Versioning policy**

Bocadillo adheres to [Semantic Versioning](https://semver.org), BUT…

Bocadillo is still in **Alpha** (< 1.0) version. As such, breaking API changes will only cause **minor** version bumps instead of major ones until v1.0 is reached.

As a result, we strongly recommend you read this document carefully before upgrading to any new alpha version. Breaking API changes will be denoted with a **BREAKING** prefix.

## [Unreleased]

### Added

- In-browser traceback of unhandled exceptions when running with `debug=True`.

### Changed

- The `before_dispatch` hook on HTTP middleware classes now takes a `Response` as second argument.
- The `bocadillo.exceptions` module has been removed:
  - `WebSocketDisconnect` has moved to `bocadillo.exceptions`.
  - `UnsupportedMediaType` has moved to `bocadillo.media`.
  - `HTTPError` has moved to `bocadillo.errors` (but is still available at the top level: `from bocadillo import HTTPError`).
- Other internal refactoring that should not affect framework users.

### Fixed

- Even if an error handler was registered for a given exception class, Bocadillo used to return a 500 error response. It will now honor what the error handler does to the `res` object.
- The `after_dispatch` hook on HTTP middleware classes is not called anymore if the inbound HTTP method is not supported by the view.

## [v0.9.1] - 2018-01-04

### Fixed

- Add missing `url` attribute on `WebSocket` objects, which prevented accessing information about the URL from WebSocket views.

## [v0.9.0] - 2018-01-03

This release has **breaking API changes** due to an overhaul of the view system.

If your application uses the features below, you are most likely affected and should review these changes thoroughly before upgrading:

- Use hooks via `@api.before()` or `@api.after()`.
- Restriction of HTTP methods via the `methods` parameter to `@api.route()`.

### Added

- Support for WebSockets, including routing with `@api.websocket_route()`.
- Send a chunk-encoded response with `res.chunked = True`.
- Support for request and response streaming with `async for chunk in req` and `@res.stream`.
- View definition utilities: `from_handler()`, `from_obj()`, `@view()`.
- In particular, the `@view()` decorator (available as `from bocadillo import view`) accepts a `methods` argument originally used by `@api.route()` . Plus,  passing the `all` built-in has the same effect as defining `.handle()` on the analogous class-based view — i.e. supporting all HTTP methods.
- Function-based views are automatically decorated with `@view()` to ensure backwards compatibility.

```python
from bocadillo import API, view

api = API()

# This:
@api.route("/")
async def index(req, res):
    pass

# Is equivalent to:
@api.route("/")
@view()
async def index(req, res):
    pass

# Which is equivalent to:
@api.route("/")
@view(methods=["get"])
async def index(req, res):
    pass

# Which is itself *strictly* equivalent to:
@api.route("/")
class Index:
    async def get(self, req, res):
        pass
```

- API reference for the `views` module.
- Various documentation additions and improvements.

### Changed

- **BREAKING**: hooks were moved to a separate module: `bocadillo.hooks`. You must now use `@hooks.before()` / `@hooks.after()` instead of `@api.before()` / `@api.after()` and `@recipe.before()` / `@recipe.after()`.
- **BREAKING**: hooks must now be placed right above the view being decorated. This affects both function-based views and class-based views (but not method views).

```python
from bocadillo import API, hooks

api = API()

async def before(req, res, params):
    print("before!")

# < 0.9
@api.before(before)
@api.route("/")
async def foo(req, res):
    pass

@api.before(before)
@api.route("/")
class Foo:
    pass

# >= 0.9:
@api.route("/")
@hooks.before(before)
async def foo(req, res):
    pass

@api.route("/")
@hooks.before(before)
class Foo:
    pass
```

### Removed

- **BREAKING**: the `methods` argument to `@api.route()` has been removed. To specify allowed methods on function-based views, you must now use the `@view()` decorator — see below.

```python
from bocadillo import API, view

api = API()

# < 0.9
@api.route("/", methods=["post"])
async def foo(req, res):
    pass

# >= 0.9
@api.route("/")
@view(methods=["post"])
async def foo(req, res):
    pass
```

- Removed dependency on `async_generator`.

## [v0.8.1] - 2018-12-27

### Changed

- `await req.json()` now returns a `400 Bad Request` error response if the input JSON is malformed, which allows to skip handling the `JSONDecodeError` manually.

## [v0.8.0] - 2018-12-26

### Added

- Show Bocadillo version using `boca -v/-V/--version/version`.
- `boca` is now accessible by running Bocadillo as a module: `python -m bocadillo`.
- `HTTPError` is now available at package level: `from bocadillo import HTTPError`.
- Built-in `HTTPError` handlers: `error_to_html`, `error_to_media`, `error_to_text`.
- `detail` argument to `HTTPError`.
- Startup and shutdown events with `api.on()`.
- Security guide.
- Deployment guide.
- `api.run()` now accepts extra keyword arguments that will be passed to `uvicorn.run()`.
- API reference for all public functionality.

### Changed

- Exceptions raised in middleware callbacks were always handled by the HTML `HTTPError` handler. If configured, the one on the `API` will now be used instead.
- The default `HTTPError` handler now returns plaintext instead of HTML.
- The `static` module was renamed to `staticfiles`.
- The `types` module was renamed to `app_types`.
- The `view` module was renamed to `views`.
- The `routing` package has been flattened into a single `routing` module.

### Fixed

- Serving static files from a non-existing directory (including the default one) used to raise an invasive warning. It has been silenced.

### Removed

- Removed example application.
- Removed dependency on `asgiref` for WSGI sub-apps.

## [v0.7.0] - 2018-12-13

### Added

- Recipes: a way to group stuff together and allow composition of bocadillos.
- Recipe books: a way to group multiple recipes into a single recipe.
- Route namespaces via `namespace` argument to `@api.route()`.
- Add GZip support through `enable_gzip`.
- Add ASGI-compliant middleware support via `api.add_asgi_middleware()`.
- Background tasks via `res.background`.

### Changed

- Exceptions raised in `before_dispatch()` and `after_dispatch()` middleware callbacks will now *always* lead to 500 error responses — they won't be handled by error handlers anymore, because these are registered on the `API` which middleware only wrap around. The only exception to this is, of course, `HTTPError`.
- All routes now have an inferred `name` based on their function or class name. Explicit route naming is still possible.
- Because of the above, names of routes in recipes now use the recipe's name as a namespace, i.e. `recipe_name:route_name` instead of `route_name`.
- Unsafe HTTP verbs used to be supported by defaults on function-based routes. Only the safe ones, GET and HEAD, are supported by default now.

### Deprecated

- `RoutingMiddleware` has been renamed to `Middleware`. It will still be available as `RoutingMiddleware` until v0.8.

### Fixed

- Errors returned by custom error handlers could have 200 status in case the handler did not set any status code. It now defaults to 500.
- If `GET` is supported, `HEAD` will automatically be implemented.

## [v0.6.1] - 2018-12-04

### Added

- Documentation on the routing algorithm.
- More documentation on how to write views.
- API reference for the `API` class.

### Changed

- Restructure documentation into 4 clear sections: Getting Started, Topics, How-To and API Reference.
- All things related to routing are now in a dedicated `bocadillo.routing` package, which provides a reusable `RoutingMixin`. This does not introduce any API changes.
- Code refactoring for the hooks and templates features. No API changes involved.
- Rewritten `CONTRIBUTING.md`.

## [v0.6.0] - 2018-11-26

### Added

- Route hooks via `@api.before()` and `@api.after()`.
- Media types and media handlers: `API([media_type='application/json'])`, `api.media_type`,
`api.media_handlers`.
- Support for async callbacks on `RoutingMiddleware`.
- Documentation for the above.
- (Development) Black auto-formatting with pre-commit.
- (Development) Documentation guide in `CONTRIBUTING.md`.

### Changed

- Documentation improvements.

### Fixed

- Exceptions raised inside a middleware callback
(`before_dispatch()` or `after_dispatch()`) are now properly handled by
registered error handlers (they were previously left uncaught).
- Middleware callbacks (especially `before_dispatch()`)
won't be called anymore if the HTTP method is not allowed.

## [v0.5.0] - 2018-11-18

### Added

- Add `boca`, Bocadillo's extensible CLI.
- Add `init:custom` command to generate files for building custom Boca commands.
- Add VuePress-powered documentation site.

### Changed

- Moved docs from README.md to docs site.

## [v0.4.0] - 2018-11-10

### Added

- Named routes. Define a named route by passing a `name` to `@api.route()`. Get the URL path to a route using `api.url_for()` or, in templates, the `url_for()` global.
- Redirections using `api.redirect()`. Can be by route name, internal URL, or external URL. Make it permanent with `permanent=True`.
- Template rendering from string using `api.template_string()`.
- Add allowed hosts configuration through `allowed_host` argument to `API()`.
- *Experimental* support for routing middleware through `bocadillo.RoutingMiddleware`.
- Add CORS support with restrictive defaults. Enable using `enable_cors = True`, configure through `cors_config`.
- Add HSTS support through `enable_hsts`.

### Changed

- Updated example app to demonstrate usage of redirects and named routes.
- Responses without content do not send an empty JSON object response anymore. Instead, an empty `text/plain` response is sent.
- Responses with 204 status code and no content do not set the `Content-Type` header anymore.

## [v0.3.1] - 2018-11-09

### Fixed

- Fixed mis-configured `setup.py` preventing Bocadillo from being installed from `pip`.

## [v0.3.0] - 2018-11-09

### Added

- Plain text responses using `res.text`.
- HTML responses using `res.html`.
- [Jinja2](http://jinja.pocoo.org)-powered template rendering through `await api.template()` and `api.template_sync()`.
- Mount ASGI or WSGI sub-apps using `app.mount(prefix, sub_app)`.
- Static assets using [WhiteNoise](http://whitenoise.evans.io). Configurable through the `static_root` and `static_dir` arguments to `API()`. By default, the `static` folder is served at `/static`. This can be disabled by passing `static_root = None` to `API()`.
- Register more static files locations by mounting a `bocadillo.static()` sub-app.
- Check (at "compile time") that a) a route pattern begins with a forward slash, and b) all parameters of a route are used on its view and vice-versa.
- Use `text/plain` content type if none was set within a view.

### Changed

- Example app in a dedicated `example/` folder.
- Allow overriding a route by reusing a route pattern. Previously, this would have raised an exception.
- Default static root is now `/static`. It previously defaulted to the static directory, which causes issues if the latter was not a relative path.
- The `res.content` attribute is now for raw response content, and will not set the `text/plain` content type anymore. Allows to send responses of arbitrary content type.
- The default error handler now sends HTML content instead of plain text.

## [v0.2.1] - 2018-11-04

### Added

- Add this `CHANGELOG.md`.
- Add error handling.
- Provide a default HTTP error handler, which catches `HTTPError` exceptions during request processing and returns the appropriate HTTP response.
- Allow to customize error handling through `@api.error_handler()` and `api.add_error_handler()`.
- Allow to restrict HTTP methods supported by a route using the `methods` argument to `@api.route()`. Ignored for class-based views: HTTP methods should be restricted by implementing/not implementing the corresponding method on the class.

### Changed

- Return a `405 Method Not Allowed` response when trying to use a non-implemented method on a class-based view. The previous behavior was to raise an uncaught `ValueError`.
- Updated `example.py`.

### Fixed

- Fixed a bug that prevented routes without parameters to be handled correctly.
- Prevent registering multiple routes on the same pattern.

## v0.1.0 - 2018-11-04

### Added

- The `API` class, an ASGI-compatible application.
- `Request` and `Response` objects, which are wrappers around Starlette's.
- Plain text responses using `res.content`.
- JSON responses through `res.media`.
- Automatic configuration of the response's `Content-Type`: `text/plain` by default, `application/json` if `response.media` was set or `res.content` was left empty.
- Route registration through `@api.route()`.
- Parametrized routes through f-string expressions, e.g. `{my_param}`. Parameters are passed directly to the view, e.g. `my_view(req, resp, my_param)`. Parameters are compliant with the [Literal string interpolation](https://www.python.org/dev/peps/pep-0498/#specification) specification. In particular, type specifiers are supported (e.g. `{age:d}`) which provides basic validation capabilities.
- Class-based views. HTTP methods (GET, POST, PUT, PATCH, DELETE) are mapped to the corresponding lowercase methods on the class, e.g. `.get()`. A generic `.handle()` method can also be given to process any request (other methods will then be ignored).
- Default bind host and port: `127.0.0.1:8000`.
- Automatic host and port based on the `PORT` environment variable. If `PORT` is set, a) the app will bind on that port, b) if no host was specified, the app will bind to known hosts (i.e. `0.0.0.0`).
- `example.py` app.
- `README.md`.
- `CONTRIBUTING.md`.


[Unreleased]: https://github.com/bocadilloproject/bocadillo/compare/v0.9.1...HEAD
[v0.9.1]: https://github.com/bocadilloproject/bocadillo/compare/v0.9.0...v0.9.1
[v0.9.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.8.1...v0.9.0
[v0.8.1]: https://github.com/bocadilloproject/bocadillo/compare/v0.8.0...v0.8.1
[v0.8.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.7.0...v0.8.0
[v0.7.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.6.1...v0.7.0
[v0.6.1]: https://github.com/bocadilloproject/bocadillo/compare/v0.6.0...v0.6.1
[v0.6.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.5.0...v0.6.0
[v0.5.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.4.0...v0.5.0
[v0.4.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.3.1...v0.4.0
[v0.3.1]: https://github.com/bocadilloproject/bocadillo/compare/v0.3.0...v0.3.1
[v0.3.0]: https://github.com/bocadilloproject/bocadillo/compare/v0.2.1.post3...v0.3.0
[v0.2.1]: https://github.com/bocadilloproject/bocadillo/compare/v0.1.0...v0.2.1.post3
