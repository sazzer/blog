title: "Javascript modules - ES5 vs ES6"
date: 2015-05-12 21:44:29
tags:
    - nodejs
    - javascript
    - es5
    - es6
category: programming
---

The latest version of the ECMAScript Language - [ES6](http://wiki.ecmascript.org/doku.php?id=harmony:specification_drafts) - introduces a lot of new features. One of the most interesting of these is the introduction of a module system that is built into the language. The way this works is, unfortunately, very different to how any of the pre-existing ES5 module systems work.

Wait, what? Most of the time when people discuss the new ES6 module system it is talked about as being one of the brand new features of ES6, and not an existing feature that has been fit into the language - such as Promises. However, there are already a number of module systems that are used and very well supported in ES5. The fact that there are different module systems, and that they aren't trivially compatible, is a big problem. There are ways of making the three major module systems work together, but it's not great to have to do that just to work around this fact. As such, the fact that there is a language-level module system in ES6 is a good step forwards. It's just that the new module system isn't a perfect solution.

<!-- more -->

### Existing ES5 Module Systems
One thing that is worth remembering here. Just because ES6 introduces a whole new module system as part of the language doesn't mean that you can't use the other module systems. If it works for you, then use it. It's even possible to mix and match to an extent, though you need to be careful there for obvious reasons.

#### AMD (Asynchronous Module Definition)
AMD is the module system that works well for browser-based Javascript, because of how the modules get loaded. The most common implementation of this is [RequireJS](http://requirejs.org/), and the code roughly looks like:
```javascript
define(['module-a', 'module-b'], function(a, b) {
    // Here, the variable 'a' is bound to module-a, and 'b' is bound to module-b

    // Here we define a brand new module that has three different fields, 'func', 'integer' and 'string'
    return {
        func: function() {...}, 
        integer: a.integer,
        string: b.string;
    };
});
```
This works well in a browser because the define() function can register a module, and then the loader can asynchronously load all of the dependencies and then call the module only when everything is available.

In this case, a single file is at least one module, but can be more than that if you so desire. If you want you can specify a name for the module in the define() function, but if you don't then the assumption is that a single file is a single module and the filename is used for the module name.

#### CommonJS
CommonJS is the module system that works well for server-side Javascript, again because of how the loading works. The most common implementation of this is [Node.JS](https://nodejs.org/), and the above code would look like:
```javascript
var a = require('module-a');
var b = require('module-b');
// Here, the variable 'a' is bound to module-a, and 'b' is bound to module-b

// Here we define a brand new module that has three different fields, 'func', 'integer' and 'string'
module.exports = {
    func: function() {...}, 
    integer: a.integer,
    string: b.string;
};
```
NodeJS works well here because the require() method is synchronous, so loading a module will pause the entire program whilst all of the dependant modules are loaded, and if a dependant module fails to load then an error occurs. 

In this case, a single file is always a single module, and the module is identified by the file path.

It can be used in a browser by means of something like [Webpack](https://webpack.github.io/) if you really want, which then can combine all of your browser-side javascript into a single file for more efficient loading. 

#### Globals
Whilst technically not a module system per se, the third major alternative that gets used is simply to use global variables for everything and to just import everything by hand. It's crap, but it works.

### ES6 Module System
In ES6, the module system that has been introduced works completely differently to the above. Both AMD and CommonJS work on the assumption that you will want all of a module, or none of it. ES6 starts out with a completely different assumption - that a module exports a number of entities, and that another module will want some subset of those entities - which may be all of them.

At it's core, the ES6 module system has two complementary concepts - Exporting and Importing. A single file is always a single module, and a single module will export a number of different entities and import a number of other entities.

The above examples can be re-written as follows using ES6:
```javascript
// Here we import the entity called 'integer' from 'module-a', and the entity called 'string' from 'module-b'. Nothing else is imported
import {integer} from 'module-a';
import {string} from 'module-b';

// Here we export three different fields, 'func', 'integer' and 'string'
export function func() {...}
export integer;
export string;
```

Firstly, note how different the above looks. The import and export keywords are brand new, and the import line isn't even valid in ES5. Secondly, note that you can declare that you export an entity at the point you declare it, rather than needing to declare everything first and then export it all later on. Finally, the import mechanism is very specific as to what you import. You pull in a very specific entity from a module and you give it a very specific name.

The export mechanism is fantastic. You no longer need to repeat yourself when you define a module, and you no longer need to write some convoluted code to get things working as you want. You simply put the word "export" before the declaration and that entity will be exported. Simple.

The import mechanism also has a lot going for it. There are a lot of variations on how you can import entities, and you only need to import the very specific pieces of a module that you actually care about. I'm not going to go into the details of how it all works - there's plenty of that already out there.

I do have some problems with all of this though. Specifically, I have the following problems:

1. The import statement is non-trivial to read and to work out exactly what you want to do. There's a lot of variation on how it works, and it can be difficult to get it exactly right. Odds are, if you are working with ES6 modules all the way then you will almost always use the above syntax, but not always. 
2. You can't import the entire module and assign that to a variable. In both AMD and CommonJS, the import mechanism works exactly by assigning the entire module to a variable in this module, and then you access the pieces that you care about. In ES6 that is not possible. At all. This means that it's difficult to write one module that is a combination of other modules in a way that is non-repetative.
3. You can't import modules dynamically. (That's not quite true. There are ways in pure ES6, but not so easily in some of the transpilers). You can do this in CommonJS as a side effect of the above, and that makes certain things really easy to implement - such as loading all of the modules in a certain directory automatically without needing to specify them.

Note that #3 above isn't possible in AMD either, but it still feels like a step backwards in comparison to the CommonJS module system in some ways. I do understand why these restrictons are in place though - namely the fact that dynamic module loading is really hard to do in a remote environment.

In general, I like the new ES6 module system. I just think it's a shame that it's not compatible with any of the existing ones, but overall it's a step in the right direction.
