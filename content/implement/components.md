+++
date = "2016-01-30T11:01:06-05:00"
title = "runtime"
description = "The goa runtime."
categories = ["Runtime"]
tags = ["dsl"]
+++

     <p>
                                The goa runtime is implemented by the goa package. It includes the
                                implementation of the goa action context which provides the means
                                to access the request state and write the response. The package
                                also contains a number of data structures and algorithms
                                that provide supporting functionality to the service. These include
                                logging, error handling, versioning support etc.
                                goa follows the "battery included" model for the supporting
                                functionality letting you customize all aspects if the provided
                                default is not sufficient.
                        </p>
                        <h3>The goa Action Context</h3>
                        <p>
                                The action context is a data structure that is provided to all goa
                                controller action implementations as first parameter. It leverages the
                                <a href="https://blog.golang.org/context">work done</a> at Google
                                around passing contexts across interface boundaries and adds to it
                                by providing additional methods tailored to the goa use case.
                        </p>
                        <p>
                                The context exposes methods to access the request state and write
                                the response in a generic way like many other Go web frameworks. For
                                example path parameters or querystring values can be accessed using
                                the method <code>Get</code> which returns a string. However goa goes
                                one step further and leverages the code generation provided by `goagen`
                                to define <i>action specific</i> fields that provide access to the
                                same state using "typed" methods. So for example if a path parameter
                                called <code>ID</code> is defined in the design as being of type
                                <code>Integer</code> the corresponding controller action method
                                accepts a context data structure which exposes a field named <code>ID</code>
                                of type <code>int</code>. The same goes for the request payload so that
                                accessing the <code>Payload</code> field of an action context returns
                                a data structure that is specific to that action as described in the
                                design. This alleviates the need for reflection or otherwise "binding"
                                the context to a struct.<br/>
                                <br/>
                                The same goes for writing responses: while the underlying http
                                ResponseWriter is available to write the response, the action context
                                also provides action specific methods for writing the responses
                                described in the design. These generated methods take care of writing
                                the correct status code and content-type header for example. They
                                also make it possible to specificy the response payload using custom
                                data structures generated from the media type described in the design.
                        </p>
                        <p>
                                As mentioned earlier each controller action context wraps a golang
                                package context. This means that deadlines and cancelation signals
                                are available to all action implemetations. The built-in
                                <a href="https://godoc.org/github.com/goadesign/goa#Timeout">Timeout</a> middleware
                                takes advantage of this ability to let services or controllers
                                define a timeout value for all requests.
                        </p>
                        <h3>Supporting Functionality</h3>
                        <h4>Service Mux</h4>
                        <p>
                                The goa HTTP request mux is in charge of dispatching incoming requests
                                to the correct controller action. It implements the <code>ServeMux</code>
                                interface which on top of the usual binding of HTTP method and path
                                to handler also provides support for API versioning.
                        </p>
                        <p>
                                The <code>ServeMux</code> interface <code>Handle</code> method associates
                                a request HTTP method and path to a HandleFunc which is a function
                                that accepts a http ResponseWriter and Request as well as a instance
                                of url Values that contain all the path and querystring parameters.<br/>
                                <br/>
                                The interface also exposes a <code>Version</code> method that gives
                                access to version specific muxes. This makes it possible to define
                                different controller actions for the same request HTTP method and
                                path but different API versions. The actual algorithm used to
                                compute the targeted API version is provided via an instance of
                                <code>SelectVersionFunc</code>. goa comes with several implementations
                                of SelectVersionFunc:
                        </p>
                        <ul>
                                <li>
                                        The <code>PathSelectVersionFunc</code> function creates a
                                        <code>SelectVersionFunc</code> that extracts the version from the request
                                        path.
                                </li>
                                <li>
                                        The <code>HeaderSelectVersionFunc</code> function creates a
                                        <code>SelectVersionFunc</code> that extracts the version from the given
                                        HTTP request header.
                                </li>
                                <li>
                                        The <code>QuerySelectVersionFunc</code> function creates a
                                        <code>SelectVersionFunc</code> that extracts the version from
                                        the given querystring value.
                                </li>
                        </ul>
                        <p>
                                The function <code>CombineSelectVersionFunc</code> makes it possible to
                                combine any number of <code>SelectionVersionFunc</code> to produce
                                arbitrarily complex lookup algorithms.
                        </p>
                        <h4>Middleware</h4>
                        <p>
                                goa defines its own type of middleware but also supports "raw" http
                                middleware. The <a href="https://github.com/goadesign/goa-middleware">goa-middleware</a>
                                repo contains a number of goa middlewares.
                        </p>
                        <h4>Logging</h4>
                        <p>
                                goa uses structured logging so that logs created at each level contain
                                all the contextual information. The root logger is the service-level
                                <code>Logger</code> field. Loggers are derived from it for each
                                controller and for each action. Finally a logger is also created for
                                each request so that log entries created inside a request contain
                                the full context: service name, controller name, action name and
                                unique request ID.
                        <p>
                        <h4>Error Handling</h4>
                        <p>
                                All goa actions return an error. Error handlers can be defined at the 
                                controller or service level. If an action returns a non-nil error
                                then the controller error handler is invoked. If the controller does
                                not define a error handler then the service-wide error handler is
                                invoked instead. The default goa error handler simply returns a 500
                                response containing the error details in the body.
                        <p>
                        <h4>Graceful Shutdown</h4>
                        <p>
                                A goa service can be instantiated via `NewGraceful` in which case the
                                http server is implemented by the <a href="https://godoc.org/github.com/tylerb/graceful">graceful package</a>
                                which provides graceful shutdown behavior where upon receving a
                                shutdown signal the service waits until all pending requests are
                                completed before terminating.
                        </p>
                        <h3>Swapping the Batteries</h3>
                        <h4>Error Handling</h4>
                        <p>
                                The service interface exposes a <code>SetHandler</code> method which
                                allows overriding the default service error handler. goa comes with
                                two built-in error handlers:
                        </p>
                        <ul>
                                <li>
                                        The <code>DefaultErrorHandler</code> returns a 400 if the error
                                        is an instance of <code>BadRequestError</code>, 500 otherwise.
                                        It also always writes the error message to the response body.
                                </li>
                                <li>
                                        The <code>TerseErrorHandler</code> behaves identically to the
                                        default error handler with the exception that it does not write
                                        the error message to the response body for internal errors
                                        (i.e. errors that are not instances of <code>BadRequestError</code>).
                                </li>
                        </ul>
                        <p>
                                Custom error handlers can be easily swapped in, they consist of a
                                function that accepts an instance of an action context and of an 
                                error.
                        </p>
                        <h4>Request Mux and Versioning</h4>
                        <p>
                                As mentioned above the goa mux supports defining version specific
                                muxes. Different versions can be defined in the design using the 
                                <code>Version</code> DSL:
                        </p>
                        <pre><code class="go">package design

import (
        . "github.com/goadesign/goa/design"
        . "github.com/goadesign/goa/design/apidsl"
)

var _ = API("cellar", func() {
        Description("A basic example of an API implemented with goa")
        Scheme("http")
        Host("localhost:8080")
})

var _ = Version("1.0", func() {
        Title("The virtual winecellar v1.0 API")
        // ... other API level properties
})

var _ = Version("2.0", func() {
        Title("The virtual winecellar v2.0 API")
        // ... other API level properties
})

var _ = Resource("bottle", func() {
        BasePath("/bottles")
        Version("1.0")
        Version("2.0")
        // ... other resource properties
})

var _ = Resource("bottle", func() {
        BasePath("/bottles")
        Version("3.0")
        // ... other resource properties
})
</code></pre>
                        <p>
                                When <code>goagen</code> sees that the design defines versions it
                                produces code that leverages the ServeMux interface Version method
                                to mount controllers onto the appropriate version mux:
                        </p>
                        <pre><code class="go">func MountBottleV1Controller(service goa.Service, ctrl v1.BottleController) {
                                // ...
                        </code></pre>
                        <p>
                                Each version defined in the design produces a different package containing
                                the corresponding generated controllers.
                        </p>
                        <p>
                                The generated code relies on the <code>ServeMux</code> method exposed
                                by the service to retrieve the top-level mux. The goa default mux
                                implementation relies on the <a href="https://github.com/julienschmidt/httprouter">httprouter</a>
                                package to implement the low level dispatch. Other low level routers
                                can easily be subsituted by providing an implementation of the
                                ServeMux interface.
                        </p>