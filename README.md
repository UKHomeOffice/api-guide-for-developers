# Home Office API Standards

This document captures The Home Office's view of API best practices and standards. 

## Contents
- [Core Principles](#core-principles)
- [Key Practices](#key-practices)
- [REST Best Practices](#rest)
- [Error Handling](#error-handling)
- [Pagination](#pagination)
- [Versioning](#versioning)
- [Authentication](#authentication)
- [Documentation, Acceptance tests, and creating a mock of your API](#documentation-acceptance-tests-and-creating-a-mock-of-your-api)
- [Further Reading](#further-reading)

## Important Note!! We want your feedback!
Before you begin to read this guide, please bear in mind that technology moves forward at a pace, and our understandings
and opinions change even faster. Where you read something that doesn't make sense for your service or as general guidance
we appreciate debate and discussion on it - raise an issue, get some developers together, or email the Centre of Excellence
at <CentreOfExcellenceCentral@digital.homeoffice.gov.uk> 

## Core principles

1. **Understand the user needs** and try to adopt a consumer-first approach. Ensure you know who your consumers are and 
talk to them. It's good to have a mailing list for them and to create a forum where the services can be discussed. 
Contracts / schemas / ICDs are all very useul but cannot replace the human conversations. Don't surprise your users 
by suddenly introducing breaking changes or non-standard responses.
1. **Hide implementation details and understand your [bounded context](https://martinfowler.com/bliki/BoundedContext.html)**. 
Hide your database structure! Do not have multiple services reading or (worse) writing to the same database. This breaks 
encapsulation and cohesion, increases coupling and makes it difficult to change schemas and data
1. **Decentralise** services so that a team owns and operates it (they build, own and support it). Be wary of 
[ESB](https://en.wikipedia.org/wiki/Enterprise_service_bus)s which can force you to use a particular, centralised 
architectural style . Keep messaging middleware dumb and ignorant of the domain. Be aware of the added complexity of 
using [API Gateways](http://microservices.io/patterns/apigateway.html) for much more than 
[API key](http://stackoverflow.com/questions/1453073/what-is-an-api-key) management. Be aware of the drawbacks of 
orchestration / "God services" / [BPM](https://en.wikipedia..org/wiki/Business_process_management) systems which tell 
other services what to do (they can become a point of centralisation and contention). Choreography via 
[event driven](https://en.wikipedia.org/wiki/Event-driven_programming) and 
[pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) patterns helps to facilitate looser coupling.
1. **Deploy independently** so that small changes to your service can be released separately from other applications. 
This requires a high degree of automated provisioning and testing (so that it's easy to create new, small services and 
release frequently). Minimise end-to-end tests, which can become test cycle bottlenecks, and instead focus on whether 
you break any of your consumers by using 
[consumer-driven contracts](http://techblog.newsweaver.com/why-should-you-use-consumer-driven-contracts-for-microservices-integration-tests/): 
API consumers write explicit expectations as tests which are made available to and tested by the service itself.  
1. **Isolate and understand inevitable failure** to improve resiliency. The last thing you want is a 'distributed single 
point of failure' where a failure in one microservice brings down an entire system; this is more difficult to deal with 
than a simple monolith failure. Avoid 'cascading failures' by failing fast and explicitly so that dependent services know 
ASAP (and don't wait longer and longer for a response). Likewise, as a consumer, don't wait too long for a response and
 deal appropriately with requests that take too long. Understand the business implications of failure so you can discuss
  and agree mitigation strategies (preferably in advance). See also 
  [stability patterns](http://www.javaworld.com/article/2824163/application-performance/stability-patterns-applied-in-a-restful-architecture.html?page=2) 
  such as bulkhead and circuit breaker.
1. **Make your service highly observable**. Record requests, response times and error codes and make sure it's easy to 
access these metrics. Ensure your services are 
[monitorable via healthcheck endpoints](https://github.com/UKHomeOffice/technical-service-requirements/blob/master/docs/monitoring_metrics.md); 
monitoring of dependencies (e.g. downstream connections) is also useful. Use a 
[Correlation ID](https://blog.logentries.com/2016/12/the-value-of-correlation-ids/) to make it easier to track requests 
across multiple microservices. 
[Semantic monitoring](https://medium.com/production-ready/implementing-semantic-monitoring-534f16e18247#.v058tpyqd) 
using end-to-end tests can be useful. Also consider outputting business metrics (e.g. number of applications) to the 
same place.
1. **Follow standards** and be consistent with versioning, pagination etc. The 
[PayPal API Style Guide](https://github.com/paypal/api-standards/blob/master/api-style-guide.md) on GitHub is a good starting point.
1. **Document your service** - several approaches including 
[OpenAPI and Spring REST Docs](#documentation-acceptance-tests-and-creating-a-mock-of-your-api) are outlined below
1. **[KISS](https://en.wikipedia.org/wiki/KISS_principle)**. Textual payloads are easier to read even if they are less 
efficient - so always favour them. Return the minimum number of fields required (it's easier to add more fields later 
than to remove them) but balance this with having multiple, small requests. Avoid versioning unless it's really needed 
(and then, use URL-based versioning e.g. v2/ then v3/ to version if you have control over your consumers). Avoid client 
libraries (the idea is that they're DRY but logic can leak into these, and new versions require new client libraries, 
so make it just about the connection if you do use these)
1. Use tech agnostic APIs (avoid [RMI](https://en.wikipedia.org/wiki/Java_remote_method_invocation)) and prefer **[REST](https://en.wikipedia.org/wiki/Representational_state_transfer)** as the default choice. However, don't use REST at the expense of usability eg for mobile clients. You could consider using [REST expansion](http://venkat.io/posts/expanding-your-rest-api) or even [GraphQL](http://graphql.org/) (see [GitHub's implementation](https://developer.github.com/early-access/graphql/)) for clients with high latency or poor network reliability.

## Key practices
### Always use HTTPS

Any new API should use and require [HTTPS encryption](https://en.wikipedia.org/wiki/HTTP_Secure) (using TLS/SSL). 
HTTPS provides:

* **Security**. The contents of the request are encrypted across the Internet.
* **Authenticity**. A stronger guarantee that a client is communicating with the real API.
* **Privacy**. Enhanced privacy for apps and users using the API. HTTP headers and query string parameters (among other
 things) will be encrypted.
* **Compatibility**. Broader client-side compatibility. For CORS requests to the API to work on HTTPS websites -- to not
 be blocked as mixed content -- those requests must be over HTTPS.

HTTPS should be configured using modern best practices, including ciphers that support 
[forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy), and 
[HTTP Strict Transport Security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security). 



### Prefer JSON

Prefer [JSON](https://en.wikipedia.org/wiki/JSON) as the default for your APIs. It reduces complexity for both the API 
provider and consumer.

General JSON guidelines:

* **Use a schema**, preferably [JSON Schema](http://json-schema.org/). This will also mean you know they keys in advance 
(i.e. they are not dynamic, derviced from data) and that you're consistent in your case
* Responses should be **a JSON object**, not an array. Using an array to return results limits the ability to include 
metadata about results.
* **Keep JSON minified in all responses**. Extra whitespace adds needless response size to requests, and many clients 
for human consumption will automatically "prettify" JSON output.

### use ISO 8601 for dates

[Use ISO 8601](https://xkcd.com/1179/), in UTC.

For dates use the format YYYY-MM-DD e.g. `2013-02-27`. For datetimes, use the form YYYY-MM-DDTHH:MM:SSZ e.g. `2013-02-27T10:00:00Z`.

### Use UTF-8

Just [use UTF-8](http://utf8everywhere.org).

An API should tell clients to expect UTF-8 by including a charset notation in the `Content-Type` header for responses, e.g.:

```
Content-Type: application/json; charset#utf-8
```

### CORS

For clients to be able to use an API from inside web browsers, the API must [enable CORS](http://enable-cors.org).

For the simplest and most common use case, where the entire API should be accessible from inside the browser, enabling 
CORS is as simple as including this HTTP header in all responses:

```
Access-Control-Allow-Origin: *
```

It's supported by [every modern browser](http://enable-cors.org/client.html), and will Just Work in many JavaScript 
clients, like [jQuery](https://jquery.com).

For more advanced configuration, see the [W3C spec](http://www.w3.org/TR/cors/) or 
[Mozilla's guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS).

**Avoid JSONP**

JSONP is [not secure or performant](https://gist.github.com/tmcw/6244497).

## REST

HTTP requests are stateless and requests should be independent; they may occur in any order, so do not attempt to 
retain transient state information between requests. Each request should be an atomic operation: a finite state machine 
where a request transitions a resource from one non-transient state to another.

Avoid designing interfaces that mirrors the internal structure of the data. Instead expose business entities and the 
operations that an application can perform on these entities.

Avoid “chatty” APIs (a large number of small resources), but balance this against the overhead of fetching excessive 
data that might not be frequently required.

Avoid requiring resource URIs more complex than collection/item/collection


### API Endpoints

An API endpoint should be an easy-to-read, self-explanatory URL that represents a single resource. Do not shoehorn 
multiple resources or operations into a single endpoint.

Operations must use the proper [HTTP methods](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html) (verbs) and [Idempotency](https://www.infoq.com/news/2013/04/idempotent) must be respected: 

Method  | Description                                                                                                                |Idempotent?
------- | -------------------------------------------------------------------------------------------------------------------------- | -------------
GET     | Return the resource | Yes
POST    | Create a new resource based on the data provided in the body, or submit a command                                                        | No
PUT     | Replace a resource (i.e. updates it) or create a named resource, based on the data provided in the body                                                               | Yes
DELETE  | Delete a resource                                                                                                           | Yes
HEAD    | Return metadata of a resource for a GET response                                                                            | Yes
PATCH   | Apply a partial update to a resource                                                                                        | No


### POST
POST operations must support the Location response header to specify the location of any created resource that was not 
explicitly named, via the Location header.

Example: the service below allows creation of hosted servers, which will be named by the service:

```http
POST http://api.contoso.com/account1/servers
```

Example response (201) with Location header:

```http
201 Created
Location: http://api.contoso.com/account1/servers/server321
```

Extra information can be passed to an endpoint either via a query string (e.g. `?year#2014`) or in an HTTP header (e.g. 
`X-Api-Key: my-key`)


### PUT
When an application sends an HTTP PUT request to update a resource, it specifies the URI of the resource and provides 
the data to be modified in the body of the request message. It specficies the Content-Type in the header.

If the modification is successful, it should ideally respond with an HTTP 204 status code, indicating that the process 
has been successfully handled, but that the response body contains no further information. The Location header in the 
response contains the URI of the newly updated resource:

```HTTP/1.1 204 No Content
...
Location: http://adventure-works.com/orders/1
...
Date: Fri, 22 Aug 2014 09:18:37 GMT
```
   


### Other best practice
* The request can include an Accept header which specifies the preferred format that the client would like to receive 
and the web service should attempt to honor this format if at all possible
* a GET request to the root endpoint should return all the endpoint categories that the API supports
* use null for blank values instead of omitting it
* lists of resources can return summary responses (a subset of all attributes for the resource, omitting attributes that 
may be expensive to return)
* use parameters for filtering or pagination

### Communication
* the best way to understand and address the weaknesses in an API's design and implementation is to use it in a 
production system. Whenever feasible, design an API in parallel with an accompanying integration of that API.
* have an clear way for clients to report issues and ask questions about the API. In addition, publish an email address 
for direct, non-public inquiries.
* have a simple way for clients to follow changes to the API, e.g. a mailing list, or a 
[dedicated developer blog](https://developer.github.com/changes/) with an RSS feed.



## Error handling

### Client errors

There are three possible types of client errors on API calls that receive request bodies:

#### Sending invalid JSON should result in a 400 Bad Request response.

HTTP/1.1 400 Bad Request
Content-Length: 35

```json
{"message":"Problems parsing JSON"}
```

#### Sending the wrong type of JSON values should result in a 400 Bad Request response.

HTTP/1.1 400 Bad Request
Content-Length: 40

```json
{"message":"Body should be a JSON object"}
```

#### Sending invalid fields should result in a 422 Unprocessable Entity response.

HTTP/1.1 422 Unprocessable Entity
Content-Length: 149

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "resource": "Issue",
      "field": "title",
      "code": "missing_field"
    }
  ]
}
```

Handle all errors (including otherwise uncaught exceptions) and return a data structure in the same format as the rest 
of the API.

For example, a JSON API might provide the following when an uncaught exception occurs:

```json
{
  "message": "Description of the error.",
  "exception": "[detailed stacktrace]"
}
```

HTTP responses with error details should use a `4XX` status code to indicate a client-side failure (such as invalid 
authorization, or an invalid parameter), and a `5XX` status code to indicate server-side failure (such as an uncaught 
exception).


## Pagination

If pagination is required to navigate datasets, use the method that makes the most sense for the API's data.

#### Parameters

Common patterns:

* `page` and `per_page`. Intuitive for many use cases. Links to "page 2" may not always contain the same data.
* `offset` and `limit`. This standard comes from the SQL database world, and is a good option when you need stable 
permalinks to result sets.
* `since` and `limit`. Get everything "since" some ID or timestamp. Useful when it's a priority to let clients efficiently 
stay "in sync" with data. Generally requires result set order to be very stable.

#### Metadata

Include enough metadata so that clients can calculate how much data there is, and how and whether to fetch the next set of results.

Example of how that might be implemented:

```json
{
  "results": [ ... actual results ... ],
  "pagination": {
    "count": 2340,
    "page": 4,
    "per_page": 20
  }
}
```

## Versioning

It is usually best to avoid versioning by enabling the continuous evolution of a schema. If adding new features to an 
API requires a new version, this makes using the API more difficult and fragile. Prefer to add capabilities via new 
types and new fields on those types; avoid breaking changes and serve a versionless API

## Authentication
How consumers authenticate with an API is important for many APIs. Whilst we've not settled on a standard the following 
approaches have been tried.

### Mutual TLS
Mutual TLS means that connecting clients need to have a client certificate before using the service. 
Typically the system terminating the mutual TLS will add headers to identify the client. The could allow your 
application to use these for example for granular permissions.

#### Pros and Cons of Mutual TLS

| Pros  | Cons
|---|
|Can use standard nginx or similar for this purpose, so your application doesn't need to worry about auth   | Client certificates will expire, and hence need to recreate these for all clients periodically, which introduces a lot of overhead and potential for errors
| | Have to maintain a CA for signing client certs


### OAuth2 with keycloak
OAuth2 is very commonly used for user authentication, but less frequently for API authentication. This is reflected by fewer client auth libraries available than you may expect. However the logic is trivial to implement.

We can implement OAuth2 authentication trivially using keycloak-proxy in front of an API, authenticating to a centralised keycloak server. This is especially beneficial as keycloak-proxy combined with a centralised keycloak give many capabilities for free such as role and user management, brute force prevention.

#### Pros and Cons of OAuth2 with keycloak

| Pros | Cons
|---|
|Can implement trivially using keycloak-proxy and keycloak |Client auth is slightly more complicated|
|Get a lot of capabilities for free with keycloak | |
|Possibility for consumers to rotate their own credentials, reducing administrative overhead | |
|Creds are only sent to the system on first auth or when the authentication token expires | | 


### Basic HTTP Auth with username and password
Basic HTTP auth requires a username and password to be sent with every request.

#### Pros and Cons of Basic HTTP Auth

| Pros | Cons
|---|
|Very simple to set up initially |Doing anything beyond basic auth is difficult
| |User accounts have to be managed by your application, introducing complexity

## Documentation, Acceptance tests, and creating a mock of your API
These topics all go hand in hand - frequently one document can be the basis for tests, documentation, and for generating 
a mock. Here we will cover the common approaches we favour, and when each is appropriate.

### Recommended approaches

#### 1. [Open API/Swagger](https://github.com/OAI/OpenAPI-Specification)
OpenAPI is an open standard for documenting APIs, and is one of the most common. It does a decent job of getting 
some API documentation available with relatively little effort required on your part. Because it is an open standard
there are a plethora of tools that can work with OpenAPI specs to produce mocks, pretty docs, tests, etc. It is also
emerging as the cross-government default choice.

The shortcomings are that:
- Some implementations can become out of sync if they rely heavily on annotations which can get out of date
- the mock you can generate from an OpenAPI spec is somewhat inflexible, as are the docs

However both of these shortcomings are likely to be addressed as the spec matures and the tools improve.

#### 2. Rest-assured tests with Spring RestDocs (you don't need Spring!), and Wiremock

**Why we like this approach**

* It enables one set of tests to generate docs and mock, helping to make sure everything ties together
* The mock generated is flexible - it can have multiple canned responses per endpoint depending on the request params, 
whereas many other automatically generated mocks only allow one canned response per endpoint
* The docs generated are flexible - A person writes the overall documentation, meaning it can be structured however 
makes most sense for your API, with as much detail as you like. The generated snippets then give real examples that you 
can embed into sections. This allows you to put much more context around the documentation than you may otherwise have 
been able to do
* Allows testing and documentation of a json schema for your API

**Example application using this approach**
Peruse the [example docs](https://ukhomeoffice.github.io/lev-api-docs/) and 
[mock](https://github.com/UKHomeOffice/lev-api-docs/tree/master/mock) for the LEV project. The 
[source code](https://gitlab.digital.homeoffice.gov.uk/lev/lev-api-scala) is available on request. If you have any 
questions ask in the developers slack channel and one of us will get back to you.

[Rest assured](https://github.com/rest-assured/rest-assured[Rest-assured]) is a library designed for making testing 
APIs straightforward. We recommend it here as it works out the box with Spring RestDocs.

[Spring Restdocs](http://projects.spring.io/spring-restdocs/) allows you to generate documentation snippets for each 
of your tests. It records the request and the response and creates snippets for each of these. You can then embed these 
snippets into a master document that describes your whole API. Documentation snippets are recorded in asciidoctor format,
 which is like a more advanced version of markdown. The main readme asciidoc file can then embed snippets from other files

[Wiremock](http://wiremock.org/) is a mock API that allows you to record a series of requests and responses (by proxying 
requests through wiremock). When running your rest-assured tests you should proxy requests through a wiremock instance 
in record mode. It will then generate a number of files that represent the requests and responses. These files can then 
be used to run wiremock as a mock, where it will then respond with the recorded responses.

### Alternative approach - [API Blueprint](https://apiblueprint.org/)
API blueprint is another alternative, but because it is less standard than OpenAPI we would typically recommend that instead.
API blueprint allows you to have one markdown file which is a specification of your API. From this file you can generate 
html docs, run tests, and generate a mock version of your API. The shortcomings are that the mock you can generate from 
an API Blueprint is very inflexible, and the docs are also very inflexible. The major benefits are that it is a common 
format that you can put together very quickly.

### Alternative approach - [RAML](https://raml.org/)
RAML is somewhat similar to OpenAPI, but following a different standard. There haven't been any uses in the Home Office
yet so we don't recommend it's use, but some other departments use it as their standard choice.

## Further reading

[HTTP 1.1 RFC](https://tools.ietf.org/html/rfc7231)

[HTTP 1.2 RFC](https://tools.ietf.org/html/rfc7540)

[Roy Fielding's original REST chapter](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)

[Microsoft API Best Practice guide](https://docs.microsoft.com/en-us/azure/best-practices-api-design)

[Microsoft's API guidelines on GitHub](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md)

[API design guide Gitbook](https://geemus.gitbooks.io/http-api-design/content/en/)

[GitHub's API guide for developers](https://developer.github.com/v3/)

[GDS API guide](https://www.gov.uk/service-manual/technology/application-programming-interfaces-apis)
