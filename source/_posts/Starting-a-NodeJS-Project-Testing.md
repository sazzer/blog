title: Starting a NodeJS Project - Testing
date: 2015-08-19 20:32:03
tags:
- nodejs
- javascript
- grunt
- mocha
- testing
category: buildsystems
---
I try to be a bit of a stickler when it comes to testing of my code. As such, I always like to make sure that I have support for Unit Testing in my projects from the very start, knowing full well that if not then the test quality just won't be as good and the project will suffer as a result.

Testing of a project comes in various different flavours, of various different levels of importance. 
* Unit Testing. This is testing every single Unit - Module in NodeJS terms - of a project in complete isolation. Everything that the module depends on should be replaced with Mocks that you have control over, so that you know exactly what is going on.
* Integration Testing. This is testing a group of units all together to make sure that the inter-operation between them is correct. The level that you do this at completely depends on the tests that you are doing. It might be simple testing 2 or 3 modules together, or it might be testing a full stack from API all the way down to a (mock) database. The key difference is that it is not testing one Unit in isolation, but testing the integration of several Units together, but isolated from everything outside of that set of Units
* Verification Testing. This is a less commonly used term, but it's one that I use to mean testing of the entire project as a whole. It's essentially Integration Testing, but where the collection of Units that you are Integration Testing is "all of them". This can be very hard to get right, because ideally you need to test against a real database engine, and using a real browser. Often it's just easier to do this level of testing by hand, but if you can automate it then you really should.

<!-- more -->

For now, I'm going to concentrate only on the first two levels, since they can be covered by the exact same setup and just varies on how you write the tests. If I get around to it I might well write up how I handle the Verification Tests at some point - though that does involve working out a setup that I'm truly happy with first.

I always end up using Mocha for my NodeJS testing. Plugging this in to Grunt is relatively easy, though there are a number of modules that you can use that work better or worse than others. Due to the way that I do {% post_link Starting-a-NodeJS-Project-ES6 my ES6 setup %}, needing to run the application with the NODE_PATH environment variable already set, this really limits the way that the tests can be run. 

The setup that I've found works best is *grunt-mocha-cli*. This is installed by running:
```bash
$ npm install --save-dev mocha grunt-mocha-cli
```

The one tricky part with this is that, because of the name of the module, it doesn't work well with *jit_grunt*. Thankfully, *jit_grunt* was written with this fact in mind and gives a way around it. 

The Grunt configuration that I use for all of this is as follows:
```javascript
require("jit-grunt")(grunt, {
    mochacli: "grunt-mocha-cli"
});
................
mochacli: {
    node: {
        options: {
            reporter: "spec",
            growl: true,
            env: [
                targetNodeMainDir, 
                targetNodeTestDir
            ].join(":"),
            files: [ 
                path.join(targetNodeTestDir, "**", "*-spec.js")
            ]
        }
    }
}
```

This setup will treat every file that has been built into the *target/node/test* directory and has a filename ending with "-spec.js". This means that, if needed, you can write helper modules that are used by the tests, and as such live under *src/node/test* instead of *src/node/main*. All you need to do is ensure that the helper modules don't have filenames that end with "-spec.js".

You'll also notice that we've passed in both the Target Node and Target Test directories as the NODE_PATH environment variable. This allows for us to find all of the modules in the main source area, and the modules in the test area as appropriate.

The only thing left to do is to wire it up. This is done simply by changing the task definition to:
```javascript
grunt.registerTask("test", ["build", "babel:nodeTest", "mochacli:node"]);
```

The end result of this is that whenever you run "grunt test", and by extension "npm test" we will get all of our tests run. 