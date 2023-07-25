# Tuning a Dapr based message oriented microservices solution in Azure Kubernetes for high throughput

## Introduction

While building any software solution it's important to ensure that the application can handle the desired load without compromising on the performance requirements. Following sections talks about a few changes we had to make while tuning our solution for high throughput.

This solution had a message oriented, microservices based architecture. The services were written in .NET and relied on Dapr for service and message invocations. Microservices were hosted in Kubernetes and messaging pub-sub used Azure Servicebus in the backend. The target of this performance tuning was to handle 500 messages per second, while processing 99% of the messages in 15 seconds. 

> Performance testing is costly both in terms of resources and time. Ensure you follow the best practices to keep your tests objective and efficient. You can find some good guidance [here](https://microsoft.github.io/code-with-engineering-playbook/automated-testing/performance-testing/).

## Little more about the architecture

The solution had a *Webhook service* exposed to the internet that received transaction IDs from a third-party system. This service immediately pushed the ID and related metadata to an *ID topic*. An *event processor* micro-service read from this topic and called a *facade* microservice to fetch the complete details of that transaction. The *facade* would internally call a third-party system to fetch the details. The *event processor* pushed the complete transaction details to the *details topic* for further processing. A *transaction receiver* microservice read from this *details topic* and invoked a *command API* microservice, that's part of a CQRS pattern, for final processing and persistence of the transaction. 

As you can see, the solution had a series of microservices and queues for enriching and transforming the event to complete the processing. For the purpose of performance tests, the third-party services were mocked to focus more on the services built by us. JMeter was used to send requests to webhook, and the *facade* invoked a dummy service instead of actual third party-service. 

                                   +--------+--------+
                                   |                 |
                                   |     Facade      |
                                   |                 |
                                   +--------+--------+
                                            ^
                                            |
                                            |
         +-----------------+       +--------+--------+       +-----------------+       +-----------------+
         |                 |       |                 |       |                 |       |                 |
    --->-+  Webhook        |       |  Event          |       |  Transaction    +------>+      CQRS       |
         |  Service        |       |  Processor      |       |  Receiver       |       |  Command API    |
         |                 |       |                 |       |                 |       |                 |
         +--------+--------+       +--------+--+-----+       +-----------+-----+       +-----------+-----+
                  |                         ^  |                         |                      |
                  |                         |  |                         |                      |
                  |                         |  |                         |                      |
                  |                         |  |                         |                      v
                  |    +--------------+     |  |   +--------v--------+   |             +--------+--------+       
                  |    |              |     |  |   |                 |   |             |                 |       
                  +--->|  ID Topic    |->---+  +->-+  Details Topic  +->-+             | Transactions DB |       
                       |              |            |                 |                 |                 |       
                       +--------------+            +-----------------+                 +-----------------+       

## System parameters and performance metrics

When we started the tests, we only had Kubernetes and Azure infra related parameters to tune.

1. Number of pods for each service.
2. CPU and memory allocated to each service.
3. Azure Service bus. 

Since this was a message-oriented system the metrics we measured were-

1. Time to process an event end to end.
2. The Number of events processed per second.

For this, we had set up a few dashboards to monitor-
1. Number of incoming and outgoing messages to service bus topics.
2. Number of active messages in each service bus topic.
3. Number of dead lettered messages for each topic.
4. CPU and Memory utilization for each Pod.
5. Number of active pods and pod restarts.

## Test load and ideal response metrics

The test had a ramp from 0 to 500 requests per second and then uniform load of 500 requests per second.

    Requests per second
    |
    | 500 ___________________
    |    /                   \
    |   /                     \
    |  /                       \
    | /                         \
    +--------------------------------> Time
    Ramp-up  Uniform load    Ramp-down

If the system performs as expected the number of incoming and outgoing messages in service bus topics should follow a similar graph with zero messages in dead letter queue and zero or comparatively smaller number of active messages.

## The story of Kubernetes pod restarts

### Part 1- setting *maxConcurrentHandlers*

In the first test, the *event processor* and *facade* pods started crashing before reaching 500 requests per second. Our assumption at this stage was that the only service that would crash was the *webhook* as the rest of the system was decoupled through a message broker- *ID topic*. But that's not what happened. 

On further digging we noticed that the *maxConcurrentHandlers* property for the Azure Service Bus pub-sub component was not set explicitly and the default was *unlimited*. This let Dapr sidecar pull more messages than the *event processor* could handle. This caused the pods to take longer to process any request, including Kubernetes liveliness probe. Eventually, Kubernetes started restarting those Pods. We could see liveliness probe failures in Kubernetes logs.

We made couple of changes after this, one by one-

> Always change only one parameter from test to test.

1. Set *maxConcurrentHandlers* to what the system could handle.
2. Increased the number of *event processor* Pods.

> Read more about *maxConcurrentHandlers* [here](https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-azure-servicebus-topics/#spec-metadata-fields).

### Part 2- Facade service optimization

With the above two changes, the *event processor* started functioning well but the *facade* service pod continued to restart until we increased the number of pods to a very high number. However, this was not an acceptable solution for two reasons- 

1. Whenever a *facade* pod failed, it never started again causing the message throughput to drop.
2. The resource utilization (both CPU and memory) on *facade* Pods were always low (less than 30%). 

This needed further investigation and optimization.

#### Facade Optimization 1- Service initialization

The *facade* was responsible for calling a few different deployments of a third party service and needed some credentials to be loaded from its database and key vault to do that. Though it had caching in place, the credentials were loaded once the first request hits the service. This was causing the service to take a while to fulfil the first set of requests. 

One might think only the very first request would take longer as the next request would find the credentials in the cache. But that would not be the case if several requests reached the service parallelly or with very little delay between then on application start-up. All of those threads processing those requests would check the cache and miss and try to fetch and cahe the credentials. 

This was the reason why the *facade* pods failed to start when the system was under load. Once Kubernetes found that the Pod was ready, it routed the traffic to the Pod. Since all of those requests got queued because of this initialization delay, Pod was not able to respond to liveliness probes. And Kubernetes tried restarting the Pods.  

The solution for this included a couple of things-

1. Ensure that the cache was loaded only once. For this, using a simple lock statement didn't help as all of those threads that missed the cache also waited to acquire lock. 

    With lock-
    ```CSharp
    private async Task<Credential?> GetCredentialForInstance(string instance)
    {
        if (!_cacheProvider.TryGetValue(instance, out Credential? credential))
        {
            lock (_credentialLock)
            {
                credential = _cacheProvider.GetOrCreateAsync(instance, LoadCredentialAsync).Result;
            }
        }

        return await Task.FromResult(credential);
    }
    ```

    Final version-
    ```CSharp
    private async Task<Credential?> GetCredentialForInstance(string instance)
    {
        Credential? credential = null;

        // Do not wait indefinitely.
        for (int i = 0; i < _gr4vyServiceOptions.CredentialRefreshTimeoutInMilliseconds / 100; i++)
        {
            // Try to Find in cache.
            if (_cacheProvider.TryGetValue(instance, out credential))
            {
                break;
            }

            // If not found, try to acquire lock.
            if (Interlocked.Exchange(ref _isCacheBeingSet, 1) == 0)
            {
                try
                {
                    // Another try. If not found, load from Mongo.
                    credential = await _cacheProvider.GetOrCreateAsync(instance, LoadCredentialAsync);
                    break;
                }
                finally
                {
                    Interlocked.Exchange(ref _isCacheBeingSet, 0);
                }
            }

            // Wait while some other thread is loading the credential.
            await Task.Delay(100);
        }

        return credential;
    }
    ```

2. Initialized the connections to database and key vault on application start-up before routing any traffic to the Pods. This reduced the time needed to process the first request.

#### Facade Optimization 2- C# best practices

This one was straight forward. We had some blocking operations in the code that stopped the app from efficiently using the CPU available. Modifying the code to use `async-await` all the solved that issue as well.

## Key takeaways

1. An instance of a service is likely to perform worse or even go down if it receives more number of concurrent requests than it can handle.
2. Implement health check endpoints correctly so that Kubernetes knows when to start forwarding traffic to that instance.
3. Complete costly initializations beforehand so that initial requests don't take too long.
4. Ensure that the code is thread safe. If there are critical code sections, ensure tasks/threads are not blocked for too long.
5. Ensure that available resources are utilized well enough before scaling out or scaling up.
6. Ensure programming best practices that help improve performance, like async-await, are implemented correctly.

## Additional thoughts

In some applications, CPU and/or memory utilizations will always be low because of the nature of the application. In those cases, number of pods can be scaled based on queue-length. You might have to use solutions like [KEDA](https://keda.sh/) for this as Kubernetes doesn't support this out of the box.
