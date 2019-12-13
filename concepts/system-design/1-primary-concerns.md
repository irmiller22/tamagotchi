# Primary Concerns of System Design

There are 3 primary concerns of system design: reliability, scalability, and maintainability.

## Reliability

Reliability refers to the ability of a system to tolerate faults / problems
in order to prevent failures or complete shutdowns. Large systems that
we know of today tend to be built with fault intolerant components. The
challenge with system design, then, is to build fault tolerant systems that
leverage fault intolerant components.

Software faults can happen at any point due to many reasons. They can be handled
through more granular understanding of business requirements, building resiliency
to handle unexpected faults, monitoring to detect issues much earlier in the process,
better unit testing, and designing better interfaces to easily isolate problems.

## Scalability

Scalability refers to the ability of the system to deliver reasonable performance
during increased load. Increased load is a bit of an arbitrary term, but it can
be applied to a specific area that can be specified as a performance criteria
for an application (say, number of reads fetching tweets for Twitter's BE).

Performance is the system's operating characteristic when load on a system
is changed. Performance metrics quite often are a part of your SLA with 
customers. There are many ways to achieve system scalability.

## Maintainability

Maintainability means writing code that can easily be understood, refactored 
and upgraded by someone who is not the original author of the code. Any 
piece of spaghetti confusing code will ultimately be understood by machines. 
Good code should be readable and easily understood so that teams can 
collaborate. Good code should also have the right level of abstractions, 
clean APIs and interfaces so that new functionality can be easily built on 
top of existing codebase.

