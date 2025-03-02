+++
title = "Graphql Java Instrumentation"
date = 2022-07-14
+++

# Disclaimer

This is my first blog post. :D

# Context

I just started a new project at work, where we need to provide GraphQL service for part of our 'Storage Engine'.

In a previous project, I used [Sangria](https://sangria-graphql.github.io/) a Scala Graphql implementation but since we
are more of a Java/Spring-boot shop, we decided on using the [DGS Framework](https://netflix.github.io/dgs/) which has
been open-sourced by Netflix early 2021.

DGS is built with SpringBoot & [graphql-java](https://github.com/graphql-java/graphql-java.) & support the Spring Boot
Reactive stack aka Webflux .

## Observability

All of our Spring Boot services have distributed tracing enabled
via [spring-cloud-starter-sleuth](https://spring.io/projects/spring-cloud-sleuth).
Spring Cloud Sleuth is a distributed tracing framework for Spring Boot and provide instrumentation on many
libraries : [available instrumentation](https://github.com/spring-cloud/spring-cloud-sleuth/tree/3.1.x/spring-cloud-sleuth-instrumentation/src/main/java/org/springframework/cloud/sleuth/instrument)
.

For example in this project, we are using Webflux, r2dbc-postgresql([https://r2dbc.io/](https://r2dbc.io/)) and
transaction management which have been instrumented in Sleuth.

*Aparté (Aside)*

(not the cool bear you're looking for)
> I rely heavily on distributed traces to analyse bug/performance issues.
>
> In production these traces are sent to Jaeger, for local dev depending on the number of event generated and the
> sampling rate,
> I use Honeycomb or Grafana Loki/Tempo (via docker-compose)

# Problem

I wrote my integration tests for each use case and implemented the first graphql query.
All the tests were failing (404 _not a good start_).

In these integrations test, I was calling the server with GET request since that is how I was writing them for my
Sangria project.

Seems like DGS is not serving GET request.

List of http endpoint exposed by the service :

![list of http endpoint exposed by the service](/img/dgs_endpoint.png)

Route definition in DGS :
![kotlin code showing the definition of the /POST route for path /graphql](/img/dgs_route.png)

Switching to POST request, I was getting different errors (graphql-errors).

I enabled the tracing in the integration tests to check if the different services were being called.
> Enabling tracing integration test helps me visualize all the different use case, it is much faster than debugging each
> test to see if it's calling the method A or B
>
There was something strange in these traces, there was no information about the graphql query processed by the engine
nor the trace of a call to the engine.

I could only see the call to POST /graphql and the call to the database.

_(I did find the cause of the failing test and fix it)_

# Research

With a quick search on the DGS documentation,
we have
more [information](https://netflix.github.io/dgs/advanced/instrumentation/#adding-instrumentation-for-tracing-and-logging)
there is no default tracing implementation for the GraphQL engine.

We must implement it by extending the SimpleInstrumentation from the graphql-java library.

```java

@Component
public class ExampleTracingInstrumentation extends SimpleInstrumentation {
}
```

_Internal monolog_

_Hmm maybe someone already did that for Sleuth_

I checked the issues and the list of available instrumentation in the Sleuth repository. I didn't find any .

I searched on google nothing...

_Maybe someone else did it for another tracing framework, Open-telemetry(Otel) ?_

_hmmm maybe I should check that._

I searched 'opentelemetry graphql java' and found there was a maven artefact.

![Search result](/img/search-opentelementry-graphql-java.png)

I checked the GitHub repository and found the instrumentation code for graphql-java
[https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/graphql-java-12.0](https://github.com/open-telemetry/opentelemetry-java-instrumentation/tree/main/instrumentation/graphql-java-12.0)

# Solution

Now I have an instrumentation for the GraphQL engine, I should be able to use it.

_Not so fast_, I need the instrumentation code to be compatible with Sleuth instrumentation & API.

_I could write a bridge open-telemetry to Sleuth ?_

_Bad idea!_

Thankfully this work is in progress by the Spring Cloud Team, but it's in the experimental
repository [https://github.com/spring-projects-experimental/spring-cloud-sleuth-otel](https://github.com/spring-projects-experimental/spring-cloud-sleuth-otel)
.

I should find another way, instead of creating a bridge I should translate the instrumentation code written with
Open-telemetry Api with Sleuth API.

The apis are not identical since Sleuth historically has inherited some design from OpenZikpin/Brave library /
OpenTracing era and Open-telemetry being the new standard.

The implementation is available bellow:

<script src="https://gist.github.com/W4lspirit/0844f010bafd0f065866892d14a6172c.js"></script>

![Trace with information about graphql query](https://user-images.githubusercontent.com/13579472/176879927-ead26414-74b6-47ac-8060-cedfc12a82a4.png)

# The end

Just wanted to share a little about Graphql Java Instrumentation.
