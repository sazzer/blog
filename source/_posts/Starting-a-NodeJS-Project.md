title: Starting a NodeJS Project - Project Setup
date: 2015-08-18 18:21:08
tags:
- nodejs
- javascript
- grunt
category: buildsystems
---
Whenever I start a new project using NodeJS, there are several things that I always do first. I thought that I'd finally get around to writing up what these things are, so that I can reference it myself, and in case anyone else might be interested. This covers part one of this writeup, setting up a base Node project, and adding Grunt in to it so that we can use Grunt as a task runner for more complicated builds. Next post will cover setting up the build so that we can write our code in ES6 instead, and later we will look at setting up some static analysis to keep code quality highter.

<!-- more -->

### Setting up package.json correctly
Node and NPM use a file called "package.json" to configure your project. This file covers a lot of details, including smongst other things the proejct dependencies and actions to perform. This is relatively simple to set up, but it's important to do it correctly. My bog standard template for this is simply:
```json
{
  "name": "template-node",
  "version": "0.0.1",
  "description": "",
  "scripts": {
  },
  "engines": {
    "node": ">=0.12"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/sazzer/template-node.git"
  },
  "author": "Graham Cox <graham@grahamcox.co.uk>",
  "bugs": {
    "url": "https://github.com/sazzer/template-node/issues"
  },
  "dependencies": {
  },
  "devDependencies": {
  
  }
}
```

This sets up the project with:
* A correct name, description and version number
* A minimum requirement for the NodeJS version to use
* Links to the Source Repository for the project
* Links to the bug tracker for the project
* Details of the author

At the very least, this works. It's nothing fancy, but it works.

### Adding Grunt into the mix
NPM has support for running certain lifecycle actions, but not many of them and not in a complicated manner. If you want to do anything more complicated then the choices are normally either Grunt or Gulp. Personally I'm a fan of Grunt - it's a system that you describe What to do, as opposed to How to do it - and so that's what I always use.

#### Installing Grunt
Adding Grunt is easy. You need a bunch of dependencies, a Gruntfile, and a few entries in your package.json file to tell NPM to run Grunt for some of it's lifecycle scripts. At a minimum the dependencies that I always use are:
* grunt-cli
* jit-grunt
* time-grunt

These are all Development Dependencies, so can be added by doing:
```bash
$ npm install --save-dev grunt-cli jit-grunt time-grunt
```

jit-grunt is a plugin that will automatically load Grunt plugins based solely on them being configured, which just makes the configuration file a bit cleaner to read. time-grunt is a plugin that will tell you at the end how long each step took so that you can see where all of the time is going. I use that purely because I find it interesting, but if you find that the build is too slow then it lets you work out where to optimise it.

#### Configuring Grunt
Grunt is configured by providing a Javascript file called "GruntFile.js". The main goal of this file is to specify the configuration of all of the plugins that you want to use, and to set up the tasks that you want to run. 

```javascript
module.exports = function(grunt) {
    require('jit-grunt')(grunt, {
    });
    require('time-grunt')(grunt);

    grunt.initConfig({
        pkg: grunt.file.readJSON('package.json')
    });

    grunt.registerTask('build', []);
    grunt.registerTask('run', ['build']);
    grunt.registerTask('test', ['build']);

    grunt.registerTask('default', ['test']);
};
```

This sets up no plugins - because we don't have any yet - and set up four tasks. The "default" task is the one that's run when you don't specify a task to run, and the other three are going to be called from NPM in a second.

Assuming that you've got the Grunt command-line tool installed globally - if not then do so now by running:
```bash
$ npm install -G grunt-cli
```
then you can test these out by running the tasks from the command line:
```bash
$ grunt test

Done, without errors.


Execution Time (2015-08-18 21:08:57 UTC)
loading tasks  109ms  ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇ 100%
Total 109ms
```

#### Calling Grunt from NPM
Once we're happy that Grunt is set up and working correctly, we can add the configuration to NPM to run the tasks automatically. I do this by adding the following into the package.json file:
```json
  "scripts": {
    "install": "grunt",
    "test": "grunt test",
    "start": "grunt run"
  },
```

This means that the default Grunt task will run when you do an "npm install", and separately to that you can run "npm test" or "npm start" to run "grunt test" and "grunt run" respectively"
