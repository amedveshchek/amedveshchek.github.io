---
title: "Clean Architecture and Tests"
layout: post
date: 2019-06-18 17:30
image: /assets/images/markdown.jpg
headerImage: false
tag:
- architecture
- testing
category: blog
author: alexmedveshchek
description: "Clean architecture and Tests"
---

### It's a draft

This is just a draft: compilation of notes and references of the above themes.

## Architecture

- [The classical explanation from Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html):
![Classical scheme](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg)
![Classical scheme cone](https://www.codingblocks.net/wp-content/uploads/2018/02/The-Clean-Architecture-Cone.jpg)

- [Clean architecture example](https://github.com/mattia-battiston/clean-architecture-example)
![quick intro](https://cdn-media-1.freecodecamp.org/images/lbexLhWvRfpexSV0lSIWczkHd5KdszeDy9a3)

- [Заблуждения Clean Architecture](https://habr.com/en/company/mobileup/blog/335382/)
![Clean architecture](https://habrastorage.org/web/531/04c/89d/53104c89d9cf44a59c95e351b7485574.png)


## [Google Testing Blog: just say "NO" to more End-To-End tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)

- Unit tests -- testing the small unit.
- Integration tests -- integration test takes a small group of units, often two units, and tests their behavior as a whole, verifying that they coherently work together.
- End-To-End tests -- user actions simulation.


> As a good first guess, Google often suggests a _**70/20/10**_ split: 70% unit tests, 20% integration tests, and 10% end-to-end tests. The exact mix will be different for each team, but in general, it should retain that pyramid shape. Try to avoid these anti-patterns:
> - Inverted pyramid/ice cream cone. The team relies primarily on end-to-end tests, using few integration tests and even fewer unit tests. 
> - Hourglass. The team starts with a lot of unit tests, then uses end-to-end tests where integration tests should be used. The hourglass has many unit tests at the bottom and many end-to-end tests at the top, but few integration tests in the middle. 

