title: I don't like Relay
date: 2016-05-06 18:48:49
tags:
- graphql
- relay
- api
category: programming
---
I've started playing with GraphQL again recently, having been looking into various other specifications for HTTP based APIs, and I've come to a conclusion. I really like GraphQL, but I don't like Relay.

Now, don't get me wrong, there are bits of Relay that I think are really good ideas. But there are also bits that I just think are either a waste of time, or worse actually make the API harder to use.

<!-- more -->

I understand that a lot of the ideas behind Relay are well founded, and they are used to make it so that a single Client side library can be used with any form of Server side API that plays by the rules. And that's fantastic. Except that it's incredibly unlikely that you're ever going to be writing any app that's got any level of complexity and *not* need some level of API understanding built into your client.

Let's start with the bits of Relay that I'm not keen on. It's not all of it by any means but there are certainly enough to ensure that any GraphQL APIs that I implement will not be properly Relay compliant - Sorry!

The simplest is the concept of "clientMutationId". This literally has no point. GraphQL lets you, on the client side, provide an alias for any query or mutation field that you are executing. You can therefore use this functionality to provide a client-side ID for any of your mutations and it will automatically work. There is no need to burden the API itself with knowledge of this, which actually is extended to the fact that any mutations need to copy the incoming value to the return value in order for the client to work - something that is automatically handled by all GraphQL libraries when you use field aliasing.

Next is forcing the mutations to only ever take a single argument that is an InputObject, inside of which are all of the actual parameters you want for your mutation. I get that this is here to make writing the client easier, because all of a sudden your client layer is only ever passing across one value, but it's also relatively pointless and only aims to make understanding the API slightly harder. Admittedly, most of the time I'd probably actually follow this pattern, but there are going to be times when I want multiple parameters and I don't want to be forced to wrap them in another object just to make the clients happy. (One example of this might be a single upsert mutation. I can have an argument representing the data to create or update, and an optional second argument representing the ID of the record to update - left out if I'm doing a create). There is also the fact here that, unless I'm mistaken, arguments can have default values whereas fields can't. I can imagine that the requirement to only have one argument means that you need to work out the default values manually in your handler code, instead of having the GraphQL schema do it for you.

Finally, the thing that I like the least about Relay. The "node" query field. This really does nothing more than add complication to the API. And quite a lot of it at that. The idea behind this query field is that you can resolve any arbitrary resource across the entire API by a single call to this. What you need to implement to support this is:
* An ID generation mechanism that encapsulates the ID and Type of a resource
* An Interface that every single resource needs to implement
* Some mechanism of actually resolving any arbitrary resource at the top level - which goes against the "Graph" part of "GraphQL"

And the benefits of this are that you can load the resource knowing only the ID. Except that you also need to know what type of resource you are loading, because you need to specify a fragment specifier for the fields that you want back. So actually, the only thing we gain from this complexity on the server is additional complexity on the client. Essentially it means that a query looks like:
```json
{
  findUser: node(id: 'dXNlcjo0Cg==') {
    ... on User {
      name
    }
  }
}
```

instead of looking like:
```json
{
  findUserById(id: 4) {
    name
  }
}
```

So, what *do* I like from the Relay spec? Pretty much the only things that are left - namely the Cursor Connections topic. The idea behind the Edges and Connections is quite nice I feel, separating out the resultset from the actual results in it. The way that Cursors work, and are used for pagination is also quite nice if you plan on doing Cursor based pagination - but the fact that Relay pretty much enforces Cursor based pagination is less nice. (Cursor based pagination is great if you are only ever going to be scrolling through. It's useless if you want to be able to jump to arbitrary records/pages since it gives no easy facility to do this). The standardised PageInfo object works well as well, meaning that there is a common knowledge about how you can manage expectations here.

And so, based on all of that, next time I'm going to be putting forward a proposal for a GraphQL based API that I feel works much better from both client and server points of view.
