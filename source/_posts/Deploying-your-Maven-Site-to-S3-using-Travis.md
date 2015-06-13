title: "Deploying your Maven Site to S3 using Travis"
date: 2015-06-13 11:16:51
tags:
    - travisci
    - s3
category: buildsystems
---
Assuming you've got a Maven Build, and you've got a correctly configured Site for your build, and then further assuming that you are using the [Travis CI](https://travis-ci.org/) system for your Continuous Integration (It's free, so why not), one potential next step is to automatically deploy your Maven Site somewhere.

<!-- more -->
This writeup shows how to publish your site to S3, and configure S3 to make it publicly accessable. There are plenty of other options out there, but S3 works well enough and turns out to be trivial to set up. All you need to make this work is:

* An AWS S3 Account
* An S3 Bucket already set up
* An AWS Access Key and Secret that can write to the bucket.

If we've got those things then we're good to go.

Firstly, we need to configure the maven build to be able to deploy to S3. I've done this using the [maven-s3-wagon](https://github.com/jcaddel/maven-s3-wagon) because it just works. The configuration that you need is twofold.

In the *build* section of your *pom.xml* file, you need to add the following. This makes it possible to use the S3 Wagon to upload your site.
```xml
<extensions>
    <extension>
        <groupId>org.kuali.maven.wagons</groupId>
        <artifactId>maven-s3-wagon</artifactId>
        <version>1.2.1</version>
    </extension>
</extensions>
```

Then, in your *distributionManagement* section of your *pom.xml* file, you need to add something like the following:
```xml
<distributionManagement>
    <site>
        <id>s3.site</id>
        <url>s3://<S3 Bucket Name>/<Site Path></url>
    </site>
</distributionManagement>
```

The S3 Bucket Name is literally the name of the bucket that you've configured in S3 - for example, I've used "grahamcox.co.uk" for mine. The Site Path is some directory path under this bucket that you want to deploy the site - for example, you might use something like "${project.artifactId}/${project.version}/site" to keep all of your sites under a directory that is both Artifact and Version specific. This has benefits that when you release later versions of your site, you get to keep all of the old sites, including all of your javadoc and everything, so that people can refer back to it. The path that you deploy under is completely up to you of course.

Once you've updated the pom file as above, you are now ready to actually deploy the site. You can test this by running *mvn site:deploy* by hand - and if you do this it will likely fail due to not having any authentication credentials available. This is the next step.

The Maven Plugin has documentation on how to configure authentication for S3 available at https://github.com/jcaddel/maven-s3-wagon/wiki/Authentication. This is work a read just to make sure you understand what's happening, and you can also set it up locally to do a test deploy to make sure you're happy with the deployment.

The way that I configured Travis to do this is using the Environment Variables, and specifically I used encrypted ones so that my AWS Key and Secret aren't just available on the internet in plaintext for anyone to see.

To do this, you first need to install the [Travis CLI tools](https://github.com/travis-ci/travis.rb). These are well documented on the Github page at https://github.com/travis-ci/travis.rb#installation, but in short you do:
```bash
$ sudo gem install travis
.....
$ travis login
.....
$
```

And then you're good to go. Once these are available, you can configure Travis to depoy your site.

First, tell Travis about your AWS Keys. This is done as follows:
```bash
$ travis encrypt --add env.glocal AWS_ACCESS_KEY_ID=abcdef AWS_SECRET_KEY=123456
```

Where the Access Key is "abcdef" and the Secret Key is "123456". (These are made up. Don't try them. They won't work).

This adds a new Encrypted setting to the Travis Build to make the Environment Variables AWS_ACCESS_KEY_ID and AWS_SECRET_KEY available throughout the build.

Secondly, tell travis to do a site deploy. I did this as follows, in my .travis.yml file
```yaml
language: java
jdk:
    - oraclejdk8
install: mvn clean install -P-quality-checks -DskipTests
script:
    - mvn install
after_script: mvn site site:deploy
```

Note that my build does not run tests or quality checks (PMD, FindBugs, Checkstyle, etc) as part of the Install phase, but does that all later on in the Script phase. I then do the site deploy after the script phase has run, whether the build was successful or not. This means that if I break the build, my site still gets deployed and tells me what my errors are. (In theory. I've not tested that yet)

Once all of this is done, you can now visit the Site deployed in S3 and marvel at it's beauty.
