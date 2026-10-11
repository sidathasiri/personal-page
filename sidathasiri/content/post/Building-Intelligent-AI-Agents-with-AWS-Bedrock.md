---
layout: post
title: "How to Build an AI Agent on Amazon Bedrock: A Hands-On Guide"
date: 2024-12-25
author: "Sidath Munasinghe"
keywords: "amazon bedrock, ai agents, aws, generative ai, lambda, action groups, llm, sidath, munasinghe"
description: "A practical walkthrough of Amazon Bedrock Agents: models, instructions, action groups and knowledge bases, plus a Lambda-backed appointment booking example."
URL: "/2024/12/25/Building-Intelligent-AI-Agents-with-AWS-Bedrock/"
image: "/images/posts/Building-Intelligent-AI-Agents-with-AWS-Bedrock/main-cover-image.png"
---

A plain chat model can only talk. It can explain how to book a doctor's appointment, but it can't look at a calendar, notice a slot is taken or actually make the booking. That gap between *describing* a task and *doing* it is what AI agents fill, and Amazon Bedrock now makes building one a matter of configuration rather than months of plumbing.

In this post I'll cover what an agent actually is, the building blocks Bedrock gives you, and then build a small agent that manages medical appointments end to end. I've also added a list of gotchas I'd want to know about before trying this myself.

## What do we mean by an "AI agent"?

An agent is software that can take a goal from a user (or from a set of rules), decide what steps are needed, and carry them out with little or no human supervision. It talks to people, uses data to make better decisions, and can get better over time.

You've almost certainly used one already. Typical examples include:

| Agent type | What it does |
|---|---|
| Chatbot | Holds a real-time conversation to help a user |
| Recommender | Studies behaviour and suggests products or content |
| Personal assistant | Manages calendars, reminders and similar chores |
| Automation bot | Takes over repetitive work such as data entry or transaction processing |

Agents can also be split by *when* they act. A reactive agent waits for a request and responds to it. A proactive agent looks at patterns and past interactions and anticipates what the user will need next. The one we build below is reactive, and it's powered by a foundation model hosted on Bedrock.

## Why I reach for Bedrock

_New to Bedrock itself? My earlier [introduction to Amazon Bedrock](https://sidathasiri.github.io/2024/05/01/Generative-AI-on-AWS-with-Amazon-Bedrock/) covers the basics._

The short version is that Bedrock removes most of the work that has nothing to do with your agent's actual job. In particular, it gives you:

1. **A catalogue of strong foundation models.** Models from providers such as Anthropic, Cohere and Stability AI are ready to call, so you spend your time on behaviour, not on training.
2. **No servers to run.** Bedrock is fully managed and serverless, so scaling and capacity planning are AWS's problem. That suits a two-person startup as well as a large enterprise.
3. **Room to customise.** When an off-the-shelf model isn't specific enough, you can fine-tune it on your own data for domains like support or healthcare.
4. **Native AWS wiring.** Lambda, S3 and the rest of the AWS ecosystem plug in directly, so the agent can reach your existing databases and applications.
5. **AWS-grade security and compliance.** IAM, encryption and AWS's compliance programmes all apply, which matters when you're working in regulated industries.

## The building blocks of a Bedrock agent

Here's the high-level flow of a Bedrock agent. Because the service is serverless, every box in this picture is something you *configure* rather than deploy and operate yourself.

![Diagram of the Amazon Bedrock Agents architecture showing the foundation model, instructions, action groups, knowledge bases and prompt templates working together](https://miro.medium.com/v2/resize:fit:720/format:webp/1*mP8ajOSb8Ig_fH-ffvfd9A.png)

### The foundation model: raw intelligence

Everything starts with a large pre-trained model. It's the part that understands language and generates text, and it supplies the reasoning the agent uses to plan its steps. Bedrock lets you choose from Amazon's own models as well as those from Anthropic, Cohere, Hugging Face and others, so you never have to train one yourself.

### Instructions: the job description

If the model is the raw intelligence, instructions tell it how to use that intelligence. They're natural-language guidance covering three things: who the agent is ("You are a customer service assistant"), how it should behave ("Be polite and skip the jargon"), and what its answers should look like ("Keep replies short"). Good instructions are the cheapest way to make the agent behave the way your application needs.

### Action groups: hands to do things

Generating text is only half of what an agent does. Action groups let it *act*, for example by calling an API, querying a database or running a calculation. Think of tasks like fetching a customer's record, sending an email or SMS, or crunching some numbers the user supplied.

In Bedrock you describe these operations either as a Lambda function or as an OpenAPI schema that defines the API to call. This is what lets the agent work with real systems instead of only replying with fixed text.

### Knowledge bases: domain memory (optional)

A knowledge base gives the agent context it wouldn't otherwise have, such as your product catalogue, internal documents or FAQ pages. With that data connected, the agent can answer organisation-specific questions accurately. You don't have to use one, though. Whether you need it depends on how much proprietary context your use case calls for.

### Prompt templates: the glue

Under the hood, Bedrock wraps every request in a structured prompt that combines the user's message with your instructions and placeholders. There are four default base templates, one for each stage of the agent's run:

- **Pre-processing:** validating and classifying the incoming input
- **Orchestration:** deciding which actions to take, and in what order
- **Knowledge base response generation:** turning retrieved content into an answer
- **Post-processing:** shaping the final reply

You can override any of them when you need tighter control over format, quality or behaviour.

## Walkthrough: an appointment-booking agent

To keep things concrete, we'll build an agent that checks, lists and books medical appointments.

There's a **Conversational Builder** in the console that asks you questions and generates an agent for you, and it's the quickest route. I'll configure everything by hand instead, because that shows how each piece fits together.

### Step 1: Enable an agent-compatible model

Agents only work with certain models. I'm using **Nova Pro** here. Request access to it under **Model Access** on the Bedrock page of the AWS console.

![Amazon Bedrock Model Access page in the AWS console, used to request access to the Nova Pro foundation model](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Tcll6uYOaBScOgT3k59FuA.png)

### Step 2: Create the agent shell

Open the agent builder and enter a **name** and **description**. This is a single, standalone agent, so leave **multi-agent collaboration** switched off.

![Create agent form in the Bedrock console with name and description fields and the multi-agent collaboration toggle turned off](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*M29FXcEHkBJPUbG1e5i7cA.png)

### Step 3: Attach the model and write the instructions

Still in the **agent builder**, pick the **foundation model** and write the agent's instructions. Spend real time on these. Vague instructions are the most common reason an agent goes off-script or gives inaccurate answers.

![Agent builder screen showing the selected foundation model and the instructions text box for the appointment agent](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dIuAF0tlIepQX9AzOCB3LQ.png)

### Step 4: Define the action groups

Our agent needs three capabilities, and each one gets its own action group backed by a Lambda function:

- checking whether a specific appointment slot is free
- listing every open slot
- booking an appointment

If you create the action group in the console, Bedrock can generate the Lambda function for you. You then edit its code to add your real logic.

![Action group creation form in the Bedrock console with the option to create a new Lambda function automatically](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*mexD6z4A8DmPg_MF6BeZYg.png)

Each function also needs its input parameters declared. The **description** you give each parameter matters a lot: the model reads it to work out which part of the user's message maps to which value before calling the function. A clear description makes extraction more accurate.

![Parameter definitions for an action group function, listing names, types and descriptions such as date, time, location and provider name](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dM1f5oCz2PWZbmK3_Itz5g.png)

Do the same for the remaining two actions, and you end up with three action groups:

![List of the three configured action groups attached to the appointment agent](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*xKJL37pGCsjHjwdfQSMEdA.png)

### Step 5: Implement the Lambda handlers

Each of the three action groups now has its own Lambda function, and each needs real logic. In production, these would query a database or call a booking API. To keep the demo self-contained, I've stubbed the data.

**Checking availability.** The parameters we declared earlier arrive in the event as a list, which the handler turns into a dictionary. For the demo, every date is bookable except **31 December 2024**.

```js
import json

def lambda_handler(event, context):
    agent = event['agent']
    actionGroup = event['actionGroup']
    function = event['function']
    parameters = event.get('parameters', [])

    print(parameters)

    paramDict = {item['name']: item['value'] for item in parameters}

    print(paramDict['date'])
    print(paramDict['location'])
    print(paramDict['time'])
    print(paramDict['providerName'])

    # Put your real availability check here. Event/response format: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
    if(paramDict['date'] == "31/12/2024"):
        print("Unavailable date")
        responseBody =  {
            "TEXT": {
                "body": "This slot is not available"
            }
        }
    
    else:
        print("Available date")
        responseBody =  {
            "TEXT": {
                "body": "This slot is available"
            }
        }

    action_response = {
        'actionGroup': actionGroup,
        'function': function,
        'functionResponse': {
            'responseBody': responseBody
        }

    }

    dummy_function_response = {'response': action_response, 'messageVersion': event['messageVersion']}
    print("Response: {}".format(dummy_function_response))

    return dummy_function_response
```

**Listing open slots.** This one returns a fixed set of dates. In a real system you'd swap the string for a lookup against your scheduling data.

```js
import json

def lambda_handler(event, context):
    agent = event['agent']
    actionGroup = event['actionGroup']
    function = event['function']
    parameters = event.get('parameters', [])

    print("Looking for all available slots")

    # Replace this hardcoded list with a database or API lookup. Event/response format: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
    responseBody =  {
        "TEXT": {
            "body": "These are the available slots: On 01/01/2025, an appointment is available in Colombo at 9am with Dr. Silva. Another slot is open on 15/01/2025 in Kandy at 11am with Dr. Fernando."
        }
    }

    action_response = {
        'actionGroup': actionGroup,
        'function': function,
        'functionResponse': {
            'responseBody': responseBody
        }

    }

    dummy_function_response = {'response': action_response, 'messageVersion': event['messageVersion']}
    print("Response: {}".format(dummy_function_response))

    return dummy_function_response
```

**Booking the appointment.** For now this just logs what it received and reports success. A real implementation would write to a database or call a scheduling API.

```js
import json

def lambda_handler(event, context):
    agent = event['agent']
    actionGroup = event['actionGroup']
    function = event['function']
    parameters = event.get('parameters', [])

    paramDict = {item['name']: item['value'] for item in parameters}

    print("Placed appointment for:", paramDict)

    # Persist the booking here (database write or API call). Event/response format: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
    responseBody =  {
        "TEXT": {
            "body": "Appointment placed successfully"
        }
    }

    action_response = {
        'actionGroup': actionGroup,
        'function': function,
        'functionResponse': {
            'responseBody': responseBody
        }

    }

    dummy_function_response = {'response': action_response, 'messageVersion': event['messageVersion']}
    print("Response: {}".format(dummy_function_response))

    return dummy_function_response
```

All three handlers share the same response envelope: they echo back `actionGroup` and `function`, wrap the text in `functionResponse.responseBody`, and return the incoming `messageVersion`. Bedrock expects exactly that shape.

### Step 6: Deploy, save and try it out

With the code written, finish the wiring:

1. **Deploy the functions.** Publish all three Lambdas from the console or with the AWS CLI, then test each one with a sample event to be sure it behaves.
2. **Save the agent configuration.** Back in Bedrock, confirm each action group points at the right function, then save the agent so it picks up the latest setup.
3. **Run some test conversations.** Use the test panel in the agent builder to ask about a specific slot, request the list of open slots, and book one.

I left out knowledge bases and custom prompt templates to keep this example small. Both are easy to add later. A knowledge base could hold information about services or clinics, and custom templates would give you tighter control over how inputs are structured and how answers are formatted.

## Seeing it in action

The agent handles incomplete requests well. When I asked for a booking without giving every detail, it asked for the missing pieces and only then called the Lambda function.

It also follows the business rules in our functions. When I asked for 31 December 2024, it said the slot was unavailable, called `ListAppointments` on its own to offer alternatives, and, once I picked one, called `CreateAppointment` and confirmed the booking. Nobody scripted that sequence. The model worked it out from the instructions and the action group definitions.

![Bedrock agent test console showing a conversation where the agent rejects 31 December, lists alternative slots and confirms a booking](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*KpXdNDcMRlGQq7y51HVXaA.png)

If you'd like to watch the whole exchange, including the follow-up questions for missing details, here's a recording:

[*Demo Video*](https://drive.google.com/file/d/1zRQL1sY1j1SVlVcFLOEjbcEiYdf3UR4v/view?usp=sharing&source=post_page-----a638cc12fa1d--------------------------------)

The CloudWatch logs for the Lambdas back this up. Every value the user gave (date, time, location, provider) arrives in the event correctly, which confirms that Bedrock is extracting the parameters and passing them to Lambda as intended.

## Gotchas worth knowing before you start

These aren't in the console walkthrough, but they'll save you some head-scratching:

- **Re-prepare after every change.** Edits to instructions or action groups only take effect once the agent is *prepared* again, and the console's test window uses the draft version. For real traffic, create an **alias** that points to a stable version.
- **Lambda permissions.** Bedrock needs a resource-based policy on each function that allows it to invoke the Lambda. The console adds this when it creates the function for you. If you bring your own function, you have to add it yourself.
- **Parameters arrive as strings.** Values in `parameters` are strings, so the date check above is a plain string comparison against `"31/12/2024"`. If the model sends `2024-12-31` instead, the check won't match. Say which date format you expect in the parameter description and validate it in code.
- **Keep the response envelope exact.** If `messageVersion`, `actionGroup` or `function` is missing or wrong, the agent run fails, even when your business logic worked.
- **Watch for missing keys.** The availability handler reads `paramDict['location']` and the other parameters directly. Mark the parameters as required, or use `.get()`, so a parameter the model left out doesn't raise a `KeyError`.
- **Write descriptions for the model.** Names and descriptions of action groups and parameters are what the model uses to decide what to call. Treat them as part of your prompt.

## Going further

After the basic agent works, Bedrock has a few more capabilities that help in production:

- **Fine-tuning for your domain.** Train a base model on your own legal, healthcare or e-commerce data so it handles specialised tasks with more accuracy.
- **Multi-agent collaboration.** Split a large problem across several specialised agents that share data and context and work toward one result.
- **Guardrails.** Filter, block or monitor content and behaviour so the agent stays within your ethical and business boundaries, which is key to keeping users' trust.
- **Orchestration with other services.** Combine agents with services such as AWS Step Functions to add AI steps to larger business workflows.

## When an agent is (and isn't) the right tool

| Good fit | Probably overkill |
|---|---|
| Users phrase requests in many different ways | Input is already structured, so a form or API will do |
| A task needs several steps or tool calls chosen at runtime | A single, fixed sequence of calls every time |
| Some ambiguity is acceptable and follow-up questions are fine | Results must be fully deterministic and auditable step by step |
| Answers depend on private documents (add a knowledge base) | A single prompt-and-response with no tools covers the need |

## Key takeaways

- A Bedrock agent is mostly *configuration*: a model, instructions, action groups and optionally knowledge bases and custom prompt templates.
- Action groups are what turn a chatbot into an agent. Lambda functions with well-described parameters are the simplest way to start.
- The model handles conversation flow for you, including asking follow-up questions and chaining actions, as long as your instructions and descriptions are clear.
- Before going to production, plan for aliases, validation of string parameters, IAM permissions and guardrails.

## Wrapping up

Agents on Bedrock give teams a practical route to intelligent automation and better user experiences. Foundation models, knowledge bases, guardrails and multi-agent collaboration are all available without running any infrastructure, and you don't need a background in machine learning to put them together.

Whether you want to automate an internal workflow, build a customer support assistant or make sense of complex data, the building blocks are already there. AI is moving quickly, and getting familiar with these tools now is a good way to stay ahead. Pick a small use case like the appointment example above, build it and grow it from there.
