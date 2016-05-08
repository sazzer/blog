title: I disagree with HATEOAS
date: 2016-05-08 11:22:01
tags:
- api
- REST
- HATEOAS
category: programming
---
HATEOAS, for anyone who's not come across it yet, is an additional constraint added on top of normal REST services whereby the only way that clients ever interact with the server is through Hypermedia that the server provides. The server will provide a single starting point, and every single interaction with the server after that is made only through URLs that the server has previously provided to the client. The advantage here is that the server can change independently of the client, and as long as the client only ever follows the URLs provided to it then everything will continue to work correctly.

That all sounds fantastic on the face of it. Completely decoupled evolution of server and client sounds almost idyllic.

Of course, in reality this doesn't actually work. There are a number of problems with the idea, ranging from the very simple - clients cut corners and just hard-code URLs to specific resources instead of following links - to the very complicated, where changes to the server need to be made that just aren't backwards compatible.

<!-- more -->

## Hard coding links
Everyones done it. You're writing a client to an API, and you know that the URL to get a users profile is "/api/user/:id". So why would you jump through the hoops of requesting the base page to be told this URL every time? It's just extra overhead for no benefit, so you decide to cut out the middle man and just hard-code the URL. And, of course, it works. Why wouldn't it? It works great up until the server decides that user profiles should now live on "/api/user/:id/profile", because they want to put some other user specific details as sub-resources of "/api/user/:id".

In the HATEOAS world, this is trivial. You change the URLs, update the base page, and everything just works. But this client isn't in the HATEOAS world, and so doesn't know that things have changed. So the servers trivial change has just unwittingly broken this client. And because in the HATEOAS world this isn't a big deal, there was no big announcement about URL patterns changing, so the client is left floundering until someone realises what's happened.

## Unsupported requirements
The HATEOAS world depends on the server providing URLs for everything that the client could ever want to do. But what if there are things that the client wants to do that the server didn't think of?

Surely though, if the server didn't think of it then it's not possible to do it. Except that in some cases that's not quite true. The main example of this I'm thinking is pagination. It's quite common for servers to provide URLs for the first, last, next and previous pages. It's less common for servers to provide URLs for every possible page, and it's (almost) unheard of for servers to provide URLs for every combination of page offset, page size, sort field, and so on. What I have seen done is the use of [URL Templates](https://tools.ietf.org/html/rfc6570) to provide a mechanism by which the client can generate the URLs it wants from the parts that the server provides. But this then gets away from what I consider to be pure HATEOAS and is back to the client generating URLs instead of using the ones provided.

## Breaking changes
It happens. Hopefully it's rare, but it happens. Something changes in the server that isn't a trivial change of URL, but is a more fundamental change in how things work. The only solution to this is that the client gets updated to cope with it. And this is then back to the world where client and server are coupled together.

## Understanding the data
This is the real killer. This is the bit that really makes a mockery of HATEOAS. No matter how well structured the server is, or how well behaved the client is. If you do everything perfectly and follow all of the rules exactly, you still end up in the situation where the client needs to understand what the data means to do something with it. Retrieving a users profile as a JSON blob is all well and good, but if you don't understand the distinction between the fields then it's worthless. This means that at the end of the day, the client is going to be hand crafted to understand the data that the server provides, so why not hand craft it to understand the API by which the data is being provided? It just makes life easier for both server and client in the long run.

## So what's the alternative?
The alternative is very simple, but a lot of people shy away from it. Documentation. Write good documentation for your API, for how to use it, for what it does. Provide this documentation in both human and machine readable format - using something like [Swagger](http://swagger.io/specification/) or [RAML](http://raml.org/), for example. Provide a mechanism by which developers can easily trial the API - using something like [Swagger UI](http://swagger.io/swagger-ui/) for example. Provide early feedback if you are going to be making large changes.

In short, talk to the developers who are writing clients to your server. You'd be amazed at what you can achieve by doing that.
