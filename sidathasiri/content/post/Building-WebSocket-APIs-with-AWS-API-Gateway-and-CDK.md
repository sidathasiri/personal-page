---
layout: post
title: 'Building WebSocket APIs with AWS API Gateway and CDK'
# subtitle:   "Hello World, Hello Blog"
date: 2023-10-15
author: 'Sidath Munasinghe'
URL: '/2023/10/15/Building-WebSocket-APIs-with-AWS-API-Gateway-and-CDK/'
image: '/images/posts/web-sockets-api-gateway.png'
relcanonical: "https://medium.com/aws-in-plain-english/real-time-magic-building-websocket-apis-with-aws-api-gateway-and-cdk-5b76734a518a"
---

## Introduction

In the continuously evolving landscape of web applications, real-time communication has become the gold standard for creating engaging and interactive user experiences. Whether you’re building a chat application, a collaborative online game, or a live dashboard, WebSocket technology has emerged as the enchanting solution that makes real-time magic happen. And when it comes to unleashing the full potential of WebSockets in a serverless and scalable manner, Amazon Web Services (AWS) has a spell of its own — AWS API Gateway.

In this article, we will go through the fundamental concepts of WebSocket API in Amazon API Gateway and create a small application to evaluate its capabilities using the AWS Cloud Development Kit (CDK).

## Understanding WebSocket APIs

WebSocket is a protocol that allows full duplex communication between a client and server using a persistent connection for continuous communication. Since the WebSocket connection is persistent, it allows extremely fast data transmission.

Unlike the short-lived stateless connections in HTTP, WebSocket leverages long-lived opened connections to exchange messages with each other independently. This feature allows WebSocket connections to go beyond the traditional request and response model, making it the ideal solution for use cases like chat applications, live broadcasts, and extremely fast data synchronization.

![WebSockets vs HTTP](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6oxb3vsp705jcfdbm34q.png)

However, when implementing a WebSockets API, some of the crucial areas that we need to pay extra attention to are,

- **connection management:** manage incoming connections appropriately when clients connect and disconnect
- **security:** ensure protection of data in transit
- **message routing:** determine how WebSocket messages will be routed to the appropriate handlers on the server side
- **monitoring and logging:** set up monitoring and logging to keep track of WebSocket connection status, performance, and potential issues

## AWS API Gateway for WebSocket APIs

![AWS API Gateway](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/byjz25vqut1blrlpjzfl.png)

Amazon API Gateway is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale. Apart from the HTTP-based restful APIs, it allows creating and managing WebSocket APIs as well by simplifying the areas we discussed above while allowing developers to focus on the core functionality of the real-time applications. Here are the key benefits of using AWS API Gateway for building your next WebSocket API.

- **Abstracts out the protocol understanding:** AWS API Gateway abstracts much of the WebSocket protocol details, making it easier to use. It manages the handshaking process and connection handling, allowing you to focus on application logic.
- **Simplified infrastructure management:** It takes care of the WebSocket server setup and configuration, eliminating the need for you to manage the WebSocket server infrastructure.
- **Integration options:** API Gateway can be integrated with multiple backend sources such as lambda functions, DynamoDB tables, or HTTP endpoints to implement any logic based on the content of the messages it receives from the client.
- **Security:** Access can be controlled by AWS IAM or lambda authorizers to implement your authorization logic. Further, it supports WSS (WebSocket Secure) to have encrypted connections for enhanced security, like protecting against man-in-the-middle attacks.
- **Easy message routing:** AWS API Gateway permits you to define routes for WebSocket messages and map them to specific AWS Lambda functions, enabling you to handle them as required.
- **Monitoring and Logging:** AWS CloudWatch Logs and Metrics monitors the WebSocket APIs created with AWS API Gateway, allowing us to monitor its statuses and get alerts when it needs our attention to investigate any critical issue.

## Basics of AWS API Gateway for WebSocket APIs

AWS API Gateway service utilizes its fundamental concepts, such as models, authorizers, and stages for WebSocket APIs as well. Below are the additional concepts we need to know when using WebSocket APIs.

###Routes
In a WebSocket API, incoming JSON messages are directed to backend integrations based on routes you configure, similar to the endpoints in RestFul APIs. API Gateway provides three predefined routes as below.

- **$connect:** Triggers when a persistent connection between the client and a WebSocket API is initiated.
- **$disconnect:** Triggers when the client or the server disconnects from the API.
- **$default:** Triggers if the route selection expression cannot be evaluated against the message or if no matching route is found. This is useful for handling invalid messages gracefully.

We can also create custom routes to cater to the business requirements and functionalities. To handle incoming requests, these routes must be connected with available backend integrations, such as Lambda functions, DynamoDB tables, etc.

In WebSocket-based communications, connection IDs are very important since they need to be presented to exchange messages. The creation of connection IDs is handled by API Gateway for us, but implementing the logic to handle those events is up to us. For example, we can integrate a lambda function to the `$connect` route to implement some logic to store the newly created connection ID in a datastore for functionality like broadcasting an event. Also, we will need to delete those entries by implementing some logic behind the `$disconnet` route.

### One-way/Two-way Communication

WebSockets are bidirectional by nature. However, AWS API Gateway allows us to configure one-way or two-way communication routes. If configured for one-way, there won’t be any response after completing the message processing. When two-way communication is enabled, we can implement some logic to receive acknowledgements after complete processing.

### Route Selection Expression

API Gateway uses the route selection expression to determine which route to invoke when a client sends a message. When creating a WebSocket API, we must provide the route selection expression. Therefore, before creating the API, we must decide what the message structure looks like. For example, we can use an attribute called action in the message to specify the route like below.

```
{
   "action": "createTodo",
   "data": {
      "name": "Create WebSocket API",
      "completed": false
   }
}
```

In this scenario, we can use `$request.body.action` as the route selection expression. Here `$request.body` refers to the message payload. We can choose an appropriate attribute we want based on the message structure.

**Building the WebSocket API with CDK**
Now let’s create a very simple WebSocket API using AWS CDK with Typescript to define and deploy the infrastructure and apply the concept we just learned. We will also add a custom route to send a message to a connection so that we can play around and test it. Below are the main steps we have to follow.

1. Create three lambda functions to handle new WebSocket connections/disconnections and send a message to a connection.
2. Create a WebSocket API
3. Create a custom route for sending a message
4. Integrate lambda functions to routes
5. Grant any required permissions
6. Test the API

###Lambda functions
Let’s first create the three lambda functions so we can directly integrate them when creating the WebSockets API routes. The below code snippet shows how to create the mentioned Lambda functions using the NodejsFunction construct in CDK.

```
// Lambda function to handle new connections
const wsConnectLambda = new NodejsFunction(this, 'ws-connect-lambda', {
      entry: join(__dirname, '../src/connect/index.ts'),
      handler: 'handler',
      functionName: 'connect-lambda',
      runtime: Runtime.NODEJS_18_X,
});

// Lambda function to handle disconnections
const wsDisconnectLambda = new NodejsFunction(
      this,
      'ws-disconnect-lambda',
      {
        entry: join(__dirname, '../src/disconnect/index.ts'),
        handler: 'handler',
        functionName: 'disconnect-lambda',
        runtime: Runtime.NODEJS_18_X,
      }
);

// Lambda function to handle sending messages
const sendMessageLambda = new NodejsFunction(this, 'send-message-lambda', {
      entry: join(__dirname, '../src/send-message/index.ts'),
      runtime: Runtime.NODEJS_18_X,
      functionName: 'send-message',
      handler: 'handler',
});
```

Now, let’s review the handler function implementations for each function. The below code snippet shows the implementation of the connection handler function.

### Connection Handler Function

```
export const handler = async (event: APIGatewayProxyEvent) => {
  const connectionId = event.requestContext.connectionId;
  console.log('connection created:', connectionId);
  return { statusCode: 200, body: 'Connected.' };
};
```

We only log the generated connection ID when a new client connects with the API to keep this as simple as possible. If you have any sophisticated logic, we can implement them here.

### Disconnection Handler Function

```
export const handler = async (event: APIGatewayProxyEvent) => {
  const connectionId = event.requestContext.connectionId;
  console.log('Disconnected:', connectionId);
  return { statusCode: 200, body: 'Disconnected.' };
};
```

We also only log the connection ID when a connection terminates for simplicity. We can extend this implementation to match the business requirements.

### Send Message Handler Function

We use this function to send messages to another connection. Below is the structure of the message format we are going to use.

```
{
   "action":"sendMessage",
   "connectionId":"<receiver's connection id>",
   "message":"<message content>"
}
```

Below is the definition of these attributes.

- **action:** Acts as the route selection attribute
- **connectionId:** Contains the receiver’s connection ID
- **message:** Message content

With this information, we can implement the below logic inside this lambda function to handle this requirement.

```
export const handler = async (event: APIGatewayProxyEvent) => {
  const apigwManagementApi = new ApiGatewayManagementApi({
    apiVersion: '2018-11-29',
    endpoint:
      event.requestContext.domainName + '/' + event.requestContext.stage,
  });

  const eventBody = JSON.parse(event.body ?? '');
  const connectionId = eventBody.connectionId;
  const message = eventBody.message;

  console.log('Payload:', { ConnectionId: connectionId, Data: message });

  try {
    await apigwManagementApi
      .postToConnection({ ConnectionId: connectionId, Data: message })
      .promise();
  } catch (error) {
    console.error('Error sending message:', error);
  }
  return { statusCode: 200, body: 'Message sent' };
};
```

Here, we get the connectionId and the message from the event body and then use AWS API Gateway Management API to post the received message to the received connection.

### WebSocket API

Since the lambda functions are ready now, let’s create the WebSockets API. The below code snippet demonstrates how to create it.

```
// Create WebSocket API with connection/disconnection route integrations
const webSocketApi = new apigw2.WebSocketApi(
      this,
      'my-first-websocket-api',
      {
        connectRouteOptions: {
          integration: new WebSocketLambdaIntegration(
            'ws-connect-integration',
            wsConnectLambda
          ),
        },
        disconnectRouteOptions: {
          integration: new WebSocketLambdaIntegration(
            'ws-disconnect-integration',
            wsDisconnectLambda
          ),
        },
      }
);

// Create API stage
const apiStage = new apigw2.WebSocketStage(this, 'dev', {
      webSocketApi,
      stageName: 'dev',
      autoDeploy: true,
});

// Add the custom sendMessage route
webSocketApi.addRoute('sendMessage', {
      integration: new WebSocketLambdaIntegration(
        'send-message-integration',
        sendMessageLambda
      ),
});
```

We first create the WebSocket API using the WebSocketApi CDK construct in the above code. Since our connection/disconnection lambda functions are ready, we have also integrated those while creating the API. Then, we defined a dev stage for the API as usual in a typical API in AWS API Gateway. Finally, we have added our custom route for the message-sending purpose. Note that since we are using sendMessage as the action in the message payload, we need to mention the same here as well when adding the route. Further, we don’t need to mention the route selection expression here, since we can use the default value ($request.body.action) provided by CDK construct.

There is one last thing missing in this setup. Our send message lambda function needs permission to post messages to connections. So we need to add the below code segment as well to grant those permissions.

```
// Get the resource ARN of the created WebSocket API
const connectionsArns = this.formatArn({
      service: 'execute-api',
      resourceName: `${apiStage.stageName}/POST/*`,
      resource: webSocketApi.apiId,
});

// Attach the required policy to the relavant lambda function
sendMessageLambda.addToRolePolicy(
      new PolicyStatement({
        actions: ['execute-api:ManageConnections'],
        resources: [connectionsArns],
      })
);
```

Now everything is complete, and we can deploy the infrastructure to AWS using the `cdk deploy` command.

### Testing

Once the deployment is complete, we can get the API’s WebSocket URL with wssprotocol from the API Gateway console. For testing purposes, we can use any WebSocket testing tool like [piesocket](https://www.piesocket.com/websocket-tester) to connect and send messages.

We can give our WebSocket API URL to this tool to test the connection creation. We should be able to see the connection ID in the Cloudwatch logs of connection handling lambda. We can connect with as many clients as we want and track their connection IDs to send messages.

![Log of different connection IDs in Cloudwatch](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/txba973daa29n95ipt58.png)

Then, we can use these connection IDs to send messages to each other using the custom route we created.

![Message history of connection 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ovr5zrpwsklhr0707nal.png)

![Message history of connection 2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/l5a9tbbds65s17q13oxe.png)

Since the message sending is working as expected, now we can try the disconnection handler’s functionality. When a client disconnects, we should be able to see the appropriate log as well.

![Appropriate logs for disconnected connections](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/29qe2fbs5zdhchwtdvn7.png)

So everything is working as expected, and it takes very little time to implement using the capabilities of AWS API Gateway. We can enhance the security of the API by introducing authorizers and access control mechanisms. Further, you can use CloudWatch to monitor the service status using different metrics, such as the connection count.

The full source code for this example application can be found in this [GitHub repo](https://github.com/sidath-munasinghe/apigateway-websocket-api).

## Conclusion

From the introduction to the basics of WebSocket APIs, we’ve navigated through the realm of real-time web development. AWS API Gateway stood as our gateway to real-time magic, simplifying the complexities and providing security and scalability. Armed with this knowledge, your journey into real-time web development begins, where you can create captivating applications with the help of AWS CDK. Embrace the magic, and let your applications come alive in the world of real-time experiences!

## Further Reading

- [About WebSocket APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-overview.html)
- [Tutorial: Building a serverless chat app with a WebSocket API, Lambda and DynamoDB](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-chat-app.html)
