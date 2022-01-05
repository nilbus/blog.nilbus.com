---
layout: post
title: "Why You Should Use Dependency Injection"
modified:
categories:
excerpt: "Dependency Injection is one practice that you may be familiar with, but how often do you actually use it in your every-day code?"
tags: [software-engineering]
comments: true
image:
  feature: https://res.cloudinary.com/dhttas9u5/image/upload/c_fill,fl_progressive,h_800,q_auto,w_1600/possessed-photography-YKW0JjP7rlU-unsplash_1_gs5mju.jpg
date: 2020-06-09T22:15:00-04:00
---
As software engineers, we generally have a strong desire to write clean, maintainable code. We want changes to take as long as they seem they intuitively should, not get hung up in a hard-to-understand mess. We want to write code that will be a pleasure rather than a pain for us and others to work with later. To this end, we attempt to use good software design principles such as [SOLID](https://en.wikipedia.org/wiki/SOLID). Dependency Injection is one practice that you may be familiar with, but how often do you actually use it in your every-day code? My goal in attending Chris Hoffman’s talk [Injecting Dependencies for Fun and Profit](https://www.youtube.com/watch?v=b5vfNcjJsLU) at RubyConf 2019 was to become inspired and motivated to better use Dependency Injection and understand the benefits it offers. I found what I was looking for!

Examine the differences between these two blocks of Ruby code for controlling a robot that can communicate with speech:

![code with dependencies injected as parameters to the initializer, or not](https://res.cloudinary.com/dhttas9u5/image/upload/Screen_Shot_2020-06-09_at_1.47.18_PM_m1tnai.jpg)

The dependency-injected version on the right is arguably more complex because of the additional indirection. Plus, it’s certainly more code. What, then, does dependency injection gain us?


## Surprising code reuse

This can be one of the most delightful outcomes of  dependency injection. Similarly to how we can chain together UNIX shell commands with pipes in ways that the original authors never imagined, we can sometimes build new things using existing classes by injecting a new dependency. Giving ourselves the ability to make changes without modifying code helps us keep the Open/Closed principle ([the O in SOLID](https://softwareengineering.stackexchange.com/questions/19627/clarify-the-open-closed-principle)). This is possible because our code relies only on the dependency's interface, not a concrete implementation or class ([the D in SOLID](https://softwareengineering.stackexchange.com/q/401769/69145)). This is what makes dependency-injected code so much easier to test. Code is expensive to write and maintain, so anytime we get reuse, that is a huge benefit to everyone.

## An explicit list of dependencies

Knowing what a class depends on has _so_ many benefits that may not be immediately obvious. Having a list of dependencies reduces the complexity of several tasks and helps the code remain more maintainable. Let's go into detail on some of these benefits that make dependency injection absolutely worthwhile.


### Optimize for understanding

Much more of our time is spent trying to understand code than writing it. This is true for developers generally but more especially for developers onboarding to a new team or codebase. Since a class’s dependencies often mirror its responsibilities, listing them explicitly makes them much more discoverable, as opposed to having to hunt through the entire class. The contrived example above is simple to scan through, but real classes of tens or hundreds of lines make this much more challenging.


### Unmask the signs of too many responsibilities

When we see the dependency list growing large, it serves as a code smell for us, prompting us to ask, “Is this code doing too much?” Perhaps the class has grown to have more than a single responsibility ([the S in SOLID](https://softwareengineering.stackexchange.com/q/17100/69145)). This signal can help us refactor toward better-designed code at an earlier stage than if we have only gut feelings to tell us when a class is doing too much. With code always growing and changing, the longer we allow single-responsibility violations to persist, the harder the code can be to untangle when it eventually gets unmanageable.


### Expose the hidden ways code can fail

In order to provide good user experience, we try to consider all the potential failure scenarios. Since unit tests eliminate most failures in the class under test, dependencies become the primary cause of failures. In many cases, surprising failures creep into production because some unexpected value or exception is returned by a third party library or service. Something we depend on failed to act as we expected. If our app code doesn’t handle these scenarios intentionally, an error or some strange behavior gets passed on to our users.

On a project using dependency injection, each class has a list of dependencies, or in other words, a list of things that might cause the class to fail. Use this list when reviewing your code. Consider the ways your dependencies could cause the class to fail, and make sure failures are handled in an appropriate way. 


### More straightforward testing

Let’s now consider how we would test our `Greet` class using RSpec.

![a test that needs to mock a method to return a double, vs a test that passes in the double](https://res.cloudinary.com/dhttas9u5/image/upload/Screen_Shot_2020-06-09_at_1.47.57_PM_uad0vt.jpg)

Our test double plays the critical role of standing in for the real `RoboIO` instance since we do not want real speech audio output when running our unit tests. The primary difference between these approaches is that the test doesn’t need to know and reproduce how to load the speech dependency. Although this example is simple, imagine if we were instead loading database objects from ActiveRecord using a chain of `where`, scopes, `includes`, `joins`, `limit`, `offset`, and other calls frequently subject to change in how they’re combined. The chain of expectations can get ridiculously long and brittle, requiring spec updates for each trivial change—change often tangential to the purpose of the method under test, since there are so many ways to accomplish the same task with ActiveRecord. Passing in a test double that stands in for the dependency can dramatically simplify test setup.

End-to-end tests should stub nothing and exercise the default arguments, but have you ever forgotten to stub a dependency that should have been stubbed in unit tests? Having an explicit list of dependencies helps prevent you from being surprised about a dependency that you didn’t notice. Also, as highlighted earlier, the dependency list shows us the list of integration points that our tests should cover success and failure scenarios for.


# Removing the boilerplate

Manually injecting dependencies into class constructors is fine to a point but gets unwieldy with many dependencies and/or dependencies that require more than a tiny amount of code to construct.

The dry-rb project maintains the [dry-auto_inject](https://dry-rb.org/gems/dry-auto_inject) gem, which moves dependency construction to a separate class, allows their import, and automatically extends the initializer and creates accessor methods.

![a code example of using the dry auto-inject library](https://res.cloudinary.com/dhttas9u5/image/upload/Screen_Shot_2020-06-09_at_3.42.19_PM_am8rzy.jpg)

Alternatively, Chris Hoffman, the talk author, maintains the [dependency_bundle](https://github.com/yarmiganosca/dependency_bundle) gem, which also extracts dependency construction to reduce initializer size. By convention, classes accept a `deps:` keyword argument and manually extract the specific dependencies for use (no automatic accessor methods). Depending on how much you prefer explicitness over magic, you may prefer this approach.


# Beware of taking it too far

Dependency injection is an excellent tool for reducing coupling between dependencies, with all the possible benefits described in this article. However, lest we turn our code into a convoluted equivalent of [Hello World Enterprise Edition](https://gist.github.com/lolzballs/2152bc0f31ee0286b722), we should use it only at appropriate times to solve the specific problems it addresses.

The most obvious use case is to inject dependencies that interface with external systems such as databases, web services, or hardware, especially when calling the dependency produces side effects. Injecting these dependencies will help keep the external systems isolated, which helps with replacement and unit testing. Dependency injection can definitely be used within the same system where you want isolation. In each case, you'll need to weigh the cost of indirection, which makes the code harder to follow, against the the other benefits you get from injecting the dependency.


# Taking it home

Hopefully, you find the benefits of dependency injection to be as compelling as I have and can find the right opportunities to use it in your work. If you're interested in this topic, you may also enjoy [Chris Hoffman’s talk](https://www.youtube.com/watch?v=b5vfNcjJsLU) (29 minutes) RubyConf talk that inspired this post.
