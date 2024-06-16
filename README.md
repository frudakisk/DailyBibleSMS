# DailyBibleSMS

A personal project that utilizes AWS and Twilio to send out text messages to subscribed users automatically. 

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Subscription](#Subscription)
- [Usage](#usage)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

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

- Feature 1: Users can subscribe to receive daily bible verses with extra material to explore by using the keyword "GOD IS GOOD"
- Feature 2: Users can unsubscribe from the service by sending the keyword "NO MORE VERSES"

## Subscription
To subscribe to this service, it is as easy as sending the correct message to the correct number.
Text "GOD IS GOOD" to +1 (888) 417-9745

### Prerequisites

The user must have an AWS account and a Twilio account with a verified toll-free number in order to replicate
this software.

In order to run this software, you need to have C# installed with the .NET Framework, or you can use a different programming language.
Packages that are required include the following and use the CLI command of dotnet add package XXX:
1. AWSSDK.DynamoDBv2
2. Twilio
3. Newtonsoft.Json

It is also recommended to install the following templates using the CLI command of "dotnet new -i XXX":
1. Amazon.Lambda.Templates

Tools to install in the CLI include:
1. dotnet tool install -g Amazon.Lambda.Tools
   This is for packaging the lambda function as a zip file using the command "dotnet lambda package --output-package insert_directory/deploy-package.zip"

## Privacy Policy
The privacy policy, as required by Twilio can be found in the following public link: https://docs.google.com/document/d/1IskYP63QBHA5z6dQ1lcupeykNZ94Wl0_hqzeZpGL9hY/edit?usp=sharing

## How It Works

#### Create Lambda Functions
To create lambda functions, you need to follow the prerequisites for this application. After doing so, the next step is to open up your IDE. I use Visual Studio Code, so all screenshots will be in this IDE.
In the bar at the top, we will create a new lambda project by typing >.NET: New Project... and then selecting "Lambda Empty Function"

One of the lambda functions will contain the code to send bible verses to subscribed numbers. I have supplimented the following code to show how I will be sending bible verses. I will note that the functionality to select a bible verse and format it is arbitrary to this project since this project can be manipulated to send any kind of message to its subscribers.

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
 * aws and twilio. Next, we access out accounts and in the FunctionHandler, we create
 * a message that contains a bible verse and some reference links. This message
 * is sent to all the numbers in out Dynamo DB.
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

        private static readonly HttpClient httpClient = new HttpClient();
        static string? fullChapter;
        static JObject? json;

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
   

