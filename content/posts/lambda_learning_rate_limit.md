---
title: "Lambda Learning: Prevent High Concurrency by Rate Limiting the Source Instead of Setting Reserved Concurrency"
date: "2024-05-13"
description: "How reserved concurrency can lead to unintended effects and what to do about it."
summary: "How reserved concurrency can lead to unintended effects and what to do about it."
tags: ["AWS", "AWS Lambda", "Lambda Learning", "Concurrency", "Rate Limit", "Scaling"]
ShowToc: true
TocOpen: true
---

This is a short article in a series about Lambda Learnings, based on my experience working with Lambdas.

### TL;DR

AWS accounts can run up to 1,000 concurrent Lambda instances, but without restrictions, one function could consume the whole pool. Configuring reserved concurrency can prevent this but can lead to surprising behavior on various event sources when the limit is reached and throttling happens. Instead, rate limit the event sources directly.

### The Unintended Consequences of Reserved Concurrency: "Dropped Events"

By default, an AWS account can run a total of [1,000 concurrent Lambda instances](https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html). Without any restrictions, one Lambda function could consume this entire pool, leaving all other functions throttled and unable to handle events, resulting in dropped events (*"What dropped means is different for each event source and can be quite unexpected as we will see below"*).

To prevent this, you can configure [reserved concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html) for each Lambda function. This sets an upper limit on the number of concurrent instances for that specific function.

However, in my experience relying on reserved concurrency is not ideal due to the unintended effects "Lambda Throttling" has on various Lambda event sources. Let's examine the expected versus actual behavior for two common event sources when throttling occurs:

**API Gateway Events:**
- **What you hope would happen:** The client receives a HTTP status code 429 (Rate Limit Exceeded) with a response body explaining the problem.
- **What actually happens:** The client receives a HTTP status code 500 (Internal Server Error) with a response body stating "Internal Server Error". There is now no way for the client to differentiate this from any other internal server error.

**SQS Events:**
- **What you hope would happen:** The message remains in the queue until throttling stops and is then processed.
- **What actually happens:** The message is marked 'unprocessable', returned to the queue, and retried based on the redrive policy, retention policy, or sent to another SQS dead-letter queue (DLQ). This could lead to 'lost' messages if not handled appropiately.

### Recommendation
Instead of setting reserved concurrency on the lambda functions, rate limit the event sources directly to achieve the desired behavior:

**API Gateway:**
- Configure [rate limits on the endpoints](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html). [Customize the status code, header, and body for rate limit responses](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-gateway-response-using-the-console.html). By default a HTTP Status Code of 429 (Rate Limit Exceeded) will be sent by API Gateway.

**SQS Events:**
- Set a [maximum concurrency on the SQS event source](https://aws.amazon.com/blogs/compute/introducing-maximum-concurrency-of-aws-lambda-functions-when-using-amazon-sqs-as-an-event-source/). Once the queue reaches the configured maximum concurrency for the consumer lambdas, no more lambda instances are created and the messages stay in the queue until capacity becomes available.

Depending on your use case, it might be useful to use reserved concurrency as a second layer of defense. It can act as a safeguard that should not be reached if the source rate limiting is correctly configured. If it is reached, it signals that you need to adjust your source rate limiting. A CloudWatch alarm can notify you when this limit is approached.

If you decide to use reserved concurrency, be sure to test what happens to events when throttling occurs, as the results may surprise you. This behavior is not always clearly documented in the AWS docs, so some experimentation is necessary.
