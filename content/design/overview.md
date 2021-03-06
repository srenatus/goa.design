+++
date = "2016-01-30T11:01:06-05:00"
title = "The goa API design language"
+++

The goa API Design Language is a DSL implemented in [Go](https://golang.org) that makes it possible
to describe arbitrary microservice APIs. While the main focus is REST based HTTP APIs, the language
is flexible enough to describe APIs that follow other methodologies as well.
[Plugins](../extend/dsls.html) can extend the core DSL to allow describing other aspects of
microservices such as database models, service discovery integrations, failure handlers etc.

## Design Definitions

At its core the design language consists of functions that are chained together to describe
*definitions*. The goa design language root definition is the API definition, the DSL to define it
looks like this:

```go
import (
        . "github.com/goadesign/goa/design"
        . "github.com/goadesign/goa/design/apidsl"
)

var _ = API("My API", func() {                           // "My API" is the name of the API used in docs
	Title("Documentation title")                     // Documentation title
	Description("The next big thing")                // Longer documentation description
	Host("goa.design")				 // Host used by Swagger and clients
	Scheme("https")					 // HTTP scheme used by Swagger and clients
	BasePath("/api")				 // Base path to all API endpoints
	Consumes("application/json", "application/xml")  // Media types supported by the API
	Produces("application/json")                     // Media types generated by the API
})
```

There are more language keywords (functions) supported by the API DSL listed in the
[reference](../reference/goa/design.html).

*A side note on "dot import" as this question comes up often: the goa API design language is a DSL
implemented in Go and is __not__ Go. The generated code or any of the actual Go code in goa does
not make use of "dot imports". Using this technique for the DSL results in far cleaner looking
code. It also allows mixing DSLs coming from plugins transparently, moving on...*

## API Endpoints

Apart from the root API definition the goa API design language also makes it possible to describe
the actual endpoints together with details on the shape of the requests and responses. The
`Resource` function defines a set of related API endpoints - a resource if the API is RESTful. Each
actual endpoint is described using the `Action` function. Here is an example of a simple `Operands`
resource exposing an `add` action (API endpoint):

```go
var _ = Resource("Operands", func() {                            // Define the Operands resource
        Action("add", func() {                                   // Define the add action
                Routing(GET("/add/:left/:right"))                // The relative path to the add endpoint
                Description("add returns the sum of the left and right parameters in the response body")
                Params(func() {                                  // Define the request parameters found in the URI (wildcards)
                        Param("left", Integer, "Left operand")   // Define left parameter as path segment captured by :left
                        Param("right", Integer, "Right operand") // Define right parameter as path segment captured by :right
                })
                Response(OK, "plain/text")                       // Define response
        })
})
```

The `Resource` and `Action` DSLs support many more keywords described in the [reference](../reference/goa/design.html).

## Data Types

The goa API design language also makes it possible to describe arbitrary data types that the API
uses both in its request payloads and response media types. Looking first at request payloads: the
`Type` function describes a data structure by listing each field using the `Attribute` function. It
can also make use of the `ArrayOf` function to define arrays or fields that are arrays. Here is what
it looks like:

```go
// Operand describes a single operand with a name and an integer value.
var Operand = Type("Operand", func() {
	Attribute("name", String, "Operand name", func() {  // Attribute name of type string
                Pattern("^x")                               // with regex validation
        })
	Attribute("value", Integer, "Operand value")        // Attribute value of type integer
        Required("value")                                   // only value is required
})

// Series represents an array of operands.
var Series = ArrayOf(Operand)
```

These types can be used to define an action payload (amongst other things):

```go
Action("sum", func() {              // Define the sum action
        Routing(POST("/sum"))        // The relative path to the add endpoint
        Description("sum returns the sum of all the operands in the response body")
	Payload(Series)             // Payload defines the action request body shape.
        Response(OK, "plain/text")  // Define response
})
```

Looking at responses next, the goa design language `MediaType` function describes media types which
represent the shape of response bodies. The definition of a media types is similar to the definition
of types (media types are a specialized kind of type) however there are two properties unique to
media types:

* Views make it possible to describe different renderings of the same media type. Often times an API
  uses a "short" representation of a resource in listing requests and a more detailed representation
  in requests that return a single resource. Views cover that use case by providing a way to define
  these different representations. A media type definition *must* define the default view used to
  render the resources.
* The other media type specific property is links. Links represent related resources that should be
  rendered embedded in the response. The view used to render links is `link` which means that
  media types being linked to must define a `link` view.

Here is an example of a media type definition:

```go
// Results is the media type that defines the shape of the "add" action responses.
var Results = MediaType("vnd.application/goa.results+json", func() {
        Description("The results of an operation")
        Attributes(func() {                                         // Define media type attributes
                Attribute("value", Integer, "Results value")        // Operation results attribute
                Attribute("requester", User, "Operation requester") // Requester attribute
        })
        Links(func() {                 // Define the links embedded in the media type
                Link("requester")      // One link to the requester,
                                       // will be rendered using the "link" view of User media type
        })
        View("default", func() {       // Define default view
                Attribute("value")     // Include value field in default view
                Links()                // And render links
        })
        View("extended", func() {      // Define extended view
                Attribute("value")     // Include value field
                Attribute("requester") // Extended view renders the default view of the requester
        })
})

// User is the media type used to render user resources.
var User = MediaType("vnd.application/goa.users+json", func() {
        Description("A user of the API")
        Attributes(func() {
                Attribute("id", String, "Unique identifier")
                Attribute("href", String, "User API href")
                Attribute("email", String, "User email", func() {
                        Format("email")
                })
        })
        View("default", func() {
                Attribute("id")
                Attribute("href")
                Attribute("email")
        })
        View("link", func() {     // The "link" view used to render links to User media types.
                Attribute("href")
        })
})
```

## Responses

Defining API responses should also include specifying their status code and describing the valid
values for other HTTP headers. The goa API design language allows defining *response templates*
at the API level that any action may leverage to define its responses. Such templates may accept
an arbitrary number of string arguments to define any of the response properties. goa provides
response templates for all standard HTTP code that define the status so that it is not required to
define templates for the simple case. Here is an example of a response template definition:

```go
var _ = API("My API", func() {                           // "My API" is the name of the API used in docs
        // ... other API DSL
        ResponseTemplate(Created, func(hrefPattern string) { // Define the "created" response template
                Description("Resource created")          // that takes one argument.
                Status(201)                              // using status code 201
                Header("Location", func() {              // and defining the "Location" header
                        Pattern(hrefPattern)             // with a regex validation.
                })
        })
})
```

Now that the response template is defined it can be used in an action definition as follows:

```go
Action("sum", func() {                         // Define the sum action
        Routing(POST("/sum"))                  // The relative path to the add endpoint
        Description("sum returns the sum of all the operands in the response body")
	Payload(Series)                        // Payload defines the action request body shape.
        Response(Created, "^/results/[0-9]+")  // The regexp that validates the Location header
})
```

## Conclusion

There is [a lot more](../reference) to the design language but this overview should have given you a
sense for how it works. It doesn't take long for the language to feel natural which makes it
possible to quickly iterate and refine the design. The [Swagger](swagger.html) specification generated
from the design can be shared with stakeholders to gather feedback and iterate. Once finalized
[goagen](../implement/goagen.html) generates the API scaffolding, request contexts and low level from
the design thereby baking it into the implementation. The design becomes a living document always
up-to-date with the implementation.
