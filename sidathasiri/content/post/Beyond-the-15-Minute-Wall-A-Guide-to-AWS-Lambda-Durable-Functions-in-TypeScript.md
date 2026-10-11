---
layout: post
title: "Long-Running Workflows in Plain TypeScript: Getting Started with AWS Lambda Durable Functions"
date: 2026-01-04
author: "Sidath Munasinghe"
keywords: "aws lambda durable functions, long-running workflows, typescript, serverless orchestration, checkpointing, step functions alternative, aws, sidath, munasinghe"
description: "Build stateful, long-running TypeScript workflows with AWS Lambda Durable Functions: checkpoints, replay, waits, callbacks and the gotchas to avoid."
URL: "/2026/01/04/Beyond-the-15-Minute-Wall-A-Guide-to-AWS-Lambda-Durable-Functions-in-TypeScript/"
image: "/images/posts/Beyond-the-15-Minute-Wall-A-Guide-to-AWS-Lambda-Durable-Functions-in-TypeScript/main-cover-image.png"
---

Every serverless developer eventually writes a Lambda that does "just one more thing", and one day it runs for 15 minutes and gets killed halfway through. If you've watched a CloudWatch log end abruptly with `Task timed out`, you know the feeling.

The trouble is that plenty of real business processes don't finish in minutes. Think of a month-long trial onboarding sequence, or a document pipeline that hands off to a human reviewer. Until recently, the options for workflows like that on AWS weren't great:

| Approach | What you get | What it costs you |
| --- | --- | --- |
| **AWS Step Functions** | A powerful, managed orchestrator | Your logic lives in Amazon States Language, which quickly grows into hundreds of lines of JSON/YAML |
| **Hand-rolled state machine** | Full control | You persist state in DynamoDB yourself, chain SQS queues together, and hope the retries never loop forever |

AWS announced a third option at re:Invent 2025: **AWS Lambda Durable Functions**. You write the stateful, long-running workflow as ordinary code, in TypeScript or one of the other supported languages. You don't write a state machine definition or build your own persistence layer.

## The Core Idea: Checkpoint, Sleep, Replay

Here's the analogy I find most useful. When you stop halfway through a long novel, you don't start again from chapter one the next day. You put a bookmark in and carry on from there.

A durable function keeps a bookmark for your code. When the workflow reaches a point where it has to wait, AWS records how far it got and releases the compute. Later, when it's time to continue, Lambda invokes the function again. The function runs from the top, but every step that already finished is skipped using its recorded result, so execution effectively resumes at the bookmark. The idea is simple, and it changes a lot about what you can build.

Does this make Step Functions obsolete? No. If you're orchestrating ten AWS services and your team relies on the visual designer, Step Functions is still the better tool. Where durable functions shine is *application logic* that needs to remember where it was.

## A Worked Example: Reserve, Charge, Ship

Let's take a typical e-commerce flow. We reserve stock for the order, charge the customer, and then create a shipment. Here's what that looks like with the durable execution SDK (`@aws/durable-execution-sdk-js`) in TypeScript:

```typescript
import { DurableContext, withDurableExecution } from "@aws/durable-execution-sdk-js";

// Placeholder services that talk to your database and external APIs
import { inventoryService, paymentService, shippingService } from "./services";

export const handler = withDurableExecution(
  async (event: any, context: DurableContext) => {
    const { orderId, amount, items } = event;
    
    // 1. Hold stock for the order
    // On failure the SDK retries this step; on success its return value is checkpointed
    const inventory = await context.step("reserve-inventory", async () => {
      return await inventoryService.reserve(items);
    });
    
    // 2. Charge the customer
    const payment = await context.step("process-payment", async () => {
      return await paymentService.charge(amount);
    });
    
    // 3. Book the shipment
    // Reached only after both earlier steps have been checkpointed
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

It looks like a normal async handler, and that's the point. But this is no longer a fire-and-forget function. It's a **stateful workflow**, and three mechanisms make that work.

### How the handler is wrapped

`withDurableExecution` does more than decorate the handler. It supervises the execution, records every call you make through `context`, and keeps a running history of the workflow.

### What happens at each step

When execution reaches `context.step("reserve-inventory", ...)`, the SDK:

1. runs the function you passed in,
2. takes its return value (here, `inventory`) and persists it to a state store that AWS manages for you, and
3. treats the step as complete from then on. If the Lambda environment dies a few seconds later, the platform already knows the reservation succeeded.

### How a resumed run catches up

Say `process-payment` runs slowly, or the execution environment gets recycled. Lambda invokes the handler again, from the first line. When the code reaches `reserve-inventory`, the SDK finds that step in the history, skips the callback and returns the stored value. `inventory` ends up with exactly the same data as before, and the workflow moves on to the step that hasn't completed yet.

## Beyond Sequential Steps

A linear list of steps is only the start. The SDK gives you a handful of primitives for non-linear flows. Your code still reads like one script, but it behaves like a distributed state machine.

**`context.wait`: pausing without paying.** This isn't a `sleep()`. Your function actually stops running. AWS stores the state and schedules a new invocation for later. You aren't charged compute while the workflow waits. You can wait for seconds, hours, or until a specific `Date`, and a single workflow can stay paused for up to a year.

**`context.waitForCallback`: waiting for a person or another system.** This is how you implement human-in-the-loop flows. The call produces a unique `taskToken`, which you hand to something outside the workflow, such as a Slack approval bot, an email link or a button in a React app. The workflow stays paused until that system calls the `SendDurableExecutionCallback` API with the token.

**`context.parallel`: concurrency that survives failures.** Running a plain `Promise.all()` inside a Lambda is risky, because one slow promise can push the whole invocation past its timeout. With `context.parallel`, AWS manages the concurrent durable operations for you. Each branch is checkpointed on its own, so if one branch fails, the finished branches keep their results and a retry re-runs only the failed one.

**`context.invoke`: composing workflows.** You can call another Lambda function, durable or regular, as a child of the current workflow. That lets you build small reusable sub-workflows (a `sendEmailWorkflow`, for example) and call them from any parent. Because AWS tracks the parent-child relationship, it can clean up child invocations when the parent is cancelled.

## Rules to Live By (and Traps to Avoid)

These primitives are powerful, but they come with rules. Ignore them and you'll get errors that are hard to diagnose.

### Keep the orchestration code deterministic

This is the most important rule. When a workflow resumes, say after a seven-day wait, the handler runs again from the top to find its place. If the code takes a different path or produces different values on that replay, the SDK raises a Non-Deterministic Error.

The usual cause is calling something non-deterministic, such as `Math.random()` or `new Date()`, directly in the handler body:

```typescript
// ❌ BAD: a new value is produced on every replay
const requestId = Math.random();
```

The fix is to put the call inside a step. The value is generated once, stored in the checkpoint, and returned unchanged on every replay:

```typescript
// ✅ GOOD: generated on the first run, then read back from history
const requestId = await context.step("get-id", async () => Math.random());
```

### The 15-minute limit still exists, just at a different level

People often assume the timeout has disappeared. It hasn't. It now applies per invocation rather than per workflow:

- the **workflow** as a whole can last up to a year;
- any single **`context.step`** still has to finish inside the normal 15-minute Lambda timeout.

If one step could run longer than that, split it into smaller steps.

### Return small values from steps

Every step result goes into the execution history, and that history has a size limit. If `inventoryService.reserve()` returns a huge JSON document, checkpointing will fail. Write large payloads to S3 and return only a reference such as the `S3_URL`.

### Don't leave workflows waiting forever

A workflow can live for a year, so it's easy to end up with thousands of executions sitting in a waiting state and adding to your state storage bill. Two habits help:

- Always set a `timeout` on `wait` and `waitForCallback` operations, so nothing waits indefinitely.
- Set `RetentionPeriodInDays` in the `DurableConfig`, so the histories of finished executions are cleaned up automatically.

### More pitfalls worth knowing

These aren't in the example above, but they follow from how checkpoint-and-replay works:

- **Design steps to be safe to retry.** A step can fail after its side effect has happened but before the result is checkpointed, and then the SDK retries it. Make external calls idempotent, for example by passing `orderId` as an idempotency key to your payment provider, so a retry can't charge the customer twice.
- **Treat step names as part of the contract.** Replay matches recorded results to steps by their order and name. Renaming, reordering or removing steps in code that in-flight executions will resume can break those executions. Ship incompatible changes as a new published version or alias and let old executions finish on the old code.
- **Don't do side effects outside steps.** Any API call, database write or log you care about that sits outside a `context.step` runs again on every replay. Anything that must happen exactly once belongs inside a step.

## Choosing Between Durable Functions and Step Functions

Step Functions is still the strongest low-code orchestration option, especially for direct service integrations such as connecting S3 to Bedrock without writing glue code. Durable functions are the better fit when:

- you want the workflow logic in **TypeScript rather than YAML/JSON**;
- your steps pass **rich objects** to each other and you'd rather not fight JSON serialization between states;
- you care about **fast feedback**: running tests locally with the `LocalDurableTestRunner` is much quicker than deploying a state machine to the cloud for every change.

For observability, the Lambda console now has a **Durable Executions** tab. It shows each execution on a visual timeline, including when it went to sleep and which `context.step` is currently running. That matters a lot when a workflow runs for weeks.

## Quick FAQ

**Do I pay while a workflow is waiting?**
You don't pay for compute during a `wait`, because the function isn't running. The stored execution state does have a cost, which is why timeouts and retention settings matter.

**Can I use `Promise.all` inside a durable function?**
It's safer to use `context.parallel`. It checkpoints each branch, so a failure doesn't throw away the work the other branches already finished.

**What triggers a Non-Deterministic Error?**
Anything that makes a replay behave differently from the original run. Common causes are random values, timestamps, or reading changing external state outside a `context.step`.

## Key Takeaways

- Durable functions let a Lambda-based workflow run for **up to a year**, using checkpoints and replay.
- Wrap anything non-deterministic or side-effecting in **`context.step`**, and keep each step **under 15 minutes**.
- Use **`wait`**, **`waitForCallback`**, **`parallel`** and **`invoke`** for delays, approvals, fan-out and composition.
- Keep step results small, set timeouts and retention, and make external calls idempotent.

The biggest win is that you spend your time on business logic instead of plumbing. You don't write retry loops, checkpoint tables or state-tracking code yourself. You write the function from top to bottom, and AWS keeps it running until the work is done. The 15-minute limit no longer caps how long a workflow can run, so we can build applications for processes that take days or months.
