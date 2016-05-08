title: Designing a GraphQL API
date: 2016-05-07 10:09:59
tags:
- graphql
- relay
- api
category: programming
---
I've just gone through some of the reasons that I don't like the Relay API requirements for GraphQL, so lets have a go at designing an API that I do like.

<!-- more -->
## Result Sets
Let's start with the bits of Relay that I do like, and use those. This means most of the Connection parts of the spec. Specifically the separation of Edges and Connections, and the idea of the Page Info object. I'm in two minds if I want to go for Cursor based pagination, or a simpler form using page numbers or offsets, so I'm going to fall back on [Postel's Law](https://en.wikipedia.org/wiki/Robustness_principle) for that. Yes, it adds a little bit of complexity to the server layer, but really not that much and the benefits to the client layer are worth it.

Firstly then, the PageInfo object. The PageInfo object is based on the one from Relay, but with some extra fields included.
```
type PageInfo {
  hasNextPage: Boolean! # False if this is the last page
  hasPreviousPage: Boolean! # False if this is the first page
  pageOffset: Int! # The 0-based offset of the first record returned in the entire resultset
  count: Int! # The total number of records in the entire resultset
}
```

These extra fields let you power a paginator on your results, because between these and the actual results array you know how many results total, and which results you have.

Next comes an Edge. This is not a simple object, because it's generic over the actual result type, but the definition will look something like:
```
type Edge<T> {
  resource: T!
  cursor: ID!
  offset: Int!
}
```
Note that we have the offset of each individual edge included if the client wishes to make use of it, and we also have a Cursor defined if the client wishes to use Cursor based pagination. This comes back to Postel's Law, being liberal in which clients we support with our API. You'll note that there is an Offset here as well as in the PageInfo object. The Offset of the first edge returned should be the same value as the offset in the PageInfo object. This means that you can choose to request every single offset individually if you desire, or you can request one single offset for the entire page and calculate the others yourself. It's trivial for the server to do this, and means the client can be developed either way as desired.

This then leads on to a Connection. Again, this is not a simple object because it's generic over the type of Edge to support, but it will look something like:
```
type Connection<T> {
  edges: [Edge<T>!]!
  pageInfo: PageInfo!
}
```

All of this gives us an example resultset of:
```json
{
  "posts": {
    "edges": [
      {
        "resource": {
          "title": "First Post"
        },
        "cursor": "b2Zmc2V0OjAK",
        "offset": 0
      }, {
        "resource": {
          "title": "Second Post"
        },
        "cursor": "b2Zmc2V0OjEK",
        "offset": 1
      }
    ],
    "pageInfo": {
      "hasNextPage": true,
      "hasPreviousPage": false,
      "pageOffset": 0,
      "count": 17
    }
  }
}
```

## Pagination
Ok then. We can get paginated results out of the API now, but how do we ask for them? We need to have support for asking for a particular page of results. Generally speaking, there are three ways that pagination can work:
* Cursor + Count + Direction
* Page Number + Count
* Initial Record Offset + Count

If we consider that Count + Direction can be replaced simply by a Count that can be negative, and if we consider that there *might* - there probably isn't, but let's be generic enough to support it anyway - be a reason to support the number of records before a given page or record offset, then we end up in the situation where we need to specify some representation of the start of the page, and the number of records to retrieve.

Unfortunately, GraphQL doesn't currently allow you to use Unions or Interfaces on Input Types, which means that if we want to support a situation where we support one of a number of different ways of specifying the start of the page we need to do it with custom server-side validation. As such, the schema could look like:
```
type Page {
  cursor: ID
  page: Int
  offset: Int
  count: Int
}
```

The other way of doing it would be to provide a value and an enum that tells the server what the value means, but then the value needs to be provided as a string even if it's got to be numeric, and it means that you've got to pass an extra value across every time regardless. I also made the schema above allow the count to be optional as well - if not provided it would just use a sensible default instead. Another sensible behaviour would be that if none of cursor, page or offset were provided then the default is the first page of results, and if multiple are provided but happen to refer to the same start point then this is accepted as well. It is only an error when it is ambiguous what is meant.

## Mutations
Next, lets think about Mutations. Mutations in GraphQL are much closer to an RPC concept than in REST. REST pretty much mandates that the only mutations you can do are Create, Update and Delete of resources. GraphQL lets you specify any mutation that you want. However, odds are you want to support the standard Create, Update and Delete operations on most of your resources anyway. For top level resources, this is easy. For nested resources this is a bit more of a challenge, because you can't nest mutations. As such, you need some way of specifying the nested resource that you want to work with. The proposal here is to define a type to identify the resource in question, which can include a number of different resource identifiers to work down to the nested resource. I would also recommend including inside of this type an optimistic lock value for the resource being edited. You probably don't need this for all of the resources in the graph, but that depends on your data model.

As an example of this, consider a blog system. You have posts, and you have comments. Comments are not a top-level concept - they make no sense to exist on their own. They are only ever a child of a post. As such, in order to edit a post itself you might use:
```
mutation editPost {
  editPost(id: {post: 1, version: 7}, patch: {title: 'The new Title') {
    id,
    version,
    title,
    body
  }
}
```

And to edit a comment you might use:
```
mutation editComment {
  editCommentOnPost(id: {post: 1, comment: 5, version: 2}, patch: {text: 'This is the updated comment'}) {
    id,
    version,
    text
  }
}
```

The first of these updates the post with an ID of 1, providing the version of the post as being 7. The second of these updates the comment with the ID of 5, that is itself a child of the post with an ID of 1, providing the version of the comment as being 2.

You might have noticed that I've included the changes as a patch, rather than providing the entire resource. I think this makes a lot of sense in a system that is deliberately designed to minimise the amount of traffic between client and server. If I need to change only one field, I shouldn't be required to provide every field to the server. I did consider using the JSON Patch standard for this, but you lose type safety and the schema guarantees that make GraphQL so powerful if you do that. You would also need to provide every field as a string and do appropriate type conversions on the server depending on the field you are writing to, which is just a bit ugly. By defining a custom patch format for each resource the schema will do a lot of work for you, and the server updates become really simple.

## Error Handling
Eventually when working with an API, something is going to go wrong. It's generally more likely that mutations will fail than queries, but technically anything could. As such, the concepts here are best suited to mutations but can be made to work with queries if needed.

GraphQL already has support for a top-level errors object returned as part of the response payload. This is wonderful for errors with the incoming request itself - e.g. if it is malformed or invalid in some way, or if the authorization details are invalid. It doesn't work well when the errors are with a specific part of the request though - e.g. an optimistic lock failure on a mutation. [Konstantin Tarkus has a proposal](https://medium.com/@tarkus/validation-and-user-errors-in-graphql-mutations-39ca79cd00bf#.fljypia9y) on a way to improve this, but I personally don't think it goes quite far enough. The basic idea is that the field would have a response that includes the actual response data and a list of errors.

I'm going to propose something very similar, but leveraging the GraphQL type system some more to make it a bit cleaner to work with. My idea here is that the field will have a response that is a union type, consisting of either the actual success response or an error response. The error response would then be a list of error objects, which are richer than just a string error message - because there are extra details that are important as well. This would look something like:
```
type Errors {
  globalErrors: [GlobalError!]!
  fieldErrors: [FieldErrors!]!
}

type GlobalError {
  code: ID!
  message: String!
}

type FieldError {
  code: ID!
  field: ID!
  message: String!
}

union EditCommentResponse = Comment | Errors
```

This does start to mean that we need to use fragment operators to actually call the mutation, but that's fine. We need to do something to know if we got a success or a failure anyway, and doing it in the GraphQL call is really no different to doing it in the actual client code. Our edit comment mutation from above now becomes:
```
mutation editComment {
  editCommentOnPost(id: {post: 1, comment: 5, version: 2}, patch: {text: 'This is the updated comment'}) {
    ... on Comment {
      id,
      version,
      text
    }
    ... on Errors {
      globalErrors {
        code
      },
      fieldErrors {
        code,
        field
      }
    }
  }
}
```

I specified the error codes as an ID, but they could equally well be an enumeration if you want to define in the schema what the possible errors are. That makes it easier to work out whats going on, but at the cost of inflating the schema definition with all of these error enumerations. I also specified the Field Errors as a list of objects that contain the field name. I could equally make it an object with a list of errors, and be more typesafe about the error codes that are supported - it makes no sense to state that a boolean field was too long, for example - but again at the cost of the schema size. This is all a balancing act at the end of the day.

I also did not request the error message in the response. This is because the error code should be enough for the client to understand, and the error message is more aimed at developers who are still getting to grips with the system. The reasoning behind this is i18n - error codes can very easily be converted into localised error messages for the user, but if the client depends on the error message coming from the server then the server needs to have support for every single language that every single possible client could support, which is difficult if you are aiming to support third party clients.

## Conclusion
That's plenty to be getting on with. It gives us a good basis for an API, but without being too restrictive in what you must and mustn't do.

One of the key parts of all of this is the fact that any API definition is specific to the application in question. There has been a lot of work recently in defining API standards that are generic enough to support any API - using things like HATEOAS, for example - and I just feel that this is doomed from the offset. There is far too much API specific knowledge that needs to be baked in to the client for a general purpose API to actually be worth the costs involved in using it. You're better off just developing a more specific API that fits your exact purposes, documenting it well, and being done with it. Clients are going to be custom anyway, so the use of a general purpose API is lost.

As such, if you've read through all of this and decide that you like some bits and not others, just use those bits. Do whatever makes the most sense at the time and actually produce something that works. Don't fall into my usual trap of trying to engineer the perfect API and ending up without actually producing anything.
