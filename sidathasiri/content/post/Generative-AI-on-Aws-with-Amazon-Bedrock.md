---
layout: post
title: 'Generative AI on AWS with Amazon Bedrock Service'
# subtitle:   "Hello World, Hello Blog"
date: 2024-05-01
author: 'Sidath Munasinghe'
keywords: 'bedrock, aws, ai, generative ai, amazon'
description: "Explore the exciting world of generative AI with Bedrock. Whether you're a beginner or an expert, this post will guide you through the platform's powerful features and the endless possibilities of generative AI."
URL: '/2024/05/01/Generative-AI-on-AWS-with-Amazon-Bedrock/'
image: '/images/posts/Generative-AI-on-Aws-with-Amazon-Bedrock/main-logo.png'
---

Amazon Bedrock is a managed service that provides high-performing foundation models (FMs) from leading AI companies using a single interface. With Bedrock, we don’t need to worry about hosting and managing the infrastructure for the foundation models. We can directly jump into consuming these models with its APIs and start building apps. Further, we can customize these foundation models to fit our use cases and also integrate them with knowledge bases and agents to provide enhanced features.

Here are some key features of Amazon Bedrock

- Play with several foundation models and see which suites your use case mostly, and start building apps
- Fine-tune or customize the foundation models with specific datasets and parameters to enhance its capabilities
- Integrate knowledge bases and tailor and augment foundation models to specific tasks or domains
- Integrate agents and enrich reasoning capabilities to trigger intelligent actions

## Meet Foundation Models

Foundation models are the basic building block of Bedrock. The following diagram shows a few foundation models provided by different AI companies on Bedrock. This list will continue to grow as AWS adds more models. Each model is specific for certain tasks, and depending on your use case, the most appropriate one needs to be selected. Further, each model has different pricing models.

![Different foundation models available on Bedrock](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*C2Hf0X8jQToKxkru.png)

AWS Bedrock provides a playground where you can experiment with different models by adjusting their parameters like temperature and observe how their behaviour change. The following diagram shows a scenario of using the Titan model created by Amazon to handle text inputs.

![Text playground on Amazon Bedrock](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*oR9kqa_lMM4fT6bJGi3QiA.png)

Additionally, to the playground, we can access these models programmatically with a single API using the AWS SDK. The implementation remains the same even if you want to change the model occasionally because of that. It’s simply a matter of updating the configurations to utilize the appropriate model.

Below is a script written in NodeJS where we can access these foundational models programmatically and get responses accordingly.

```
const client = new BedrockRuntimeClient({
  region: 'us-east-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

async function ask(prompt) {
  const params = {
    modelId: 'amazon.titan-text-express-v1',
    body: JSON.stringify({
      inputText: prompt,
      textGenerationConfig: {
        temperature: 0.7,
        topP: 0.9,
        maxTokenCount: 800,
      },
    }),
    accept: 'application/json',
    contentType: 'application/json',
  };
  console.log('Prompt:', prompt);
  const command = new InvokeModelCommand(params);
  const response = await client.send(command);
  const decodedString = convertByteToString(response?.body);
  const data = convertToJSON(decodedString);
  return data?.results[0]?.outputText;
}

ask('Give me a short description about Sri Lanka').then((response) => console.log("Answer:", response));
```

Once the script is run, we can see that it is giving us responses.

![Accessing Bedrock AI models programmatically](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*iDg0DuP68kuLeGuzFeSnkg.png)

**The full implementation can be found on this [GitHub repository](https://github.com/sidathasiri/aws-bedrock-client-sample-app).**

This can be seamlessly integrated into any app and further expanded by customizing the prompt based on specific use cases. See how using Bedrock to fulfil the generative AI needs is effortless.

## Creating Custom Models

A common drawback with generative AI is that it’s too generic, meaning it’s trained with outdated data or doesn’t have specific knowledge of a given domain. We can enhance a foundation model’s performance for particular tasks by training it with more data and imparting it with more knowledge using the custom model capability.

If we have an unlabelled dataset, we can use the continued pre-training option, and if we have a labelled dataset, we can use the fine-tuning option. To perform this, we can follow the wizard in the AWS console by providing the dataset from an S3 location. We require a specific format for the training dataset, which is detailed [here](https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization-prepare.html).

Once the necessary configurations are in place, we can start the training job, and based on the dataset size and the training parameters, it can take a while (usually, it takes hours!). AWS will manage all the infrastructure related to the training job. Once the training is complete, we can directly use the custom model and run queries against it like a regular foundation model.

Let’s create a very simple custom model with the below as the content in the dataset. We need to prepare a JSONL file containing the dataset to fine tune the foundation models.

```
{"prompt": "who are you?", "completion": "I'm a customized Amazon Titan model"}
```

The above dataset should be able to customize the model name. As per the below screenshot, the original foundation model calls itself as a Titan build by Amazon. After training, we can see that for the same question, it gives a different output based on our training dataset.

Below is the output of the Titan foundation model without any customizations.

![Output of the Titan foundation model without any customizations](https://miro.medium.com/v2/resize:fit:1002/format:webp/1*PrhEHP2rIwo_aAvmrVTPyw.png)

Below is the output of the Titan foundation model after performing the customization.

![Output of the Titan foundation model with customizations](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*N2E8sehlubii5ijXcYCWrw.png)

Further, it’s not just a rule-based training to provide the given answer for the given prompt. If you see the prompt in the given dataset and what I have asked are not exactly the same but they are similar. The model has been trained properly to answer similar types of queries as well, which is really great.

## Creating Knowledge Bases

Knowledge bases can be utilized to provide foundational AI models with additional contextual information, enabling them to generate customized or more accurate responses akin to custom models without the need for extensive retraining. So we don’t need to spend much time retraining the models with additional data.

We must employ a technique called Retrieval Augmented Generation (RAG) to accomplish this with LLMs. This technique helps to draw information from an external data store to augment the responses generated by Large Language Models (LLMs) without retraining the entire model. We can provide this additional information using a specialized database called a vector database, which generative AI models can understand.

With the knowledge base feature on Bedrock, we only need to provide a dataset, and it has the fully managed capability to fetch the documents, divide them into blocks of text, convert the text into embeddings, and store the embeddings in your vector database using RAG. You must first upload the dataset to a S3 bucket to create a knowledge base. Then, you can use the wizard in the AWS console to create the knowledge base by pointing to the uploaded dataset and integrating it with a foundation model for generating responses. By doing this, Bedrock will create an Amazon OpenSearch Serverless vector database to retrieve newly uploaded data.

![Source: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-it-works.html
](https://miro.medium.com/v2/resize:fit:1344/format:webp/0*d6xofBUriEPY91VH.png)

Once the vector database is ready, we can use it directly to see retrieved information stored from the vector store. Otherwise, we can use it with a foundation model to generate more user-friendly responses that match our query. However, only the Anthropic Claude models are currently [supported](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-supported.html) in generating responses.

The diagram below illustrates how the vector database can be utilized as an input to a foundation model to generate augmented responses.

![Source: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-it-works.html](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*kSgGKXAbN-tbesXp.png)

I have created a knowledge base using the AWS documentation for bedrock using its PDF version. Once the knowledge base is ready, we can query it as shown below. Since I’m not utilizing a foundation model to create responses, I retrieve the row information from the vector database without performing any post-processing. Nonetheless, a pleasing feedback can be attained by employing a foundation model to generate a response.

![Testing the created knowledge base](https://miro.medium.com/v2/resize:fit:870/format:webp/1*NQHVY5FLK4sls4IcrRcqJA.png)

## Creating Agents

Bedrock agents allow the triggering of actions based on specific inputs and the creation of autonomous agents. For instance, you could create an agent to accept hotel room reservations from customers by configuring an agent with a knowledge base of room availability and other relevant data and the respective backend to place reservations. When configuring the backend, we need to provide an OpenAPI specification of the backend services so that it knows which endpoints to call to satisfy the request.

To have this capability, we need to configure the below components in Bedrock.

- **Foundation model:** This is needed to interpret the user input and continue the orchestration. Currently, only the Anthropic Claude models are supported.
- **Instructions:** Instructions are prompts describing what the agent is supposed to do. Having a clear and detailed prompt for the instruction is crucial for getting accurate results from the agent.
- **Action groups:** Here, we need to define the agent's actions. This consists of a lambda function and an OpenAPI specification. The lambda function has the implementation to act, and the OpenAPI specification provides the agent details on invoking the function. For example, we could implement a POST /reservation endpoint in the lambda function to create a reservation and provide the API specification on the details of the request, such as URL, request body, validation requirements, etc.
- **Knowledge base:** Knowledge base is optional but is mandatory in most cases. This can be used to provide contextual information to the agent. For example, in this case, it would be some information about the room availability, pricing details, etc, so that the agent knows to perform the actions as intended.

Once the agent is correctly configured, it understands its responsibilities based on the provided instructions. The knowledge base contains comprehensive information about the specific domain. The action group provides details on initiating each action and achieving the desired outcomes. Then, the foundation model can do the magic by orchestrating the workflow to handle a given request and provide the output to the user.

![Source: https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html](/images/posts/Generative-AI-on-Aws-with-Amazon-Bedrock/bedrock-agents.png)

**You can see a cool demonstration of an agent working with a knowledge base [here](https://www.youtube.com/watch?si=dH0YrYSdzaAuf-gF&v=P9n8BE693go&feature=youtu.be).**

Besides these, Bedrock offers additional features for developing responsible AI policies, including guardrails and watermark detection. We can anticipate introducing more capabilities to Bedrock as the potential of generative AI continues to unfold.

In conclusion, Amazon Bedrock offers a powerful platform for leveraging generative AI capabilities on AWS. With its Foundation models and easy-to-use APIs, developers can quickly integrate AI-driven features into their applications. Additionally, the ability to create custom models, knowledge bases, and agents opens up endless possibilities for tailoring AI solutions to specific needs. By harnessing the power of Bedrock, developers can unlock new levels of innovation and create intelligent, personalized experiences for their users.

## Learn More

- [What is Amazon Bedrock?](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
- [Custom Models](https://docs.aws.amazon.com/bedrock/latest/userguide/custom-models.html)
- [What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/#:~:text=RAG%20extends%20the%20already%20powerful,and%20useful%20in%20various%20contexts.)
- [Knowledge bases for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [How Agents for Amazon Bedrock Works](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html)
