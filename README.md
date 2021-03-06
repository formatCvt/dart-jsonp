Dart JSONP
==========

JSONP handler for Dartlang. Allows you to make individual and multiple requests, as well as providing some support for automatically converting the responses to Dart classes.

Usage
------

This library handles requesting a URL and handling the subsequent callback. To achieve this a URL must be provided which can be updated with a callback method name which is determined by the library. There are two ways to do this.

The easiest way is to provide a complete URL which includes a callback query parameter. The query parameter value which can be replaced by the callback name has the placeholder value ?. The library is not smart about this, and will replace _all_ query parameters which have that value.

    jsonp.fetch( uri: "http://example.com/rest/object/1?callback=?" );

If you need more control over the creation of the URL, then you can take the more advanced approach. This means providing a function which takes a String (the callback method name) and returns a string (the request URL including the callback parameter). As you define the function, this approach allows total control over the constructed URL.

    jsonp.fetch( uriGenerator: (String callback) =>
        "http://example.com/rest/object/1?callback=$callback" );

### Single Requests

The _fetch_ method can be used to make a single request.

When you use _fetch_ to request a URL a future will be returned. This future will complete with the response from the JSONP request.

    import "dart:async";
    import "package:jsonp/jsonp.dart" as jsonp;

    // In this example the returned json data would be:
    // { "data": "some text" }
    Future<dynamic> result = jsonp.fetch(
        uri: "http://example.com/rest/object/1?callback=?"
      );

    result.then((var proxy) {
      print(proxy['data']);
    });

#### Type Conversion

The proxy objects can be time consuming to handle, as you don't get things like autocompletion for proxy fields. It's best to wrap the data in an object and then return that from your method:

    class ExampleData {
      var data;

      fromProxy(var proxy) {
        this.data = proxy['data'];
      }
    }

    jsonp.fetch(
        uri: "http://example.com/rest/object/1?callback=?"
      )
      .then((var proxy) => new ExampleData.fromProxy(proxy))
      .then((ExampleData object) => print(object.data));

### Many Requests

The _fetchMany_ and _disposeMany_ methods can be used to handle many requests.

The _fetchMany_ method will return a named Stream which receives individual results. This Stream is identified by the _name_ parameter in the request, sharing single Streams across multiple requests. This means you only need to set up result handling code once, as all results will be handled by the same Stream.

By default the Stream provides the response from the JSONP request.

    Stream<dynamic> object_stream = jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/1?callback=?"
      );

    // The uri is optional when making a fetchMany request
    // as you may just want to configure the Stream
    Stream<dynamic> user_stream = jsonp.fetchMany("user");

    object_stream.forEach(
        (var data) => print("Received object!")
      );
    user_stream.forEach(
        (var data) => print("Received user!")
      );

    // You just need to refer to the stream by name to make further requests
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/2?callback=?"
      );
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/3?callback=?"
      );
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/4?callback=?"
      );
    jsonp.fetchMany(
        "user", uri: "http://example.com/rest/user/1?callback=?"
      );

    // Release the stream if you don't need it any more
    jsonp.disposeMany("object");

#### Type Conversion

Converting the type of your stream is also straight forward. You can convert the values on demand by mapping the stream, which returns a typed stream:

    Stream<ExampleData> example_stream = jsonp.fetchMany(
        "object"
      )
      .map((var proxy) => new ExampleData.fromProxy(proxy));

    example_stream.forEach(
        (ExampleData object) => print("Received ${object.data}")
      );

    // No need for the type when you don't use the returned stream
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/1?callback=?"
      );
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/2?callback=?"
      );
    jsonp.fetchMany(
        "object", uri: "http://example.com/rest/object/3?callback=?"
      );

Examples
--------

An example of using the library can be found in the examples folder. This
example makes some calls to working and broken jsonp endpoints. The results are
printed in the console.

General Note
------------

This library is not required because you can make JSONP requests very easily with pure Dart. This library does reduce the work required to make JSONP requests.

Using javascript has become significantly easier since this was originally written. To perform a JSONP request without using this library is as easy as:

    import 'dart:html';
    import 'dart:js';

    context['callbackMethod'] = (JsObject response) {
      // Do something with the response object. The JsObject can be treated like a dictionary.
    }

    document.body.children.add(
        new ScriptElement()
          ..src = 'http://some.service/?callback=callbackMethod'
      );

To get a Future for this do the following:

    import 'dart:async';
    import 'dart:html';
    import 'dart:js';

    Completer callbackCompleter = new Completer();

    context['callbackMethod'] = (response) {
      callbackCompleter.complete(response);

      // if you just want to use this once, then you can clear the callback method:
      context['callbackMethod'] = null;
    }

    document.body.children.add(
        new ScriptElement()
          ..src = 'http://some.service/?callback=callbackMethod'
      );
    callbackCompleter.future.then((JsObject response) {
      // Do something with the response object. The JsObject can be treated like a dictionary.
    });

You can read more about Dart Javascript interoperability [here](https://www.dartlang.org/articles/js-dart-interop/).
