---
layout: default
title: About
permalink: /about/
---

# About library

**Changeset** is lightweight CQRS/ES library for PHP. It is based little bit on [Prooph](https://github.com/prooph) and
little bit on [Broadway](https://github.com/broadway). However it is very much simplified and minimum viable
functionality contains maybe dozen of classes.

### Project motivation

#### Simplicity

Having extensively [worked with Prooph](https://github.com/nixilla/cqrs-bundle) (mostly with version 6) in the past
I found it as pretty good and logical implementation of the CQRS/ES concept. Nevertheless I didn't like some of
the design concepts they decided to implement. Although Prooph is good, I think we can have even better/simpler design,
which can be easier to understand by developer.

#### Based on VCS changeset concept

**Changeset** concept is based more on VCS changeset where a single action is applying change to read data in your system.
Every change is persisted in the write model, in the way that it does not depend on previous changeset. It means that
write model does not carry the state of data, it is only a data change log. It means that, there is no way to implement
some of the CQRS/ES concepts which rely on write model state, like sagas. On the other hand **Changeset** never reads
from write model before adding new event, therefore makes it faster.

#### Can be easily added and removed from the project

The read model and write model are completely independent. It means that when you develop your app, you still design
your data model as you would do normally with Doctrine for example, then you add write model layer and you get CQRS/ES.
It also means that you can remove CQRS/ES easily from your project, if you decide this is not required for your project.

#### Education

The CQRS/ES concept is not easy to grasp. Most of the blog posts are not really helping, neither the libraries.
By using **Changeset** you will learn basic concepts of CQRS/ES and by analysing the source code, you will quickly
get up to speed. 

### Project structure

Project has been split into couple of sub projects to allow you to override almost everything in your app. All code
relies on interfaces defined in [chgst/common](https://github.com/chgst/common). It means you can write your own
implementation of about everything in **Changeset** library if you want.

You will find the main implementation in [chgst/chgst](https://github.com/chgst/chgst). Currently there is only 5 PHP
classes there.

There is also a [datastore Doctrine implementation](https://github.com/chgst/persistence-doctrine) which is an optional
dependency. It contains literally 1 PHP class. You can implement your own persistence to any datastore you want. All you
need to do is to implement 1 interface.

And finally there is [a Symfony bundle](https://github.com/chgst/chgst-bundle) if you want to use **Changeset** with
your Symfony projects.



