title: What is GraphQL?
date: 2016-01-24 18:24:12
tags:
- graphql
- api
category: programming
---
You may or may not have already come across GraphQL by now. GraphQL is a new concept coming out of the software people at Facebook, and like their other ideas recently - React and Flux mainly - it looks like it could really change the way we think about our HTTP based API design.

So, what is it exactly? And why is it a big deal?

<!-- more -->

GraphQL is a way of structuring your API such that the API Client is more in control of exactly what information is requested and how the information is returned. It also allows much more structured data to be returned in far fewer requests, because of how it represents the data that is being retrieved.

That all sounds very good, but what does it mean? And how does it compare to what we're already used to?

Let's start with the second question and go from there. It seems that the go-to mechanism for writing HTTP based APIs will be something resembling REST, often with all kinds of extra bits added on as well (HAL, HATEOAS, etc). REST gives you a hierarchical API pattern, in which you specify Resources and allow the client to access these resources in various different ways. Generally speaking, this breaks down to:
* List/Search - GET /resource
* Retrieve - GET /resource/1
* Create - POST /resource
* Update - PUT /resource/1
* Delete - DELETE /resource/1

The hierarchical nature of this allows for some flexibility in the URL patterns to reflect how the data is structured. Let's be a bit more realistic now and use a more real-world example. I'm going for a DVD Catalogue, since it's a bit more unusual than what you normally see. In this, we have the following resources available:
* /users - Access to the user database
* /films - Access to the film database
* /users/&lt;id&gt;/films - Access to the film database for a particular user

So far, so good. Although already we can see that we need to do extra work if we want to return a list of users and the films that each user has. And what happens if we want to see a list of other users that also have a particular film? The obvious way of doing that is:
* /films/&lt;id&gt;/users - Access to the users that all own a particular film

This works, but again it makes it a bit clunky to see details across a number of films in one go. And what about when you want to find users that own films that are owned by a particular user? That's quite a nasty case, but it's perfectly feasible. In order to do that, we would need to:
1. GET /users - find the user in question.
2. GET /users/&lt;id&gt;/films - Find the films that the user has.
3. GET /films/&lt;id&gt;/users - Find the users for each film returned. 

This means that if we do a search in #1 that returns 15 users, and each user has an average of 10 films, in order to find all of the users with shared films we have to:
* Make request #1 once
* Make request #2 15 times - once per user
* Make request #3 150 times - once for each film for each user

That's a total of 166 requests to get this data. Of course, if it's a common use case you could write a custom endpoint to do it all in a single request, but then you are writing server-side code to handle client-side concerns.

Enter GraphQL. When you write a GraphQL API, you don't think of each resource as being independent any more. You need to instead think of all the resources as being an inter-connected graph. You define some entry points into the graph, and then the client can navigate as they wish. This all sounds very good, but what does it look like? The above query - to find all of the users that share films with another set of users - could look like:
```json
query UsersSharingFilms {
  users() {
    id,
    name,
    films() {
      id,
      name,
      ownerBy() {
        id,
        name
      }
    }
  }
}
```

So, that's one query. That means one single request to the server, and it will return:
* All of the matching users, including the ID, Name and Films they own
* For each Film, the ID, Name and the Users that own the film
* For each owner, the ID and Name.

Notice as well how there is a nesting from User -> Film -> User, which is where the Graph part comes in. Because the client dictates the query to be executed, and the server just provides the graph schema to query, you can navigate this as deeply or widely as you want.

Also notice how the query is specifying which fields to return. This means that you only ever retrieve the details that you actually care about, and nothing more. In a RESTful API, you may find that the server returns significantly more information than you care about, just in case you do want it. This then takes longer to produce the response, longer to transmit the response and longer to parse the response. (Generally speaking, the time we're talking about here is negligible, but as the resultset grows in size, this has more impact)

Now, what happens if you want to get only a single user instead? You do this exactly the same:
```json
query GetUser {
    user(id: <id>) {
        id,
        name
    }
}
```

In fact, all Query mechanisms work exactly the same. You use a server-defined query that gives you the entry point into the graph, and you go from there. What's even better, the top level request in the query works exactly the same as all other levels, so you can actually specify multiple queries in a single request:
```json
query GetUsersAndFilms {
    users() {
        id,
        name
    },
    films() {
        id,
        name
    }
}
```

This query will return a response containing to lists, one of all the users and the other of all the films. And again, this is one request and one response.

This all means that you can reproduce everything that REST does with a GET, but with more flexibility and more control. But what about changing data? How are we meant to do anything that causes the data to change?

That's simple. GraphQL defines two different top-level options. Above we've seen Query, but the other option is Mutation. This works identically to Query, but it has the understanding that the data will be mutated as a side-effect of the query. From the client point of view, the only difference is that you use the "mutation" keyword instead of the "query" keyword, and you need to use a defined Mutation instead of a defined Query. Because of this, we have the added benefits that we can specify multiple mutations in a single request, and we can specify the fields that we want to get back in the response to the mutation. For example:
```json
mutation AddFilmsToUser {
    addFilmToUser(user: <userId1>, film: <filmId1>) {
        name,
        dateAdded
    },
    addFilmToUser(user: <userId2>, film: <filmId2>) {
        name,
        dateAdded
    },
    addFilmToUser(user: <userId3>, film: <filmId3>) {
        name,
        dateAdded
    },
}
```

Now, all this sounds fantastic, but how do you know what you can do with the API? One of the remaining problems that REST APIs have that GraphQL helps to fix is working out what you can actually do with the API. GraphQL offers a standard set of query fields that you can use to query the actual GraphQL Schema as opposed to the API Itself. This means that you can as a GraphQL Service to describe itself, and you can then have documentation and API Clients automatically generated based on this schema definition. This is actually quite complicated so I'm not going to describe it here, but there's a fair few details to find already.