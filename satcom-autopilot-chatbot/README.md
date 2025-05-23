# Satellite Capacity chatbot

## Description
The Satellite Capacity bot is a chatbot which either returns a satellite capacity forecast, or provides targeted responses 
to satellite related questions, depending on the user’s input questions. 

The bot is intended to show customers how they can (a) get inference results from a SageMaker endpoint in an
easy to use interface i.e. a chatbot, (b) leverage an Image to Text Foundational Model to detect 
interference or noise impairments in digital IQ constellations, (c) integrate Bedrock Knowledgebase agents to existing applications to 
retrieve answers to domain-specific questions. 

This sub-project is associated with the blog [here](https://aws.amazon.com/blogs/publicsector/maximizing-satellite-communications-usage-with-amazon-forecast/).
It focuses on a Maritime shipping use-case, 
using [Amazon SageMaker Autopilot Time-series](https://docs.aws.amazon.com/sagemaker/latest/dg/timeseries-forecasting-deploy-models.html) to build 
a time series predictor model for each satellite beam in the ship(s) path, accounting for the impact of weather
conditions in a given location. It then renders results in [Amazon QuickSight](https://aws.amazon.com/quicksight/) BI tooling, displaying
forecasted capacity needs, accuracy metrics, and which attributes most impact the model.

## Architecture
The architecture of the system is as follows: -

![satcapacitybot-arch drawio](https://github.com/user-attachments/assets/e5fd45d2-c7bd-430f-8703-8740c252cc23)

Users interact with the application via either Amazon Lex directly or a Messaging platform such as Facebook, Slack or Twilio. This is achieved
via a [Lex channel integration](https://docs.aws.amazon.com/lexv2/latest/dg/deploying-messaging-platform.html).

This project provides the infrastructure, via a series of Cloudformation templates, for Lex, Lambda, Sagemaker endpoint, 
Bedrock runtime, knowledgebase and agent, and S3 assets. 

Depending on the user’s question the bot goes one of 3 different paths: -
* capacity forecast - if the user asks “get capacity” or similar utterances, we invoke the BeamForecast Lex intent, which in turn calls a Lambda function to invoke a Sagemaker Autopilot timeseries model endpoint on [spot beams](https://en.wikipedia.org/wiki/Spot_beam). The model was trained on generated, synthetic satellite data - the Notebook is available [here](https://github.com/aws-samples/satellite-comms-forecast-aws/tree/main/autopilot-notebook)

![Capture_fig1](https://github.com/user-attachments/assets/78297a2e-e75e-46ea-abe2-0acd05b229df)
  
* image summarization - if the users asks "get noise type", or similar utterances, we invoke the ImageIntent Lex intent. This uses a multi-mode LLM to describe the content of IQ constellation modulation satcom images. Phase noise, compression noise and interference are all common [examples](https://rahsoft.com/2022/10/16/understanding-constellation-distortions/).

![IQ_plots](https://github.com/user-attachments/assets/9e0a4952-a474-4fcb-baff-00ab63f5a83b)
 
* satellite communication topics - if the user asks a question related to the field of satellite communication, but not directly requesting a specific forecast, the FallbackIntent is invoked. The same Lambda handles this path, but instead invokes a Claude3 Bedrock agent associated with a Bedrock knowledge-base consisting of AWS Aerospace and Satellite blogs. 

As illustrated in the architecture diagram, we use the following AWS services:

- [SageMaker](https://aws.amazon.com/sagemaker/) to invoke the Satellite Capacity Autopilot inference endpoint for spot beam predictions
- [Bedrock](https://aws.amazon.com/bedrock/) for access to the FMs for
  embedding and text generation as well as the knowledge base agent.
- [Lambda](https://aws.amazon.com/lambda/) to call either the Sagemaker endpoint with the BeamForecast Lex intent or the Bedrock LLM KB agent via the Fallback intent.
- [OpenSearch Service Serverless with vector
  search](https://aws.amazon.com/opensearch-service/serverless-vector-engine/)
  for storing the embeddings of the enterprise knowledge corpus and
  doing similarity search with user questions.
- [S3](https://aws.amazon.com/pm/serv-s3/) for storing the raw knowledge
  corpus data (PDF, HTML files).
- [AWS Identity and Access Management](https://aws.amazon.com/iam/)
  roles and policies for access management.
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/) for
  creating the entire solution stack through infrastructure as code.


## Deployment Guide 

In the following sections, we discuss the key steps to deploy the solution, including pre and post-deployment.

## Pre-Deployment
An AWS account to deploy the resources is required. Please use the link to sign-up if you do not have an account [AWS
account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup)

**Note** - navigate to Amazon Bedrock Console and ensure that you have access to the models you are going to use in 
this solution e.g. Claude 3.5 Sonnet v2 or Claude 3.7 Sonnet v1

![Capture-Bedrock-Claude](https://github.com/user-attachments/assets/8fe406ab-2e80-469a-9e41-36496937d4dc)

We require a multi-modal LLM (i.e. image and text) to perform all of the functions in the bot.

Clone the entire repository using the command 
`git clone https://github.com/aws-samples/satellite-comms-forecast-aws.git`

The SageMaker endpoint needs to be available for this project to use. Follow the steps 
[here](https://github.com/aws-samples/satellite-comms-forecast-aws/tree/main/autopilot-notebook) to 
create a SageMaker Autopilot time-series real-time inference endpoint. At the end you should have a model as follows: -

![Capture-Sage-endpoint-bluracctid](https://github.com/user-attachments/assets/86e68bda-f557-45c4-9ce3-f2f3c349e111)


## Deployment Steps
**Note** - this project shares many of the same deployment steps as [this project](https://github.com/aws-samples/amazon-bedrock-rag-knowledgebases-agents-cloudformation) hence
feel free to use it as an additional reference.

The solution deployment automation script uses 3 parameterized CloudFormation template, OpenSearch_serverless.yaml, satcom-ts-kb-agent.yaml, and satcom-ts-lexbot.yaml to automate provisioning of the following solution resources:
 
 1. OpenSearch Service Serverless collection
 2. Amazon Bedrock Agent and KnowledgeBase
 3. Amazon Bedrock Runtime
 4. AWS Lambda 
 5. Amazon Lex chatbot intents
 6. IAM Roles


## Cloudformation to deploy OpenSearch_serverless.yaml stack
AWS CloudFormation prepopulates stack parameters with the default values provided in the template except for ARN of the IAM role with which you are
currently logged into your AWS account which you must provide. To provide alternative input values, you can specify parameters as environment variables that are referenced in the `ParameterKey=<ParameterKey>,ParameterValue=<Value>` pairs in the following shell script’s `aws cloudformation create-stack --stack-name <stack-name> --template-body file://OpenSearch_serverless.yaml --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=<parameter key>,ParameterValue=<parameter value>` ....

**Note** - make sure you have sufficient IAM permissions in the IAM role you pass in to use Amazon OpenSearch

**Note** - some of the assets may not be available in all regions. The stacks were tested in us-east-1 and us-west-2.

Once the Cloudformation stack creation is successful navigate to the Output section of the stack and grab the following output values `AmazonBedrockExecutionRoleForKnowledgeBasearn`, `AOSSIndexName`. We will use these values as parameters for our next stack satcom-ts-kb-agent.yaml to deploy Amazon Bedrock Knowledgebase and agents.

### Create Vector index in OpenSearch Serverless
The previous CloudFormation stack creates an OpenSearch Service Serverless collection, but the next step will require us to create a vector index in the generated collection. Follow the steps outlined below: 

1.  Navigate to OpenSearch Service console and click on `Collections`.
    The `satcom-aoss-coll` collection created by the CloudFormation stack
    will be listed there.

2.  Click on the `satcom-aoss-coll` link to create a vector index for
    storing the embeddings from the documents in S3. Next, click `Create vector index`

3. Grab the vector index name from the output values of the previous stack, the default value is`satcom-aoss-index`. Input the vector
    field name as `vector` [dimensions](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-setup.html) as `1024`,
    choose engine types as `FAISS` and distance metric as
    `Euclidean`. **It is required that you set these parameters exactly
    as mentioned here because the Bedrock Knowledge Base Agent is going
    to use these same values**.
  
5.  Once created the vector index is listed as part of the collection.

![Capture-aoss-vector](https://github.com/user-attachments/assets/d3046262-98b6-4afd-95af-67824694abb3)


## CloudFormation to deploy satcom-ts-kb-agent.yaml

Deploy the next stack using the following commands to provision the Bedrock knowledge base and agent resources in your AWS account. 

`aws cloudformation create-stack --stack-name <stack-name> --template-body file://satcom-ts-kb-agent.yaml --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=<parameter key>,ParameterValue=<parameter value>` ....

**Note** - change the default S3 bucket name to your desired S3 bucket using the parameter `ParameterKey=KnowledgeBaseArticlesBucket,ParameterValue=my-bucket-name`

**Note** - grab the values of parameters from the output of the previous stack. Use these keys, `AmazonBedrockExecutionRoleForKnowledgeBasearn`, `CollectionArn`

### Test the RAG App in Amazon Bedrock Agents Console.

1. Navigate to the Amazon Bedrock console and click on `Knowledge bases`
The `satcom-kbase-bedrock` knowledgebase created by the CloudFormation stack will be listed there. Click on this,
and then click on the S3 `Source Link` under Data source.

Load up some relevant articles in the S3 bucket. For satellite capacity Q&A we added the following: -
* [AWS joins the Digital IF Interoperability (DIFI) Consortium](https://aws.amazon.com/blogs/publicsector/aws-joins-the-digital-if-interoperability-difi-consortium/)
* [Creating satellite communications data analytics pipelines with AWS serverless technologies](https://aws.amazon.com/blogs/publicsector/creating-satellite-communications-data-analytics-pipelines-aws-serverless-technologies/)
* [Maximizing satellite communications usage with Amazon Forecast](https://aws.amazon.com/blogs/publicsector/maximizing-satellite-communications-usage-with-amazon-forecast/)
* [Virtualizing satellite communication operations with AWS](https://aws.amazon.com/blogs/publicsector/virtualizing-satcom-operations-aws/)

Production use-cases would typically ingest thousands of articles.

Flip back to the Knowledge base Data source and click `Sync` to synchronize the Knowledge base with the S3 bucket contents.

2. Click on `Agents`, and select the `satcom-fm-agent` created by the CFN stack. In the `Test` window ask a few questions e.g.

![Capture-Bedrock-agent-test](https://github.com/user-attachments/assets/49a28a0a-4ca2-4f11-9b4e-a0c44d3e0bd1)

3. Also notice that each response from the Amazon Bedrock agent is accompanied by a **trace** that details the steps being orchestrated by the agent. The **trace** helps you follow the agent's reasoning process that led it to the response given at that point in the conversation.

Use the **trace** to track the agent's path from the user input to the response it returns. The trace provides information about the inputs to the action groups that the agent invokes and the knowledge bases that it queries to respond to the user.


## CloudFormation to deploy satcom-ts-lexbot.yaml

The final stack deploys the Amazon Lex and Lambda resources. Lex controls the entire chatbot dialog but reaches out to a 
Lambda to provide the Fulfillment logic for the Satellite Capacity forecasting intent `BeamForecast`, the `ImageIntent` 
IQ Constellation decoding via Bedrock runtime, and the Bedrock LLM agent invocation `FallbackIntent`. 

The following paramaters should be modified: -
* SatComBotS3Bucket - the name of the bucket holding the zipped Lambda function
* SatComBotLambdaZipName - name of the lambda zip file invoking Sagemaker endpoint and Bedrock agent
* SatComInferenceEndpoint - the SageMaker Autopilot inference endpoint as indicated in [Pre deployment](#Pre-Deployment)
* BedrockAgentId - the id of the Bedrock agent deployed by `satcom-ts-kb-agent.yaml`
* BedrockAgentAliasId - the alias id of the Bedrock agent deployed by `satcom-ts-kb-agent.yaml`
* BedrockImageModelId - the model id of the Bedrock image model for Lex ImageIntent IQ constellation (defaults to Claude 3.7 Sonnet v1)

1. Deploy the Lambda function to the S3 bucket listed above. There are several ways to automate the deployment of Lambdas: one is to embed the code directly in the yaml file, another is to reference the code as a zip file in an Amazon S3 bucket. We use the latter mechanism. Simply zip up the Python function, `lambda_function.py` in the satcom-ts-bot-intent folder, and upload it to the SatComBotS3Bucket.

![Capture-S3-lambda-zip](https://github.com/user-attachments/assets/0ee48f43-7155-4c97-bfab-9582681dbfdb)

2. In addition to the SageMaker model endpoint name, the [invoke_endpoint](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker-runtime/client/invoke_endpoint.html) API requires a `Body` parameter. Autopilot timeseries inference requires a small (8 or more rows) amount of input data with the same format as the desired prediction output (same column headings etc).

Sample input files have been provided at [endpoint-sample-input-data](endpoint-sample-input-data). Copy all of these csv files to your `dataset/rtinf` folder in your S3 bucket.
![Capture-s3-input-sample-files](https://github.com/user-attachments/assets/cb308625-4968-4a4a-95ea-e540a2c30e10)

**Note** - if you upload to a different folder structure, change the `S3_FOLDER_PREFIX` in the [Bot Lambda function](satcom-ts-bot-intent/lambda_function.py)

**Note** - if you wish to use the generated results from the `inf` mode of [satcom-timeseries-autopilot-gen-fxn](https://github.com/aws-samples/satellite-comms-forecast-aws/tree/main/satcom-timeseries-autopilot-gen-fxn), change the csv filenames in the [Bot Lambda function](satcom-ts-bot-intent/lambda_function.py)

3. To test the ImageIntent we supply a sample of noise pattern images e.g. `interference-0.jpeg`, `phase_noise-3.jpeg` etc. Copy all of these JPEG files to a new folder `iq-constellation-images` in your S3 bucket.

![Capture-iq-images-s3](https://github.com/user-attachments/assets/d2bbd173-add6-4a81-82fe-4461a6f4453a)

**Note** - if you upload to a different folder structure, change the `S3_FOLDER_IQ_IMAGES_PREFIX` in the [Bot Lambda function](satcom-ts-bot-intent/lambda_function.py)

4. Deploy the next stack using the following commands to provision the resources in your AWS account. Copy all of these csv files to your dataset/rtinf folder in your S3 bucket.

`aws cloudformation create-stack --stack-name <stack-name> --template-body file://satcom-ts-lexbot.yaml --capabilities CAPABILITY_NAMED_IAM --parameters ParameterKey=<parameter key>,ParameterValue=<parameter value>` ....

**Note** - the chatbot implementation is specific to the Satellite Capacity forecasting use-case. Use it as a reference for your own Lex intents, [slot types](https://docs.aws.amazon.com/lexv2/latest/dg/add-slot-types.html) etc.

5. Open Amazon Lex and click on the generated Bot `SatelliteCapacityChatbot`. Navigate to `Intents` which should look as follows: -

![Capture-intents](https://github.com/user-attachments/assets/b4f85c86-c233-4fad-a3e2-dc86291c3a85)

The CFN creates all of the Lex resources but does not build the bot. Hence click `Build`. 

6. Click `Test` in the Lex console upper right corner, and test the `BeamForecast` intent. You should get the first few lines of a Satellite Capacity forecast for different spot beams. 

![Capture-Lex-beamforecast](https://github.com/user-attachments/assets/d2b91534-08d3-4959-a887-420c2e55f282)

Now test ImageIntent - type in "get noise type" then select one of the sample noise types to evaluate the LLM responses for noise or interference type. The model should also tell you the modulation scheme (eg 16APSK)

![Capture-iq-1](https://github.com/user-attachments/assets/09e9bc95-38b9-42d0-86a8-4283e17d2ee0)
![Capture-iq-2](https://github.com/user-attachments/assets/59f0eee8-d779-4b60-aa7c-8508d8739364)

Finally, test the Bedrock LLM Agent integration with Lex by asking a satellite related question: -

![Capture-Lex-bedrock](https://github.com/user-attachments/assets/05fa051e-a8e9-4a11-a48e-3a2e7d740f1d)

This completes the satcom-autopilot-chatbot deployment. Happy chatting!

## Clean up

To avoid incurring future charges, delete the resources. You can do this
by first deleting all the files from the S3 buckets, and then deleting the CloudFormation stacks. 

## Troubleshooting

1. No satellite capacity forecast results - if the bot merely returns "Intent BeamForecast is fulfilled" there may be an issue with the `Body` input data being passed to the the [invoke_endpoint](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker-runtime/client/invoke_endpoint.html) API. Recheck your S3 folder and file structure in Step 2 of the final CFN deployment.
2. LLM path returns 'Sorry, I am unable to assist you with this request.' - this can occur if you have not loaded a sufficient number of articles in the Amazon Bedrock KnowledgeBase. Also, don't forget to `Sync` the articles after uploading.
3. It takes a few seconds to get a response from the SageMaker endpoint - this is normal. This problem type requires a larger payload for the history (not a single record as in regression) — then it has to do predictions for all the underlying base models.  Predictions are per quantile and for each step. Finally it has to look up weights from the ensemble and do that math.
4. It takes a few seconds to get a response from the Bedrock agent - this is also normal. First the embeddings model is invoked, then semantic search occurs on the Opensearch vector database. Finally the LLM (Claude 3.5 in this case) summarizes the response.
5. ImageIntent fails - likely reasons are (i) you have chosen a Model which is not multi-modal, or (ii) the Model is not yet available in your region of choice.


