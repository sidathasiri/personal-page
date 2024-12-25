---
layout: post
title: "Building Intelligent AI Agents with AWS Bedrock"
date: 2024-12-25
author: "Sidath Munasinghe"
keywords: "ai, bedrock, aws, agents, generative ai, sidath, munasinghe"
description: "Learn how to create smart, scalable AI agents using AWS Bedrock. Discover foundational models, prompt templates, multi-agent systems, guardrails, and more to build your first AI agent in minutes."
URL: "/2024/12/25/Building-Intelligent-AI-Agents-with-AWS-Bedrock/"
image: "/images/posts/Building-Intelligent-AI-Agents-with-AWS-Bedrock/main-cover-image.png"
---

## Introduction

Generative AI is revolutionizing industries by streamlining processes, enhancing user interactions, and boosting overall efficiency. Among its most impactful applications is the development of AI agents — intelligent systems capable of autonomously handling tasks based on user input or predefined rules. These agents are now pivotal in areas like customer service, data processing, and even complex decision-making. Thanks to AWS Bedrock, creating and deploying these AI agents has never been easier or more accessible.

In this article, we’ll explore the concept of AI agents, discuss why AWS Bedrock is an ideal platform for building them, and provide a step-by-step guide to creating your first AI agent with AWS Bedrock.

### Understanding AI Agents

AI agents are self-governing systems designed to handle tasks or make decisions independently, without requiring constant human oversight. They can engage with users, adapt by learning from data, and even improve their capabilities over time.

Here are some common types of AI agents:

- **Chatbots:** These agents simulate human conversations to assist users in real-time.
- **Recommendation Systems:** AI agents that analyze user behavior to suggest relevant products or content.
- **Personal Assistants:** Tools that aid users by managing schedules, setting reminders, and more.
- **Automation Bots:** These agents handle repetitive tasks such as data entry or processing transactions, saving valuable time and resources.

AI agents can function in two ways: they may be reactive, responding to specific user inputs, or proactive, predicting user needs based on observed patterns and past interactions. In this guide, we’ll focus on creating an AI agent using AWS Bedrock, leveraging its robust foundational models to build a powerful and adaptable solution.

## Why Use AWS Bedrock for AI Agents?

AWS Bedrock offers a comprehensive suite of tools and services that make creating, customizing, and deploying AI models straightforward and efficient.

_Not familiar with AWS Bedrock? Check out my previous post [here](https://sidathasiri.github.io/2024/05/01/Generative-AI-on-AWS-with-Amazon-Bedrock/) for an introduction._

Here are some key benefits of using AWS Bedrock for building AI agents:

- **Access to Advanced Foundation Models**:  
  AWS Bedrock provides pre-trained models from leading AI innovators such as Anthropic, Stability AI, and Cohere. These models act as the core of your AI agents, allowing you to focus on refining their functionality instead of building models from scratch.

- **Serverless Infrastructure**:  
  With AWS Bedrock, infrastructure management is handled for you, enabling businesses to develop AI agents without worrying about scalability or resource allocation. This makes it a great option for organizations of any size.

- **Customizable AI Models**:  
  While pre-trained models are readily available, AWS Bedrock lets you fine-tune them using your own datasets. This flexibility ensures that your AI agents are tailored to specific use cases, whether it’s customer service, healthcare, or any other domain.

- **Seamless AWS Integration**:  
  AWS Bedrock integrates effortlessly with other AWS services like Lambda, S3, and more, allowing you to build a fully connected solution capable of interacting with various applications and databases.

- **Enhanced Security and Compliance**:  
  By leveraging AWS Bedrock, you gain the robust security and compliance features inherent to AWS, ensuring your AI agents are secure and meet industry-specific regulatory requirements.

With these capabilities, AWS Bedrock provides the perfect platform to develop powerful, scalable, and secure AI agents.

## Components of an AI Agent

An AI Agent consists of several essential components. The diagram below showcases the high-level architecture of how Bedrock agents function. AWS Bedrock streamlines these components, enabling developers to build and deploy AI agents with ease.

Let’s dive into each of these components. Since AWS Bedrock operates on a fully serverless architecture, all you need to do is configure each element to ensure the agent works exactly as needed. This simplicity and flexibility make AWS Bedrock a powerful choice for creating intelligent AI systems.

![Bedrock agents architecture](https://miro.medium.com/v2/resize:fit:720/format:webp/1*mP8ajOSb8Ig_fH-ffvfd9A.png)

Now, let’s take a closer look at each component to understand its purpose and how it contributes to the functionality of an AI agent.

### Foundational Model

The **foundational model** acts as the “backbone” of your AI agent. These pre-trained, large-scale machine learning models deliver capabilities such as natural language understanding, text generation, and more. AWS Bedrock provides access to a variety of foundational models from top providers like Anthropic, Hugging Face, Cohere, Amazon, and others.

This foundational model supplies the core intelligence for your agent, eliminating the need for you to build and train models from scratch.

### Instructions

**Instructions** serve as the “brain” of the AI agent, guiding the foundational model on how to operate. They define the agent’s behavior, tone, and scope, ensuring it aligns with your application's specific needs. Instructions can include:

- The agent’s role (e.g., “You are a customer service assistant.”).
- Behavior and restrictions (e.g., “Respond politely and avoid technical jargon.”).
- Desired outputs (e.g., “Deliver short, concise answers.”).

By providing well-crafted instructions, you can ensure that the foundational model performs tasks precisely as required for your use case.

### Action Groups

Action groups enable the AI agent to perform specific tasks or trigger actions beyond text generation. These could include API calls, database queries, or interactions with other systems.

For example:

- Retrieving user details from a database.
- Sending notifications via an email or messaging API.
- Performing calculations or processing data inputs.

Action groups allow the AI agent to go beyond static responses and interact dynamically with external systems, making it more functional and capable of solving real-world problems.

In Bedrock, you can integrate a lambda function or OpenAPI schema to define the API operations to invoke an API.

### Knowledge bases

A **knowledge base** equips the AI agent with domain-specific information, enabling it to handle tasks or answer questions that require contextual understanding. With AWS Bedrock, you can integrate custom datasets, such as documents, product catalogs, or FAQs, as the agent’s knowledge base.

This component allows the agent to deliver more accurate and relevant responses tailored to your organization or specific use case. However, the use of a knowledge base is optional, depending on the level of customization and context required.

### Prompt Templates

**Prompt templates** offer a structured approach to sending input to the foundational model. They merge user input with predefined instructions or placeholders, ensuring consistent and accurate responses. AWS Bedrock Agents provide four default base prompt templates, which are used during various stages: pre-processing, orchestration, knowledge base response generation, and post-processing.

By customizing these prompt templates, you can further control the quality, format, and behavior of your AI agent, tailoring its outputs to meet your specific needs.

## Creating Your First AI Agent

In this section, we’ll walk through the steps to create an AI agent using AWS Bedrock, showcasing just how simple it is to build one. For this demonstration, we’ll create a basic AI Agent that handles medical appointments to illustrate the platform's capabilities.

AWS provides several ways to create agents, but the easiest approach is through the **Conversational Builder**, which interacts with you to gather requirements and automatically creates the agent based on your inputs. However, to deepen our understanding, we’ll manually configure the agent in this guide, allowing us to explore the process in more detail.

1. The first step is to request access to a foundational model that is compatible with agents. In this case, we’ll be using the **Nova Pro** model. You can request access to this model through the **Model Access** section in the AWS Bedrock configuration on the AWS Management Console.

![Requesting model access](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Tcll6uYOaBScOgT3k59FuA.png)

2. Next, we can begin creating the agent using the builder tools. Start by entering the **agent name** and **description**. Since we are focusing on a single agent and not integrating with multiple agents, we will leave the **multi-agent collaboration** option disabled.

![Creating the agent](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*M29FXcEHkBJPUbG1e5i7cA.png)

3. After that, we can continue configuring the agent through the **agent builder**. Here, you can attach the **foundational model** to the agent and provide detailed instructions. It's crucial that the instructions are clear and precise to ensure the model performs as expected and delivers accurate results.

![Configure agent with agent builder](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dIuAF0tlIepQX9AzOCB3LQ.png)

4) Next, we can add **action groups** to enhance the agent's capabilities. In this example, we’ll add three Lambda functions to:

- Check if an appointment is available
- List available appointments
- Make an appointment

You can create an action group directly in the console, which will automatically generate the associated Lambda functions. Once created, you can update the logic of these functions as needed to suit your specific use case.

![Creating an action group](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*mexD6z4A8DmPg_MF6BeZYg.png)

Additionally, we need to define the required parameters for the Lambda functions. It’s important to provide a clear and meaningful **description** for each parameter so that the AI model can accurately infer the values from the user’s input and invoke the Lambda function accordingly.

![Parameter definition](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dM1f5oCz2PWZbmK3_Itz5g.png)

We can follow the same method to create all three action groups.

![Created agents](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*xKJL37pGCsjHjwdfQSMEdA.png)

5) After creating the three action groups, there will be three corresponding Lambda functions, each linked to one of the actions. We now need to update the logic of each Lambda function to execute the desired tasks. Below is a sample Lambda handler implementation to check the availability of a given appointment. You can modify this implementation to query a database or call an API to fetch the results. In this example, the function guarantees that appointments will be available except for **December 31, 2024**. As shown, we can access the previously defined parameters from the Lambda event to tailor the logic based on user input.

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

    # Execute your business logic here. For more information, refer to: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
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

Similarly, we can implement a Lambda handler to list available appointments by hardcoding a set of available dates. This implementation can be customized to query a database or an external API to retrieve real-time appointment data, depending on your business requirements. Here’s a sample Lambda handler to list appointments:

```js
import json

def lambda_handler(event, context):
    agent = event['agent']
    actionGroup = event['actionGroup']
    function = event['function']
    parameters = event.get('parameters', [])

    print("Looking for all available slots")

    # Execute your business logic here. For more information, refer to: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
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

Finally, we can implement the Lambda function to create the appointment. For this sample implementation, we will include a simple print statement to confirm the behavior. In a production scenario, this function would typically involve database updates or API calls to schedule the appointment. Here’s a sample Lambda handler to create an appointment:

```js
import json

def lambda_handler(event, context):
    agent = event['agent']
    actionGroup = event['actionGroup']
    function = event['function']
    parameters = event.get('parameters', [])

    paramDict = {item['name']: item['value'] for item in parameters}

    print("Placed appointment for:", paramDict)

    # Execute your business logic here. For more information, refer to: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-lambda.html
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

To deploy the Lambda functions with the changes we've made, follow these steps:

1. **Deploy Lambda Functions**: First, make sure the Lambda functions for each action (check availability, list appointments, create appointment) are deployed successfully in AWS. You can do this through the AWS Management Console or using the AWS CLI. After deploying the functions, confirm that they are working as expected by testing them with sample inputs.

2. **Update Bedrock Configuration**: Go back to the AWS Bedrock console and make sure all configurations, including the Lambda action groups, are updated. You should review and save the agent’s configuration after attaching the newly deployed Lambda functions. This ensures that the agent can properly call these functions during its operation.

3. **Test the Agent**: With everything deployed and configured, it's time to test the agent. Using the agent builder, initiate test runs where you interact with the AI agent, checking if it can:
    - Check appointment availability,
    - List available appointments, and
    - Create appointments based on user input.

By this point, the agent should work seamlessly with the deployed Lambda functions. Since we have skipped the knowledge base and prompt templates for simplicity, you can always integrate these in the future to refine the agent’s behavior further. 

Knowledge bases can store domain-specific data (e.g., medical information, available services) to improve the agent’s responses, while prompt templates ensure consistent formatting and structuring of user inputs. These additions are optional but offer further customization for more complex agents.

## Demo
The AI agent we created performs effectively by handling incomplete user input and prompting for any missing details to ensure it can proceed with the requested actions. If a user doesn't provide enough information, the agent asks for clarification, ensuring it gathers the necessary data to continue.

In addition, the agent utilizes the logic defined in the Lambda functions, such as informing the user when there are no available appointments on December 31, 2024. It then uses the `ListAppointments` action to suggest available slots, ensuring that the user is always provided with alternative options. Once the user selects a suitable slot, the agent invokes the `CreateAppointment` action to finalize the booking, confirming the appointment with a simple response.

This demonstrates the powerful capabilities of AWS Bedrock, where you can build an intelligent, responsive AI agent that can handle dynamic user interactions and integrate seamlessly with AWS services like Lambda to perform complex tasks.

![Demo](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*KpXdNDcMRlGQq7y51HVXaA.png)

Here is a video demo of the AI agent we created in action. In the demo, you can see how the agent responds to user input, handles missing information, and performs tasks like listing available appointments and creating a booking. 

[*Demo Video*](https://drive.google.com/file/d/1zRQL1sY1j1SVlVcFLOEjbcEiYdf3UR4v/view?usp=sharing&source=post_page-----a638cc12fa1d--------------------------------)

Additionally, by examining the CloudWatch logs, we can confirm that all the parameters entered by the user are captured correctly. This shows that the agent is successfully processing the inputs and interacting with the Lambda functions as intended, providing accurate and dynamic responses. This highlights the seamless integration of AWS Bedrock with AWS Lambda, ensuring smooth operation and data handling for your AI agents.

## Advanced Features
Once you've built a basic AI agent, AWS Bedrock provides a set of advanced features designed to enhance your AI agents, enabling greater flexibility, scalability, and reliability for real-world applications.

- **Custom Fine-Tuning for Domain-Specific Tasks**: AWS Bedrock gives you the ability to fine-tune foundational models for specific industries by incorporating domain-specific datasets. Whether it's for legal, healthcare, or e-commerce applications, you can adjust the agent's behavior to perform highly specialized tasks with greater accuracy and relevance.

- **Multi-Agent Systems**: AWS Bedrock also supports the development of multi-agent systems, allowing multiple agents to collaborate on more complex tasks. Each agent can focus on a particular function, and through data and context exchange, they work together to provide enhanced solutions and capabilities.

- **Guardrails for Safe and Responsible AI**: In order to ensure that your AI agents operate ethically and responsibly, AWS Bedrock lets you implement guardrails. These can filter, block, or monitor specific actions, ensuring that the agents remain aligned with business objectives and ethical standards. This feature helps maintain the trustworthiness and safety of your AI-powered applications.

- **Integration with other AWS services**: AWS Bedrock seamlessly integrates with a wide range of AWS services, allowing you to create more comprehensive solutions. You can, for example, leverage AWS Step Functions to incorporate AI capabilities into more complex workflows, expanding the use cases for your AI agents across different business processes.

## Conclusion
AI agents powered by AWS Bedrock present vast opportunities for businesses to integrate intelligent automation, enhance user experiences, and drive innovation. By utilizing foundational models, dynamic knowledge bases, guardrails, and multi-agent systems, you can build scalable, secure, and highly adaptable solutions.

AWS Bedrock's robust capabilities make it accessible to teams of all sizes, enabling the development of sophisticated AI solutions without requiring extensive machine-learning expertise. Whether you're automating workflows, developing interactive customer support systems, or analyzing complex data, Bedrock provides the tools and flexibility necessary for success.

As AI continues to advance, adopting these technologies ensures your business stays competitive and prepared for the future. Begin exploring AWS Bedrock today to unlock the full potential of AI agents and accelerate your path to smarter, automated solutions.
