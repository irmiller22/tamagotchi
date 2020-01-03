# Microservice Architects

## Outline

- [Role of Architects](#role-of-architects)
- [Decisionmaking](#decisionmaking)
  - [Strategic Goals](#strategic-goals)
  - [Principles](#principles)
  - [Practices](#practices)
- ["Good Citizen" Service](#good-citizen-service)
  - [Monitoring](#monitoring)
  - [Interface](#interface)
  - [Safety](#safety)
- [Ways to Share Knowledge](#ways-to-share-knowledge)
  - [Documentation](#documentation)
  - [Exemplars](#exemplars)
  - [Service Templates](#service-templates)
- [Responsibilities](#responsibilities)

## Role of Architects

Architects are in charge of making sure that an organization
has a technical vision that lines up with the systems that the organization's
customers need. In short, they are responsible for developing a framework
in which the right systems can emerge and continue to evolve over time.
Architects, to borrow an example from Sam Newman, are like town planners
in that they define architectural direction in broad strokes, and only
get involved in defining implementation details in specific cases.

Instead of thinking about what happens inside a microservice, architects
tend to think more about what happens _in between_ microservices. Choices
made in between microservices tends to be what leads to a messy service
implementation. What happens when your service leverages multiple
communication protocols (HTTP, SOAP, gRPC, etc)? Once architects determine
an optimal course of action, then it's up to them to ensure that the
trade-offs are worth it.

Here's a good guideline to follow:

> Be worried about what happens between the boxes, and be liberal about
> what happens inside.

## Decisionmaking

Making decisions as an architect is all about trade-offs:

1. Is it ok for us to have multiple technology stacks in our system?
1. Do we pick a datastore that we have less experience with, but scales well?

When making decisions, it's important to use a decision framework to determine
the best course of action:

- Strategic Goals
- Principles
- Practices

### Strategic Goals

These are high level goals that are defined at an organizational or group level.
They are meant to be directional goals that communicate where your organization
or group is headed, and are more aligned around strategy than anything else. It's
up to the architect to come up with principles that adhere to these streategic
goals.

### Principles

Principles are rules that are made in order to align with your strategic goals.
Generally, principles tend to be constraints that are necessary to reach some
necessary outcome. For example, Heroku's 12 Factors are mostly a list of constraints
that define design decisions necessary for optimal usage of Heroku's application
deployment process. By rule of thumb, it's useful to have at most 10 principles
for your organization.

### Practices

Our practices are how we ensure our principles are being carried out. They are a
set of detailed, practical guidance for performing tasks. They will often be
technology-specific, and should be low level enough that any developer can
understand them. Practices could include coding guidelines, the fact that all
log data needs to be captured centrally, or that HTTP/REST is the standard
integration style. Due to their technical nature, practices will often change
more often than principles.

## "Good Citizen" Service

A "Good Citizen" service is essentially an exemplar service within your system
that details what an ideal service should look like. The idea is that it
should encapsulate the engineering best practices within your organization to
allow for replication of those best practices. As Ben Christensen from Netflix
puts it, when we think about the bigger picture, “it needs to be a cohesive
system made of many small parts with autonomous lifecycles but all coming
together.” So we need to find the balance between optimizing for autonomy of
the individual microservice without losing sight of the bigger picture.
Defining clear attributes that each service should have is one way of being
clear as to where that balance sits.

### Monitoring

One of the most critical functions of an architect is to ensure that there
is visibility throughout the entire system. To start, implementing health
checks, logging, and monitoring-related metrics are critical. In order to keep the
implementation details simple, they should both be emitted in the same
manner.

For example, you might choose a push mechanism (Graphite, Nagios) or
polling mechanism. It's critical to keep it as simple as possible.

### Interfaces

Ensure that the number of interfaces chosen for the system is as small as
possible in order to reduce complexity. Having one standard is good - having
two isn't terrible. But having more than 10 standards is _not_ good.

For example, if you have 5 different standards for HTTP/REST protocols, will
you use verbs or nouns? How will you handle pagination? What about endpoint
versioning?

### Safety

In the context of services, we want to ensure that all services within the
system act in the best interest of other services. This means that when a
service goes down, it should not take the rest of the system down with it.
At a minimum, this means that an architect would want to mandate that each
downstream service gets its own connection pool, and also that each service
also leverages a circuit breaker.

Define a set of rules and guidelines for safe interoperability between services
to ensure that your system is robust and fault-tolerant.

## Ways to Share Knowledge

### Documentation

Documentation is the most straightforward way to share knowledge, especially
when there is a varied audience. For example, you can have technical documentation
that breaks down implementation details for engineers, and you can also have
non-technical documentation that describes a high-level overview for how a
specific system works.

Documentation, however, tends to be abstract, and it serves as another translation
layer that engineers need to translate in order to put that information to use.
Enter exemplars and service templates.

### Exemplars

An exemplar is a real-world service that incorporates most, if not all, of
the organization's best practices that align with the organization's
principles.

Developers like code, and code they can run and explore. If you have a set of
standards or best practices you would like to encourage, then having exemplars
that you can point people to is useful. The idea is that people can’t go far
wrong just by imitating some of the better parts of your system.

Ideally, these should be real-world services you have that get things right,
rather than isolated services that are just implemented to be perfect examples.
By ensuring your exemplars are actually being used, you ensure that all the
principles you have actually make sense.

### Service Templates

A service template is a template that incorporates all of the organization's best
practices, and is a means of bootstrapping new services/components in such a
way that it aligns with the organization's principles. It is different from an
exemplar in that it is not a real-world service - it is merely a template.

One downside of a service template is that it corrals engineering choices into a
small list of "accepted" practices. For example, if I have a service template
that leverages Python 3.7 and gRPC, that may subconsciously communicate to
other engineers that they shouldn't consider other technology choices. This
isn't as significant an issue for smaller organizations, but it does start to
become an issue for larger organizations once an inflection point is reached
in terms of system support (for example, a metrics service template that has
been used in X microservices - it makes it more difficult to innovate over
time).

## Responsibilities

Here are the core responsibilities of an architect:

> Vision

Ensure there is a clearly communicated technical vision for the system that
will help your system meet the requirements of your customers and organization.

> Empathy

Understand the impact of your decisions on your customers and colleagues.

> Collaboration

Engage with as many of your peers and colleagues as possible to help define,
refine, and execute the vision.

> Adaptability

Make sure that the technical vision changes as your customers or organization
requires it.

> Autonomy

Find the right balance between standardizing and enabling autonomy for your
teams.

> Governance

Ensure that the system being implemented fits the technical vision.

An architect is one who understands that pulling off this feat is a constant
balancing act. Forces are always pushing you one way or another, and
understanding where to push back or where to go with the flow is often
something that comes only with experience. But the worst reaction to all these
forces that push us toward change is to become more rigid or fixed in our
thinking.
