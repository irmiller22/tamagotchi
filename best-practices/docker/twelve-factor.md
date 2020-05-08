# Docker - Twelve Factor Application

## Outline

- [Codebase](#codebase)
- [Dependencies](#dependencies)
- [Configuration](#configuration)
- [External Services](#external-services)
- [Build, Release, Run](#build-release-run)
- [Processes](#processes)
- [Port Binding](#port-binding)
- [Concurrency](#concurrency)
- [Disposability](#disposability)
- [Development and Production Parity](#development-and-production-parity)
- [Logs](#logs)
- [Admin Processes](#admin-processes)

## Codebase

WIP: https://github.com/docker/labs/blob/master/12factor/01_codebase.md

One application is analogous to one codebase.

If there are several code bases, it is a distributed system containing multiple applications.

One code base is used for multiple deployments of the same application:

- development
- staging
- production

## Dependencies

The application's dependencies must be declared and isolated.

## Configuration

The application's configuration must be stored in the environment via environment variables.

## External Services

The application should handle external services as external resources of the application.

Examples of external services include:

- database(s)
- logging services
- error reporting tools

This allows the application to be loosely coupled with the external services, so that it's straightforward to switch vendors out if necessary.

## Build, Release, Run

The application's build, release, and run stages must be kept separate. By isolating these three stages, you decrease the risk of failure in your deployment pipeline.

This way, once the build and release stages have completed, then we can update the execution environment (the image version) in the release stage.

## Processes

The application consists of several processes.

Each process must be stateless and must not have local storage (such as sessions). This is crucial for 2 reasons: scalability and fault tolerance.

If you have data that needs to be persisted, the data should be stored in a stateful system, such as a database.

## Port Binding

The application must be exposed to the outside world via port binding.

In order to be compliant with 12 Factor guidelines, an application must use specialized dependencies, such as a HTTP server, and expose the service through a port.

The host has a responsibility to route the request to the correct application through port mapping.

## Concurrency

The application can achieve concurrency via horizontal scalability with processes.

For example, the application can be seen as a set of processes across different types:

- web server
- worker
- cron

Each process would need to be able to scale horizontally. Additionally, each process can leverage its own multiplexing.

## Disposability

The application must have processes that can be easily disposable.

In order to be disposable, it must meet the following guidelines:

- it must have a quick startup
  - makes scalability more performant
- it must ensure a clean shutdown
  - stop listening on the port
  - finish handling the current request
  - leverage a queueing system for long-lived processes

## Development and Production Parity

The application's separate environments must be as close as possible.

Docker is very good at reducing the gap as the same services can be deployed on the developer machine as they could on any Docker Hosts.

A lot of external services are available on the Docker Store and can be used in an existing application. Using those components enables a developer to use Postgres in development instead of SQLite or other lighter alternative. This reduces the risk of small differences that could show up later, when the app is on production.

This factor shows an orientation toward continuous deployment, where development can go from dev to production in a very short timeframe, thus avoiding the big bang effect at each release.

## Logs

The application needs to handle logs as a timeseries of textual events.

The application should not handle or save logs locally but must write them in stdout / stderr.

## Admin Processes

Admin process should be seen as a one-off process, as opposed to long running processes that make up an application.

Usually used for maintenance task, through a REPL, admin processes must be executed on the same release (codebase + configuration) as the application.
