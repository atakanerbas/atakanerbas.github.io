---
layout: post
title: "Go vs Python for Serverless: Performance, Cost, and Ease of Use"
description: "A comparison of Go and Python for serverless applications in terms of performance, cost, and ease of use."
date: 2024-08-29 12:00:00 +0300
categories: programming
tags: go python serverless
image: "/assets/image/gopher.jpg"
---

In this article:
- [Introduction](#introduction)
- [Lambda Performance](#lambda-performance)
- [Cost](#cost)
- [Ease of Use](#ease-of-use)
    - [Runtime](#runtime)
    - [Development and Developer Experience](#development-and-developer-experience)
    - [Administration](#administration)
- [Conclusion](#conclusion)


## Introduction
I've been using Python for serverless applications for a while now. Although I'm mostly happy with it, I think Golang is a strong competitor in terms of performance and cost. 
I tried to compare the two languages in various aspects, some of them are subjective, but I tried my best to be as objective as possible. **Please note that Lambda performance may vary depending on the assigned machine, workload of AWS and the region.**

Golang is known for its performance and getting more popular among developers. However, I still think Golang is not the best language for handling complex data and implementing business logic.
On the other hand, Python is great for data processing and business logic implementation with its simplicity and readability, and rich libraries.
## Lambda Performance
In general, Golang is much faster than Python in benchmarks, **[sometimes 50-100 times faster](https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/python3-go.html)**. However, in serverless, the handlers does not do much computation as they are expected to return a HTTP response in a short time.  
I created two simple Lambda functions, which are triggered by an API Gateway event, querying a single-table design implemented DynamoDB table, and return a JSON response. The objective is to measure serialization/deserialization, and database query performance.
The DynamoDB table has 1000 items, and the Lambda functions query a single item by its primary key. Each item also has a map, which contains a nested map to test the deserialization performance.
I used [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) for the tests.

<figure>
  <figcaption>Invocation Times (Lower is better)</figcaption>
  <img src="/assets/image/invocation-times.png" alt="Invocation Times">
</figure>

AWS Lambda Power Tuning discards the top/bottom outliers, by default, 20% of the sample set. So we expect the results to be more stable, and, since sequential invocations will mostly cancel the effect of cold starts, let's take a look of the startup times of the functions.
The difference between the two languages is also visible in the startup times. **Cold starts with Python are 5-6 times slower than Golang for nearly all memory configurations.**
Python cold starts are starting from 350ms and does not change much with the memory configuration. Golang cold starts are starting from 50ms.  

The performance difference is also important for scalability, not just for response times. If you need to handle a burst of requests, Golang will be more suitable. 
The Lambda service has a limit of concurrent executions which is 1000 by default, but can be increased. For the same limit of concurrent executions, **the results imply that Golang can handle 2 to 5 times more concurrent requests than Python for the same memory configuration.**

## Cost

As Python is slower than Golang for the same workload, it consumes more CPU time for the same request and memory config.
If you look at the figure, Python needs 2048MB memory to be as fast as Golang for 512MB memory. The figure shows the cost difference between two(not too meaningful but gives an idea).

<figure>
  <figcaption>Cost Comparison</figcaption>
  <img src="/assets/image/cost-comparison.png" alt="Cost Comparison">
</figure>

### Why the graph is not telling the whole story
If you look at the graph, you will tend to directly compare the languages by fixed memory configurations. However, the performance is not the same for two languages for the same memory.
You need to compare the cost of the same performance, not the same memory configuration.
For example, let's assume that you need nearly the same performance for an endpoint.
Golang with 512MB memory returns the response in 744ms, and Python with 2048MB memory returns in 900ms, assume they are identical for your use case.
For 10,000,000 requests, Golang costs $57.13, and Python costs $295.13. Even Golang is still %20 faster, it costs %80 less.

## Ease of Use
Python is the undisputed winner in terms of ease of use. Why?

### Runtime
You can run Python code on your local machine just by calling the handler function with a dictionary, event. You can test your code without deploying it, you can debug it line by line and profile.
Running Golang code you wrote in your local machine, and vice versa, is not that easy. Golang has a different Lambda runtime environment. You need to wrap your code with a handler function, and you need to use a library to simulate the Lambda environment.
You can use [AWS SAM](https://aws.amazon.com/serverless/sam/) for local testing, but it is not as easy as running Python code.  

You may also need to cross-compile your Golang code for Lambda, depending on your local machine. 

### Development and Developer Experience
There are lots of IDE plugins, free and paid services, libraries, and documentation for Python Serverless. Golang lacks these; development tools are not as mature as Python's.
Python has a rich library ecosystem. Go also has great libraries but finding the library and the documentation is not as easy as with Python.
You can look at [Awesome Go](https://github.com/avelino/awesome-go) for a curated list of Golang libraries.  

Language consistency is another point to consider. If you are using CDK, you can use Python for both infrastructure and application code.
CDK for Golang does not feel as natural as Python. You may want to reuse some functions between infrastructure and application code, and it is easier with Python.
If your team is already using Python, it is easier to onboard new developers, or switch between projects/stacks. 
### Administration
Python is a scripting language, and it makes it a great tool for application administration tasks.
You can directly invoke your functions from other packages and emulate your whole flow. You can just create a notebook and import your functions, and run them.
This might be a DB migration, a data processing task, or a simple monitoring script. Building all these in Golang is not as easy as in Python.

## Conclusion
My very subjective conclusion is:
- Python is better for:
    - When performance and scalability are not big concerns
    - Prototyping
    - Data processing
    - Business logic implementation
    - Application administration tasks
- Golang is better for:
    - Performance and scalability
    - Handling burst of requests
    - Cost efficiency
    - Complex applications. Implementing complex business logic is easier with Python, but the performance of Golang will reduce the complexity of the application.

I think Golang is a great language for serverless applications, but it is not a silver bullet. You need to consider the trade-offs and your use case.
