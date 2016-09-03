title: OpenID Connect Response Types
date: 2016-09-03 13:40:28
tags:
    - oauth2
    - openid
    - oidc
    - openid connect
category: programming
---
The [OpenID Connect](https://openid.net/connect/) Specification extends OAuth2 in a number of ways, one of which is to define some new Response Types that can be used. Unfortunately it doesn't do the best job in explaining what these all mean. It actually is explained, if you know [where to look](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html), but even then it's not the clearest document in the world.

Part of the complexity is that the specification allows for combinations of response types, but doesn't make this as obvious as it could. Instead it makes it look like the combinations are actually whole new response types in their own right, which isn't quite right.

The actual set of response types are:
* code - The requester would like an Authorization Code to be returned to them
* token - The requester would like an Access Token to be returned to them
* id_token - The requester would like an ID Token to be returned to them
* none - The requester doesn't want any of the above to be returned to them

The "none" response type is a special case in that it can not be combined with any of the others. The other three can be combined in any way that you want, and you will be requesting all of the details for the combination that you specify. For example:
* code token - The requester would like both an Authorization Code and an Access Token to be returned to them
* token id_token - The requester would like both an Access Token and an ID Token to be returned to them
* code token id_token - The requester would like an Authorization Code, an Access Token and an ID Token to be returned to them

The further confusion is in the fact that the specification will allow these to be combined in any order, but you will almost always see them in the same order in the various documents. The requests for "code token id_token", "code id_token token" and "token id_token code" are all actually the same, and should be treated as such.

So why would you want to combine these? The obvious reason is because of the [OpenID Connect Implicit Flow](https://openid.net/specs/openid-connect-core-1_0.html#ImplicitFlowAuth) and [OpenID Connect Hybrid Flow](https://openid.net/specs/openid-connect-core-1_0.html#HybridFlowAuth) requiring it. Implicit allows for the client to specify either "id_token" or "id_token token", and Hybrid allows for the client to specify either "code id_token", "code token", or "code id_token token".

The uses of "id_token token" and "code id_token" are fairly straightforward - they allow for the client to get the ID Token directly and also get an Access Token - either directly or indirectly - to further make API calls to the server. The use of "code token" and "code id_token token" is slightly less obvious - why would you want to get the Access Token AND an Authorization Code, when the only use of an Authorization Code is to swap it for an Access Token? The reason is that an Authorization Code actually does get you more than an Access Token. It also gets you a Refresh Token, which gives you much longer term access to the resources on the server than the Access Token on it's own likely will.
