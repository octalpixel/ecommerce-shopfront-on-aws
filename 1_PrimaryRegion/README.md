# Building the Bookstore in your Primary Region

In this module, you will deploy Bookstore application in Irelad region. This components include followings:
1. S3 - Web statci content
2. API Gateway and Cognito - App layer with authentication
3. DynamoDB - Books, Order, Cart table
4. Lambda - multiple functions

You will also create the IAM polices and roles required by these components.

**Frontend**

Build artifacts are stored in a S3 bucket where web application assets are maintained (like book cover photos, web graphics, etc.). Amazon CloudFront caches the frontend content from S3, presenting the application to the user via a CloudFront distribution.  The frontend interacts with Amazon Cognito and Amazon API Gateway only.  Amazon Cognito is used for all authentication requests, whereas API Gateway (and Lambda) is used for all API calls interacting across DynamoDB, ElasticSearch, ElastiCache, and Neptune. 

**Backend**

The core of the backend infrastructure consists of Amazon Cognito, Amazon DynamoDB, AWS Lambda, and Amazon API Gateway. The application leverages Amazon Cognito for user authentication, and Amazon DynamoDB to store all of the data for books, orders, and the checkout cart. As books and orders are added, Amazon DynamoDB Streams push updates to AWS Lambda functions that update the Amazon Elasticsearch cluster and Amazon ElasticCache for Redis cluster.  Amazon Elasticsearch powers search functionality for books, and Amazon Neptune stores information on a user's social graph and book purchases to power recommendations. Amazon ElasticCache for Redis powers the books leaderboard. 

### AWS Lambda

AWS Lambda is used in a few different places to run the application, as shown in the architecture diagram.  The important Lambda functions that are deployed as part of the template are shown below, and available in the [functions](/functions) folder.  In the cases where the response fields are blank, the application will return a statusCode 200 or 500 for success or failure, respectively.

### Amazon ElastiCache for Redis

Amazon ElastiCache for Redis is used to provide the best sellers/leaderboard functionality.  In other words, the books that are the most ordered will be shown dynamically at the top of the best sellers list. 

For the purposes of creating the leaderboard, the AWS Bookstore Demo App utilized [ZINCRBY](https://redis.io/commands/zincrby), which *“Increments the score of member in the sorted set stored at key byincrement. If member does not exist in the sorted set, it is added with increment as its score (as if its previous score was 0.0). If key does not exist, a new sorted set with the specified member as its sole member is created.”*

The information to populate the leaderboard is provided from DynamoDB via DynamoDB Streams.  Whenever an order is placed (and subsequently created in the **Orders** table), this is streamed to Lambda, which updates the cache in ElastiCache for Redis.  The Lambda function used to pass this information is **UpdateBestSellers**. 

### Amazon CloudFront and Amazon S3

Amazon CloudFront hosts the web application frontend that users interface with.  This includes web assets like pages and images.  For demo purposes, CloudFormation pulls these resources from S3.

**Developer Tools**

The code is hosted in AWS CodeCommit. AWS CodePipeline builds the web application using AWS CodeBuild. After successfully building, CodeBuild copies the build artifacts into a S3 bucket where the web application assets are maintained (like book cover photos, web graphics, etc.). Along with uploading to Amazon S3, CodeBuild invalidates the cache so users always see the latest experience when accessing the storefront through the Amazon CloudFront distribution.  AWS CodeCommit. AWS CodePipeline, and AWS CodeBuild are used in the deployment and update processes only, not while the application is in a steady-state of use.

![Developer Tools Diagram](assets/readmeImages/DeveloperTools.png)

<summary><strong>CLI step-by-step instructions </strong></summary>


Navigate to the `1_API` folder within your local Git repository and take a look at the
files within. You will see several files - here are descriptions of each:

* `wild-rydes-api.yaml` – This is a CloudFormation template (using SAM syntax) that
  describes the infrastructure needed to for the API and how each component should be configured.
* `tickets-get.js` – This is the Node.js code required by our Lambda function needed
  to retrieve tickets from DynamoDB
* `tickets-post.js` – This is the Node.js code required by our second Lambda function
  to create new tickets in DynamoDB
* `health-check.js` - Lambda function for checking the status of our application health


There is no modification necessary to this application code so we can go ahead and
deploy it to AWS. Since it comes with a CloudFormation template, we can use this to
upload our code and create all of the necessary AWS resources for us rather than doing
this manually using the console which would take much longer. 
<!--We recommend deploying the
primary region using the Console step-by-step instructions and then deploying the failover
region using the CloudFormation template. 
--> 
Feel free to open the template and take a look at the resources it is creating and how they are defined.

## 1. Create an S3 bucket to store the app code

We'll first need a bucket to store our AWS Lambda source code in AWS.

#### High-level Instructions

Go ahead and create a bucket for each region using the CLI. S3 bucket names must be
globally unique so choose a name for your bucket using something unique to you such as
your name e.g. `wildrydes-firstname-lastname`.

Let's create 2 buckets using the CLI with the following command:

*Ireland Bucket* (choose a unique bucket name)
     
     aws s3 mb s3://multiregion-<firstnamelastname>-ireland --region eu-west-1

*Singapore Bucket*
     
     aws s3 mb s3://multiregion-<firstnamelastname>-singapore --region ap-southeast-1

## 2. Package up the API code and push to S3

Because this is a [Serverless Application Model Template](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html), we must first package it. This process will upload the source code to our S3 bucket and generate a new template referencing the code in S3 where it can be used by AWS Lambda.

#### High-level instructions

Go ahead and create two SAM deployment package in Ireland and Singapore regions respectively.
You can do this using the following CLI command. Note that you must replace
`[bucket-name]` in this command with the bucket you just created:

Execute this command to enter the correct subdirectory:

    cd /home/ec2-user/environment/MultiRegion-Serverless-Workshop/1_API

*Ireland*

    aws cloudformation package \
    --region eu-west-1 \
    --template-file wild-rydes-api.yaml \
    --output-template-file wild-rydes-api-output.yaml \
    --s3-bucket [Ireland bucket_name_you_created_above]

<!--
**IMPORTANT** DO NOT deploy any resources to Singapore during your initial pass
on Module 1. You will come back in Module 3 and then deploy the same components
to Singapore. We are providing the commands for both regions here for your
convenience.
-->

*Singapore* 

    aws cloudformation package \
    --region ap-southeast-1 \
    --template-file wild-rydes-api.yaml \
    --output-template-file wild-rydes-api-output-ap-southeast-1.yaml \
    --s3-bucket [Singapore bucket_name_you_created_above]


<!-- If all went well, you should get a success message and instructions to deploy your new template. -->
If all went well, there will be 2 cloudformation templates generated: `wild-rydes-api-output-ap-southeast-1.yaml` for Singapore and `wild-rydes-api-output.yaml` for Ireland. follow the instruction below to deploy the cloudformation template.

## 3. Deploy a stack of resources

Let's spin up the resources needed to run our code and expose it as an API using the 2 cloudformation templates.

#### High-level instructions

You can now take the newly generated Cloudformation templates (`wild-rydes-api-output.yaml` for Ireland and `wild-rydes-api-output-ap-southeast-1.yaml` for Singapore) and use them to create resources in AWS.
Go ahead and run the following CLI command:

*Ireland*

    aws cloudformation deploy \
    --region eu-west-1 \
    --template-file wild-rydes-api-output.yaml \
    --stack-name wild-rydes-api \
    --capabilities CAPABILITY_IAM

<!--
**IMPORTANT** DO NOT deploy any resources to Singapore during your initial pass
on Module 1. You will come back in Module 3 and then deploy the same components
to Singapore. We are providing the commands for both regions here for your
convenience.
-->

*Singapore* 

    aws cloudformation deploy \
    --region ap-southeast-1 \
    --template-file wild-rydes-api-output-ap-southeast-1.yaml \
    --stack-name wild-rydes-api \
    --capabilities CAPABILITY_IAM


This command may take a few minutes to run. In this time you can hop over to the AWS console
and watch all of the resources being created for you in CloudFormation. Open up the AWS Console in your browser
and check you are in the respective regions (EU Ireland or Asia Pacific. You should check your stack listed as `wild-rydes-api`. 

Once your stack has successfully completed, navigate to the Outputs tab of your stack
where you will find an `API URL`. Take note of this URL as we will need it later to configure
the website UI in the next module.

You can also take a look at some of the other resources created by this template. Under
the Resources section of the Cloudformation stack you can click on the Lambda functions
and the API Gateway. Note how the gateway was configured with the `GET` method calling
our `TicketGetFunction` Lambda function and the `POST` method calling our `TicketPostFunction`
Lambda function. You can also see that an empty DynamoDB table was set up as well as IAM
roles to allow our functions to speak to DynamoDB.

Now, you can confirm that your API is working by copying your `API URL` and appending `/ticket`
to it before navigating to it into your browser. It should return the following:

For example: 

    https://XXXXXX.execute-api.eu-west-1.amazonaws.com/prod/ticket
    https://XXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/ticket

 It should return the following:

    {"Items":[],"Count":0,"ScannedCount":0}

You can also run the health check by copying your `API URL` and appending `/health`
to it before navigating to it into your browser.

For example:

    https://XXXXX.execute-api.eu-west-1.amazonaws.com/prod/health
    https://XXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/health

 It should return the following (for Ireland):

    {
        "region":"eu-west-1",
        "message":"Successful response reading from DynamoDB table."
    }

Make note of the API Endpoint URL - you will need this in Module 2_UI.

#### Enable DynamoDB Global Table using CLI

**IMPORTANT** DyanmoDB Global Tables doesn't support the CloudFormation yet, we need to create a
global table using the console or AWS CLI (To create a global table using the console, you can refer 
to the '2. Create the DynamoDB Global Table' section under 'Console step-by-step instructions'). 

Follow the steps to create a global table (SXRTickets) consisting of replica tables in the Ireland and 
Singapore regions using the AWS CLI. 

<!--
*Enable Streaming on DynamoDB*
_Ireland_

    aws dynamodb update-table --table-name SXRTickets \
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
    --region eu-west-1 

_Singapore_

    aws dynamodb update-table --table-name SXRTickets \
    --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
    --region ap-southeast-1
-->

*Enable DynamoDB Global Table between Ireland and Singapore*

    aws dynamodb create-global-table \
    --global-table-name SXRTickets \
    --replication-group RegionName=eu-west-1 RegionName=ap-southeast-1 \
    --region eu-west-1

## Completion

Congratulations you have configured the backend components required by the
ticketing application. In the next module you will deploy a frontend that uses
these components.

Module 2: [Build a UI layer](../2_UI/README.md)