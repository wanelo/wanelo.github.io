---
layout: default
title: "Decoupling Distributed Ruby applications with RabbitMQ"
author: Eric Saxby
author_username: sax
---

A lot of people might be surprised to hear how long it took us to stand up our message bus infrastructure at Wanelo. One of the downsides of focusing on iteration is that fundamental architecture changes can seem extremely intimidating. Embracing iteration means embracing the philosophy that features should be developed with small changes released in small deployments. The idea of spinning up unfamiliar technology, with its necessary Chef code, changing our applications to communicate in a new way, then production-izing the deployment through all the attendant failures and misunderstandings seems... dangerous.

Fortunately "Danger" is our middle name at Wanelo. (Actually, our middle name is "Action." Wanelo Action Jackson). Having now developed using a messaging infrastructure for almost a year and a half, I would no longer develop applications in any other way.


## Asynchronous work vs. decoupled messaging

We’ve used Sidekiq for asynchronous work since the beginning of Wanelo, and for a long time it dominated our understanding of application messaging. For that reason, I’d like to take a brief detour to touch on the difference between asynchronous workloads and messaging.

In Sidekiq (and other background worker frameworks such as Resque) when asynchronous work is required by an application, it possesses explicit knowledge of the code that will execute the work. Event X occurs, so the application adds a message to the queue for Worker X. When Event Y occurs, the application adds a message to the queue for Worker Y. Despite the fact that workers run in separate processes, they are tightly bound to the rest of the application… they often use the same models, and they usually talk to the same databases. They share the same configuration, and are almost always deployed at the same time…

Tools such as Sidekiq *are* often used for purposes that might not depend on application models, such as sending emails or push notifications. They may also be deployed in a way that separates the producer code from the consumer code. We should not be confused into thinking that message producers and consumers are decoupled from each other, however. Each message is consumed by a single worker. When a message is produced, it must specify the queue and **class** of the consumer. If multiple consumers care about a message, the producer must direct a message to each one. *The producer knows about every consumer.* When adding a new consumer, the producer needs to change to accommodate this. When refactoring the class name of a consumer, the producer code must necessarily change.

When using a message bus to send messages between applications, the producers should not know about the consumers. When our API endpoint notifies our message bus about Event X, I can create a new application that listens for Event X without making any changes to our API code.


## RabbitMQ

RabbitMQ is a message bus written in Erlang. Packages are available on most platforms, and it has been run in many high traffic deployments.

RabbitMQ provides a lot of flexibility with regard to the topology of its deployment and the configuration of different types of exchanges. We’ve found the documentation to be clear and the tutorials to be very enlightening, but in our current deployment we’ve made a few choices that might be helpful for others to consider. I’m sure that some will disagree and suggest different choices, but so far this works very well for us.

### Multiple independent instances

RabbitMQ allows for topologies where some instances serve as primaries and others as replicas. Instances can be configured differently to distribute disk I/O and memory load in ways that allow for different reliability guarantees.

For the near term, we’ve chosen to stand up multiple independent instances of RabbitMQ proxied by HAProxy. This removes the need for instances to communicate with each other and simplifies many possible failure scenarios. Producers send messages to HAProxy on localhost, and do not ever receive producer acknowledgements. When we deploy message consumers for an application, we configure a duplicate set of consumers for each RabbitMQ instance. These consumers talk directly with RabbitMQ without a proxy.

When a RabbitMQ instance crashes or its server reboots, producers receive an error and reconnect via HAProxy, attaching to a different instance. Since TCP connections in HAProxy are long-lived, we can expect that processes will stick to a single RabbitMQ instance until they reboot or suffer a connection error. For this reason, if a single RabbitMQ instance restarts we expect it to see negligible load until our message producers restart.

Durability of messages will be done at the queue level, rather than relying on replication. Messages that have been successfully published to the instance but not consumed will have to wait for the instance to recover and consumers to reconnect. This is a risk that we are happy to accept in exchange for the simplicity of the setup.

The primary failure scenario that we anticipate is the possibility that messages may be produced faster than they can be consumed. If this were to happen, then our RabbitMQ instances would become memory bound, at which point they would page out memory to disk and suffer a massive performance degradation. We've avoided this situation through a few solutions. First, we over-provision our RabbitMQ cluster in terms of memory. They're still comparatively small and cheap compared with other of our systems, and the benefits outweigh the financial cost. Second, we do as little work as possible in the consumer code, to ensure that they are as fast as possible. I'll get into this later. Third, we have very aggressive alerting based on queue depth. The only time that queues should back up is when we're restarting our consumer daemons. So, while our Sidekiq queues might back up to thousands or tens of thousands of messages before we alert, a few hundred backlogged messages in a Rabbit queue will trigger a critical alert.

A long term concern is that when a message matches multiple queues, it is written into each queue. As queue and message counts increase, the resource requirements of our RabbitMQ infrastructure can not be expected to scale linearly... we'll need to watch out for this, though for the near future it's not a problem.

### Topic exchanges vs fanout or direct

When messages are published to RabbitMQ, they are published to an exchange. RabbitMQ allows for several different types of exchanges, each of which provides different behavior. For instance, if an exchange is defined as `fanout`, then each message published to the exchange will be sent to each queue in the exchange. Your message literally fans out to every consumer listening on the exchange.

We exclusively use topic exchanges for the purposes of inter-application messaging. Messages published to a topic exchange include a routing key, in the dot-delimited format, `arbitrary.routing.information`. When a queue is registered in the exchange, bindings are registered matching one or more routing key. For instance, a queue can be bound to the routing key `arbitrary.routing.*` or `arbitrary.#`. The `*` is a wildcard character matching a single word. The `#` wildcard matches 0 or more words. So, a queue could be bound to `abitrary.routing.*` or `arbitrary.#` to match the example routing key. `arbitrary.*` would match `arbitrary.stuff` but not `arbitrary.things.and.stuff`.

We formed the opinion that topic exchanges best suit our needs because of our strong opinion that producers should not know about consumers. With a fanout exchange, in order to target a message to multiple sets of consumers, a producer would need to send a message to multiple targeted exchanges (thus demonstrating implicit knowledge of its consumers). With a topic exchange, the producer sends a message to a single exchange, using a routing key to signal the type of message. If successful, the producer can reasonably expect that the message has been sent to any listening consumers. If no consumers care about the message, then RabbitMQ will discard the message and return a success response to the producer. This is a good thing, as it allows us to deploy new producer code before we finish writing consumer code, without worrying about filling up memory on the RabbitMQ instances.

### Durable queues

When a consumer process starts, it declares a queue in which messages are retained, and the bindings that determine which messages are placed on the queue. For a topic exchange, the queue bindings include the routing key pattern matcher as described above. After a queue is created, many properties are read-only. Durability is one of those.

Since we do not use replication to ensure persistence of messages on restart, we use queue durability as one way to increase reliability. If RabbitMQ tells a producer that a write was successful to a durable queue, we trust that it has been written to disk. Our public cloud of choice has ZFS local storage with battery-backed write log devices—this gives us a reasonable guarantee that when the file system receives a write, it can be read back later even on an unexpected restart. More than enough guarantee for our use cases, and we can bolster our confidence by including retry logic and data consistency checks within our applications.

### Fast consumers, asynchronous workers

Our primary concern with consumer daemons is to process messages from our message bus as quickly as possible, so that we never run out of memory on the RabbitMQ instances. This informs the work (and lack of work) done directly in our message bus consumer daemons.

When we deploy a new queue, we configure our consumer daemons to forward each message to one or more classes. This configuration lives in the consumer application. In almost all cases, the classes that process message payloads are Sidekiq worker classes. This dramatically reduces the work done directly in the RabbitMQ consumer processes. All that they do is receive a message, write its payload to the relevant Redis instance for the target Sidekiq worker.

This separates the risk of queue overloads into two buckets: application-specific and platform-specific. If a Sidekiq Redis queue is backed up, that is application specific and can be scaled by the individuals or teams responsible for that application. If our RabbitMQ queues are backed up, then we have a larger problem, often with a higher priority.

This also allows us to use RabbitMQ in a specific capacity that it is very good at: inter-application messaging. We can use Sidekiq for what it is very good at: doing asynchronous work in an application-specific context.


## Comparison with other message buses

There are a number of options when trying to find the right message bus for you. Here is a non-exhaustive list of a few that we've looked at.

### Kafka

Many of the concerns, pros and cons that I've written about can be applied to Kafka. One major difference between RabbitMQ and Kafka is that the latter was written with extremely high throughput in mind from the very beginning. The listed benchmarks show it to outperform RabbitMQ once you get to the point of producing millions of messages per second.

A Kafka cluster is deployed with a pre-defined number of partitions, and with a (modifiable) replication factor to allow operators to configure their reliability guarantees. The number of processes in a consumer group must equal the number of partitions in the Kafka cluster. Partitions are configured to expire messages after a certain amount of time, allowing consumers to rewind if desired. The current location of a partition consumer in its partition is stored in Zookeeper, as is other real-time information about the cluster.

So... why did we not use Kafka?

For our current purposes, RabbitMQ is more than able to keep up with the scale of our message bus use. We may not have the reliability guarantees of Kafka, nor the available message throughput, but we trade that off by having a much simpler infrastructure with understandable failure cases. We also appreciate that we can scale reads on the fly by adding more consumer processes, whereas with Kafka we would need to deploy a new cluster with a different partition count.

Even if in the future we deploy a Kafka cluster, RabbitMQ will probably stick around in some capacity. There's no reason not to have both, and we appreciate how little continuing oversight our current setup requires.

### Amazon SQS & SNS

There are a few major differences between RabbitMQ and SQS. One of the biggest differences is that messages are pushed to RabbitMQ consumers, whereas SQS consumers must poll for messages. This is likely only important when setting up the messaging infrastructure, as once the initial development cost is made in writing and deploying consumer processes it should be fairly transparent to developers.

Another difference with SQS is that the queue is chosen by the message producer. Using only SQS, if a message must target multiple consumers the producer code must be configured to write to each queue. To us, this feels like tight binding between producers and consumers. While SQS can be wired up to SNS to reduce the coupling, it's a little more complicated.

The real reason we did not view SQS as a viable solution for us is that we run our code in the Joyent Public Cloud instead of AWS. Now that I have operated RabbitMQ for the time that I have, I might choose to do so again even for an AWS deployment. Familiarity breeds fraternity... or friendliness... or something.

### ZeroMQ

To be honest, we didn't even consider ZeroMQ. We were specifically looking for a brokered solution.


## What can go wrong?

Nothing could go wrong, right? It's all rainbows and unicorns on the message bus!

No, wait, those are HTTP servers.

There are a number of problems that we've run into. For the most part they were based on misunderstandings, but they've led to a few policies that we follow, in some cases automatically via code frameworks.

### Pattern matching

`*` matches a single word in a routing key. It's very clear in the documentation, but we still got caught out a few times when we should have used a `#` but accidentally typed `*`.

One of the features of RabbitMQ that we like is that if no queue bindings match a given message, it is discarded with low overhead. This gives us a lot of leeway in developing new features, as we can deploy producer code before we have consumer code ready to receive messages. The downside is that a simple typo can go uncaught, when consumers are mis-registered and the expected messages instead disappear into limbo.

We've solved this by writing middleware in our message bus consumer daemons and our Sidekiq processes that emit Statsd counters for every message. It's easy to know whether a consumer is actually receiving messages; we just look in our Graphite dashboards. This still relies on a human to double check when deploying new features, but it provides considerable help with post-mortem debugging.

### Changing queue properties

Once you define a queue in RabbitMQ, you're stuck with its properties. Want a durable queue, but forgot to set it on queue creation? Unfortunately, that queue is permanently non-durable, even if you've already changed your code and everything now appears to be working. The only way forward is to create a new, durable queue, and delete the old queue.

### Renaming queues

Same problem as changing queue properties. You need to create a new queue with a new name. Oh, and while you're doing this, remember that you do actually have to delete the old queue.

Once you have a queue with bindings that match real messages, those messages are going to be written to the queue. If you switch over your consumers to a new queue, but forget to delete the old one, it will start filling up. This is one of the reasons our alerts are so aggressive about queue backlogs... a queue depth of 500 can very likely mean that someone forgot to delete something, or accidentally changed a queue name in a yaml file without meaning to.

### Naming queues to avoid confusion

One thing we've found is that once the infrastructure is there, it's very easy to build new features on top of our message bus. After a few months of adding code without forethought, it's also easy to accidentally collide on queue names. The routing key of your message is `product.created`, so why not name your queue `product.created`? Do multiple applications care about `product.created`?

Naming queues based entirely on purpose can also obfuscate the location of consumer code. You've just received a critical alert saying that the `product.created` queue is backed up. What application owns that queue?

We've found that it's much easier to name queues based on our consumer applications. Instead of `product.created`, we'll name our queues `product-feeds.product.created` or `website.product.created`. This greatly reduces confusion, and renders queue name collisions extremely unlikely.

### Consistent grammar in routing keys and queue names

How many times have we created a queue listening to `product.create` only to realize a day later that the actual routing key is `product.created`? More often that I would like to admit.

Consistent grammar can save a lot of wasted effort. Decide a few rules in advance, and keep the rules in a wiki or README that your developers will stumble upon repeatedly to help remind them.

* Resource names: singular or plural?
* Action name: present tense or past tense?
* Data source: Should routing keys include this information? Will it add to or reduce confusion? Does the message payload even make sense without it?

Note that this concern also applies to naming of RESTful API endpoints and webhook payloads. Inconsistent naming not only wastes your time, it can waste the time of other products consuming data from your services.
