# DailyBibleSMS

A personal project that utilizes AWS and Twilio to send out text messages to subscribed users automatically. 

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Subscription](#Subscription)
- [Privacy Policy](#privacy-policy)
- [In Action](#in-action)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Create Lambda Functions](#create-lambda-functions)
- [Setting up AWS Environment](#setting-up-aws-environment)

## Introduction

This project is a combination of many moving parts. It utilizes different aspects of AWS such as
Lambda, DynamoDB, and API Gateway. DailyBibleSMS is designed for people who want to be in the word of God.
By sending daily bible verses to subscribed users, they are introduced to scripture that they may not have read
otherwise and gain more understanding of who God really is. In addition, extra material, such as a link to the chapter
that the verse came from and a synopsis of that chapter, is included in the sms message. This program is particularly useful 
because of its SMS capabilities. Typically, people are more likely to view a text messages than an app notification, email, or
smap phone call. DailyBibleSMS is a non-intrusive way to spread the word of God to those who want to hear about it.

This Github will not include the source code for this project because there is some sensitive information in there api keys, identifications, etc.

## Features

- Feature 1: Users can subscribe to receive daily bible verses with extra material to explore by using the key phrase "GOD IS GOOD"
- Feature 2: Users can unsubscribe from the service by sending the key phrase "NO MORE VERSES"

## Subscription
To subscribe to this service, it is as easy as sending the correct message to the correct number.
Text "GOD IS GOOD" to +1 (888) 417-9745

## Privacy Policy
The privacy policy, as required by Twilio can be found in the following public link: https://docs.google.com/document/d/1IskYP63QBHA5z6dQ1lcupeykNZ94Wl0_hqzeZpGL9hY/edit?usp=sharing

## In Action
As a preview (and proof that this works) I have supplied some screenshots of the messages send to subscribed users and how the subscribing and unsubscribing system works/looks like

![IMG_2876](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/df550b30-9e8c-4910-8e21-a4e3e82ee14a)
![IMG_2877](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/fa9feb07-1a21-4fb0-a4fc-970927904917)
![IMG_2879](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/e6027aa9-57f7-44d7-a773-59d472afb2f4)



## How It Works
In this section, I will go over how this whole system works so that other developers can create similar projects without having to go through all the headaches I went through.

### Prerequisites
The user must have an AWS account and a Twilio account with a verified toll-free number in order to replicate
this software. In order to send messages via Twilio, you must allow yourself some funds. I started out with just $20.

To create this program, I used C# and the .NET Framework. Feel free to use a different programming language, but the snippets of code here are going to be in C#.
Packages that are required include the following. Use the CLI command of ```dotnet add package XXX``` in the appropriate directory:
1. AWSSDK.DynamoDBv2
2. Twilio
3. Newtonsoft.Json
4. Amazon.Lambda.APIGatewayEvents

It is also recommended to install the following templates using the CLI command of ```dotnet new -i XXX```:
1. Amazon.Lambda.Templates

Tools to install in the CLI include:
1. ```dotnet tool install -g Amazon.Lambda.Tools```
   This is for packaging the lambda function as a zip file using the command ```dotnet lambda package --output-package yourFileName.zip```

### Create Lambda Functions
To create lambda functions, you need to follow the prerequisites for this application. After doing so, the next step is to open up your IDE. I use Visual Studio Code, so all screenshots will be in this IDE.
In the bar at the top, we will create a new lambda project by typing >.NET: New Project... and then selecting "Lambda Empty Function"

One of the lambda functions will contain the code to send bible verses to subscribed numbers. We will call this file DailyBibleSMS.cs I have supplied the following code to show how I will be sending bible verses. I will note that the functionality to select a bible verse and format it is arbitrary to this project since this project can be manipulated to send any kind of message to its subscribers.

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Amazon.Lambda.Core;
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.DocumentModel;
using Twilio;
using Twilio.Rest.Api.V2010.Account;
using Newtonsoft.Json.Linq;

/*
 * What happens when this file runs?
 * First, we get all of our static strings set up so we can access out accounts in
 * aws and twilio. Next, we access our accounts and in the FunctionHandler, we create
 * a message that contains a bible verse and some reference links. This message
 * is sent to all the numbers in our Dynamo DB.
 */


// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace DailyBibleSMS
{
    public class Function
    {
        private static readonly string tableName = "TableName";
        private static readonly string accountSid = "yourAccountSid";
        private static readonly string authToken = "yourAuthToken";
        private static readonly string fromPhoneNumber = "yourTwilioPhoneNumber";

        public async Task FunctionHandler(ILambdaContext context)
        {
            //init Twilio
            TwilioClient.Init(accountSid, authToken);

            var dynamoDbClient = new AmazonDynamoDBClient();
            var table = Table.LoadTable(dynamoDbClient, tableName);

            var scanFilter = new ScanFilter();
            var search = table.Scan(scanFilter);
            List<Document> documentList = new List<Document>();

            string message = await CreateMessageAsync();

            do
            {
                documentList = await search.GetNextSetAsync();
                //for each number in our db we send a message
                foreach (var document in documentList)
                {
                    string? phoneNumber = document["PhoneNumber"].ToString();
                    await SendMessage(phoneNumber, message);
                }
            } while (!search.IsDone);
        }

        private async Task SendMessage(string toPhoneNumber, string messageBody)
        {
            var message = MessageResource.Create(
                body: messageBody,
                from: new Twilio.Types.PhoneNumber(fromPhoneNumber),
                to: new Twilio.Types.PhoneNumber(toPhoneNumber)
            );
        }
```

This code shows us how to essentially access our twilio account, access our table in aws (more on perissions later), and send a message to numbers in our table. 
Be sure to zip the lambda function using the command line code ```dotnet lambda package --output-package yourFileName.zip``` where yourFileName is your zip file name.

For our second lambda function, we need to have a process for allowing people to subscribe to the service. We will call this program SmsSubscriptionHandler.cs. To do this, we start a separate lambda project just like we did for the first one.

```
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Amazon.Lambda.Core;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.DocumentModel;
using Twilio.TwiML;
using Twilio.TwiML.Messaging;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace SmsSubscriptionHandler
{
    public class Function
    {
        private static readonly AmazonDynamoDBClient dynamoDbClient = new AmazonDynamoDBClient();
        private static readonly Table subscribersTable = Table.LoadTable(dynamoDbClient, "yourTableName");

        //End Point for API to access
        public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest request, ILambdaContext context)
        {

            try
            {
                string msgResponse;
                var formData = System.Web.HttpUtility.ParseQueryString(request.Body);
                string? incomingMsg = formData["Body"].Trim().ToUpper();
                string? fromNumber = formData["From"];

                var response = new MessagingResponse();

                if (incomingMsg == "SUBSCRIBE PHRASE HERE")
                {
                    //check if this number already is subscribed
                    if(await IsNumberInDB(fromNumber) == false)
                    {
                        await AddSubscriberAsync(fromNumber);
                        msgResponse = "your success response here";
                    }
                    else
                    {
                        msgResponse = "You have already subscribed!";
                    }
                }
                else if (incomingMsg == "UNSUBSCRIBE PHRASE HERE")
                {
                    msgResponse = await DeleteSubscriberAsync(fromNumber);;
                }
                else
                {
                    msgResponse = "UNKNOWN INPUT RESPONSE HERE";
                }

                response.Message(msgResponse);

                return new APIGatewayProxyResponse
                {
                    StatusCode = 200,
                    Body = response.ToString(),
                    Headers = new Dictionary<string, string> { { "Content-Type", "application/xml" } }
                };
            }
            catch (Exception ex)
            {
                context.Logger.LogLine($"Error: {ex.Message}");
                return new APIGatewayProxyResponse
                {
                    StatusCode = 500,
                    Body = $"Internal Server Error: {ex.Message}",
                    Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
                };
            }
            
        }
```

This code basically takes any incoming messages to our twilio number and checkes the body of it. If the body text is one of our phrases, we add/remove that number to/from our DynamoDB. Some arbitrary functionality has been left out. Be sure to zip the lambda project by using the same command line code as the last lambda function.

### Setting up AWS Environment

There are four services we have to set up in AWS to make this whole thing work. This includes Lambda, DynamoDB, APIGateway, and EventBridge (Cloudwatch Events).

#### DynamoDB

The table I am using for this project is extremely simple. It only contains one column called PhoneNumber. It is of type string and will just contain a list of phone numbers that have subscribed to this service. 
To create a table, log in to your aws account and search for DynamoDB. Once you are there, you click on "Create a Table" and you should be presented with the following GUI:

![Screen Shot 2024-06-15 at 8 08 36 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/38509110-7728-44d4-98e8-7cb70e864382)

Name the table and partition key appropriatley for your application. Below is a sample of what my table looks like with a couple of numbers subscribed to you. I have redacted all phone numbers for privacy purposes

![Screen Shot 2024-06-15 at 8 12 26 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/6f555e4b-43f0-4449-a02e-22cefbf527ca)

Here you can see I have 5 items in my table.

Once you created your table in DynamoDB, you are ready to move onto the next step, which is importing your lambda code into AWS Lambda.

#### Lambda

Lambda is a service provided by Amazon that allows you to store functions on a cloud and call them in various ways. In our case, we have two lambda functions working for us. One that will send an sms message at a certain time every day (DailyBibleSMS.cs), and another one that waits for an sms from a user to try and subscribe or unsubscribe them (SmsSubscriptionHandler.cs). Creating an lambda function on AWS is pretty easy, so I will touch on it breifly. Essentially, you follow the prompt and insert your zip file into the code section of the lambda function. Your page should looks something like this when youre done:

![Screen Shot 2024-06-15 at 8 53 41 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/6f567e4f-b69b-49bf-89cb-b0da0038839e)

##### SmsSubscriptionHandler.cs Lambda

In the Code Source section, you are going to want to upload the zip file of the appropriate lambda functio we created eariler. The SmsSubscriptionHandler lambda function will be dealing with the API Gateway and the DailyBibleSMS lambda function will be dealing with EventBridge. In this section, we are going to focus setting up the SmsSubscriptionHandler lambda function. After you upload your zip file to the lambda function, you are going to want to make sure you update the runtime settings. Specifically, the default Handler or else you'll run into a time consuming error...

![Screen Shot 2024-06-15 at 8 56 46 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/c1137b53-1361-4f89-a0e5-e088a171c63b)

The syntax for the Handler is a little odd, so lets break it down: ```SmsSubscriptionHandler::SmsSubscriptionHandler.Function::FunctionHandler```
Basically, we tell Lambda what function to run when we call this lambda function (much like the main() method in c++). We start off with the namespace of the project, then we tell lambda that we are going to give it a function. Finally, we tell lambda which function we are going to run. In the case of SmsSubscriptionHandler, there is a method called FunctionHandler which is specifically equipped to be run by Lambda in AWS.

Once we have successfully set up the handler, we need to make sure we have something to test our lambda function with. In the case of the subscription service, we need to imitate a message being sent to us. The formated that a message is sent is in JSON as follows:

```
{
  "resource": "/sms",
  "path": "/sms",
  "httpMethod": "POST",
  "headers": {
    "Content-Type": "application/x-www-form-urlencoded"
  },
  "body": "ToCountry=US&ToState=CA&SmsMessageSid=SMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX&NumMedia=0&ToCity=&FromZip=94016&SmsSid=SMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX&FromState=CA&SmsStatus=received&FromCity=SAN+FRANCISCO&Body=hello world!&FromCountry=US&To=%2B1234567890&ToZip=&NumSegments=1&MessageSid=SMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX&AccountSid=ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX&From=%2B19876543210&ApiVersion=2010-04-01",
  "isBase64Encoded": false
}
```
We can use this template to test out any kind of response our subscription service will use based on the body of the text message.

But before we test our function, we need to give our lambda permissions to access the DynamoDB, to invoke itself, and use API Gateway. In order to add the permissions we must go to configuration -> permissions -> role name -> Add Permission -> Create inline policy. The following JSON are the permissions I have given to each policy

DynamoDB permissions:
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": [
				"dynamodb:PutItem",
				"dynamodb:DescribeTable",
				"dynamodb:DeleteItem",
				"dynamodb:GetItem"
			],
			"Resource": "*"
		}
	]
}
```

Lambda permissions:
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": "lambda:InvokeFunction",
			"Resource": "*"
		}
	]
}
```

API Gateway permissions:
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": [
				"apigateway:DELETE",
				"apigateway:PUT",
				"apigateway:PATCH",
				"apigateway:POST",
				"apigateway:GET"
			],
			"Resource": "*"
		}
	]
}
```

With these permissions added to your lambda function, we should be able to test it out without any permission errors. If you need permission to other services, apply the fundamental rules of doing so.

After we are sure that our code is working as we want it to, we need to set a trigger. In the case of the SmsSubscriptionHandler lambda function, our trigger is going to be an API Gateway. There is a section about how to set up the API Gateway. Here I will explain how to set it up as a trigger. 

For our lambda function, we want to head to the configuration tab, which is just below the description of the lambda function. From here, we want to click on the "Triggers" tab to the left. Create a new trigger and set the type as an API Gateway. Find the API Gateway that we have [created](#API-Gateway) and select is to be our trigger. Now, this subscription lambda function should be triggered by anything that calls our API Gateway endpoint URL.


##### DailyBibleSMS Lambda

This lambda function focuses on sending a message to the subscribed users every day at the same time. Follow the steps for setting up a lambda function (uploading zip file and editing Handler). To test the code for sending messages, we do not need any JSON. all we need to do is have empty brackets ```{}``` in the test section because this process of sending messages should be automatic without any need for input. But, in order to test this function, we need to give it permission to access our DynamoDB. To do this we go to configurations tab, then to the permissions tab. From here, we click on the role name and we should be sent to the IAM console of AWS. Click on "Add Permissions" and then click on "Create inline policy". You want to give this lambda function permission to read and write to your DynamoDB table. I have given my table the following permissions, but your situations might require different selections:

![Screen Shot 2024-06-15 at 9 22 53 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/8901527d-1fb3-4473-b424-902ad3a07160)

You will also have to select a table to have these permissions applied to in the "resources" section. With the permissions added, you should be able to test your lambda function with ease.

Now we need a trigger. To trigger our sms sender, we are going to use EventBridge. At the landing page for our sms sender lambda function, click on "Add Trigger". Set the type as EventBridge and start creating a new rule. The page should look like this:

![Screen Shot 2024-06-15 at 9 15 11 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/a57d45fb-07f7-43c2-84ba-ad0bbf5d9876)

All you really need to do is give the event a name and a schedule. To schedule a time to run this lambda function, we need to know Cron expressions. I did not know how to write Cron expressions, so I looked up my specific case of sending a message every day at the same time, which resulted in the following: ```0 11 * * ? *```. This runs my functions every day at 11AM UTC.

After you have completed setting up your trigger, you are essentially done working with this lambda function. Test your function and make sure that the trigger works!

#### API Gateway

the API Gateway is essentially how you are going to connect your sms messages sent to your twilio number to your SmsSubscriptionHandler lambda function. When a user texted your Twilio number, a POST response is sent to your APIGateway, which is a URL. This URL then takes the information from the text message and sends it over to your integrated lambda function. In our case, this lambda function will be dealing with subscribing and unsubscribing numbers.

To begin this process, you must be logged in to your AWS account. Search for APIGateway and then click on Create API. you will be given a couple of choices on what type of API to create. Click on the "Build" button for Rest API. Create a new API and name it what you want. Leave all other options to their default value. Finally, click on "Create your API". Your API page should look similar to this.

![Screen Shot 2024-06-15 at 8 22 21 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/ab7ced33-7d59-4b69-8eaf-3776feb21f1d)

After you created an API, we need to configure it. There are two things we need to do on this page: resource and stage. Resources in API Gateway refer to the various endpoints or paths that make up your API. These are essentially the different URL paths that the API will respond to. Each resource can represent a collection or a single item within your API, much like endpoints in a RESTful API. Stages in API Gateway are used to manage different versions or environments of your API. They are snapshots of your API configuration at a given point in time and can be used to deploy your API to different environments such as development, testing, staging, and production. Lucky for us, we only need to create one stage and one resource.

Click on the resource tab on the far left and start creating a new resource. All you have to do is name the resource. Leave all other options on their default setting. 

To make a stage, we need to implement a "method" for our API Gateway to run. This method is going to be our lambda function that focuses on subscriptions (SmsSubscriptionHandler). On the resource page, click on the "Create Method" button.

![Screen Shot 2024-06-15 at 8 44 36 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/4a74eb1e-19b8-4302-8ad4-043babcbf098)

On this page, we set the method type as POST, integration type as lambda function, and select our appropriate lambda function in the "Lambda Function" section at the bottom of the screen. Leave everything else as is and create the method.
Once your method is created, you can create a stage. Click on the "Deploy" button at the top right of the resource page and create a new stage. Name it whatever you want. Then deploy the API! To find the URL we will be using, click on the API Settings tab on the far left. Here you should find your default URL under the "Default Endpoint" section. Use this, but add in your stage name and resource name. It should look like this: ```https://<APIid>.execute-api.<region>.amazonaws.com/<stage>/<resource>```. This url is what will connect your lambda function to your text messaging service.

This endpoint is needed to connect your Twilio number to the SmsSubscriptionHandler lambda function. So in Twilio, go to your active numbers and select the phone number you want to work with to send and receive messages with. Make sure your number is verified (follow the verification steps for toll-free numbers on Twilio's website). Head to the Message Configuration section and set your webhook to be the API Gateway endpoint URL. Make sure that the HTTP type is POST.

![Screen Shot 2024-06-15 at 9 09 45 PM](https://github.com/frudakisk/DailyBibleSMS/assets/52004052/2326fe41-b6d0-40c6-8450-ccf28338a47c)

### Conclusion
At this point, Everything should be set up properly and any phone number in your database should be receiving text messages at the same time every day. Also, people can subscribe to your service by entering the key phrase when they text your number.


