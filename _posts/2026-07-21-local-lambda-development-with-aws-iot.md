---
layout: post
title: "Running CDK Lambda Functions Locally with AWS IoT"
date: 2026-07-20 10:00:00 -0500
categories: engineering
author: Jose Rodriguez
---

If you're working with AWS CDK and Lambda functions, you know the pain: change a line, run `cdk deploy`, wait 3-5 minutes, test, repeat. No breakpoints, no hot reload, just waiting. What if your Lambda could run on your laptop while real AWS services invoke it?

## The Problem

Traditional CDK Lambda development means deploying on every change. Over a day of development, that's hours wasted waiting for CloudFormation. Local testing tools like SAM Local simulate AWS services, but subtle differences from production can catch you off guard.

## The Solution: IoT Core as a Bridge

Deploy a stub Lambda that forwards invocations to your machine via AWS IoT Core. Your real API Gateway, SQS, or EventBridge invokes the stub, which forwards events over MQTT to your laptop where your handler runs. Results flow back the same way.

**Why IoT Core?**
- MQTT over WebSocket with SigV4 auth (no ports to expose)
- Works behind corporate firewalls
- Native AWS service (no extra infrastructure)
- Both Lambda and your laptop authenticate the same way

## CDK Setup

First, create the stub Lambda infrastructure in your CDK stack:

```typescript
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as iam from "aws-cdk-lib/aws-iam";
import * as s3 from "aws-cdk-lib/aws-s3";

export class MyStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: MyStackProps) {
    super(scope, id, props);

    const stage = props.stage || "dev";
    const isLiveMode = this.node.tryGetContext("live") === "true";

    // S3 bucket for large payloads (MQTT has 128KB limit)
    const scratchBucket = new s3.Bucket(this, "ScratchBucket", {
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    // IoT endpoint for your account/region
    const iotEndpoint = `${cdk.Aws.ACCOUNT_ID}-ats.iot.${cdk.Aws.REGION}.amazonaws.com`;

    // Deploy stub in live mode, real handler otherwise
    const handler = isLiveMode
      ? lambda.Code.fromAsset("./stub-handler")
      : lambda.Code.fromAsset("./functions");

    const myFunction = new lambda.Function(this, "MyFunction", {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: isLiveMode ? "stub.handler" : "index.handler",
      code: handler,
      environment: {
        IOT_ENDPOINT: iotEndpoint,
        STAGE: stage,
        FUNCTION_ID: "my-function",
        SCRATCH_BUCKET: scratchBucket.bucketName,
      },
    });

    if (isLiveMode) {
      // Grant IoT permissions to stub
      myFunction.addToRolePolicy(new iam.PolicyStatement({
        actions: ["iot:Connect", "iot:Publish", "iot:Subscribe", "iot:Receive"],
        resources: ["*"],
      }));
      scratchBucket.grantReadWrite(myFunction);
    }

    // Wire up to API Gateway, SQS, etc.
    new apigw.LambdaRestApi(this, "Api", { handler: myFunction });
  }
}
```

Deploy with live mode:
```bash
cdk deploy -c live=true -c stage=dev
```

## The Stub Handler

Create `stub-handler/stub.ts`:

```typescript
import type { Handler, Context } from "aws-lambda";
import { auth, iot, mqtt5 } from "aws-iot-device-sdk-v2";

let iotClient: any;

async function getIoTClient() {
  if (!iotClient) {
    const config = iot.AwsIotMqtt5ClientConfigBuilder
      .newWebsocketMqttBuilderWithSigv4Auth(process.env.IOT_ENDPOINT!, {
        credentialsProvider: auth.AwsCredentialsProvider.newDefault(),
        region: process.env.AWS_REGION,
      });

    iotClient = new mqtt5.Mqtt5Client(config.build());
    await new Promise((resolve, reject) => {
      iotClient.on("connectionSuccess", resolve);
      iotClient.on("connectionFailure", (e: any) => reject(e.error));
      iotClient.start();
    });
  }
  return iotClient;
}

export const handler: Handler = async (event, context: Context) => {
  const client = await getIoTClient();
  const requestId = context.awsRequestId;
  const requestTopic = `live-lambda/${process.env.STAGE}/${process.env.FUNCTION_ID}/request`;
  const responseTopic = `live-lambda/${process.env.STAGE}/${process.env.FUNCTION_ID}/response/${requestId}`;

  const response = await new Promise((resolve, reject) => {
    const timeout = setTimeout(() => {
      reject(new Error("No local dev session running"));
    }, context.getRemainingTimeInMillis() - 1500);

    client.subscribe({
      subscriptions: [{ topicFilter: responseTopic, qos: mqtt5.QoS.AtLeastOnce }],
    });

    client.on("messageReceived", (msg: any) => {
      if (msg.message.topicName === responseTopic) {
        clearTimeout(timeout);
        resolve(JSON.parse(Buffer.from(msg.message.payload).toString()));
      }
    });

    client.publish({
      topicName: requestTopic,
      payload: JSON.stringify({ requestId, event, context }),
      qos: mqtt5.QoS.AtLeastOnce,
    });
  });

  return response;
};
```

## The Local CLI

Create a local development CLI that connects to IoT and runs your handler:

```typescript
import { auth, iot, mqtt5 } from "aws-iot-device-sdk-v2";
import { watch } from "chokidar";

async function dev(stage: string, functionId: string, handlerPath: string) {
  const endpoint = `${process.env.AWS_ACCOUNT_ID}-ats.iot.${process.env.AWS_REGION}.amazonaws.com`;

  const config = iot.AwsIotMqtt5ClientConfigBuilder
    .newWebsocketMqttBuilderWithSigv4Auth(endpoint, {
      credentialsProvider: auth.AwsCredentialsProvider.newDefault(),
      region: process.env.AWS_REGION,
    });

  const client = new mqtt5.Mqtt5Client(config.build());
  await new Promise((resolve) => {
    client.on("connectionSuccess", resolve);
    client.start();
  });

  let handler = (await import(handlerPath)).handler;

  // Watch for changes
  watch(handlerPath).on("change", async () => {
    delete require.cache[require.resolve(handlerPath)];
    handler = (await import(handlerPath)).handler;
    console.log("Reloaded handler");
  });

  const requestTopic = `live-lambda/${stage}/${functionId}/request`;

  client.subscribe({
    subscriptions: [{ topicFilter: requestTopic, qos: mqtt5.QoS.AtLeastOnce }],
  });

  client.on("messageReceived", async (msg: any) => {
    const { requestId, event, context } = JSON.parse(
      Buffer.from(msg.message.payload).toString()
    );

    try {
      const result = await handler(event, context);
      await client.publish({
        topicName: `live-lambda/${stage}/${functionId}/response/${requestId}`,
        payload: JSON.stringify(result),
        qos: mqtt5.QoS.AtLeastOnce,
      });
    } catch (error: any) {
      console.error("Handler error:", error);
    }
  });

  console.log(`Listening for ${functionId} invocations...`);
}

dev("dev", "my-function", "./functions/index.ts");
```

Run it:
```bash
ts-node local-dev.ts
```

## How It Works

1. API Gateway invokes your deployed Lambda
2. The stub publishes the event to IoT Core topic
3. Your local CLI receives the event via MQTT subscription
4. Your handler executes on your laptop
5. The CLI publishes the result back
6. The stub returns it to API Gateway

The round trip adds milliseconds. To API Gateway, it's just a slow Lambda.

## Benefits

**Instant feedback**: Edit, save, test in under a second.

**Real AWS services**: Unlike simulators, you test against actual API Gateway, SQS, DynamoDB.

**Standard debugging**: Use breakpoints in VS Code like any Node.js app.

**Team friendly**: Each developer uses their own stage (alice, bob) to avoid conflicts.

## Production Deployment

Deploy without the live flag and your real handler gets deployed:
```bash
cdk deploy -c stage=prod
```

No IoT permissions, no stub code, just your production Lambda.

## Conclusion

By using AWS IoT Core as a transport layer between a stub Lambda and your local machine, you get instant feedback loops for CDK Lambda development. Real AWS services invoke your code, but it runs locally with full debugging and hot reload.

The setup is straightforward: modify your CDK stack to deploy a stub in dev mode, create the stub handler that forwards to IoT, and run a local CLI that executes your real handler. Edit your code, save, and test immediately without waiting for `cdk deploy`.
