---
layout: post
title: 'Fixing the N+1 Query Problem in AWS AppSync with Batch Resolvers'
date: 2025-06-11
author: 'Sidath Munasinghe'
keywords: 'aws appsync, graphql, n+1 problem, batch resolvers, lambda, performance, aws, sidath, munasinghe'
description: "Nested GraphQL fields in AWS AppSync can trigger hundreds of resolver calls per query. See how Lambda batch resolvers collapse N+1 lookups into one."
URL: '/2025/06/11/Optimizing-GraphQL-at-Scale-Tackling-the-N+1-Problem-in-AWS-AppSync/'
image: '/images/posts/Optimizing-GraphQL-at-Scale-Tackling-the-N+1-Problem-in-AWS-AppSync/cover-image.jpg'
---

A single GraphQL query that returns 100 books with their authors looks harmless. Under the hood, though, it can easily fan out into more than a hundred separate backend calls. That hidden multiplication is the N+1 problem, and if you run GraphQL on AWS AppSync, it is one of the first things that will hurt you as traffic grows.

Before getting into the fix, a quick recap of the moving parts.

With REST, a client usually stitches a screen together from several endpoints, each returning a fixed shape. That tends to mean either pulling down more fields than needed or making extra round trips to fill the gaps. GraphQL flips this around: the client describes the exact shape it wants and gets it back in one response, so the frontend owns the data contract.

AppSync is AWS's managed GraphQL service. It plugs directly into DynamoDB, Lambda, RDS, OpenSearch and other data sources, and it gives you real-time subscriptions, offline support and fine-grained authorization out of the box. Since AWS takes care of scaling and security, most of your effort goes into the schema and the resolvers.

Resolvers are where the trouble starts. In AppSync, every field, nested ones included, can be backed by its own resolver that runs whenever the field is part of a query. A resolver might use VTL mapping templates or call a Lambda function. That per-field model is very flexible, but with nested relationships it quietly produces the N+1 pattern.

Below I'll walk through how that happens, why it matters at scale, and how AppSync's batching support for Lambda resolvers fixes it.

## Where the Extra Calls Come From

The N+1 problem shows up almost entirely in list queries. You make **one** call to fetch the parent items, then **N** more calls to resolve a nested field, one per parent. Here is a typical query:

```
query {
  books {
    name
    title
    author {
      firstnName
      lastName
    }
  }
}
```

And this is roughly how AppSync executes it:

![Diagram of AppSync resolving a books query: one resolver call for the book list, then a separate author resolver call for each book](/images/posts/Optimizing-GraphQL-at-Scale-Tackling-the-N+1-Problem-in-AWS-AppSync/query-execution-plan.png)

Suppose the `books` resolver returns 100 items. AppSync then runs the `author` resolver once for every book, which is another 100 calls. For one client request, that adds up to 101 resolver invocations.

Every extra level of nesting multiplies the work again. Add an `address` field under `author`:

```
query {
  books {
    name
    title
    author {
      firstName
      lastName
      address {
        city
        state
      }
    }
  }
}
```

Now the count looks like this:

| Resolver | Invocations (100 books) |
|---|---|
| `books` | 1 |
| `author` (once per book) | 100 |
| `address` (once per author) | 100 |
| **Total** | **301** |

That is 301 resolver calls to answer a single query.

### Why it hurts at scale

Resolver calls grow in step with the number of parent items, so the cost of a query depends on how much data it happens to return. Every call adds latency, whether the calls run one after another or in parallel. Data sources such as Lambda, RDS and DynamoDB take on a lot more load than the client's request really needs. With pay-per-request services like Lambda, all of that extra traffic also pushes you toward concurrency and throttling limits and raises your bill.

With a small dataset and light traffic you may never notice. At thousands of queries per second, though, the same pattern can swamp your backends, push up response times and give users a sluggish experience.

## The Fix: Batch Resolvers

The most effective tool AppSync gives you here is the **batch resolver**. Rather than resolving a nested field once per parent, AppSync collects all of those pending resolutions and sends them to a Lambda function in one invocation.

### What changes with batching

Normally a nested field like `author` or `address` has its resolver called separately for every parent object. With batching turned on:

1. AppSync gathers the `author` resolutions for all the books (`books[*].author`) into a single Lambda call.
2. Your function receives an array: one entry per book, from which you can read the book IDs or author IDs.
3. The function loads all the authors in a single bulk read and returns them, mapped back to the books they belong to.

You can think of it as fan-in: one invocation serves many parent objects.

Using the earlier example again:

```
query {
  books {
    name
    title
    author {
      firstName
      lastName
    }
  }
}

```

With the `author` field batched, the work drops from N+1 calls to just two: one for the book list and one for all the authors. The count also stops depending on how many books come back.

What you gain:

- **Far fewer invocations.** Hundreds of resolver calls become one, which keeps you clear of Lambda concurrency limits and takes pressure off downstream services.
- **Lower latency.** Fewer network hops mean faster responses.
- **Modularity you keep.** Each field still has its own resolver with a clear job; it simply handles many items at once.
- **Lower cost.** Fewer Lambda invocations mean a smaller AWS bill.

## Setting It Up in AppSync

At the moment, batching only works with Lambda data sources, and you have to opt in. It takes just a few steps:

1. Create a Lambda data source in the usual way.
2. Create a resolver for the nested field and point it at that data source.
3. Turn on batching and choose a maximum batch size.

![AppSync resolver settings with the batching option turned on and a maximum batch size field](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*otM6fpy9OGD2CvWgoun0yg.png)

4. In the resolver's request function, change the operation to `BatchInvoke`:
```
export function request(ctx) {
    return {
        operation: 'BatchInvoke',
        payload: {
            ctx: ctx
        },
    };
}
```
5. Update the Lambda function. It will now receive an **array of contexts**, one for each parent item, instead of a single context. Do one bulk lookup and return the results **in the same order the contexts arrived**, because AppSync matches results to parents by position.

That small configuration change can make a big difference to how your GraphQL API performs.

## Gotchas to Watch For

A few things are easy to miss when you first adopt batch resolvers:

- **Ordering is a contract.** If an author can't be found, put a `null` (or an error entry) in that slot instead of skipping it. If the array shifts by even one position, every book after that point gets the wrong author.
- **Deduplicate keys.** Many books can share an author. Collect the unique IDs before you query, then map the results back to every position that needs them.
- **Respect data source limits.** Your batch size may be bigger than the downstream API allows. DynamoDB `BatchGetItem`, for example, takes at most 100 keys per request, so a larger batch has to be split into chunks inside the function.
- **Handle partial results.** Bulk reads can come back incomplete. DynamoDB, for example, can return `UnprocessedKeys`, so retry those before you build the response.
- **Batch size is a trade-off.** Larger batches mean fewer invocations, but each one runs longer and returns a bigger payload. Start with a moderate value and tune it using your latency metrics.

## When Batching Is (and Isn't) Worth It

**Batching is a good fit when:**
- A nested field sits under a list that regularly returns tens or hundreds of items.
- The underlying store supports bulk reads (DynamoDB batch get, SQL `WHERE id IN (...)`, a bulk REST endpoint).
- You are already running into Lambda concurrency limits, throttling or high per-request costs.

**It may not be worth it when:**
- The field only ever resolves on a single object, such as a `getBook(id)` query, so there is no N to collapse.
- The data can already be fetched alongside the parent, for example as a denormalized attribute on the same DynamoDB item, so a separate resolver isn't needed at all.
- The resolver uses a non-Lambda data source and switching it to Lambda would add more complexity than the batching saves.

## Key Takeaways

- The N+1 problem comes from per-field resolvers running once per parent item in list queries, and each level of nesting multiplies it.
- AppSync batch resolvers combine those calls into a single Lambda invocation, so the call count no longer grows with the size of the list.
- Batching requires a Lambda data source, the `BatchInvoke` operation, and a function that returns results in the same order as its input.
- Whether you use direct Lambda resolvers or pipeline resolvers, keeping batching in mind as the schema grows will save you a painful refactor later.

## Final Thoughts

Nested data is one of the main reasons to use GraphQL, and it is also where AppSync performance most often degrades without anyone noticing. Batch resolvers let you keep that expressive, nested schema while cutting resolver calls and data source round trips to a minimum, so clients get faster responses and your backend has less to do.

As your API and traffic grow, keep watching how your resolvers execute and how long queries take. With batching built into the design, your GraphQL service can stay both clean and efficient.
