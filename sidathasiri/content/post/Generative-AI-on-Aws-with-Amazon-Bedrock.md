---
layout: post
title: 'Amazon Bedrock Explained: Foundation Models, Fine-Tuning, RAG and Agents'
# subtitle:   "Hello World, Hello Blog"
date: 2024-05-01
author: 'Sidath Munasinghe'
keywords: 'amazon bedrock, aws, generative ai, foundation models, rag, knowledge bases, bedrock agents, fine-tuning, sidath, munasinghe'
description: "A hands-on tour of Amazon Bedrock: call foundation models from Node.js, fine-tune a custom model, build a RAG knowledge base and wire up agents on AWS."
URL: '/2024/05/01/Generative-AI-on-AWS-with-Amazon-Bedrock/'
image: '/images/posts/Generative-AI-on-Aws-with-Amazon-Bedrock/main-logo.png'
---

Adding generative AI to an application used to mean picking a model, finding GPUs to run it on, and keeping that infrastructure alive. Amazon Bedrock removes most of that work. It is a fully managed AWS service that gives you models from several leading AI providers behind one consistent API, so you can skip hosting entirely and start calling models from your code straight away.

Bedrock also goes past plain inference. In this post I'll walk through the four building blocks I find most useful:

1. **Foundation models**: try several models, compare them, and call the one that fits your use case.
2. **Custom models**: adapt a foundation model with your own data and training parameters.
3. **Knowledge bases**: ground a model's answers in your own documents without retraining it.
4. **Agents**: let a model reason over a request and call your APIs to actually get something done.

## Starting point: the foundation models

Everything in Bedrock is built on foundation models (FMs). AWS hosts models from a range of AI companies, and the catalogue keeps growing as new ones are added. They are not interchangeable. Some are built for text generation, some for embeddings, some for images, and each has its own pricing. Part of the job is matching the model to the task.

![A selection of foundation models from different AI providers offered on Amazon Bedrock](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*C2Hf0X8jQToKxkru.png)

### Experiment in the playground first

Before writing code, I suggest spending some time in the Bedrock console playground. You can send prompts to a model, change inference parameters such as temperature, and see how the output shifts. The screenshot below uses Amazon's own Titan model in the text playground.

![The Amazon Bedrock text playground running a prompt against the Amazon Titan model](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*oR9kqa_lMM4fT6bJGi3QiA.png)

### Then call it from code

Once you've settled on a model, the AWS SDK lets you invoke it through a single runtime API. The call looks the same whichever model sits behind it, so moving to another model is mostly a configuration change (the model ID and its request body format) and not a rewrite.

Here is a small Node.js script that sends a prompt to Titan and prints the answer:

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

When I run it, the model's answer is printed in the terminal:

![Terminal output showing a prompt sent to Bedrock and the generated answer about Sri Lanka](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*iDg0DuP68kuLeGuzFeSnkg.png)

**The complete sample is on [GitHub](https://github.com/sidathasiri/aws-bedrock-client-sample-app).**

There isn't much more to it. Wrap the call in a function, build the prompt from your app's context, and you have a generative AI feature in any service that can use the AWS SDK.

## Teaching a model your domain with custom models

Out of the box, foundation models know a lot about everything and very little about your business. Their training data also stops at a fixed point in time. Bedrock's custom model feature addresses this by training an existing FM further on data you supply.

You have two options, depending on what your data looks like:

| Your data | Bedrock option | What it does |
| --- | --- | --- |
| Unlabelled (raw domain text) | Continued pre-training | Exposes the model to more domain content |
| Labelled (prompt/completion pairs) | Fine-tuning | Teaches the model how to respond to specific kinds of input |

The process runs through a console wizard. You upload the training data to S3 in the [required format](https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization-prepare.html), choose the base model and hyperparameters, and start the job. AWS provisions and manages all the training infrastructure. Expect it to take a while, since jobs commonly run for hours depending on dataset size and settings. When the job finishes, you query the custom model just like any other FM.

### A tiny fine-tuning experiment

To see the effect clearly, I fine-tuned Titan on a JSONL dataset with one deliberately obvious record:

```
{"prompt": "who are you?", "completion": "I'm a customized Amazon Titan model"}
```

The aim was to change how the model describes itself. Before training, the base model introduced itself as Titan, built by Amazon:

![The base Titan model describing itself as a model built by Amazon before customization](https://miro.medium.com/v2/resize:fit:1002/format:webp/1*PrhEHP2rIwo_aAvmrVTPyw.png)

After fine-tuning, the same kind of question got the answer from my dataset:

![The fine-tuned Titan model replying that it is a customized Amazon Titan model](https://miro.medium.com/v2/resize:fit:1000/format:webp/1*N2E8sehlubii5ijXcYCWrw.png)

The interesting part is that this isn't a lookup table. The question I asked was worded differently from the prompt in the dataset, and the model still answered the way it was trained to. It generalised to similar phrasings, which is exactly what you want from fine-tuning.

## Grounding answers with knowledge bases (RAG)

Retraining isn't always the right tool. If you mainly need the model to use up-to-date or private information, a knowledge base is usually faster and cheaper. It gives the model extra context at query time, so the answers become more specific and accurate without changing the model's weights.

The technique behind this is **Retrieval Augmented Generation (RAG)**. When a question comes in, relevant passages are retrieved from an external store and passed to the LLM together with the prompt. That store is normally a vector database, which holds text as embeddings that can be searched by meaning, not just by keyword.

Building a RAG pipeline yourself involves a fair amount of plumbing. Bedrock knowledge bases take care of it:

- You upload your documents to an S3 bucket.
- In the console wizard, you point the knowledge base at that bucket and choose a model to generate responses.
- Bedrock fetches the documents, splits them into chunks, converts the chunks into embeddings, and writes them to a vector store. By default it provisions an Amazon OpenSearch Serverless collection for this.

![Diagram of Bedrock knowledge base ingestion: documents are chunked, embedded and stored in a vector database (source: AWS documentation)](https://miro.medium.com/v2/resize:fit:1344/format:webp/0*d6xofBUriEPY91VH.png)

You can use the result in two ways. You can query the vector store directly and get back the raw matching chunks, or you can let a foundation model turn those chunks into a readable answer. At the time of writing, response generation was [supported](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-supported.html) only with Anthropic Claude models.

The second approach looks like this, with retrieved context fed into the model to produce an augmented response:

![Diagram of the RAG query flow: a user query retrieves relevant chunks from the vector store, which are passed to a foundation model to generate the answer (source: AWS documentation)](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*kSgGKXAbN-tbesXp.png)

To try it, I built a knowledge base from the PDF version of the Bedrock developer guide. I didn't attach a model for generation, so the test below shows the raw text retrieved from the vector store with no post-processing. Adding a model would turn this into a cleaner, conversational answer.

![Querying the Bedrock knowledge base in the console and seeing raw passages retrieved from the documentation](https://miro.medium.com/v2/resize:fit:870/format:webp/1*NQHVY5FLK4sls4IcrRcqJA.png)

## From answers to actions with Bedrock agents

Knowledge bases help a model *say* the right thing. Agents let it *do* something. A Bedrock agent reads a user's request, works out the steps needed, and calls your backend to complete them.

Take a hotel booking assistant. It would need to know which rooms are available and be able to create reservations. To build it, you configure four pieces:

- **A foundation model** that understands the user's input and drives the orchestration. When I wrote this, agents only supported Anthropic Claude models.
- **Instructions**, a prompt that describes the agent's job. The more specific and detailed this is, the more reliably the agent behaves.
- **Action groups**, which define what the agent can do. Each one pairs a Lambda function that does the actual work with an OpenAPI specification that tells the agent how to call it. In the hotel example, the Lambda might implement `POST /reservation`, and the spec would describe the URL, the request body and its validation rules.
- **A knowledge base**, technically optional but needed in most real cases. Here it would hold room availability, pricing and similar details, so the agent has the facts it needs before it acts.

At runtime these pieces work together. The instructions tell the agent what its role is, the knowledge base gives it domain context, and the action groups tell it which operations exist and how to invoke them. The foundation model then plans the steps, calls the right actions, and returns the result to the user.

![How Amazon Bedrock agents orchestrate a request using the foundation model, knowledge base and action groups (source: AWS documentation)](/images/posts/Generative-AI-on-Aws-with-Amazon-Bedrock/bedrock-agents.png)

**For a good walkthrough of an agent using a knowledge base, watch [this demo](https://www.youtube.com/watch?si=dH0YrYSdzaAuf-gF&v=P9n8BE693go&feature=youtu.be).**

Bedrock also includes responsible-AI tooling such as guardrails and watermark detection, and I expect the feature set to keep expanding as generative AI matures.

## Choosing the right tool

People often aren't sure whether they need a custom model, a knowledge base or an agent. Here is how I usually decide:

| Your problem | Reach for |
| --- | --- |
| The model needs to answer from your documents, and those documents change often | Knowledge base (RAG) |
| The model needs a consistent tone, format or domain-specific behaviour | Fine-tuning |
| You have lots of unlabelled domain text and want the model to "speak" that domain | Continued pre-training |
| The model needs to take actions such as creating, updating or booking | Agent with action groups (plus a knowledge base) |
| A well-crafted prompt already gets you close | Plain foundation model call, so start here |

## Gotchas worth knowing

- **Model access is opt-in.** You have to request access to each model in the Bedrock console before you can invoke it. Otherwise the API call fails with an access error.
- **Request bodies differ per model.** The `InvokeModel` API is shared, but the JSON body is not. The `inputText`/`textGenerationConfig` shape above is specific to Titan, and other providers expect different fields.
- **Regional availability varies.** Not every model or feature exists in every AWS region, so check before you hard-code a region.
- **Custom models have their own serving costs.** To run inference on a custom model you generally need to buy Provisioned Throughput, which is billed differently from on-demand calls to base models.
- **Keep credentials out of code.** The sample reads keys from environment variables. In Lambda, ECS or EC2, prefer the IAM role attached to the compute so you don't handle access keys at all.

## Key takeaways

Amazon Bedrock makes generative AI on AWS easy to get started with. A single API gives you a range of foundation models without managing any infrastructure. When a generic model isn't enough, you can fine-tune or continue pre-training it, ground it in your own data with a RAG knowledge base, or give it actions through agents. My advice is to start with the simplest option that works, which is usually a good prompt, and only add knowledge bases, customisation or agents when the use case actually needs them.

## Further reading

- [What is Amazon Bedrock?](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
- [Custom Models](https://docs.aws.amazon.com/bedrock/latest/userguide/custom-models.html)
- [What is RAG?](https://aws.amazon.com/what-is/retrieval-augmented-generation/#:~:text=RAG%20extends%20the%20already%20powerful,and%20useful%20in%20various%20contexts.)
- [Knowledge bases for Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [How Agents for Amazon Bedrock Works](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-how.html)
