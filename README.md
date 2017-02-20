# Home Office API Standards

This document captures **The Home Office's view of API best practices and standards**. We aim to incorporate as many of them as possible into our work.

This document provides a mix of:

* **High level design guidance** that individual APIs interpret to meet their needs.
* **Low level web practices** that most modern HTTP APIs use.

## Authentication
How consumers authenticate with an API is important for many APIs. Whilst we've not settled on a standard the following approaches have been tried.

### Mutual TLS
Mutual TLS means that connecting clients need to have a client certificate before using the service. Typically the system terminating the mutual TLS will add headers to identify the client. The could allow your application to use these for example for granular permissions.

##Pros and Cons of Mutual TLS

| Pros  | Cons
|---|
|Can use standard nginx or similar for this purpose, so your application doesn't need to worry about auth   | Client certificates will expire, and hence need to recreate these for all clients periodically, which introduces a lot of overhead and potential for errors
| | Have to maintain a CA for signing client certs


### OAuth2 with keycloak
OAuth2 is very commonly used for user authentication, but less frequently for API authentication. This is reflected by fewer client auth libraries available than you may expect. However the logic is trivial to implement.

We can implement OAuth2 authentication trivially using keycloak-proxy in front of an API, authenticating to a centralised keycloak server. This is especially beneficial as keycloak-proxy combined with a centralised keycloak give many capabilities for free such as role and user management, brute force prevention.

##Pros and Cons of OAuth2 with keycloak

| Pros | Cons
|---|
|Can implement trivially using keycloak-proxy and keycloak |Client auth is slightly more complicated|
|Get a lot of capabilities for free with keycloak | |
|Possibility for consumers to rotate their own credentials, reducing administrative overhead | |
|Creds are only sent to the system on first auth or when the authentication token expires | | 


### Basic HTTP Auth with username and password
Basic HTTP auth requires a username and password to be sent with every request.

## Pros and Cons of Basic HTTP Auth

| Pros | Cons
|---|
|Very simple to set up initially |Doing anything beyond basic auth is difficult
| |User accounts have to be managed by your application, introducing complexity

## Acceptance tests, documentation, and creating a mock of your API
These topics all go hand in hand - frequently one document can be the basis for tests, documentation, and for generating a mock. Here we will cover the common approaches we favour, and when each is appropriate.

### Recommended approach - Rest-assured tests with Spring RestDocs (you don't need Spring!), and Wiremock

*Why we like this approach*

* It enables one set of tests to generate docs and mock, helping to make sure everything ties together
* The mock generated is flexible - it can have multiple canned responses per endpoint depending on the request params, whereas many other automatically generated mocks only allow one canned response per endpoint
* The docs generated are flexible - A person writes the overall documentation, meaning it can be structured however makes most sense for your API, with as much detail as you like. The generated snippets then give real examples that you can embed into sections. This allows you to put much more context around the documentation than you may otherwise have been able to do
* Allows testing and documentation of a json schema for your API

### Example application using this approach
Peruse the [example docs](https://ukhomeoffice.github.io/lev-api-docs/) and [mock](https://github.com/UKHomeOffice/lev-api-docs/tree/master/mock) for the LEV project. The [source code](https://gitlab.digital.homeoffice.gov.uk/lev/lev-api-scala) is available on request. If you have any questions ask in the developers slack channel and one of us will get back to you.

[Rest assured](https://github.com/rest-assured/rest-assured[Rest-assured]) is a library designed for making testing APIs straightforward. We recommend it here as it works out the box with Spring RestDocs.

[Spring Restdocs](http://projects.spring.io/spring-restdocs/) allows you to generate documentation snippets for each of your tests. It records the request and the response and creates snippets for each of these. You can then embed these snippets into a master document that describes your whole API. Documentation snippets are recorded in asciidoctor format, which is like a more advanced version of markdown. The main readme asciidoc file can then embed snippets from other files

[Wiremock](http://wiremock.org/) is a mock API that allows you to record a series of requests and responses (by proxying requests through wiremock). When running your rest-assured tests you should proxy requests through a wiremock instance in record mode. It will then generate a number of files that represent the requests and responses. These files can then be used to run wiremock as a mock, where it will then respond with the recorded responses.

### Alternative approach - [Swagger](http://swagger.io/)
Swagger is a very common way to document APIs. It does a reasonable job of getting some API documentation available. The shortcomings are that it can become out of sync (it relies heavily on annotations which can get out of date), the mock you can generate from a swagger spec is very inflexible, and the docs are also very inflexible. The major benefits are that it is a common format that you can put together very quickly.

### Alternative approach - [API Blueprint](https://apiblueprint.org/)
API blueprint allows you to have one markdown file which is a specification of your API. From this file you can generate html docs, run tests, and generate a mock version of your API. The shortcomings are that the mock you can generate from an API Blueprint is very inflexible, and the docs are also very inflexible. The major benefits are that it is a common format that you can put together very quickly.

## API Design

### Using one's own API

The best way to understand and address the weaknesses in an API's design and implementation is to use it in a production system. Whenever feasible, design an API in parallel with an accompanying integration of that API.

### Point of contact

Have an clear way for clients to report issues and ask questions about the API. In addition, publish an email address for direct, non-public inquiries.

### Notifications of updates

Have a simple way for clients to follow changes to the API, e.g. a mailing list, or a [dedicated developer blog](https://developer.github.com/changes/) with an RSS feed.

### API Endpoints

An "endpoint" is a combination of two things:

* The verb (e.g. `GET` or `POST`)
* The URL path (e.g. `/articles`)

Information can be passed to an endpoint in either of two ways:

* The URL query string (e.g. `?year#2014`)
* HTTP headers (e.g. `X-Api-Key: my-key`)

Generally speaking:

* **Avoid single-endpoint APIs.** Don't jam multiple operations into the same endpoint with the same HTTP verb.
* **Prioritize simplicity.** It should be easy to guess what an endpoint does by looking at the URL and HTTP verb, without needing to see a query string.
* Endpoint URLs should advertise resources, and **avoid verbs**.

Some examples of these principles in action:

* [OpenFDA example query](https://open.fda.gov/api/reference/#example-query)
* [Sunlight Congress API methods](https://sunlightlabs.github.io/congress/#using-the-api)

### Just use JSON

[JSON](https://en.wikipedia.org/wiki/JSON) should be the default for web APIs; it reduces complexity for both the API provider and consumer.

General JSON guidelines:

* Responses should be **a JSON object** (not an array). Using an array to return results limits the ability to include metadata about results, and limits the API's ability to add additional top-level keys in the future.
* **Don't use unpredictable keys**. Parsing a JSON response where keys are unpredictable (e.g. derived from data) is difficult, and adds friction for clients.
* **Use consistent case for keys**. Whether you use `under_score` or `CamelCase` for your API keys, make sure you are consistent.

### use ISO 8601 for dates

[Use ISO 8601](https://xkcd.com/1179/), in UTC.

For dates use the format YYYY-MM-DD e.g. `2013-02-27`. For datetimes, use the form YYYY-MM-DDTHH:MM:SSZ e.g. `2013-02-27T10:00:00Z`.

### Other best practice

* a GET request to the root endpoint should return all the endpoint categories that the API supports
* use null for blank values instead of omitting it
* lists of resources can return summary responses (a subset of all attributes for the resource, omitting attributes that may be expensive to return)
* use parameters for filtering or pagination


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

Handle all errors (including otherwise uncaught exceptions) and return a data structure in the same format as the rest of the API.

For example, a JSON API might provide the following when an uncaught exception occurs:

```json
{
  "message": "Description of the error.",
  "exception": "[detailed stacktrace]"
}
```

HTTP responses with error details should use a `4XX` status code to indicate a client-side failure (such as invalid authorization, or an invalid parameter), and a `5XX` status code to indicate server-side failure (such as an uncaught exception).


### Pagination

If pagination is required to navigate datasets, use the method that makes the most sense for the API's data.

#### Parameters

Common patterns:

* `page` and `per_page`. Intuitive for many use cases. Links to "page 2" may not always contain the same data.
* `offset` and `limit`. This standard comes from the SQL database world, and is a good option when you need stable permalinks to result sets.
* `since` and `limit`. Get everything "since" some ID or timestamp. Useful when it's a priority to let clients efficiently stay "in sync" with data. Generally requires result set order to be very stable.

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

### Always use HTTPS

Any new API should use and require [HTTPS encryption](https://en.wikipedia.org/wiki/HTTP_Secure) (using TLS/SSL). HTTPS provides:

* **Security**. The contents of the request are encrypted across the Internet.
* **Authenticity**. A stronger guarantee that a client is communicating with the real API.
* **Privacy**. Enhanced privacy for apps and users using the API. HTTP headers and query string parameters (among other things) will be encrypted.
* **Compatibility**. Broader client-side compatibility. For CORS requests to the API to work on HTTPS websites -- to not be blocked as mixed content -- those requests must be over HTTPS.

HTTPS should be configured using modern best practices, including ciphers that support [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy), and [HTTP Strict Transport Security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security). 


### Use UTF-8

Just [use UTF-8](http://utf8everywhere.org).

An API should tell clients to expect UTF-8 by including a charset notation in the `Content-Type` header for responses, e.g.:

```
Content-Type: application/json; charset#utf-8
```

### CORS

For clients to be able to use an API from inside web browsers, the API must [enable CORS](http://enable-cors.org).

For the simplest and most common use case, where the entire API should be accessible from inside the browser, enabling CORS is as simple as including this HTTP header in all responses:

```
Access-Control-Allow-Origin: *
```

It's supported by [every modern browser](http://enable-cors.org/client.html), and will Just Work in many JavaScript clients, like [jQuery](https://jquery.com).

For more advanced configuration, see the [W3C spec](http://www.w3.org/TR/cors/) or [Mozilla's guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS).

**Avoid JSONP**

JSONP is [not secure or performant](https://gist.github.com/tmcw/6244497). If IE8 or IE9 must be supported, use Microsoft's [XDomainRequest](http://blogs.msdn.com/b/ieinternals/archive/2010/05/13/xdomainrequest-restrictions-limitations-and-workarounds.aspx?Redirected#true) object instead of JSONP. There are [libraries](https://github.com/mapbox/corslite) to help with this.


## Further reading
[Microsoft's API guidelines on GitHub](https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md)

[API design guide Gitbook](https://geemus.gitbooks.io/http-api-design/content/en/)

[GitHub's API guide for developers](https://developer.github.com/v3/)

[GDS API guide](https://www.gov.uk/service-manual/technology/application-programming-interfaces-apis)
