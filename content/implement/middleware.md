+++
date = "2016-01-30T11:01:06-05:00"
title = "Request Middleware"
+++

goa defines its own type of middleware but also supports "raw" http middleware. The
[middleware](https://github.com/goadesign/middleware) repo contains a number of goa middlewares.

The `middleware` package provides middlewares that do not depend on additional packages other than
the ones already used by `goa`. These middlewares provide functionality that is useful to most
microservices:

* [LogRequest](https://godoc.org/github.com/goadesign/middleware#LogRequest) enables logging of
  incoming requests and corresponding responses. The log format is entirely configurable. The default
  format logs the request HTTP method, path and parameters as well as the corresponding
  action and controller names. It also logs the request duration and response length, as well as the
  the request payload if the DEBUG log level is enabled. Finally if the RequestID middleware is
  mounted LogRequest logs the unique request ID with each log entry.

* [LogResponse](https://godoc.org/github.com/goadesign/middleware#LogResponse) logs the content
  of the response body if the DEBUG log level is enabled.

* [RequestID](https://godoc.org/github.com/goadesign/middleware#RequestID) injects a unique ID
  in the request context. This ID is used by the logger and can be used by controller actions as
  well. The middleware looks for the ID in the [RequestIDHeader](https://godoc.org/github.com/goadesign/middleware#RequestIDHeader)
  header and if not found creates one.

* [Recover](https://godoc.org/github.com/goadesign/middleware#Recover) recover panics and logs
  the panic object and backtrace.

* [Timeout](https://godoc.org/github.com/goadesign/middleware#Timeout) sets a deadline in the
  request context. Controller actions may subscribe to the context channel to get notified when
  the timeout expires.

* [RequireHeader](https://godoc.org/github.com/goadesign/middleware#RequireHeader) checks for the
  presence of a header in the request with a value matching a given regular expression. If the
  header is absent or does not match the regexp the middleware sends an HTTP response with a given
  HTTP status.

Other middlewares listed below are provided as separate Go packages.

#### JWT

Package [jwt](https://godoc.org/github.com/goadesign/middleware/jwt) contributed by [@bketelsen](https://github.com/bketelsen)
adds the ability for goa services to use [JSON Web Token](http://jwt.io/) authorization.

#### CORS

Package [cors](https://godoc.org/github.com/goadesign/middleware/cors) adds
[Cross Origin Resource Sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) support
to goa services.

#### Gzip

Package [gzip](https://godoc.org/github.com/goadesign/middleware/gzip) contributed by [@tylerb](https://github.com/tylerb) adds the ability to compress response bodies using gzip format as specified in RFC 1952.

#### Defer Panic

Package [dpgoa/middleware](https://godoc.org/github.com/deferpanic/dpgoa/middleware) contributed
by [Defer Panic](https://github.com/deferpanic) adds the ability for goa services to leverage the
[Defer Panic service](https://deferpanic.com/).
