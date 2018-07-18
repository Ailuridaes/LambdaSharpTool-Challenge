# July2018-LambdaSharp

In this challenge we're going to learn how to use MindTouch's λ# tool to turn simple yaml markup in to a CloudFormation stack and quickly itterate on a stack of functionality provided by a number of Lambda functions and API Gateway

The architecture should look like this:
![stack](flow.png)

Please follow the teardown instructions at the bottom of this readme when the challenge is complete to avoid unnecessary charges from AWS.

## Pre-requisites

The following tools and accounts are required to complete these instructions.

- [Complete Step 1 of the AWS Lambda Getting Started Guide](http://docs.aws.amazon.com/lambda/latest/dg/setup.html)
  - Setup an AWS account
  - [Setup the AWS CLI](https://docs.aws.amazon.com/lambda/latest/dg/setup-awscli.html)
- .NET Core 2.1 is required
  - <https://www.microsoft.com/net/learn/get-started>

## Level 0 - Setup λ#
- Ensure you have your AWS credentials file set up in the .aws folder under your home directory
- Clone λ# repo <https://github.com/LambdaSharp/LambdaSharpTool>
- Clone Messages App repo <https://github.com/LambdaSharp/July2018-LambdaSharp>
- Setup command alias for the LambdaSharp tool
  - `$LAMBDASHARP=/path/to/LambdaSharpRepo`
  - `alias lst="dotnet run -p $LAMBDASHARP/src/MindTouch.LambdaSharp.Tool/MindTouch.LambdaSharp.Tool.csproj --"`
- Bootstrap deployment
  - This step provides some necessary infrastructure to your AWS account.
  - `lst deploy --bootstrap --deployment {DeploymentName} --input $LAMBDASHARP/BootStrap/LambdaSharp/Deploy.yml`
- Deploy the Messages stack to the same deployment
  - `lst deploy --deployment {DeploymentName} --input /path/to/MessagesAppRepo/Deploy.yml`

Now that you have deployed the Messages stack you will have the following infrastructure in your AWS account:
- (DynamoDB Table) MessageTable
  - Stores messages with a unique identifier (_MessageId_) and a source field
- (S3 Bucket) IngestionBucket
  - A bucket to drop text files in to to be loaded into _MessageTable_. Sends event notifications to _LoadMessages_
- (Lambda Function) PostMessages
  - Recieves a message from API Gateway and writes the body as a message to _MessageTable_
- (Lambda Function) GetMessages
  - Reads from _MessageTable_ and returns list of all messages available
- (Lambda Function) LoadMessages
  - Reads from _MessageTable_ and returns list of all messages available. This function is not complete and is to be completed by the user.
- (API Gateway Endpoint) POST:/
  - Sends payload to _PostMessages_ Lambda function
- (API Gateway Endpoint) GET:/
  - Sends payload to _GetMessages_ Lambda function
- (IAM Role)
  - All necessary permissioning is granted

Once the stack is deployed you can post to the API endpoint that is output by the tool. The body of the request will be stored as a message in DynamoDB. All messages can be retrieved by issuing a GET request to the same endpoint.

## Level 1 - Implement Bulk Load Lambda Function

The S3 bucket that we created sends CreateObject notifications to the LoadMessages Lambda function. This function, however, does not do anything with the message. The challenge here is to complete the Lambda function code so that it will read the contents of the text file that was uploaded and consider each line a message that should be written into the DynamoDB table.

NOTES: 
- The "source" field should be "S3" here.
- To simplify things, all needed DynamoDB functionallity has been abstracted in the Messages.Tables library so you will not need to interact with it directly.
- Be cautious of the fact that the DynamoDB calls are asyncronous and must be completed before the Lambda function returns.

## Level 2 - Add DeleteMessages Lambda Function

The next piece of functionality we would like to add is the ability to clear out all messages from the DynamoDB table. Add a new entry under the `Functions` section of Deploy.yml for _DeleteMessages_. A new C# project similar to those that exist for the other Lambda functions will need to be added. The λ# tool uses naming conventions to find the project and provides a utility to create a new project which follows the convention and contains all necessary references. In the directory with the Deploy.yml file execute the following command:

`lst new function --name Messages.DeleteMessages`

The new function inherits from ALambdaFunction, which has limited functionality. Since we would like this to be an API Gateway function you can replace `ALambdaFunction` at the class definition with `ALambdaApiGatewayFunction`. You can then remove the `Task<object> FunctionHandlerAsync(Stream stream, ILambdaContext context)` method. Next add a method with the signature `override Task<APIGatewayProxyResponse> HandleRequestAsync(APIGatewayProxyRequest request, ILambdaContext context)` as that is required by `ALambdaApiGatewayFunction`.

## Level 3 - Add Another Source For Messages

We can now add messages to the DynamoDB table via API Gateway and S3. Choose another source (SNS, SQS, etc) to populate messages and add another lambda function to do so. Examples of sources can be found in the LamdbaSharp repo

## Boss Challenge

![boss](http://images2.fanpop.com/image/photos/10400000/Bowser-nintendo-villains-10403203-500-413.jpg)

Create a new stack (new Deploy.yml file and corresponding csprojs) that will interact with the DynamoDB table. Adding the key `Export` with a value of `/{{Deployment}}/dynamo` will populate the value of the DynamoDB table's name in that location within parameter store. In your new deploy file add the following section under `Parameters`:

```
  - Name: MessageTable
    Description: Imported DynamoDb table for storing received messages
    Import: /{{Deployment}}/dynamo
    Resource:
      Type: AWS::DynamoDB::Table
      Allow: Read
```

With this section you will have read access to the DynamoDB talbe from the Messages stack and you will be able to access the name of the table within the Lambda function with the line `var tableName = config.ReadText("MessageTable");` in the `InitializeAsync` method.

NOTES:
- The Name value in the root of the Deploy file must be different than any other stack you have defined (in this case, Messages).


## Teardown

Since all of the infrastructure was deployed with CloudFormation it is easy to get rid of all of it. In the AWS console navigate to the CloudFormation dashboard. Find your stack (it will me named {Deployment}-Messages). Click the checkbox next to it, click the `Actions` button, and select `Change termination protection`. When the dialog pops up click the `Yes, Disable` button. After the page refreshes click `Actions` again and select `Delete Stack` then confirm when the dialog pops up. The stack will change status to `DELETE_IN_PROGRESS` for about a minute then you will see that it changes to `DELETE_FAILED` after a refresh of the page. The delete fails since we have created an S3 bucket which cannot be automatically deleted by CloudFormation. The next step is to once again select `Actions` and `Delete Stack`. This time when the confirmation dialog pops up click the checkbox for IngestionBucket, acknowledging that it will not be deleted before clicking `Yes, Delete`. Repeat this process for the {Deployment}-LambdaSharp stack then, navigate to the S3 console to delete the S3 buckets that were created for those two stacks.