title: "Deploying Hexo to Github Pages with Travis"
tags:
  - hexo
  - github
  - travisci
---
I've just recently been working on getting my Hexo blog set up, and importantly on getting it automatically deploy to Github Pages every time I do a commit. Given that this is a generated site, there are intermediate steps involved between the source that is committed and the site that is deployed. 

The obvious way to achieve the actual Build part of this is to use the fantastic [Travis CI](https://travis-ci.org/), which can be set up to perform a build every time you do a commit to a Github Repository. The challenge involved in this is getting Travis to be able to push the deployed site back to Github for it to be accessed. However, it turns out that Github have a solution that can work for this as well.

From this point on, I'm going to assume that you've already got Hexo configured as you want it, and you know how to configure Travis to build a Github repository correctly.

### Configure Travis to build Hexo
The first thing that I did was to configure Travis to build the Hexo site, but not deploy it. This is simply a case of enabling the Travis hooks for this repository, and adding a .travis.yml file as follows:
```yaml
language: node_js
branches:
  only:
  - master
before_install:
- npm install -g hexo
- npm install
install:
- hexo generate
```

This configures Travis to treat the repository as a Node.JS build, to only ever build when the "master" branch changes (This is important, otherwise we get an infinite loop caused by Travis updating the repo itself), and to actually install Hexo and run the generate step.

### Configure Hexo to deploy to Git
Next we need to configure Hexo to deploy to Github correctly. This is best done using the existing "hexo-deployer-git" plugin, which does exactly what we need. Installing this is as simple as:

```bash
$ npm install --save hexo-deployer-git
```
and then entering the following into the \_config.yml file in the Hexo root. 

```yaml
deploy:
    type: git
    repo: git@github.com:sazzer/blog.git
    branch: gh-pages
```

The branch needs to be "gh-pages" to work with the Github Pages system, and the repo needs to be the repository that you wish to deploy to. Note that I've used the SSH version of the URL here, which means that you can deploy to it manually but Travis will not be able to, unless you're daft enough to configure your Github account with an SSH Key that Travis uses. (Don't do that)

### Set up a Github Token to use for deployments
Travis is unable to push to your Github repository by default. This makes a lot of sense from a security point of view - if Travis can push to your repository then it can change your code without your permission. However, Github have an OAuth style system where you can use an alternative URL to push to that uses an OAuth token instead of an SSH key. 

Firstly you need a Token. You can generate these by visiting the [Personal access tokens](https://github.com/settings/tokens) page in your Github settings. Simply configure a new token here, and make sure that it has the "repo" permissions at least.

When we want to use this token to actually push to Github, we use a different URL to the usual one. This isn't instantly obvious, but the URL that you need to use is: https://<TOKEN>:x-oauth-basic@github.com/<user>/<repo>.git. So, for this blog the URL is https://<TOKEN>:x-oauth-basic@github.com/sazzer/blog.git.

### Rewrite the Hexo config to use the Github Token
We obviously don't want to ever commit the Github Token to the repository in plain text - that would be silly. If we do that then anyone who can read the repo, which for Public repositories means everyone, can see the token and then push changes to it. What's worse, the token is applied to the entire account, not just to a single repository so if someone compromises this token then they can write to *every* repository you've got.

Thankfully, Travis has a solution here too. Travis supports setting Encrypted settings in your .travis.yml file, so you can safely add an encrypted setting that stores your Github token safe in the knowledge that only Travis builds of this specific Github repository - not forks of it - can read it.

To do this you need to use the Travis command line tools. This is a simple Ruby gem, so you can install it with:

```bash
$ gem install travis
$ travis login
```

Once installed you then add a new setting to your .travis.yml file with:
```bash
$ travis encrypt 'GH_TOKEN=<TOKEN>' --add
```

This will add a new section to your .travis.yml file along the lines of:
```yaml
env:
  global:
      secure: X7+8XAko5oQgDTuPbLEQVlb0qwXjLkz1Pgia6eGL4Q/0kZ89hGzGtOrKAhmtZRpcmg+PBxrLo5bdziAP/rvFslIabHHYBYv8ES8VrA81B/Q+t1VFbQEfhZdSq/L0wVsyBe2p6OOu2bNOLjPMG//aaynLxXstEEVSHh/lSuzsE4A=
```

That long string is the encrypted environment settings that we want to use. When the Travis build executes then an environment variable called "GH\_TOKEN" will exist with the Github token in it.
Steps: 
1. Set up Hexo
2. Set up Travis to build Hexo
3. Set up Hexo to deploy to Github
4. Configure a Github Token
5. Configure Travis to rewrite the config using this token
6. Configure Travis to deploy
