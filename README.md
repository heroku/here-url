# URInception
[Data URIs](http://en.wikipedia.org/wiki/Data_URI_scheme) over HTTP.

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/heroku/urinception)

## Overview
[Data URIs](http://en.wikipedia.org/wiki/Data_URI_scheme) are great for encoding 
small amounts of data directly in a URI without requiring a server or storage; 
however, they are mostly only supported in web browsers. URInception allows
Data URIs to be used with any HTTP client, such as `curl`.

For example, this is a Data URI of an image of a red dot:

    data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==

Modern web browsers can load this directly, but many clients don't understand the `data` scheme. 
URInception is a simple service that serves Data URIs over HTTP. Here is the same image over HTTP:

    http://example.com/?uri=data%3Aimage%2Fpng%3Bbase64%2CiVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4%2F%2F8%2Fw38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg%3D%3D

## Motivation
This is primarily used for integration testing file downloads with contents that can be dynamically
generated by the tests themselves. Instead of creating static test fixtures, hosting them somewhere, 
and hard coding the URI in tests, this service allows the tests to generate Data URIs on the fly
without having the host the file anywhere. If clients like `curl` supported Data URIs directly, 
this service would not be needed, but this acts an adapter for such clients.

## Usage
URIs for use with this service can be constructed by the client or server. It is more efficient and simple enough to create them client-side, but a server-side URI builder is provided as convenience and for RESTful symmetry. The `urinceptiontest` package is also included that runs URInception locally for use with tests in other Go applications that need to test HTTP callouts without external dependencies or network latency.

### Client-Side

1. Create a Data URI as described in [RFC 2397](https://www.ietf.org/rfc/rfc2397.txt).
2. URL encode the Data URI as described in [RFC 2396](http://www.ietf.org/rfc/rfc2396.txt).
3. Create a URI with encoded Data URI as the value of the `uri` query parameter on any path this service.

See the [syntax](#syntax) below for details.

### Server-Side

POST data to any path on this service and a URI will be returned as a 
[`text/uri-list`](http://tools.ietf.org/html/rfc2483) of one. 
If a `Content-Type` header is include, it will be used as the Data URI's `mediatype`;
otherwise, the `mediatype` will be guessed from the data. 

For example, to create the URI of the red dot above:

    $ curl http://example.com -X POST --data-binary @reddot.png
    http://example.com/?uri=data%3Aimage%2Fpng%3Bbase64%2CiVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4%2F%2F8%2Fw38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg%3D%3D

The `mediatype` will automatically be detected as `image/png`; however, if you want to specify it yourself, set the `Content-Type` header

    $ curl http://example.com -X POST --data-binary '<tag/>' -H 'Content-Type: text/xml'
    http://example.com/?uri=data%3Atext%2Fxml%3Bbase64%2CPHRhZy8%2B

So far these example have not specified a path; however, if one is specified, it will be returned in the URI. This is helpful for clients that expect certain paths or file extensions. The XML example again with a path:

    $ curl http://example.com/example.xml -X POST --data-binary '<tag/>' -H 'Content-Type: text/xml'
    http://example.com/example.xml?uri=data%3Atext%2Fxml%3Bbase64%2CPHRhZy8%2B

### Testing Go with `urinceptiontest` package

Package `urinceptiontest` provides other Go applications
a way to run URInception locally for use in tests.
Importing `urinceptiontest` automatically starts an HTTP
server on a random port on the local machine. Several
methods are provided to create URI fixtures that can be
be passed the system under test which will call the
local server.

For example, if a test that wants to assert `http.Get` works,
`urinceptiontest` can be used to create a URI that will return
a given response body:

    import "github.com/heroku/urinception/urinceptiontest"

    // create the URI fixture
    txt := "hello world"
    uri := urinceptiontest.StringUri(txt)

    // pass the URI to the system under test
    res, _ := http.Get(uri)
    defer res.Body.Close()
    bytes, _ := ioutil.ReadAll(res.Body)

    // assert the result
    obtained := string(bytes)
    expected := txt
    if obtained != expected {
        t.Errorf("Obtained: '%v'; Expected: '%v'", obtained, expected)
    }

### Custom HTTP Statuses

By default, all URIs return an HTTP status of `200 OK`; however, this can be overridden with the `status` parameter.

## Syntax
```
httpuri    := "http" [ "s" ] "://" host *urlchar "?uri=" datauri [ "&status=" httpstatus ]
datauri    := "data:" [ mediatype ] [ ";base64" ] "," data
mediatype  := [ type "/" subtype ] *( ";" parameter )
data       := *urlchar
parameter  := attribute "=" value
httpstatus := int 
```

## Testing

    $ go test

## Deployment

    $ heroku create
    $ git push heroku master
    
... or just:

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/heroku/urinception)
