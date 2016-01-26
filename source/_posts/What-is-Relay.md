title: What is Relay?
date: 2016-01-24 22:22:28
tags:
- graphql
- relay
- api
category: programming
---
Having just had a very brief overview of GraphQL, the next thing that might be of interest is Relay. Relay in this sense means one of two things. Either it means the [Relay Javascript Framework](https://facebook.github.io/relay/) that is used in your frontend layer to communicate with a GraphQL Server, or - and this is what's more interesting to myself right now - it means the set of concepts that your GraphQL Server needs to adhere to in order for it to be Relay Compatible.

<!-- more -->

Ultimately, Relay defines three specific concepts that you need to follow in your schema definition. However, one of these concepts is quite sizable and I tend to think of it more as four concepts that have some overlap. These four concepts are:
* Entity Loading
* Connections between Entities
* Pagination
* Mutations

### Entity Loading
This is covered in the Relay Specification under [Object Identification](https://facebook.github.io/relay/graphql/objectidentification.htm). 

In brief, Relay requires that all of your identifiers for all of your resources are globally unique, and a single query field that can be used to load any resource of any time from this globally unique identifier. 

When we talk about a Globally Unique Identifier, we do really mean it. It must not be possible for the server to be ambiguous between the kind of resource to load when it is given just the ID. How you achieve this is entirely up to you, but my preference here is to have the ID be a Base64 encoded representation of the Resource Name and Database ID. (The idea of Base64 encoding the ID is so that it's obvious to the client that it's just an Opaque ID). So, for example:
* The first User might have an ID of "dXNlcjoxCg==" (user:1)
* The first Film might have an ID of "ZmlsbToxCg==" (film:1)

Notice here that even though the database IDs are both the same, the ID that is used over the API is totally different, and when the server receives this ID it has enough information to know whether it represents a User or a Film.

The other part of this concept is that there is a single uniform way of loading any resource knowing only it's ID. This is the Query field *node*. This works by returning an instance of the Node interface - all resources in Relay must implement the Node interface, though all that means is that they have a unique ID - and uses some more complicated parts of the GraphQL specification to be able to extract the desired fields from it - namely the use of Inline Fragments to extract fields from a specific subtype of Node. This means that we can write a query as follow:
```json
query GetFilm {
    node(id: 'ZmlsbToxCg==') {
        id,
        ... on Film {
            name
        }
    }
}
```

What this query says is "Get the resource with an ID of 'ZmlsbToxCg==', returning the ID and - iff the resource is a Film - the Name of the film". We know that it's a Film, because somehow we got hold of the ID and when we got hold if it we will have known it's a Film from the context. This does mean that you need to keep track of what each ID means on the client side or else you'll get very confused, but that shouldn't be too difficult.

### Connections between Entities
This is covered in the Relay Specification under [Cursor Connections](https://facebook.github.io/relay/graphql/connections.htm).

Relay requires a particular pattern when one resource links to a collection of other resources. This is represented by the concept of a Connection and Edges, where the outermost resource contains a Connection, the Connection contains a list of Edges, and each Edge contains a Node. For example, the User resource may look like:
```json
{
    id: 'dXNlcjoxCg==',
    name: 'Graham',
    filmConnection: {
        edges: [
            {
                node: {
                    id: 'ZmlsbToxCg==',
                    name: 'The Shawshank Redemption'
                }
            },
            {
                node: {
                    id: 'ZmlsbToyTCg==',
                    name: 'The Godfather'
                }
            }
        ]
    }
}
```

The benefit out of this is that the Connection and Edge entries can contain other data that relates to the related edges, and then the Node is the actual resource that was wanted. Specifically, the mandated extra information that can be included on Connection and Edge are related to Pagination, but you can put anything you like there. Specifically it can make sense to put the total number of records on the Connection.

Note that each Node here has an ID on it. This must be usable in a call to the *node()* query - described above - to re-request the same resource again, possibly with different fields this time.

### Pagination
This is covered in the Relay Specification under [Cursor Connections](https://facebook.github.io/relay/graphql/connections.htm).

Pagination builds on top of the Connections and Edges concepts by adding some mandatory fields to the response, and some mandatory attributes to the queries that allow you to make use of them.

On the Connection, we add a field called *pageInfo* which contains some details about the current page of records that was returned. This must contain fields *hasNextPage* and *hasPreviousPage*, but can contain other fields as well if that makes sense to your pagination model. On the Edge, we add a field called *cursor* which allows us to refer to this exact point in the Connection. This means that the above can then become:
```json
{
    id: 'dXNlcjoxCg==',
    name: 'Graham',
    filmConnection: {
        pageInfo: {
            hasNextPage: true,
            hasPreviousPage: false
        },
        edges: [
            {
                cursor: 'MQo=',
                node: {
                    id: 'ZmlsbToxCg==',
                    name: 'The Shawshank Redemption'
                }
            },
            {
                cursor: 'Mgo=',
                node: {
                    id: 'ZmlsbToyTCg==',
                    name: 'The Godfather'
                }
            }
        ]
    }
}
```

The values returned in the *cursor* field are used for requesting more results. The second part of the Pagination concept is that there are certain required query attributes that you need to include to request a page of results. There are four additional attributes that are required to be supported, though only in certain combinations:
* first - The number of records to return reading **forward** from the *after* cursor, or the start of the resultset if not specified.
* after - The cursor to start reading forward from, if *first* was also specified.
* last - The number of records to return reading **backwards** from the *before* cursor, or the end of the resultset if not specified.
* before - The cursor to start rreading backwards from, if *last* was also specified.

It is technically possible to specify both *first* and *last*, but it's not obvious what should happen if you do. Note that none of these attributes are required, and if none of them are set then the specification states that the entire resultset should be returned. It may make more sense to instead return a default subset - e..g the first 10 records - so that the payload and server processing is not overwhelming.

For example, requesting the above user with the next 2 films would look like:
```json
query UserFilms {
    node(id: "dXNlcjoxCg==") {
        ... on User {
            id,
            name,
            filmConnection(first: 2, after: "ZmlsbToyTCg==") {
                ........
            }
        }
    }
}
```

### Mutations
This is covered in the Relay Specification under [Input Object Mutations](https://facebook.github.io/relay/graphql/mutations.htm).

The only thing that the Relay Specification mandates is that each mutation must accept only a single argument, named *input*, and that this input must support a field called *clientMutationId*. The response from the mutation must return the exact same value in an output field also called *clientMutationId*. All of the rest of the inputs to the mutation are just included as fields on the *input* argument.

### Putting it all together
That seems like a lot that needs to be done. It's not quite as bad as it seems though, since you only *need* to do these if you are wanting to support Relay as a client layer, and you may decide that you want to implement them anyway. In particular, the Connections and Edges are a useful way of grouping together resources in a graph without introducing differences in the resources when they are present at different parts of the graph.