+++
title = "Racket-MUD 0.1.0 Development Log"
author = ["emsenn"]
draft = false
+++

(**_Note:_** This document was converted from Pollen/Racket to Org-mode and then HTML via Hugo. There's probably missing formatting and formatting errors.)

Milestones:

-   [X] A function that calls tick() until broken, after loading and starting the engine.
-   [X] A test service that is loaded, started, and run (ticked).

This document is the first in what I plan to be a series of documents where I "literately program" a MUD engine implemented in Racket.

Each document will have a list of milestones to be achieved in that version, and then some discussion about how I plan to implement it, and then finally, the narrative of implementing it.

(This is my first attempt at programming something in this sort of log-driven development method, and I don't know much about Pollen or Racket, the languages I'm using to document and implement what I'm doing. So, bear with me!)

The goal is for this documentation - in addition to being the source of the source code - to be a useful resource for people wanting to learn the reason why procedures and data structures exist the way they do, which might help in changing them in the future. And more generally, it might be of interest to someone to watch how someone does this.

So, Racket-MUD - which I'll usually just call "the MUD engine" - is my implementation of an LPC-style MUD in the Racket language. It'll follow how my Python 3 MUD, qtMUD, worked, as much as it can, which is with a tick-based central engine that triggers events and runs services.

I'd prefer to stay away from jargon, but it's kind of inevitable with a project like this, so let me just get some of it out of the way:

-   A MUD is a "multi-user dimension" (or "dungeon" if you're old,) and it's a type of server that specializes in accepting text commands (like a command line) and sending back text descriptions.

-   The "tick" is the basic unit of game time. Each tick, every scheduled event is triggered, and every service is told to tick.

-   "Events" are procedures (functions) which work with in-game data.

-   "Services" are things that handle continously-active things like interfaces (the MUD socket or web servers, for example.)

-   "Things" are the data structure of in-engine objects.

(Note: I'll switch between "in-game" and "in-engine" and similar terms often; they mean the same thing. MUDs are traditionally games, but they don't have to be.)

So, the engine starts up, populates a list of services, and ticks: ticking each service and calling each scheduled event. This implies a way for events to be scheduled.

Things - working with in-game data, can probably come in version 0.1.1. For this version, I'm going to focus on the ticking of the engine.

It sounds like "service" should be one datatype.

```pollen
◊(struct service/0.1.0 (id load-proc start-proc tick-proc stop-proc))
```

It needs an ID to be referenced by, and then references to its various load, start, tick, and stop procedures.

So for a test service,

```pollen
◊(define (load-test-service/0.1.0) #t)
◊(define (start-test-service/0.1.0) #t)
◊(define (tick-test-service/0.1.0) #t)
◊(define (stop-test-service/0.1.0) #t)

◊(define services-to-load/0.1.0 (list
  (service/0.1.0 "test-service"
    load-test-service/0.1.0
    start-test-service/0.1.0
    tick-test-service/0.1.0
    stop-test-service/0.1.0)))
```

The goal of the main engine's load procedure is to then, for each service, call its load- function. After playing around in the REPL, I've got:

```pollen
◊(define (load-services/0.1.0 services)
  (map (lambda (service)
    ((service/0.1.0-load-proc service))) services))
```

So the MUD engine's load service would be:

```pollen
◊(define (load/0.1.0)
  (load-services/0.1.0 services-to-load/0.1.0))
```

Simple enough. Oh - but what if a service doesn't have a load procedure? (Or more likely, start, but we'll improve our load first.)

```pollen
◊(define services-to-load/0.1.1 (list
  (service/0.1.0 "test-service"
  #f
  #f
  tick-test-service/0.1.0
  #f)))
```

Here we have a version of the services-to-load list, where the test-service only has a tick function. Testing this with "(load-services/0.1.0 services-to-load/0.1.1)" in a REPL gets us an error that #f isn't a procedure. (I'm enjoying, so far, the suffix of micro-versions on this stuff.)

```pollen
◊(define (load-services/0.1.1 services)
   (map (lambda (service)
     (let ([load-proc (service/0.1.0-load-proc service)])
       (if (procedure? load-proc)
	 (load-proc)
	 #f)))
     services))
```

Now running "(load-services/0.1.1 services-to-load/0.1.1)" gets me "'(#f)". Alright!

We'll need a similar function for starting and ticking each service. It should be assumed that down the line, each of these will be significantly more complex, but for now, this'll get the job done.

```pollen
◊(define (start-services/0.1.0 services)
  (map (lambda (service)
    (let ([start-proc (service/0.1.0-start-proc service)])
      (if (procedure? start-proc)
	(start-proc)
	#f)))
    services))

◊(define (tick-services/0.1.0 services)
  (map (lambda (service)
    (let ([tick-proc (service/0.1.0-tick-proc service)])
      (if (procedure? tick-proc)
	(tick-proc)
	#f)))
    services))
```

And, corresponding procedures for the MUD. (We'll have these use services-to-load/0.1.0 instead of /0.1.1, since it has real (fake) procedures.

```pollen
◊(define (start/0.1.0)
  (start-services/0.1.0 services-to-load/0.1.0))
```

When we "run" the MUD engine, we want to look for every service with a tick procedure, and add it to a hashtable of tickable services, where the key is the service ID and the value is the service tick function. We'll also want to call the tick procedure for the first time.

```pollen
◊(define tickable-services/0.1.0 (make-hash))

◊(define (run/0.1.0)
  (map (lambda (service)
    (let ([tick-proc (service/0.1.0-tick-proc service)])
      (when (procedure? tick-proc)
	(hash-set! tickable-services/0.1.0 (service/0.1.0-id service) tickproc))))
    services-to-load/0.1.0)
  (tick/0.1.0))

◊(define (tick/0.1.0)
  (map (lambda (tick-proc) (tick-proc)) (hash-values tickable-services/0.1.0))
  (tick/0.1.0))
```

So, now we have:

-   load, start, run, and tick procedures for our MUD.
-   a test service with load, start, and tick procedures.
-   a tick procedure that calls the tick procedures of specified services.

That's all that I specified as needed for version 0.1.0, so I'm going to put a bow on this.

```pollen
◊(define service service/0.1.0)
◊(define load-test-service load-test-service/0.1.0)
◊(define start-test-service start-test-service/0.1.0)
◊(define tick-test-service tick-test-service/0.1.0)
◊(define stop-test-service stop-test-service/0.1.0)
◊(define services-to-load services-to-load/0.1.0)
◊(define load load/0.1.0)
◊(define load-services load-services/0.1.1)
◊(define start-services start-services/0.1.0)
◊(define tick-services tick-services/0.1.0)
◊(define start start/0.1.0)
◊(define tickable-services tickable-services/0.1.0)
◊(define run run/0.1.0)
◊(define tick tick/0.1.0)
◊(provide
  service
  load-test-service
  start-test-service
  tick-test-service
  stop-test-service
  services-to-load
  load
  services-to-load
  load-services
  start-services
  tick-services
  start
  tickable-services
  run
  tick
```

(I'd love to automate that mess!)

In version 0.1.1, I'll be adding functions for creating basic things, in preparation for making a proper service: a MUDSocket server.

(I don't need "things" to work until after the MUDSocket is able to accept connections, but I'll be happy I have them in place when the time comes.)

Oh - and in a version between things and the MUDSocket, I'll have to add a proper "stop" to the engine, so that it'll be able to be told to let go of the ports it claims when I shutdown the engine.