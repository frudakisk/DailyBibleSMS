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

In order to run this software, you need to have C# installed with the .NET Framework.
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
TO create lambda functions, you need to '''install Amazon.Lambda.Tools'''
   

