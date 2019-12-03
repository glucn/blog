# Refactoring Legacy Code

*Originally posted at https://vendasta-blog.appspot.com/blog/BL-QT79H4QW/*

We broke accounts microservice a couple of weeks before. 120,000 demo RM accounts returned from death, and this brought heavy traffic to the Core Services.

It happened when we refactored a function for converting data to protobuf. We assigned deactivation date to the field `deactivationOn`, but it turned out that VBC and our legacy projects expected expiry date in this field. 

In this post, I won't go into the detail of the post-mortem report, but I want to discuss some rules of thumb about refactoring legacy code.

![tangled wires](https://i.pinimg.com/originals/2c/a4/8f/2ca48f98ffc7d51c7b0c47e661a353a1.jpg)

## Understand & Test it before refactor it

The legacy code may look like these wires, but it is working fine, probably. Before we plug out any wire, we need to know where it goes.

The first thing we can do is to run the tests. If the code you are going to refactor have a good test coverage, god bless you, it will help you a lot in understanding the existing code and verifying the code after the refactoring.

> To me, legacy code is simply code without tests. ~ Michael Feathers

But things do not always go right. This quote reflects the perspective of legacy code being difficult to work with in part due to a lack of automated tests. If we spot any weird logic in the legacy code and there is no test around it, we'd better ask for some suggestions from the domain experts. Our team leads, tech leads, or open the annotation, the one who changed the code last time could be the good candidates of the experts who can explain to us why the code is like that. 


## Have a separate PR for refactoring

I usually do the refactoring when I'm working on some related new features. It can be okay if the refactoring is minor, but it is not the best practice.

Imagine when we review a PR, there are always some red blocks here and green blocks there, and in some cases, Github is just not smart enough to get them aligned as we expected. For example, when we move some functions from a file to another file, the red block and green block will probably be several screens away from each other. If we mix and match our refactoring and some other changes together in one PR, it will become a game of "Spot the difference" for the reviewers.

So, if we are going to have some significate refactoring on our existing code, I'd suggest having a separate PR, and in this PR, we should make sure the current behavior will not change. 

Another thing to consider about is breaking down the whole refactoring into baby steps. "Divide and Conquer" can also be applied here. With smaller commits and PRs, we can replace every part of the Ship of Theseus without nobody even noticing that.


## Consider fallbacks
Refactoring can be risky, any mistake can damage the existing process. In order to reduce the risk, here are some approaches we may want to try.

#### Parallel Run
The idea is to use the legacy version of the code as a reference for your refactoring, run both and check that they are doing the same thing. We can implement parallel run in many different ways. 

https://github.com/vendasta/IAM/pull/146 is an example of hedging old and new elastic indices, which can be an excellent reference if we want to refactor code in this way.

#### Strangler pattern
Strangler vines seed in the branches of a tree, gradually work their way down to the soil, and kill their host. This is also a metaphor of how we refactor or rewrite the legacy code. Here is an example I steal from [Philippe Bourgau's blog](http://philippe.bourgau.net/):
>> For example, we had to refactor a rather complex parser for an internal DSL. It was very difficult to incrementally change the old parser, so we started to build a new one aside. It would try to parse, but delegate to the old one when it failed. Little by little, the new parser was handling more and more of the grammar. When it supported all the inputs, we removed the old one.


## Ask for more eyeballs
> Given enough eyeballs, all bugs are shallow.  ~ Linus's Law

Refactoring can be boring and difficult if we do it alone, and it will be a lot easier if we team up. First, it creates a mindset of "we are all in this together" and all participants share the same understanding of the logic before and after the refactoring. Second, the pair or mob programming can prevent some silly mistake an individual can make.

In addition, we should make our PR useful and friendly for the reviewers. Some brief explanation of the background, some comments on specific lines to explain the details ("Why", not "What"), or maybe some screenshots, all of these will make the code review effective and efficient. Besides, as I mentioned above, try to have separate PRs for code change and code refactoring.


## Write tests before you go
> Keep the camping ground cleaner than you found it. ~ Boyscout Rule

Before the refactoring, there might not be enough tests for the legacy code, but it can not be the status when we finish our refactoring.  After our work, the code should be isolated enough to test part by part. And a good test coverage can build the confidence for us that the refactoring we make will not break the production environment.

On the other hand, we are not going to test everything. An unrealistic target of the test coverage is totally a waste of time. We need to test all the non-trivial business logic. In addition to that, the test should also cover the functions that would cause havoc should they malfunction, even the code itself is as simple as assigning the value to a variable.


## Tiny Tips
* Some extra log can help you understand the code you are going to work on or detect the problem after the refactoring. Just remember to remove them;
* Before the refactoring, set up some dashboards or monitors in Datadog, so that you can know in the first moment if something goes wrong.


## Conclusion
There is no silver bullet for refactoring legacy code. It’s no promise that refactoring will become easy or fast. I hope this post can be a starting point for you to consider your refactoring plan.

Some more references:

* [Code Smells](https://blog.codinghorror.com/code-smells/)
* [Catalog of Refactorings](https://refactoring.com/catalog/)
