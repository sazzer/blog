title: Unit Testing ES6 Modules - Mockery
date: 2015-08-20 21:45:10
tags:
- javascript
- es6
- testing
- mocks
- mockery
category: programming
---
For a long time, one of the big problems I've had with NodeJS is that of Unit Testing. Specifically with the fact that, by definition, a Unit Test is testing a single unit in complete isolation of the rest of the world. In Node terms, a Unit would be a single Module. 

<!-- more -->

In Java, this is solved by the fact that you will often use an [IoC container](https://en.wikipedia.org/wiki/Inversion_of_control) to wire your application together. That way, every object that you are testing is constructed by passing in the objects that it depends on. That way, in a Unit Test you can simply pass in a [Mock](https://en.wikipedia.org/wiki/Mock_object) version of the object that is expected, and so you have absolute control of the way the Mock object reacts. This means that you are able to test your object in terms of the API that it's dependencies expose, instead of needing to worry about implementation details of those other objects. This also means that you can test things that are otherwise hard to test (Ever wanted to test that you're correctly handling an IOException that realistically will never happen?)

In Node, the presence of the module system means that you write your code the other way around. Instead of wiring up everything from the outside, you tend to write it so that modules pull their dependecies in. There are IoC containers for Node, but they're often clunky and I've yet to find one that works well with ES6 modules. (If anyone knows of one, let me know!). The fact that modules pull in their dependencies means that it's very difficult to provide a mock version of them instead.

Enter [Mockery](https://github.com/mfncooper/mockery). Mockery is a very clever little module that hooks into the Node module system and makes it so that a call to require a module actually does something different. This is evil, but awesome, and instantly means that even though modules pull in their dependencies we can intercept that and give them a version that we control.

Then we come to ES6. If you are using the ES6 module system then you'll know that the new "import" statements can only go at the top level. This means that your object is imported *before* you can intercept it with Mockery, and the whole thing falls apart.

So what do we do? Well, as it happens, there's no reason that we *can't* use the old "require" statement in ES6 code, and this statement isn't restricted to being at the top level like the "import" statement is. This means that we can simply require the module under test after we've set Mockery up, and we're back in business.

This example is written in Mocha. There's no reason why it can't apply to anything else though.

```javascript
// first.js
import {name} from "second";

export function hello() {
    return "Hello, " + name();
}

// second.js
export function name() {
    return "Graham";
}


// first-spec.js
import assert from "assert";
import mockery from "mockery";

describe("first", () => {
    let secondMock;
    let first;

    before(() => {
        mockery.enable();
        mockery.registerAllowable("first");

        secondMock = {};
        mockery.registerMock("second", secondMock);

        first = require("first");
    });

    after(() => {
        mockery.disable();
    });

    it("should return the right name", () => {
        secondMock.name = () => {
            return "Fred";
        };

        assert("Hello, Fred" === first.hello());
    });
});
```

So, what are we doing here? We have three files, imaginatively named "first", "second" and "first-spec". The "first" module explicitly depends on the "second" module, and the "first" module is the one we're testing. By using Mockery, we override what the "second" module looks like before we load the "first" module. This means that instead of returning a name of "Graham" like it's meant to, it instead returns a name of "Fred". 

There is a slight oddity in there. Because we're enabling Mockery and then doing a "require" call for the module "first", which we haven't mocked out, we need to tell Mockery that this is allowed otherwise it will log a warning when it runs. It will still work, but it's a bit noisy, and what's worse it obscures when we really did miss mocking out a module for real.

In more complicated code you could then use something like [Sinon](http://sinonjs.org/) to implement your Mocks behaviour, but for the sake of this simple example that was a bit too much.