title: Some notes on supporting CORS
date: 2015-10-24 10:57:17
tags:
  - webdev
  - cors
  - ajax
category: programming
---
If you didn't know, {% link CORS http://www.w3.org/TR/cors/ %} - or Cross-Origin Resource Sharing - is a W3C specification for allowing a page loaded from one domain to access resources on a different domain via XMLHttpRequest calls. This is important for a number of reasons, but the main one for me right now is because it means that you can split your application into multiple smaller services - microservices - that are hosted on different subdomains and still access them all from your Javascript.

<!-- more -->

So how does it work? There's a lot of good details about it already on the web, so I won't go into too many details. Briefly though, when you make an XMLHttpRequest call, the *response* contains some special headers that indicate if it is acceptable for the browser to consume the data.

Yes. The Response. What this means is that the request may go through regardless, but the response might not always make it back to the browser. I've just tested this and can conform that this is exactly what happens.

Now, of course, there is more to it than that. There are two types of Cross-Origin Requests that can be made - Simple Cross-Origin Requests and Cross-Origin Requests with Preflight. 

A Simple request is one that follows a set of rules, which state:
* The HTTP method is one of GET, HEAD or POST
* The custom headers on the request only include Accept, Accept-Language, Content-Language and Content-Type
* If a Content-Type header is included, the value is one of application/x-www-form-urlencoded, multipart/form-data, or text/plain

If all of these rules are met then the browser will make the request, the server will receive and process the request, and the browser may or may not receive the response depending on the presence of CORS headers in the response.

If any of these rules are not met then the browser will instead make a Pre-Flight request first - which is an OPTIONS request to the same URL, and if the Pre-Flight response contains the appropriate CORS headers then the browser will make the actual request.

All seems well and good. We now have a mechanism by which we can write a web application such that the browser makes requests to a selection of servers and it all works correctly.

For more details on how all of this works, the {% link "HTML5 Rocks page" http://www.w3.org/TR/cors/ %} has a lot of good details.
