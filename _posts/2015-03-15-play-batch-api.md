---
layout: post
title: Batch API for Play
---

While we are waiting for [HTTP/2](http://http2.github.io) to be widely adopted, there is a simple trick that can make
our applications faster - batch API. It allows clients to encode multiple API calls into one HTTP request.
Here are some examples of different batch API implementations:
[Facebook](https://developers.facebook.com/docs/graph-api/making-multiple-requests),
[Google](https://cloud.google.com/storage/docs/json_api/v1/how-tos/batch),
[Dropbox](https://blogs.dropbox.com/tech/2014/01/retrieving-thumbnails/)

The idea is that instead of making multiple HTTP requests to get different pieces of data the client just makes one.
This one request contains information about the different real endpoints requested. And the response from the server
contains the individual responses combined. This approach can make the client faster because it reduces the overhead of
multiple HTTP requests (TCP connection time, SSL handshake time, sequential execution due to a limit on the number of
concurrent connection to the same host).

Batching of API requests is very easy to implement on top of the [Play Framework](https://www.playframework.com). The
key feature that enables us to do it is that application code has access to the global router. That makes it possible to
receive a batch request, extract individual calls encoded into it, create fake HTTP requests for them and ask the router
to process these fake requests.

For this post I chose to implement a Facebook API inspired request batching protocol. Let's say there are multiple
endpoints returning JSON responses. The goal is to create a batch endpoint that takes a list of individual endpoints in
the query parameters and returns JSON containing responses from all of them. For example there are endpoints `/foo` and
`/bar`. A call to `/batch?f=/foo&b=/bar` should return `{ "f": <foo resonse>, "b": <bar response> }`. In the batch call
query parameter names are used to give names to the sections of the resulting JSON document.

Let's start from the top level batch controller action. It defines the high level algorithm: extract batched calls,
fetch them individually, combine into the response.

```scala
def batchGet(): Action[AnyContent] = Action.async { implicit request =>
  val resultFutures = request.queryString.map { case (name, values) =>
    fetch(values.head).map(name -> _)
  }
  Future.sequence(resultFutures).map(combineResults)
}
```

The next function is the most important part - fetching an individual request locally. It creates a fake request using
the given URL, routes to the corresponding action and invokes the action to produce a response.

```scala
private def fetch(path: String)(implicit request: RequestHeader): Future[Result] = {
  val fetchRequest = request.copy(path = path, uri = path)
  val handler = Play.current.global.onRouteRequest(fetchRequest)
  handler.map {
    case action: EssentialAction => action(fetchRequest).run
    case x => Future.failed(new IllegalArgumentException(s"Unexpected handler type"))
  } getOrElse {
    Future.failed(new IllegalArgumentException(s"No handler for path '$path'"))
  }
}
```

The last part is combining individual responses into the final JSON document. Responses are assumed to be valid JSON
documents, so no validation is done.

```scala
private def combineResults(results: Iterable[(String, Result)]): Result = {

  def bytesEnumerator(s: String) = Enumerator(s.getBytes)
  def openBrace = bytesEnumerator("{")
  def closeBrace = bytesEnumerator("}")
  def comma = bytesEnumerator(",")
  def namedBlock(name: String) = bytesEnumerator(s""""$name":""")
  def isLast(index: Int) = index == results.size - 1

  val body = results.zipWithIndex.foldLeft(openBrace) { case (acc, ((name, result), index)) =>
    acc
      .andThen(namedBlock(name))
      .andThen(result.body)
      .andThen(
        if (isLast(index)) {
          closeBrace
        } else {
          comma
        }
      )
  }
  Result(ResponseHeader(OK), body)
}
```

Some improvements to the code above would be:

 * support for HTTP methods other than `GET`
 * better error handling (what if one of the batched requests fails, but the rest of them succeed)
 * better batching protocol (e.g. send not only the body, but also the headers and the status code for individual responses)

Full source code of the batch controller together with a sample application available
[here](https://github.com/dmitriy-yefremov/play-batch-api). Please check out and let me know what you think.