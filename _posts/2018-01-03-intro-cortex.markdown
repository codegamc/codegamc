---
layout: post
title:  "Introduction to Cortex"
date:   2018-01-02 19:32:45 -0500
categories: intro cortex
---

# Introduction 

Over the past year, I have been working on a series of projects as the foundation for future Automation + SmartHome projects.  The goal was creating a hub or framework that would enable me to easily create future projects.  Ideally, this framework would have some sort of event system (eg. when the door opens, trigger this function), and some sort of knowledge of state (eg. the light is currently on). Anything additional to those would be nice-to-have but less critical. Other goals, but secondary goals, was a focus on making it distributed across devices (eg. my Pi's and Desktop) and an easy 'plugin' model, so I can create new functions for it (eg. new SmartHome devices, Twilio connectivity, etc).

### The History

Last January, I started this project as a Python webapp.  The goal was creating HTTP endpoints that cooresponded to trigger functions (eg. /Devices/DeskLight/TurnOff), and dynamically load python modules that listen for those endpoints. I quickly ran into limits with this, since HTTP is a heavy protocol, especially for microcontrollers and small devices, and the architecture was limiting the use for complex automation tasks (eg. if its the morning, when I leave, turn off the lights).  This led me to rethink the project to use software defined events, where after a quick python prototype, I transitioned to using Nodejs's EventEmiiters.  This allowed me to chain together complicated flows of events.  This model showed its limits when I tried to add new functionality, and had to completely stop the entire thing and restart it (since my architecture did not allow easy hot-swapping or dynamically changing the code). It was at this time that I was taking a robotics class and learned about The Robotic Operating System ("ROS"). ROS works by runnning a broker, called the `roscore` that handles HTTP messagaes to and from nodes, which can publish events, subscribe to events, offer "services" (request - reply messages), as well as several other functions.  This platform was very similar to what I was looking for, except ROS can only have one core, on one machine, and I found the build process for ROS to be cumbersome and slow. I used ROS as inspiraton for my newest rendition of the project, and the final architecture.

### The Solution 

Like ROS, I decided that using a "core" and individual nodes would offer me the flexibility I needed to add and remove processes without stopping the whole thing.  Additionally, I addopted the Pub-Sub and Service model from ROS, giving nodes the following actions: Publish, Subscribe, Create Service, Make Request to Service.  This model allows me to easily create nodes on the system which can respond to events, as well as expose triggers. Meanwhile, the core would track what events are being listened on or published to, and what services are available.  For a transport medium instead of HTTP, I decided to use Nanomsg, a light weight messaging library that can operate across a variety of transports.  Fortunately IPC is one of those transports, allowing Cortex to bypass any networking overhead for same-device nodes.  To handle nodes on different devices, I decided to have Cores work as routers, passing messages from local nodes on one devices, to local nodes on the other.  This functionality is not yet implemented, but will use simple TCP. This adds the overhead of a core for every devices, but since I expect only a few devices with many nodes, it was a tradeoff I thought was worth it. 

### Implementation details

I implemented this core and node framework using Nodejs and Typescript. Nodejs has an amazing asyncronous event loop, which is perfect for this type of event-driven work, and Typescript keeps large projects organized with a simple, out of the way, type system.  As I mentioned above, I used Nanomsg as the message transport protocol. Nanomsg has some cool features, like the different trasnports it allows, and the ability to classify the sockets as "Bus", "Req/Rep, and "Pub/Sub".  I made heavy use of the Bus and Request Reply methods. The events are IPC Bus's, allowing any node to write or read a message on the bus, meaning that the core is not required to be a broker, it simply creates the Bus for any event requested, and manages that IPC channel.  

You can find the code for these projects on (gitlab)[gitlab.com/michaelccodega].

# Project Status

The project currently "works".  It is very alpha software, but with errors that occur in most of its functions, but it generally can run and operate, with supervision. I am currently taking a break from writing the functionality in order to write out a test suite for lib-cortex (the sub-project that contains the node, as well as supporting classes) and for the core.

# Plugins in progress

Since I have several projects under progress (I'll save that for another post), I have been writing several plugins. These include a plugin to subscribe to MQTT topics, and convert them to events, and a Twilio service to allow any node to send texts.  Additionally, I am writing a plugin that accepts webhooks, primarily for the accepting texts from twilio, but functional for other webhook based services.

# Problems to Address

I may look to change the request-reply sockets - I currently use them for the Services.  The problem that I ran into with the Req/Rep sockets stems from the blocking nature of the socket in Nanomsg. When a socket gets a request, it does not expose the return address, but the next message sent on that socket goes to the address of the last request, unless that address already got a response. I have not tested how this works under load, but if a future system has long waiting async calls to make before replying, errors may occur if that socket recieved additional messages. While a simple test may indicate that nanomsg handles this behavior gracefully, I plan to implement message routing in the core to allow messages to be fowarded from core to core, which could dramatically change the reply time of messages, and will require messages be decoupled from the socket anyways. I currently plan to overcome this by using IPC channels encoded with the node's handle, a uniquely identifying string for each node, to work as a mailbox for messages sent to a node.  This would require some means to understand what is a response message, and what it is replying to.  This could be solved with some message lookup table, which I plan to use, but maintenance of that table would likely have its own challenges. 