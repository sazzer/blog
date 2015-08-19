title: Starting a NodeJS Project - ES6
date: 2015-08-19 06:35:43
tags:
- nodejs
- javascript
- grunt
- es6
category: programming
---
In the previous post, I set up a new NPM project to work with, and added in Grunt as a task runner so that we can do complex tasks if we want to. Next is setting it up to build ES6 code into ES5 via Babel, so that we get to use the newer features - Classes, Arrow Functions, Destructuring, Let/Const, and so on - whilst running on a runtime that doesn't yet support them - because no runtime yet supports ES6 fully.

[Babel](https://babeljs.io/) is a Transpiler that we can use to automatically convert ES6 code into ES5 code for exactly this purpose. It literally takes existing ES6 source code as input, and converts it into ES5 source code as output - in the same way that a compiler would covert source code into machine code.

If we so desired, we could use the exact same setup for working with [CoffeeScript](http://coffeescript.org/), [Dart](https://www.dartlang.org/), [TypeScript](http://www.typescriptlang.org/), or any of a [growing set of languages](https://github.com/jashkenas/coffeescript/wiki/List-of-languages-that-compile-to-JS) that we can convert into Javascript.

<!-- more -->

I always set up my projects with a consistent filesystem structure, so that I know what I'm talking about. This is largely stolen from Maven, simply because I know it and it works for me. As such, I work with the following structure:
```
| package.json
| GruntFile.js
+ src
  + node // This is where all of the Server-side code goes
    | main
    | test
  + js // This is where all of the Client-side code goes
    | main
    | test
+ target
  + node // This is where all of the Server-side output goes
    | main
    | test
```

Notably in comparison to Maven, I have the language and scope folders swapped around. This simply keeps things closer together.

Now, hwo to actually make this work? Everything is done in Grunt, which makes life easier. I'm only going to concentrate on the src/node stack for now - the src/js stack is different because it builds a single output file using Webpack instead of building lots of files using Babel. I'll cover how that works in a later post.

Firstly, because we have some explicit areas that we work in, I always set up some variables at the top of my Gruntfile to handle this:
```javascript
var path = require("path");

var srcDir = "src";
var srcNodeDir = path.join(srcDir, "node");
var srcNodeMainDir = path.join(srcNodeDir, "main");
var srcNodeTestDir = path.join(srcNodeDir, "test");

var targetDir = "target";
var targetNodeDir = path.join(targetDir, "node");
var targetNodeMainDir = path.join(targetNodeDir, "main");
var targetNodeTestDir = path.join(targetNodeDir, "test");
```

Next, because we're creating a target area to work, it's only polite to tidy it up as well. This is done with the grunt-contrib-clean module, configured as such:
```json
clean: {
    build: {
        src: targetDir
    }
}
```

Literally, this means that if you run "grunt clean" then it will delete the "target" directory. That's it.

Next comes actual compilation of the source code from ES6 to ES5. This is done using Babel, and specifically with the grunt-babel module. Note that there are certain ES6 constructs that you may want to use that will also require the babel-runtime module being installed, and this as an actual dependency as opposed to a development dependency.

The configuration of Babel is a bit more involved, but not by much:
```json
babel: {
    options: {
        sourceMap: true,
        optional: "runtime"
    },
    nodeMain: {
        files: [{
            expand: true,
            cwd: srcNodeMainDir,
            src: ["**/*.js"],
            dest: targetNodeMainDir
        }]
    },
    test: {
        files: [{
            expand: true,
            cwd: srcNodeTestDir,
            src: ["**/*.js"],
            dest: targetNodeTestDir
        }]
    }
},
```

Having done that, you can now run "grunt babel", and you should see all of your ES6 files that you've written in src/node/main turn up as ES5 code in target/node/main, and likewise for src/node/test into target/node/test.

Unfortunately, it doesn't quite work. Almost everything is fine, but module loading is a bit tricky. This is because of the module loader needing to know where to find the modules. The way that I get around this is a bit of a hack, but works, and that is to set the *NODE_PATH* environment variable. The way that I do this has the disadvantage that I can only easily run the code from Grunt, but that's a small price to pay I find.

I do this by using the grunt-env and grunt-execute modules, and configuring as follows:
```json
env: {
    node: {
        NODE_PATH: targetNodeMainDir
    }
},
execute: {
    node: {
        src: path.join(targetNodeMainDir, "main.js")
    }
}
```

I now need to change the task registrations to execute all of the plugins in the correct order. This now looks like:
```javascript
grunt.registerTask("build", ["babel:node"]);
grunt.registerTask("run", ["build", "env:node", "execute:node"]);
```

As of this point, executing "npm install" will correctly build all of your ES6 code into ES5 code, and executing "npm start" will do this and then execute the code you wrote in "src/node/main/main.js". 