Routes
======

In Scalatra, a route is an HTTP method paired with a URL matching pattern.

{pygmentize:: scala}
    get("/") { 
      // show something 
    }
   
    post("/") { 
      // submit/create something 
    }

    put("/") { 
      // update something 
    }

    delete("/") { 
      // delete something 
    }
{pygmentize}


## Route order

The first matching route is invoked.  Routes are matched from the bottom up.  
_This is the opposite of Sinatra._  Route definitions are executed as part of 
a Scala constructor; by matching from the bottom up, routes can be overridden 
in child classes.

## Path patterns

Path patterns add parameters to the `params` map.  Repeated values are 
accessible through the `multiParams` map.

### Named parameters

Route patterns may include named parameters:

{pygmentize:: scala}
    get("/hello/:name") {
      // Matches "GET /hello/foo" and "GET /hello/bar"
      // params("name") is "foo" or "bar"
      <p>Hello, {params("name")}</p>
    } 
{pygmentize}

### Wildcards

Route patterns may also include wildcard parameters, accessible through the 
`splat` key.

{pygmentize:: scala}
    get("/say/*/to/*) {
      // Matches "GET /say/hello/to/world"
      multiParams("splat") // == Seq("hello", "world")
    }

    get("/download/*.*) {
      // Matches "GET /download/path/to/file.xml"
      multiParams("splat") // == Seq("path/to/file", "xml")
    }
{pygmentize}

### Regular expressions

The route matcher may also be a regular expression.  Capture groups are 
accessible through the `captures` key.

{pygmentize:: scala}
    get("""^\/f(.*)/b(.*)""".r) {
      // Matches "GET /foo/bar"
      multiParams("captures") // == Seq("oo", "ar") 
    }
{pygmentize}

### Rails-like pattern matching

By default, route patterns parsing is based on Sinatra.  Rails has a similar, 
but not identical, syntax, based on Rack::Mount's Strexp.  The path pattern 
parser is resolved implicitly, and may be overridden if you prefer an 
alternate syntax:

{pygmentize:: scala}
    import org.scalatra._

    class RailsLikeRouting extends ScalatraFilter {
      implicit override def string2RouteMatcher(path: String) =
        RailsPathPatternParser(path)

      get("/:file(.:ext)") { // matched Rails-style }
    }
{pygmentize}

### Path patterns in the REPL

If you want to experiment with path patterns, it's very easy in the REPL.

    scala> import org.scalatra.SinatraPathPatternParser
    import org.scalatra.SinatraPathPatternParser

    scala> val pattern = SinatraPathPatternParser("/foo/:bar")
    pattern: PathPattern = PathPattern(^/foo/([^/?#]+)$,List(bar))

    scala> pattern("/y/x") // doesn't match 
    res1: Option[MultiParams] = None

    scala> pattern("/foo/x") // matches
    res2: Option[MultiParams] = Some(Map(bar -> ListBuffer(x)))

Alternatively, you may use the `RailsPathPatternParser` in place of the
`SinatraPathPatternParser`.

## Conditions

Routes may include conditions.  A condition is any expression that returns 
Boolean.  Conditions are evaluated by-name each time the route matcher runs.

{pygmentize:: scala}
    get("/foo") {
      // Matches "GET /foo"
    }

    get("/foo", request.getRemoteHost == "127.0.0.1") {
      // Overrides "GET /foo" for local users
    }
{pygmentize}

Multiple conditions can be chained together.  A route must match all 
conditions:

{pygmentize:: scala}
    get("/foo", request.getRemoteHost == "127.0.0.1", request.getRemoteUser == "admin") {
      // Only matches if you're the admin, and you're localhost
    }
{pygmentize}

No path pattern is necessary.  A route may consist of solely a condition:

{pygmentize:: scala}
    get(isMaintenanceMode) {
      <h1>Go away!</h1>
    }
{pygmentize}

## Actions 

Each route is followed by an action.  An Action may return any value, which 
is then rendered to the response according to the following rules:

`Array[Byte]` - If no content-type is set, it is set to `application/octet-stream`.  
The byte array is written to the response's output stream.

`NodeSeq` - If no content-type is set, it is set to `text/html`.  The node 
sequence is converted to a string and written to the response's writer.

`Unit` - This signifies that the action has rendered the entire response, and 
no further action is taken.

`Any` - For any other value, if the content type is not set, it is set to 
`text/plain`.  The value is converted to a string and written to the 
response's writer.

This behavior may be customized for these or other return types by overriding 
`renderResponse`.

### Parameter handling

Parameters become available to your actions through two methods: `multiParams` 
and `params`.

`multiparams` are a result of merging the standard request params (query 
string or post params) with the route parameters extracted from the route 
matchers of the current route. The default value for an unknown param is the 
empty sequence. Keys return `Seq`s of values. 

`params` are a special, simplified view of `multiparams`, containing only the
head element for any known param, and returning the values as Strings. 

Hitting a URL with a GET like this:
{pygmentize::}
  /articles/52?foo=uno&bar=dos&baz=three&foo=anotherfoo
{pygmentize}
(note that there are two "foo" keys in there)

Assuming the following route, we get the following results inside the action:
{pygmentize:: scala}
  get("/articles/:id") {
    params("id") // => "52"
    params("foo") // => "uno" (discarding the second "foo" parameter value)
    params("unknown") // => generates a NoSuchElementException
    params.get("unknown") // => None - this is what Scala does with unknown keys in a Map

    multiParams("id") // => Seq("52")
    multiParams("foo") // => Seq("uno", "anotherfoo")
    multiParams("unknown") // => an empty Seq
  }
{pygmentize}
