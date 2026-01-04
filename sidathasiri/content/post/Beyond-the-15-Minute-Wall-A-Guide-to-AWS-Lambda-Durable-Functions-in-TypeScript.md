---
layout: post
title: "Beyond the 15-Minute Wall: A Guide to AWS Lambda Durable Functions in TypeScript"
date: 2026-01-04
author: "Sidath Munasinghe"
keywords: "aws lambda, durable functions, typescript, serverless, aws, cloud computing, state management, sidath, munasinghe"
description: "Learn how AWS Lambda Durable Functions enables stateful, long-running workflows in TypeScript without Step Functions complexity. Build workflows that can run for up to a year with automatic checkpointing and state management."
URL: "/2026/01/04/Beyond-the-15-Minute-Wall-A-Guide-to-AWS-Lambda-Durable-Functions-in-TypeScript/"
image: "/images/posts/Beyond-the-15-Minute-Wall-A-Guide-to-AWS-Lambda-Durable-Functions-in-TypeScript/main-cover-image.png"
---

Let's be real: we've all had that moment where our AWS Lambda function hit the dreaded 15-minute timeout, and we died a little bit inside.

Up until recently, if you wanted to build a long-running process — say, a 30-day trial onboarding flow or a multi-step document processing pipeline — you had two choices:

1. **AWS Step Functions**: Powerful, but you had to learn Amazon States Language (ASL). Let's just say writing logic in 500 lines of JSON/YAML isn't exactly "fun."
2. **The "DIY" State Machine**: Manually saving state to DynamoDB, triggering SQS queues, and praying your retry logic didn't create an infinite loop.

But the game changed at re:Invent 2025. Enter **AWS Lambda Durable Functions**. We can finally write stateful, long-running workflows directly in TypeScript or in other supported languages. No YAML, no external state management, just pure code.

## What is "Durable Execution" Anyway?

Imagine you're reading a massive 1,000-page book. If you have to put it down to sleep, you don't start from page one the next morning — you use a bookmark.

Durable Functions act as that bookmark for your code. When your function hits a "wait", AWS "checkpoints" the progress and shuts down the compute. When the event is ready, it "replays" the function, skips the parts it already finished, and picks up exactly where it left off. The idea is very simple but it's very powerful.

I know what you're thinking: "Is Step Functions dead?" Not quite. If you are building complex micro-service choreography between 10 different AWS services, Step Functions' visual designer is still king. But if you are building application logic that just needs to be stateful, Durable Functions is your new best friend.

## Order Processing With Durable Functions

Let's look at a real-world scenario: An e-commerce workflow that needs to validate an order, wait for payment, and then ship the goods.

Here is how you would write this using the new `@aws-sdk/client-lambda-durable` in TypeScript.

```typescript
import { DurableContext, withDurableExecution } from "@aws/durable-execution-sdk-js";

// Imagine these services interact with your DB and APIs
import { inventoryService, paymentService, shippingService } from "./services";

export const handler = withDurableExecution(
  async (event: any, context: DurableContext) => {
    const { orderId, amount, items } = event;
    
    // 1. Reserve inventory
    // If this fails, the SDK handles the retry. If it succeeds, the result is checkpointed
    const inventory = await context.step("reserve-inventory", async () => {
      return await inventoryService.reserve(items);
    });
    
    // 2. Process payment
    const payment = await context.step("process-payment", async () => {
      return await paymentService.charge(amount);
    });
    
    // 3. Create shipment
    // This step only runs once the previous two are successfully checkpointed.
    const shipment = await context.step("create-shipment", async () => {
      return await shippingService.createShipment(orderId, inventory);
    });
    
    return {
      orderId,
      status: 'completed',
      paymentId: payment.id,
      shipmentTracking: shipment.trackingNumber
    };
  }
);
```

This code represents a significant shift in how we build on AWS. By using the `@aws-durable-execution-sdk-js`, you are moving away from "fire-and-forget" functions toward **Stateful Workflows**.

Here is a breakdown of what is actually happening under the hood while this code runs.

### 1. The Wrapper (`withDurableExecution`)

This isn't just a decorator; it's a manager. It tracks every interaction you have with the `context` object. It maintains a "History Log" of your execution.

### 2. Checkpointing (`context.step`)

When your code hits `await context.step("reserve-inventory", ...)`:

- **Execution**: The SDK runs your code.
- **Persistence**: Once the code returns the `inventory` object, the SDK saves that result into a managed state store (hidden from you).
- **Reliability**: If the Lambda crashes 10 seconds later, AWS knows that "reserve-inventory" is already done.

### 3. The Replay Mechanism

If your `process-payment` step takes too long or the Lambda environment restarts, AWS triggers the function again.

- The code starts from the very first line.
- When it hits `context.step("reserve-inventory")`, the SDK looks at the history log and says: "I already have the result for this!"
- Instead of running the inventory code again, it simply injects the saved result into the `inventory` variable and moves to the next line.

## Other Powerful Features

Once you move beyond basic sequential steps, the `@aws-durable-execution-sdk-js` reveals its true power. It allows you to build complex, non-linear systems that feel like a single script but behave like a distributed state machine.

Here is a breakdown of the advanced primitives you can use to orchestrate high-scale applications.

### `context.wait`: The Time-Traveler

This is more than just a `sleep()` command. When you call `wait`, your Lambda function stops executing entirely. AWS saves the state and schedules a fresh invocation for the future.

- **Cost Efficiency**: You are charged $0.00 for the time spent waiting.
- **Precision**: You can wait for seconds, hours, or even a specific `Date`.
- **Max Duration**: A single workflow can wait for up to 1 year.

### `context.waitForCallback`: The Human Bridge

This is the "Human-in-the-Loop" pattern. It pauses the code until an external system tells it to continue.

- **How it works**: When called, it generates a unique `taskToken`. You send this token to an external service (like a Slack bot, an Email, or a React UI).
- **The Resume**: The code stays "frozen" until your external system calls the `SendDurableExecutionCallback` API with that specific token.

### `context.parallel`: The Turbo-Charger

Standard `Promise.all()` is dangerous in serverless because one long-running promise could time out the whole Lambda. `context.parallel` is different. It tells AWS to manage multiple durable operations simultaneously.

- **Persistence**: Each branch in the parallel array is checkpointed individually.
- **Resilience**: If one branch fails, the other branches' results are already saved, so a retry only re-runs the failed path.

### `context.invoke`: The Modular Architect

This allows you to call other Lambda functions (durable or standard) as a child process of your current workflow.

- **Encapsulation**: You can build a library of "Micro-Workflows" (e.g., a `sendEmailWorkflow`) and call them from any parent.
- **Parent-Child Link**: If the parent is canceled, AWS can automatically handle the cleanup of child invocations.

## Best Practices & Gotchas
The durable functions are so powerful. But at the same time you'll need to follow the best practices to avoid the chaos.

### The Golden Rule: Orchestration Must Be Deterministic

When your function "wakes up" from a 7-day wait, it runs from the very beginning to find its place. If your code produces a different result during this replay, the SDK will throw a Non-Deterministic Error.

**The Mistake**: Using `new Date()` or `Math.random()` in the main body.

```typescript
// ❌ BAD: This will be different every time the function replays
const requestId = Math.random();
```

**The Fix**: Wrap these calls in `context.step()`. This ensures the value is generated once, saved in the checkpoint, and then "replayed" as the exact same value.

```typescript
// ✅ GOOD: The result is saved to the history after the first run
const requestId = await context.step("get-id", async () => Math.random());
```

### The "15-Minute" Misconception

It's a common trap to think the 15-minute Lambda limit is gone. It isn't.

- **Workflow Limit**: Can last up to 1 year.
- **Step Limit**: Any single `context.step` must finish within the 15-minute Lambda timeout.

### State Size & The "Fat Payload" Problem

AWS stores your execution history (the results of every `context.step`) in a managed state store.

- **Limit**: This history is typically limited
- **Gotcha**: If you return a massive JSON object from `inventoryService.reserve()`, the checkpointing will fail.
- **Solution**: Store large data in S3 and return only the `S3_URL` from your steps.

### Avoiding "Zombie" Workflows

Since workflows can live for a year, it's easy to accidentally leave thousands of executions "Waiting" and racking up state storage costs.

- **Best Practice**: Always set a `timeout` in your `waitForCallback` and `wait` operations.
- **Retention Policy**: Configure your `RetentionPeriodInDays` in the `DurableConfig` so that finished histories are automatically scrubbed.

## Conclusion

With the release of AWS Lambda Durable Functions, the boundary between "simple functions" and "complex orchestrators" has finally vanished. We no longer have to choose between the simplicity of a single code file and the resilience of a multi-step state machine.

### When to Use Durable Functions vs. Step Functions

Even in 2026, Step Functions remains the king of low-code orchestration and massive service integrations (like connecting S3 to Bedrock without any code). However, Durable Functions is now the clear winner for:

- **Pure Developer Workflows**: If you want your logic in TypeScript, not YAML.
- **Complex Data Handling**: When you need to pass rich objects between steps without JSON serialization headaches.
- **Fast Iteration**: Testing locally with the `LocalDurableTestRunner` is significantly faster than deploying state machines to the cloud.

### Monitoring Your "Marathons"

Because these functions can run for months, AWS introduced a dedicated Durable Executions tab in the Lambda Console. You can now see a visual timeline of every execution, including exactly when a function "went to sleep" and which specific `context.step` is currently running.

### Final Thought: Write Code, Not Config

The true power of this SDK is that it lets you focus on business logic. You don't have to write "plumbing" code for retries, database checkpoints, or state management. You simply write your function from top to bottom, and AWS ensures that it stays alive until the job is done.

The 15-minute wall has been torn down. It's time to start building applications that can truly go the distance.
