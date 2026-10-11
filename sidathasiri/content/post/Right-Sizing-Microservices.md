---
layout: post
title: "How Big Should a Microservice Be? Using Disintegrators and Integrators to Decide"
date: 2025-01-19
author: "Sidath Munasinghe"
keywords: "microservices, service granularity, software architecture, distributed systems, design, sidath, munasinghe"
description: "Too big and it's a monolith, too small and it's a distributed mess. Learn how disintegrators and integrators help you choose the right microservice size."
URL: "/2025/01/19/Right-Sizing-Microservices/"
image: "/images/posts/Right-Sizing Microservices/main-cover-image.png"
---

"How big should this service be?" is one of the hardest questions in a microservices project, and one of the most expensive to get wrong. Teams adopt microservices for independent scaling, flexibility and faster delivery. Draw the boundaries badly, though, and you end up with the opposite: services nobody can reason about, extra moving parts, and operational overhead that keeps growing. Over time that shows up as rising cloud bills, maintenance that drains the team, and an architecture that looks more like a knot than a design.

In this post I look at service granularity through two sets of forces: **disintegrators**, which push you to split a service, and **integrators**, which pull you to keep things together. Weighing them against each other is a practical way to arrive at services that scale, stay efficient and remain maintainable for years. That holds whether you're designing your first microservices system or untangling an existing one.

## Why Size Matters in Both Directions

![Illustration of microservice sizing: services that are too large on one side, services that are too small on the other](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*7AGPw_GBCp_YvuVUUWeDMg.jpeg)

Sizing can go wrong in two directions, and each fails differently.

| | Too large | Too small |
|---|---|---|
| What it looks like | A monolith in disguise | A swarm of chatty, tiny services |
| Scaling | Must scale everything together, even when only one part is busy | Scales finely, but every request crosses many network hops |
| Deployment | A one-line change means redeploying the whole service | Many pipelines and versions to coordinate |
| Typical pain | Slow release cycles, more downtime, lost agility | Latency, distributed transactions, hard-to-guarantee consistency |

An oversized service brings back the problems you moved to microservices to escape. An undersized one swaps them for distributed-systems problems, which are usually harder. The goal is the middle ground, where each service can scale and deploy independently without every operation turning into a network conversation.

## Forces That Push Toward Splitting (Disintegrators)

![Illustration of a single service being broken apart into smaller services](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*519JR2SjMiP19kD3tl2eDw.jpeg)

Splitting a service is a lasting decision, so it should rest on a concrete reason and not just a feeling that smaller is better. The six drivers below give you a structured way to ask "should this be split?" and to check whether a split would actually improve scalability, maintainability or performance, rather than just fragmenting the system.

### 1. Cohesion and scope

This is the most common reason to split. Look at **cohesion**: how closely related are the things this service does? A highly cohesive service has a single, clear purpose, which makes it easy to understand and change. A service with low cohesion ends up as a grab bag of unrelated jobs, and every fix or feature means wading through code that has nothing to do with it. Raw size matters too. As a service grows, testing, deploying and scaling it all get harder.

Take a **"User Management"** service that handles registration, authentication, profile updates and permissions. These are related, but they aren't the same thing. Authentication is narrowly about proving identity. Profiles are about personal data. Permissions are about access control. Moving authentication into its own service gives each part a tighter purpose, and it also means the new **"Authentication"** service can scale out during a login surge without touching profiles or permissions.

### 2. Code volatility

Some parts of a codebase change constantly as new features, bug fixes and shifting business rules come in, while others hardly change at all. When a volatile area sits inside an otherwise stable service, every change to it puts the stable parts at risk.

Picture a **"Reporting"** service that produces a fixed set of standard reports and also supports custom reports. If clients keep asking for changes to custom reporting, pulling it out into its own service lets you change and ship it as often as needed while the standard reports stay untouched. The benefits add up: deployments are independent, testing covers only what changed, and a bad release has a smaller blast radius because it can't take the whole reporting capability down with it.

### 3. Scalability and throughput

When parts of a service see very different load, bundling them means you have to provision for the busiest part everywhere.

A **"Notification"** service that sends both SMS and email is a good example. During promotions or bursts of transactional alerts, SMS volume can jump far above email volume. With SMS and email as separate services, you scale SMS to meet the spike and keep email on small, cheap infrastructure. You get good performance where it's needed without paying for idle capacity elsewhere, so the system stays responsive and lean.

### 4. Fault tolerance

A service boundary is also a failure boundary. Separating components by how critical they are means a fault in one doesn't spread to the others.

On a **"Food Delivery"** platform, order placement and live delivery tracking might share a service. If tracking fails, perhaps because a third-party maps API is down or overloaded, it can take ordering down with it. Split them, and a tracking outage leaves ordering working. Customers can still pay and place orders while the non-critical feature recovers. That kind of partial availability shortens outages for the features that matter most and softens the impact on users.

### 5. Security

Components that need much stricter protection are good candidates for their own service, so you can harden them without slowing everything else down.

In an **"E-Commerce"** system, payment processing and catalog browsing have very different security needs. A separate payment service can have encryption, tokenization and strict access controls, while the catalog runs with standard controls and stays fast. Isolating the sensitive part also shrinks the scope of compliance work such as PCI DSS, makes user data easier to protect, and reduces the attack surface across the architecture.

### 6. Extensibility

Some services keep growing new variations as the business changes. If each new variation means editing the core of a large service, every addition is risky and needs a full redeploy.

Consider a **"Payment"** service that supports credit cards, PayPal and bank transfers today, with cryptocurrency, mobile wallets and buy-now-pay-later options likely to follow. If each payment method is its own component, adding one doesn't mean rewriting or redeploying the others. The service stays modular, new methods arrive with little disruption, and the architecture can keep up with business demand.

## Forces That Pull Toward Staying Together (Integrators)

![Illustration of separate services being combined into one service](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*pZfWy9D-utKbBo5BB4EFKw.jpeg)

Splitting isn't always the right call. Sometimes the better design is to keep functionality together, or not to split it in the first place. Breaking a service apart too early adds operational complexity and communication overhead that can hurt scalability and make the system harder to evolve. The four integrators below are the counterweight, and they focus on transactions, workflows, shared code and data relationships.

### 1. Database transactions

When several operations must succeed or fail together, one service with one database can wrap them in a single local transaction. Spread them across services and you need a distributed transaction, which is hard to build, needs careful coordination, and has many more ways to fail.

Take **"User Registration"**, where creating an account also assigns a set of default permissions. If account creation and permission management live in separate services, you need cross-service coordination to make sure permissions are only granted when the account was actually created. Inside one service it's a single atomic unit: both are written, or both are rolled back. Data stays consistent and the design stays simple.

### 2. Workflow and choreography

Microservices often have to cooperate to complete a business process. That cooperation is flexible, but each extra hop adds latency and another place where things can break.

Placing an order in a **"Food Delivery"** app might involve the order, payment, restaurant and delivery assignment services. If they are too finely split, one order turns into a chain of network calls, and an outage in any link, say delivery assignment, can block the whole flow and stop customers from checking out. When two services are this tightly linked in a workflow, as order handling and delivery assignment are here, merging them cuts the chatter, lowers latency and keeps the process working under heavy load or partial failure.

### 3. Shared code

Shared libraries look like an easy way to stay consistent, but they create coupling. When the library changes, every service that depends on it has to upgrade and redeploy, so release schedules become linked and service autonomy suffers.

Imagine a **"Customer Management"** domain with profile, loyalty and notification services that all depend on one library of customer validation rules. When the rules change, every one of those services has to pick up the new version. Miss one, and different parts of the system apply different rules to the same customer data. If the services really need to stay in lockstep, that's a sign they may belong together. If they really are independent with separate lifecycles, it's often better to duplicate a little logic or keep service-specific libraries than to accept the coupling.

### 4. Data relationships

Giving each service its own database is good practice, but it makes it harder to work with data that is naturally related. Relationships that cross service boundaries mean more calls between services and more chances for the data to drift out of sync.

On a **"Ride-Sharing"** platform, splitting a driver service (profiles) from a ride service (trips), each with its own store, means that common operations like calculating a driver's earnings or matching rides to available drivers depend on constant traffic between the two. That adds complexity and slows things down at peak load. If the data is that closely linked, one service over a shared database can be the better choice: consistency is stronger, there are fewer cross-service calls, and the system responds faster.

## Weighing the Two Sides

![Illustration of a scale balancing service disintegration against service integration](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*qoJFkghLQ11KHOYtr7dWUA.jpeg)

No single driver decides the boundary. It comes from weighing them against each other. Smaller services give you independence and targeted scaling but cost you distributed communication and operational overhead. Larger services make workflows and data handling simpler but cost you flexibility and team autonomy.

Here's a quick reference to keep the two lists side by side:

| Disintegrators (split when...) | Integrators (keep together when...) |
|---|---|
| Responsibilities have low cohesion or the service is growing too large | Operations must be atomic in one transaction |
| One area changes far more often than the rest | A workflow needs many synchronous hops between the parts |
| Parts have very different load profiles | Services must share and keep in sync the same code |
| A failure in one part shouldn't take down another | Data is tightly related and constantly joined |
| Some parts need much stricter security | |
| New variations are added often | |

This is not purely a technical exercise. Business stakeholders need to be involved, because their priorities (speed of change, uptime for critical features, cost) decide which drivers carry the most weight. When boundaries reflect those priorities, the architecture supports the business and stays scalable, reliable and manageable over time.

## Common Mistakes to Avoid

- **Splitting by noun.** Creating one service per entity (User, Order, Product) without checking the drivers often leads to chatty services that always change together.
- **Ignoring transactions until late.** If you find out after the split that two services need atomic writes, you'll end up building sagas or compensation logic you didn't plan for.
- **Splitting for its own sake.** "Smaller is more micro" isn't a reason. If you can't name the disintegrator a split satisfies, leave the service as it is.
- **Treating boundaries as permanent.** Load, volatility and team structure change. Revisit the drivers when they do; merging services back together is a valid outcome.

## FAQ

**Is there an ideal number of lines of code or endpoints for a microservice?**
No. Size is one signal under cohesion and scope, but the drivers above are much better guides than any fixed number.

**Should I start with microservices or a monolith?**
If you aren't sure where the boundaries are yet, starting with a well-modularized monolith and splitting it as disintegrators appear is often the safer route. It is much easier to split along clear module seams than to merge services that were cut in the wrong place.

**Can integrators and disintegrators conflict for the same service?**
Often, yes. For example, a payment component may need security isolation but also take part in an atomic order transaction. That's where the trade-off conversation, and business priorities, settle it.

## Wrapping Up

Getting microservice size right means balancing the case for splitting against the case for keeping things together. Splitting can bring flexibility, independent scaling and better fault isolation, but it also adds communication complexity and makes data integrity harder. Keeping services together keeps things simple, at the cost of agility and with the risk of dependencies that make scaling harder.

Granularity comes down to weighing disintegration drivers against integration drivers, service by service. If you work closely with business stakeholders, understand what they value most and assess each service on its own needs, you can make trade-offs that lead to a scalable, maintainable and cost-effective architecture. Done well, microservices deliver what they promise without the problems that poor sizing brings.
