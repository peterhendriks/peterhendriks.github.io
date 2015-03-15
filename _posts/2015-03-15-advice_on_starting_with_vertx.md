---
title:  "Advice on starting with Vert.x"
date:   2015-03-15 22:15:00
categories: java
---
[Vert.x](http://www.vertx.io) is a Java based platform to run serverside programs. It is very similar to Node.js, and very different from Java Enterprise application servers.

Although Vertx has some good documentation to get you started, there are a few pitfalls the documentation does not make you aware of. In this article, I'll
highlight a few I ran into myself in the past few months of working with Vertx in a real project.

Verticles, Modules, Containers and Clusters
===========================================

Vertx is pretty flexible in splitting up code in several ways, following a Russian Doll pattern. Although all the details are documented, no overview is available.
This will likely confuse you as you move beyond deploying your first Verticle. This is a good way of looking at the different dolls, starting from small to big:

 * **Verticle** will perform actual work, by listening to events on the event bus, or opening an HTTP socket, or doing other things. Note that a Verticle may also deploy other Verticles or Modules in its Container.
 * **Module** provides an isolated classloader context for all Verticles started in this module, and may contain additional bundled dependencies and metadata for ease of deployment.
 * **Container** a runtime JVM process of Vertx. Typically constructed by start Vertx from command line: e.g. vertx runMod mymod.
 * **Cluster** a Container may be configured to be considered part of a Cluster. If it is, the event bus events will be connected across containers, allowing Verticles to communicate with Verticles in other Containers.

Vertx also encourages the following (but also allows other models):

 * Deploying one Verticle per Module.
 * Running one Module per Container.
 * Deploying a Module is recommended over deploying a Verticle directly, mainly because it is much easier to manage dependencies.
 * A Verticle should be focused on a specific task.

In practice, if you can constrain yourself to this one Verticle to one Module to one Container, this makes for a very simple model.
Just run everything in different Containers and configure at least a Cluster for the local machine to connect everything together.

Threading and pooling
=====================

Threading in Vertx is pretty straightforward, but again, it is good to get a bit more overview before diving in the docs. One of the staples of Vertx is to encourage "non blocking" coding styles. In typical Java programs,
for instance, the thread is blocked while waiting for external dependencies like the file system, database or external services. By not blocking, less threads are needed to process the same user load. Since threads
are relatively expensive, it means in practice that you'd be able to service more users at once with the same hardware.

 * **Verticle** This is considered to be safe, non-blocking code for directly executing event handling on. For each Verticle instance, only a single thread will ever execute event processing on this instance.
 * **Worker Verticle (not multi-threaded)** Worker Verticles are for legacy code that needs blocking. For each Worker Verticle instance that is not multi-threaded, a worker thread will execute event processing one at a time. Unlike a normal Verticle, this thread may be a different one over time.
 * **Worker Verticle (multi-threaded)** For blocking code which requires a lot of shared state. A single Worker Verticle instance that is multi-thread may be executed by multiple worker threads at once. Be prepared to think about multi-threading in your code again!

For tuning, you need to think about Verticle instance count and the thread pool affected. Although this will be wildly different for each application and you will need to tune yourself, the following basic rules do apply:

 * Vertx has two thread pools: the "event loop pool" (for normal Verticles) and the "background pool" (for Worker Verticles).
 * A thread pool is shared across all Verticles in the Container. This is an important detail if you deploy multiple Verticles in a single Container.
 * A thread pool will determine the maximum amount of parallel processing. There is no "overflow" where more threads are made on demand.
 * The event loop pool is set to 2 times the number of cores in the machine by default. If you have truly non-blocking code, it does not make not much sense to set this to a higher value. If you run many Containers on the same machine, you may want to lower this to prevent starvation issues.
 * The background pool will be set to 20. In my experience, if you need a worker verticle, this is a pretty low number and you may need to raise it.
 * For Verticles and Worker Verticles (not MultiThreaded), the instance count should be lower or equal to the pool size. The lower number of the two determines for a Verticle the amount of parallel processing possible.
 * For a Multi-threaded Worker Verticle, setting the instance count to a higher number than 1 usually does not make sense, as the MTWV upper limit of parallel requests is limited to the background pool already.

Error handling
==============

Something Vertx does not provide in its documentation is some consideration on how to do error handling. Vertx, by default, does not render a response to the receiver when an error occurs. Any uncaught Exceptions will be
propagated to the Vertx log, but the client of the event that is waiting for the response will never receive any response. At best, this results in an unnecessary timeout at the client. Vertx, however, has default API
that does not explicitly timeout. When you never receive an answer, the handler waiting for a response will never be called or cleaned up, resulting in a memory leak.

By now, you should have become rightfully scared about dealing with Exceptions correctly in Vertx. Do not trust the example code on this! A few practical considerations:

 * Prefer to use Vertx operations with a timeout. This way, you get exceptions as you timeout or messages cannot be delivered, allowing much more options on how to deal with these problems yourself.
 * At least set the "default timeout" on the eventbus of each Vertx Container. This enables a global timeout that is used for Vertx operations without a timeout, preventing memory leaks and at least provides some logging on when things went wrong.
 * Always return an error reply back to the client. This allows the client to fail faster, and allows proper logging of why something went wrong.

There is some more useful things to say about error handling in non-blocking code in general, but I'll save those for another post. For now, happy coding with Vertx, and let me know what other useful tips you may have!