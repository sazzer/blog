title: TDD with Static Typed Languages
date: 2016-05-29 15:12:50
tags:
  - testing
  - TDD
category: programming
---
TDD, or Test Driven Development, is fairly well acknowledged to be a great idea. The general theory behind it is that you write your tests first, which will initially fail, and then you write just enough code to make the tests pass. Once the tests pass, the code is finished and you can move on to the next bit of functionality. There have been studies done that show that working this way drastically reduces the number of issues in the overall product.

And for some languages, this is all fantastic. You write the tests, the tests fail, you move on to the actual development.

And for yet other languages, this just doesn't work at all. Specifically I'm thinking compiled, statically typed languages here. The problem with this scenario is, when you write your tests against non-existent types then you break the compilation of the test suite, which in turn breaks *every* test in the system. And that in turn means that until you've implemented at least enough for all of the tests to compile you can't be sure that you've not broken anything else.

<!-- more -->

Historically, I have to confess that this has led me down the path of thinking that TDD is just a non-starter in these situations. And so I've always carried on working in the same way, writing the code first and the tests after. And, of course, sometimes I then miss some tests out or just don't get around to them. And as such the whole test suite suffers. Not good.

I've just recently come up with a thought to reconcile all of this though. Namely, TDD doesn't have to apply to all levels of testing. As such, my new plan is to apply the TDD principles to Integration and E2E testing only. Unit Testing I will continue to do after the code being tested has been written, and that's fine.

Why this line in the sand? E2E testing is literally testing the application as a whole. You write the tests as if a user was using the application, driving the user interface and asserting only on things that are visible to the user. Integration testing is testing at the code level, but it is testing an entire set of work in one go. As such, that entire set should have a single concrete API, and you should test against that API. You are therefore able to define the API first, and write tests against this API before you implement it. The end result here is that everything that you are testing in this way is done so from the outside in, and generally with no access to the innards that you've not yet written. This is important, because it means that the entire test suite can be written without any implementation code existing yet, which is exactly the idea behind TDD.
