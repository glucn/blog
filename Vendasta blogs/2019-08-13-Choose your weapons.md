# Choose your weapons

*Originally posted at https://vendasta-blog.appspot.com/blog/BL-LKTZ33DP/*

In a dark dungeon, we will want to use a divine hammer to knock down skeletons and use a silver sword to deal with devils.

We also have a lot of kinds of weapons in our arsenal to deal with message passing and asynchronous workflow: Odin, Cloud Tasks, Cloud Pub/Sub, Cloud Scheduler, Event Broker...(not to mention Fantasm, vpipeline, etc. of the good old times)

In this blog, I want to share some of my understanding of all these weapons.

## TL;DR
Here are some key features that I feel important when choosing weapons:

- Message passing

    - **Cloud Pub/Sub**: The publisher of a message doesn't care about the consumers; the message will be processed as fast as possible without too much control options; high throughput.

    - **Cloud Tasks**: The publisher of a message needs to know the consumer; we can control when the message will be delivered or how to retry.

    - **Event Broker**: In-house tech; we can join messages from different sources, and filter/replay messages

- Asynchronous workflow

    - **Cloud Tasks**: We can defer a task in the workflow to a certain time with Cloud Tasks.

    - **Odin Pipeline**: In-house tech; Pipelines will be processed as fast as possible; Handy UI.

    - **Cloud Scheduler**: It's cron; it's the best fit to kick off a workflow with a certain interval (daily/monthly etc.).


## Google Cloud Pub/Sub
- The official doc: https://cloud.google.com/pubsub/docs/
- gosdks: https://github.com/vendasta/gosdks/tree/master/pubsub

As the name implies, pub/sub decouples the publisher and the subscriber of the message.

When we publish a message, we just need to know the topic and the message. It's like we are speaking aloud in a dark room, the room might be empty or full of ghosts. The publisher knows nothing about the subscribers/consumers, so when we want to add a new subscriber or remove an existing subscriber, we don't need to update the code on the publisher side.

The subscribers can register the subscriptions as Push or Pull. The push subscription is like Google making an HTTP POST call to a specified endpoint with the message. With our gosdks, a subscription will be registered as a Pull subscription, where we should have some workers that receive messages from the topic and Ack/Nack the message after processing the messages. Each the subscribers under a shared subscription name will receive a portion of all the messages, it's good for load balancing when there are multiple pods running, but please don't be as stupid as I was to have the same subscription name on prod and demo environments.

Pubsub can guarantee that all messages will be delivered at least once, but messages can be delivered multiple times or out of order. This is something we must be very careful about.

## Event Broker
- SDK: https://github.com/vendasta/event-broker/tree/master/sdks/go/v1

This is our in-house technology of message passing.

To me, Event Broker feels like pubsub using Push subscription. The publisher (who emits events) does not need to care about who will consume the events or how the events will be processed. Event Broker will save the event in vstore, react to vstore pubsub to kickoff Odin pipeline which sends out messages to subscribers.

I personally haven't used Event Broker a lot, but I feel the neatest features in Event Broker are, when we subscribe to an event, we can filter messages with certain conditions, or join messages send asynchronously by different sources.


## Google Cloud Tasks
- The official doc: https://cloud.google.com/tasks/docs/
- gosdks: https://github.com/vendasta/gosdks/tree/master/taskqueue

Dustin has been asking people to move away from the to-be-deprecated pull queue for a while, so I don't want to talk about pull queue anymore here.

Cloud Tasks is a great tool to pass messages asynchronously from one service to another, and it can also be used to drive workflow with the scheduler/worker pattern that is described in [Aaron's blog](https://vendasta-blog.appspot.com/blog/BL-NMVZWZM9/).

Google has a comprehensive [article](https://cloud.google.com/tasks/docs/comp-pub-sub) comparing Cloud Tasks and Pubsub. I won't write it out again, but my key takeaways from that article are that (1) Cloud Tasks couples the publisher and the consumer of a message to some extent (2) Configurable retries, scheduled delivery, and explicit rate controls are the features that Cloud Tasks has, but Pubsub doesn't.


## Odin Pipeline
- odin: https://github.com/vendasta/odin (might become an opensource tool at some point)
- vOdin: https://github.com/vendasta/vodin (odin with Vendasta's flavors, like initializing vbigtable and vstore input reader, etc.)

Odin is also one of our in-house technologies. Essentially, it uses bigtable to store the state of pipelines and uses pubsub message to drive the pipelines moving forward.

It worth writing a whole separate blog post to talk about details in using odin. Here I just want to touch a few key points that I feel important:

- When we return a non-nil error from a "step", which is the function called with `Call` or `After`, odin will retry the operations in that step until it eventually goes through. So, it is very important to make sure the "step" is **idempotent** and **retriable**. Also, if we don't want the external API quota run out because of those retries, we will also need to put some thought to design the "step".

- Although it is very very rare, storing data in bigtable or publishing pubsub message can also fail. So, it is possible, very unlikely though, that the "step" is successful but it still gets retried. Again, a step must be idempotent and retriable.

- Usually, odin will try to execute the pipeline as fast as possible. It is doable to defer the pipeline to a certain time point in the future using `CheckIn` function on the pipeline state. But during that period, the pipeline manager thread will be kicked off over and over again, checking if it is the time to execute a certain pipeline. It will consume all sorts of resources, like CPU, memory, I/O, etc. So, I'd suggest using Cloud Task in the case where a task in the workflow needs to be deferred.

Fun fact: we have a microservice called Ragnarok which is for load testing of odin. hmm...

## Google Cloud Scheduler
- The official doc: https://cloud.google.com/scheduler/docs/quickstart

Cloud Scheduler help us execute something with a specified interval. We can create a job that is executed every minute (`* * * * *`), or every 4 hours (`0 */4 * * *`), or at 6:00AM on the first every month (`0 6 1 * *`).

Now, we usually create scheduler jobs manually in the console, as there is not supposed to be a ton of jobs. The downside is that we cannot config retry strategies, which is available in the API. For example, without retry strategies, if the job is to publish a pubsub message and the topic does not exist, the job will just log the error and wait for the next scheduled time.
When a job runs, it can either make an HTTP call to an endpoint with a pre-defined body, or publish a pubsub message in a topic with a pre-defined payload.

We probably cannot use Cloud Scheduler in the middle of your workflow, but it can be a very good option to kick off a workflow.

## Takeaways
Talking about choosing the right weapon, it is hard to say "we should always use xxx". It is all depending on the problem we are trying to solve. I hope all the features I highlighted in this blog can help a bit when we want to choose the weapons to deal with a problem.

Usually, our in-house technologies can be some easy choices, especially when we want to reduce some toil. But, it is very important for us to be aware of the limitation of any tool we picked and make the call to use another tool when necessary.

Besides, it is also legit to make all of those tools work together. For example, we can setup a Cloud Scheduler task which sends out a pubsub message, the pubsub message triggers an odin pipeline, and a step in the pipeline is to defer a task in Cloud Task queue. In that case, we might want to think again about the workflow, as it is usually not supposed to be that complicated, and using all the services together means degradation of any service can break your workflow.

#### References
- [Aaron: Status-driven and Task-based Asynchronous Workflow in Microservices](https://vendasta-blog.appspot.com/blog/BL-NMVZWZM9/)
- [Google: Choosing between Cloud Tasks and Cloud Pub/Sub](https://cloud.google.com/tasks/docs/comp-pub-sub)
- [Google: Cloud Tasks versus Cloud Scheduler](https://cloud.google.com/tasks/docs/comp-tasks-sched)
