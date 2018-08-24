---
title: OpenAPI (fka Swagger) 3 support in Eclipse Vert.x now in test stage!
date: 2017-07-04
template: post.html
author: slinkydeveloper
draft: false
---
[IMPORTANT Now on upstream! | We have published this package with name `vertx-web-api-contract` ]

As GSoC 2017's student, I'm actually working on an embedded support to OpenAPI 3 standard inside Eclipse Vert.x framework. Now, after a lot of work, you can try it!

## Why OpenAPI 3?
OpenAPI 2 is the most important industry-grade standard for API Specifications. As you can see on [official blog of OpenAPI Initiative](https://www.openapis.org/blog/2017/05/25/the-open-api-initiative-is-sending-you-a-save-the-date-card?utm_source=Blog&utm_medium=Twitter&utm_campaign=SaveTheDate), the release of version 3 is behind the corner, so we want to give to our community the latest tools for the latest standards!

Vert.x project objective is to give you more integrated tools. With this new support, it gives you the ability to use the [Design Driven](https://swaggerhub.com/blog/api-design/design-first-or-code-first-api-development/) (or Design First) approach **without loading any thirds parties libraries**.

## Features
The actually supported features are the following (we refeer to document Version 3.0.0-rc2):
* OpenAPI 3 compliant API specification validation (thanks to [Kaizen-OpenApi-Parser](https://github.com/RepreZen/KaiZen-OpenApi-Parser)) with **loading of external Json schemas**
* Automatic request validation
* Automatic mount of security validation handlers
* Automatic 501 response for not implemented operations
* Router factory to provide all this features to users

Automatic request validation is provided by a new handler: `ValidationHandler`. You can also define your own `ValidationHandler` without API specifications, but I will discuss it later.

The request validation (provided by subclass `OpenAPI3RequestValidationHandler`) actually supports:
* Parameters defined in [Parameter object](https://github.com/OAI/OpenAPI-Specification/blob/OpenAPI.next/versions/3.0.md#parameter-object). We support every type of parameter, including `object` and `array`. We also support every type description field (for example `format`, `minimum`, `maximum`, etc). Also, at the moment, we support every combination of `style` and `explode` field (excluded styles `matrix` and `label`)
* Body defined in new [RequestBody object](https://github.com/OAI/OpenAPI-Specification/blob/OpenAPI.next/versions/3.0.md#request-body-object). In particular:
  - For `application/json` the validation handler will take schema that you have defined in `schema` object and will validate json bodies with it
  - For `application/x-www-form-urlencoded` and `multipart/form-data` the validation handler will take care of validate every parameters in form attributes. It actually supports only comma separated values for `object` and `arrays`
  - For other parameter types it will check `Content-Type` header
  
Request validation errors will be carried with `RoutingContext` encapsulated in an object called `ValidationHandler`, so you have to attach failure handler to check if something went wrong during validation. Also the `RoutingContext` carry a new object called `RequestParameters` that encapsulate all request parameters deserialized and parsed.
  
Router factory is intended to give you a really simple user interface to use OpenAPI 3 support. Most important features are:
* Async loading of specification and its schema dependencies
* Automatic convert OpenAPI style paths to Vert.x style paths
* Lazy methods: operations (combination of paths and HTTP methods) are mounted in definition order inside specification
* Automatic mount of security validation handlers

Also, it's planned to release a **project skeleton generator** based on API spec.

## Startup your project
We are in a testing stage, so the vertx-web official repo doesn't contain it. To include the modified version of vertx-web replace your vertx-web maven dependency with this one:
```xml
<dependency>
    <groupId>com.github.slinkydeveloper</groupId>
    <artifactId>vertx-web</artifactId>
    <version>89d6254d50</version> <!-- Or <version>designdriven-SNAPSHOT</version> for latest versions -->
</dependency>
```
Then you have to add this maven repository in your `pom.xml`
```xml
<repositories>
  <repository>
      <id>jitpack.io</id>
      <url>https://jitpack.io</url>
  </repository>
</repositories>
```

You can also use it with [gradle](https://jitpack.io/#slinkydeveloper/vertx-web/designdriven-SNAPSHOT)

Now you can start using OpenAPI 3 inside your Vert.x powered app!

First of all you need to load the specification and construct the router factory:
```java
// Load the api spec. This operation is asynchronous
OpenAPI3RouterFactory.createRouterFactoryFromFile(this.vertx, "src/main/resources/petstore.yaml", ar -> {
    if (ar.succeeded()) {
        // Spec loaded with success
        OpenAPI3RouterFactory routerFactory = ar.result();
    } else {
        // Something went wrong during router factory initialization
        Throwable exception = ar.cause();
        logger.error("Ops!", exception);
    }
});
```

## Handlers mounting

Now load your first path. There are two functions to load the handlers:
* `addHandler(HttpMethod method, String path, Handler<RoutingContext> handler, Handler<RoutingContext> failureHandler)`
* `addHandlerByOperationId(String operationId, Handler<RoutingContext> handler, Handler<RoutingContext> failureHandler)`

This two functions wants an handler and a failure handler. You can, of course, **add multiple handlers to same operation**, without overwrite the existing ones.

[IMPORTANT Add operations with operationId | Usage of combination of path and HTTP method is allowed, but it's better to add operations handlers with operationId, for performance reasons and to avoid paths nomenclature errors]

This is an example of `addHandlerByOperationId()`:
```java
// Add an handler with operationId
routerFactory.addHandlerByOperationId("listPets", routingContext -> {
    // Handle listPets operation (GET /pets)
}, routingContext -> {
    // Handle failure
});
```

This is an example of `addHandler`:
```java
// Add an handler with a combination of HttpMethod and path
routerFactory.addHandler(HttpMethod.POST, "/pets", routingContext -> {
    // Handle /pets POST operation
}, routingContext -> {
    // Handle failure
});
```

## Request parameters
Now you can freely use request parameters. To get the `RequestParameters` object:
```java
RequestParameters params = routingContext.get("parsedParameters");
```

The `RequestParameters` object provides all methods to access to query, cookie, header, path, form and entire body parameters. Here are some examples of how to use this object.

Parameter with name `awesomeParameter` with type `integer` in `query`:
```java
RequestParameter awesomeParameter = params.queryParameter("awesomeParameter");
if (awesomeParameter != null) {
    // awesomeParameter parameter exists, but we are not sure that is empty or not (query parameters can be empty with allowEmptyValue: true)
    if (!awesomeParameter.isEmpty()) {
      // Now we are sure that it exists and it's not empty, so we can extract it
      Integer awesome = awesomeParameter.getInteger();
    } else {
      // Parameter exists, but it's empty value
    }
} else {
    // Parameter doesn't exist (it's not required)
}
```

As you can see, every parameter is mapped in respective objects (`integer` in `Integer`, `integer` with `format: int64` in `Long`, `float` in `Float` and so on)

Comma separated array with name `awesomeParameters` with type `integer` in `query`:
```java
RequestParameter awesomeParameters = params.queryParameter("awesomeParameters");
if (awesomeParameters != null && !awesomeParameters.isEmpty()) {
    List<RequestParameter> awesomeList = awesomeParameters.getArray();
    for (RequestParameter awesome : awesomeList) {
      Integer a = awesome.getInteger();
    }
} else {
  // awesomeParameters not found or empty string
}
```

JSON Body:
```java
RequestParameter body = params.body();
if (body != null)
  JsonObject jsonBody = body.getJsonObject();
```

## Security handling
You can mount only one security handler for a combination of schema and scope.

To add a security handler only with a schema name:
```java
routerFactory.addSecurityHandler("security_scheme_name", routingContext -> {
    // Handle security here and then call next()
    routingContext.next();
});
```

To add a security handler with a combination of schema name and scope:
```java
routerFactory.addSecuritySchemaScopeValidator("security_scheme_name", "scope_name", routingContext -> {
    // Handle security here and then call next()
    routingContext.next();
});
```

[NOTE You can define security handlers where you want but define it! | During Router instantiation, if factory finds a path that require a security schema without an assigned handler, It will throw a `RouterFactoryException` ]

## Error handling
Every time you add an handler for an operation you can add a failure handler. To handle a `ValidationException`:
```java
Throwable failure = routingContext.failure();
if (failure instanceof ValidationException)
    // Handle Validation Exception
    routingContext.response().setStatusCode(400).setStatusMessage("ValidationError").end(failure.getMessage());
```

Also the router factory provides two other tools:
* It automatically mounts a 501 `Not Implemented` handler for operations where you haven't mounted any handler
* It can load a default `ValidationException` failure handler (You can enable this feature via `routerFactory.enableValidationFailureHandler(true)`)

## And now use it!
Now you are ready to generate the `Router`!
```java
Router router = routerFactory.getRouter();

// Now you can use your Router instance
HttpServer server = vertx.createHttpServer(new HttpServerOptions().setPort(8080).setHost("localhost"));
server.requestHandler(router::accept).listen();
```

[NOTE Lazy methods! | `getRouter()` generate the `Router` object, so you don't have to care about code definition order]

## And now?
You can find a complete example here: [OpenAPI 3 Vert.x example gists](https://gist.github.com/slinkydeveloper/bdf5929c2506988d78fc08205089409a)

You can access to documentation (WIP) [here](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/asciidoc/java/index.adoc#openapi-3) (for others languages, check out [here](https://github.com/slinkydeveloper/vertx-web/tree/designdriven/vertx-web/src/main/asciidoc)), but you can also check Javadoc inside code. These are the most important ones:
* [DesignDrivenRouterFactory (Base interface of OpenAPI3RouterFactory)](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/java/io/vertx/ext/web/designdriven/DesignDrivenRouterFactory.java)
* [OpenAPI3RouterFactory](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/java/io/vertx/ext/web/designdriven/OpenAPI3RouterFactory.java)
* [RequestParameters (container of all parameters)](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/java/io/vertx/ext/web/RequestParameters.java)
* [RequestParameter (container of a parameter instance)](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/java/io/vertx/ext/web/RequestParameter.java)
* [ValidationException](https://github.com/slinkydeveloper/vertx-web/blob/designdriven/vertx-web/src/main/java/io/vertx/ext/web/validation/ValidationException.java)

Follow [my fork of vertx-web project](https://github.com/slinkydeveloper/vertx-web) to get last updates.

[IMPORTANT We want you! | Please give us your feedback opening an issue [here](https://github.com/slinkydeveloper/vertx-web/issues) ]