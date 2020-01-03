# Introduction to Microservices

## Outline

- [What Are Microservices](#what-are-microservices)
- [Why Microservices](#why-microservices)
- [Benefits of Microservices](#benefits-of-microservices)
  - [Heterogeneity](#heterogeneity)
  - [Resilience](#resilience)
  - [Scaling](#scaling)
  - [Organizational Alignment](#organizational-alignment)
  - [Composability](#composability)

## What are Microservices

Microservices are small, autonomous services that work together in tandem.
There are several principles that a microservice adheres to:

- Single Responsibility
- Autonomous
- Maintainable

A microservice embodies the Single Responsibility Principle (SRP) - they are focused on
only one thing, and doing it well. They are also autonomous and maintainable.

An important question to answer is _how small_ should microservices be? There's
no set standard for this question, but a good rule of thumb for size is that
it could be refactored in 2 weeks. The smaller the service, the more you're
able to maximize the benefits of microservices. The trade-off with smaller
services is that you have to then manage the interdependence of small
services with others, which leads to more moving parts in your system.

In short, microservices should aim to be autonomous and maintainable.
Communication with other services should happen via network calls in order
to better enforce separation and prevent tight coupling.

## Why Microservices

Where did microservices come from? In short, they came about as a result of
rapid iteration and advancement in how engineers manage their software.
Microservices have emerged With the rise of domain-driven design, continuous
delivery, virtualization, and automation tooling.

By embracing the principles around microservices, engineers can deliver
software much more quickly and reliably. This gives engineers more freedom
to react and make decisions independently of other groups / parties / code
boundaries.

## Benefits of Microservices

### Heterogeneity

Microservices tend to leverage homogenous technology, especially when
compared to monolithic services. As a result, it's much easier to
implement new technology in microservices due to the surface area
being much smaller. For example, if I have a monolithic application and I
want to test out a new database technology, this might require changes in
multiple places across the application. However, with a microservice, there
are much fewer areas where change is necessary due to the smaller surface
area, which also makes microservices more suitable as a testing ground.
Long story short, it's much easier to absorb new technology in an organization
that adopts the microservice paradigm.

### Resilience

There's a key concept in engineering called the bulkhead. If one service in
your system fails and that failure doesn't cascade, you can isolate the problem
and allow the rest of your system to continue operating normally. In this
context, your bulkheads are your service boundaries. In a monolithic system,
if a service fails, then everything stops working. Microservices allow your
system to become much more resilient, assuming that errors and exceptions are
handled safely.

### Scaling

Monolithic services are known for scaling issues, particularly due to the fact
that monoliths have to be scaled up or down in tandem. It's difficult to scale
a sub-service within a monolith independently. With microservices, it's possible
to just scale only each microservice independently, allowing for much more
granular fine-tuning.

### Organizational Alignment

Microservices allow us to better align our architecture to our organization,
helping us minimize the number of people working on any one codebase to hit the
sweet spot of team size and productivity.

### Composability

Microservices allow us to better align our architecture to our organization,
helping us minimize the number of people working on any one codebase to hit
the sweet spot of team size and productivity.
