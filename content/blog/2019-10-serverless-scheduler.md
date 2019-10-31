---
title: "Serverless scheduler"
date: 2019-10-11T11:17:14+02:00
publishdate: 2019-10-11T11:17:14+02:00
image: "/images/blog/1.jpg"
tags: ["aws", "serverless", "scheduling", "lambda", "dynamodb", "sqs", "sns"]
comments: false
draft: false
---
AWS offers many great services, but when it comes to ad hoc scheduling there is still potential. We use the term ad hoc scheduling for irregular point in time invocations, e.g. one in 32 hours and another one in 4 days.

{{< figure src="/images/blog/2019-10/1.png" >}}

[Zac Charles](https://medium.com/u/e523a4c1158d) has shown [a couple ways to do serverless scheduling](https://medium.com/@zaccharles/there-is-more-than-one-way-to-schedule-a-task-398b4cdc2a75), each with their own drawbacks in terms of cost, accuracy or time in the future.

In [Yan Cui](https://medium.com/u/d00f1e6b06a2)’s [analysis](https://theburningmonk.com/2019/06/step-functions-as-an-ad-hoc-scheduling-mechanism/) of step functions as an ad hoc scheduling mechanism he lists three major criteria:

1. *Precision*: how close to my scheduled time is the task executed? The closer, the better.
2. *Scale — number of open tasks*: can the solution scale to support many open tasks. I.e. tasks that are scheduled but not yet executed.
3. *Scale — hotspots*: can the solution scale to execute many tasks around the same time? E.g. millions of people set a timer to remind themselves to watch the Superbowl, so all the timers fire within close proximity to kickoff time.

This article shows my [serverless scheduler](https://github.com/bahrmichael/aws-scheduler) and how it performs against these criteria. Follow-up articles will take a closer look at scaling and cost.

## Overview

The service’s two interfaces are an SNS input topic which receives events to be scheduled and an output topic which is hosted by the consuming account.

Each input event must contain an ARN to the SNS output topic where the payload will be published to once the scheduled time arrives.

{{< figure src="/images/blog/2019-10/2.png" >}}

Internally the service uses a DynamoDB table to store long term events. Events whose scheduled time is less than ten minutes away are directly put into the short term queue.

This queue uses the DelaySeconds attribute to let the message become visible at the right time. The event loader function is basically a cron job. The emitter function finally publishes the events to the desired topics.

## Usage

The scheduling service takes an event with a string payload along with a target topic, the scheduled time of execution and a user agent. The latter is mainly to identify callers.

{{< figure src="/images/blog/2019-10/3.png" >}}

The above python code publishes an event with a custom string payload. Please make sure that you fill out all four fields or the event may be dropped. See more in the Troubleshooting section at the end of the article.

Note that we have to create our own SNS topic which must grant publish rights to the serverless scheduler. The [quickstart project](https://github.com/bahrmichael/aws-scheduler-testing#prerequisites) contains a script that helps you with creating the SNS topic. The AWS role of the public serverless scheduler is `arn:aws:sts::256608350746:assumed-role/aws-scheduler-prod-us-east-1-lambdaRole/aws-scheduler-prod-emitter`

{{< figure src="/images/blog/2019-10/4.png" >}}

You may also manually assign an additional access policy to your SNS topic.

{{< figure src="/images/blog/2019-10/5.png" >}}

After that you need a [lambda function](https://docs.aws.amazon.com/en_pv/lambda/latest/dg/with-sns-example.html) that consumes events from your output topic.

{{<figure src="/images/blog/2019-10/6.png">}}

That’s it. You can use the [quickstart project](https://github.com/bahrmichael/aws-scheduler-testing) to quickly schedule and receive your first events. Go try it out!

## Evaluation

Let’s come back to the criteria mentioned in the intro: Precision, Scale and Hotspots.

### Precision

Precision is probably the most important of all. Not many use cases are tolerant to events arriving minutes to hours late. A sub second delay however is viable for most use cases.

Over a period of five days I create a base load of roughly 1000 events per hour. The emitter function logs the target timestamp and the current timestamp which are then compared to calculate the delay. Plotting this data gives us the graph below.

{{< figure src="/images/blog/2019-10/7.png" >}}

As you can see, the vast majority is well below 100 ms with the maximum getting close to 1000 ms. This gets clearer when you take a look at the histogram.

{{< figure src="/images/blog/2019-10/8.png" >}}

### Scale

Scaling for many open tasks is an easy one here. SQS and DynamoDB don’t put a hard limit on how many items you can process. Therefore the serverless scheduler can hold millions and billions of events in storage for later processing.

{{<tweet 1182405324222779392>}}

Based on a discussion with [Daniel Vassallo](https://medium.com/u/1894ef0f7671) I don’t believe SQS to become a bottleneck.

The only bottleneck is the event loader function. It does however use a dedicated index which helps to identify which items are soon to be scheduled. It then only loads the database IDs and hands them over to a scalable lambda function which then loads the whole event into the short term queue.

Tests with varying loads showed that the bottleneck lambda function is able to process more than 100.000 events every minute or 4.3 billion events per month. Due to increased costs I did not run tests at higher scales. Contributions are welcome ;)

### Hotspots

Hotspots can arise when a lot of events arrive at the input topic or are to be emitted within a very short time.

DynamoDB is configured to use pay-per-request which should allow for nearly unlimited throughput spikes, however I had to add a retry mechanism when DynamoDB does internal auto scaling. The most time critical function, the emitter, does any possible database operations after emitting the events to the output topic.

Both the SNS input topic and the SQS short term queue are not expected to become slow under high pressure, but the consuming lambdas could.

{{<tweet 1182410032727482368>}}

[Randall Hunt](https://medium.com/u/c8eb1c0e04da) wrote an [AWS blog article](https://aws.amazon.com/blogs/aws/aws-lambda-adds-amazon-simple-queue-service-to-supported-event-sources/#additional-info-lambda-sqs) which takes a deep dive on concurrency and automatic scaling in this situation.

> […] the Lambda service will begin polling the SQS queue using five parallel long-polling connections. The Lambda service monitors the number of inflight messages, and when it detects that this number is trending up, it will increase the polling frequency by 20 ReceiveMessage requests per minute and the function concurrency by 60 calls per minute.

While cold starts of lambda functions can result in a slight increase of delays, the polling behaviour or an eventual lambda concurrency limit could lead to major delays.

To test this I scheduled 30.000 events to be published within a couple seconds. While the median went well up (probably due to cold starts), this was still not enough to hit any limits.

{{<figure src="/images/blog/2019-10/9.png">}}

To sum up the Hotspots section: Very sharp spikes with very high loads can become an issue, but those are so big I couldn’t test them yet.

I’m curious about an upcoming talk at re:Invent 2019 which takes a deep dive into SQS.

{{<tweet 1182472004185645056>}}

## Troubleshooting and Error Handling

Due to the asynchronous nature of the service, error handling is a bit trickier. I decided against an API Gateway endpoint to publish events due to its cost of 3.5$ per million events (compared to SNS’ 0.5$ per million events). Errors can be published to another output topic, the failure topic.

If the event is not published at your output topic, please first make sure you pushed a correct event to the input topic. It must contain all of the four fields `payload`, `date`, `target` and `user`. All of those must be strings.

You may additionally add a field `failure_topic` to the event which contains the ARN of another SNS topic where you’d like to be informed about errors. Note that this must have the same publish permission for the serverless scheduler as your output topic does.

## Conclusion

The serverless scheduler appears to perform excellent both in precision and scale. Hotspots might become an issue at very sharp spikes and exceptional volume, but the actual limits remain to be discovered.

I’d be happy to see your testing results and what you think about the serverless scheduler. Go try it out with the [quickstart project](https://github.com/bahrmichael/aws-scheduler-testing) or check out the [source code](https://github.com/bahrmichael/aws-scheduler). Would you attach it to your own project?

More tests and insights will be published over the next weeks.