#Salesforce Process Automation

## Introduction

In this project we will be using three different process automation tools and combining them to provide a solution to managing Instructors and Teaching Assistants for web development classes. 

TBD: screen shot of data model

There are three process automation tools that you will use:
* Process Builder - orchestrate all automation actions; recruit Teaching Assistants via Chatter, and prep for emailing the Instructor by creating a Campaign Member child record 
* Flow - append Instructor Bio from Contact record to Class Session Description
* Apex - thank each Instructor and Teaching Assistant by posting a Thanks badge

Let's get started! 

this is how to make a URL in markup [heavy lifting](http://coenraets.org/blog/2016/04/salesforce-slack-bot/) 

## 0 - Defining the Data Model
There is some pre-work before you get to the process automation tools. You need to define the data model. Class details are stored in a custom object. Class sessions are modeled as Campaigns, and the Instructor is added to the Campaign. Instructors and Teaching Assistants are modeled as Contacts and also need to be listed as Campaign Members.

### Pre-Work Steps
1. Create a custom object named Class with one Long Text field named Description.
2. Customize Campaign: add field for Class (Lookup to Class); add field for Instructor (Lookup to Contact); edit Campaign Type picklist to include DEV Class
3. Customize Campaign Member: add field for Class Role (Picklist with values Instructor, Teaching Assistant)
4. Create Women in Tech Chatter Group: Private, Allow Customers
5. Create sample data: at least one Instructor, create as Chatter External user and as Contact
6. Create sample data: at least one Teaching Assistant, create as Chatter External user and as Contact
7. Add Instructors and Teaching Assistants to the Women in Tech Chatter Group

## 1 - Automating Processes for New Class Sessions
As the Chapter Leader, you've done the legwork to identify the starting point for a new class session. Now you need to get it into the system and start recruiting Teaching Assistant volunteers. You've been doing this manually, but it is always the same thing: post to the Women in Technology Chatter Group, provide the details of the class, and ask volunteers to email you. Also, the Instructor and the Teaching Assistants need to be Campaign Members so that you can send group emails for the class. Currently, you do that manually - first adding the Instructor at the Campaign level for easy visiblity and then creating a Campaign Member record for the instructor. Let's save you some time and automate that part.

### What you will do
1. Create a process in Process Builder for the Campaign object
2. Add the process Criteria
3. Add an Action to post to Chatter to ask for volunteers
4. Add an Action to create a Campaign Member record for the Instructor
5. Activate and test the process

![Salesforce to Slack](5.6 - salesforceToSlack.png)

## Automating with Point & Click...
### Create Process and Define Criteria
These two automations can be done right within the Process Builder, completely declaratively! 

Lets fire up our Process Builder and create this rule.
![Create Process](5.0 - create process.png)

Click New and populate the details of your new Process.

![New Process](5.2 - new process window.png)

Select the Campaign object, then add the selection criteria. i.e. When does the action need to fire? In our case, Campaigns are used for many things, so we only want to fire the actions when it is a DEV Class we want to notify our project teams through Slack when an Opporunity record changes stage. Notice here we could add plenty of different conditions if required, we might not want to spam our team with every minor detail? 

![Process Criteria](5.3 - Process Criteria.png)

Now we can add an action that calls our fresly minted Apex class, ready to accept the two parameters that we annotated with the @InnvocableVariable annotation.
![Set the Action](5.4 - Configure Apex Class.png)

### Test
Your functioning integration should now be ready to test. Go ahead and login to Slack

[![Process Builder](6.3 - Step1Video.png)](https://youtu.be/M8gEkDk0bto)



Create your own Slack account for this exercise, your company might already have teams on Slack so when you register with your email address you could be prompted to join one of these existing channels.. They may not appreciate your positing updates from your Salesforce Developer environment so lets create a new team from scratch. Go ahead and let the wizard guide you through creation of your own Slack Team, you should end up with something like this.

![Create Slack Team](3 - Create Team.png)

Once you have created your team, add yourself a new Channel. Slack Channels are an effective way of broadcasting information to a group of users who are interested, they express their interest by subscribing to channels they are interested. Our channel we are creating now will allow members of the project team to be alerted, if they are interested (i.e. if they subscribe!)

![Create Slack Channel](1 - Create Slack Channel.png)

### Create Slack Integration
Once you have your team and channel in place you will need to add an integration, this is an endpoint for Salesforce to call when it has something interesting to report. Click on the gear icon in the top right hand corner and select "Add an app or integration".

![Add the Integration](4.1 - Add an integration.png)

Select the integration you need from the list provided, we want to add the "Incoming Webhooks" app to our Team. Click "Install" to add this to your environment.

![Install Webhooks](4.2 - Install Webhook.png)

Lastly, we need to configure an endpoint for Salesforce to call. The configuration allows you to select a Channel to POST into, set some labels and also provides the endpoint URL that you will need later in the tutorial.

![Configure Webhook](4.3 - Webhook settings.png)

Now you should have a new Slack Team, Channel and a configured webhook that is ready to go. Before we get to the fun stuff (i.e. Salesforce!) we can test our Webhook is working correctly with a simple cURL command from the commandline. 

```
curl -X POST -H 'Content-type: application/json' --data '{"text":"This is a test post from cURL"}' https://hooks.slack.com/services/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
```
Should result in something like this.

![Testing With cURL](4.4 - cURL test.png)

## The Salesforce Bit...
Now we have an endpoint configured and listening for something, its time to configure Salesforce to send the messages to Slack. We are going to do this by writing a small piece of Apex code that will be fired from a process we define in the Process Builder

###Apex Class
Now we can create the Apex code that is capable of posting a message to this newly configured Webhook URL. The methods and classes here will allow Opportunity Name and Stage fields to be posted out to the the URL. 

```
public with sharing class SlackOpportunityPublisher {
     
    private static final String slackURL = 'YOUR_WEBHOOK_URL';
     
    public class Oppty {
        @InvocableVariable(label='Opportunity Name')
        public String opptyName;
        @InvocableVariable(label='Stage')
        public String stage;
    }
     
    @InvocableMethod(label='Post to Slack')
    public static void postToSlack(List<Oppty> oppties) {
        Oppty o = oppties[0]; // If bulk, only post first to avoid overloading Slack channel
        Map<String,Object> msg = new Map<String,Object>();
        msg.put('text', 'The following opportunity has changed:\n' + o.opptyName + '\nNew Stage: *' + o.stage + '*');
        msg.put('mrkdwn', true);
        String body = JSON.serialize(msg);    
        System.enqueueJob(new QueueableSlackCall(slackURL, 'POST', body));
    }
     
    public class QueueableSlackCall implements System.Queueable, Database.AllowsCallouts {
         
        private final String url;
        private final String method;
        private final String body;
         
        public QueueableSlackCall(String url, String method, String body) {
            this.url = url;
            this.method = method;
            this.body = body;
        }
         
        public void execute(System.QueueableContext ctx) {
            HttpRequest req = new HttpRequest();
            req.setEndpoint(url);
            req.setMethod(method);
            req.setBody(body);
            Http http = new Http();
            HttpResponse res = http.send(req);
        }
 
    }
    
}
```
Notice the [@InvocableVariable](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableVariable.htm) and [@InvocableMethod](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableMethod.htm) annotations on this class that allow these methods to be exposed to the configuration tools in the Salesforce system. Other interesting Apex features in this code are the use of [System.Queueable](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_Queueable.htm%23apex_class_System_Queueable) and [Database.AllowsCallouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm) interfaces that are being used. 


### Build the Process
Now we can define the business process that will cause the notifcation to fire and be propogated into Slack. Here we get to see one of the really powerful features of Salesforce when you writing code, the amount of code you don't have to write! As a developer all we needed to do was build a small module that took some parameters and passed them to our Slack endpoint, the who, when and why of this integration is now completely declarative! 

Lets fire up our Process Builder and create this rule.
![Create Process](5.0 - create process.png)

Click New and populate the details of your new Process.

![New Process](5.2 - new process window.png)

Select the Opportunity object, then add the selection criteria. i.e. When does the action need to fire? In our case, we want to notify our project teams through Slack when an Opporunity record changes stage. Notice here we could add plenty of different conditions if required, we might not want to spam our team with every minor detail? 

![Process Criteria](5.3 - Process Criteria.png)

Now we can add an action that calls our fresly minted Apex class, ready to accept the two parameters that we annotated with the @InnvocableVariable annotation.
![Set the Action](5.4 - Configure Apex Class.png)

### Test
Your functioning integration should now be ready to test. Go ahead and login to Slack

[![Process Builder](6.3 - Step1Video.png)](https://youtu.be/M8gEkDk0bto)


## 2 - View Salesforce Data Using Slash Commands
Now our project team is fully aware of the latest and greatest news on deals in real time from Salesforce, but what if they wanted to interact with Salesforce data themselves? Should these users really have to leave their beloved Slack interface if they just wanted to see a few opportunities, contacts or to create a simple case? Luckily we have a few handy developers on staff who can pull together a little integration that will allow just this. Lets have a look at [Slack Slash Commands](https://api.slack.com/slash-commands) and another awesome component from the Salesforce App Cloud, Heroku. 

We want to setup a few different scenarios here :
* Show the top opportunities from Salesforce  (/pipeline[number to show])
* Search for a Salesforce Contact in the Slack UI (/contact[search key])
* Create a customer service Case from the Slack UI (/case[subject:description])

To achieve this we are going to :
* Setup a Salesforce Connected App
* Create a Node.js application that will serve as the proxy between Salesforce and Slack (well, actually we are just going to copy one!)
* Configure Slash Commands in Slack

![Slack to Salesforce](6.4 - Slack to Salesforce.png)

### Architecture
We need to setup a small Heroku app to broker the communcation between Slack and Salesforce. This app is going to use [Node.js](https://nodejs.org/) and the [nForce](https://github.com/kevinohara80/nforce) module to provide convenience methods for accessing Salesforce. If you have yet to get your [Heroku](www.heroku.com) account, head over and sign up for the free tier. 

![Heroku Sign Up](7.1 - sIgnupHeroku.png)

We need to let Salesforce know that an application is going to want to use the API, so we have to configure a the Connected App in our developer environment. Go to Apps to create a new Connected App. 

![New Connected App](7.2 - ConnectAppConfig.png)

Configure, don't worry about the callback URL yet as we are going to change that later! 

![Configure Connected App](7.3 - ConnectedAppConfig.png)

Now we can deploy this [application](https://github.com/ccoenraets/slackforce) into our newly (or well used) Heroku environment, it is a trivial thing to deploy this appliction thanks to Heroku... try it, just click this button. 

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/ccoenraets/slackforce)

You will be asked to configure some properites on this application

* SF_CLIENT_ID - enter the Consumer Key of your Salesforce Connected App
* SF_CLIENT_SECRET - enter the Consumer Secret of you Salesforce Connected App
* SF_USER_NAME - this is the username of the Salesforce integration user
* SF_PASSWORD - this is the password for this user
* SLACK_OPPORTUNITY_TOKEN, SLACK_CONTACT_TOKEN and SLACK_CASE_TOKEN are blank for now (we get back to this)

Once you have deployed, you should see the an output similar to this. 

![Success Heroku](7.4 - SuccessDeploy.png)

### Create the Slack Commands
We now need to do some configuration work in the Slack UI to create the actual Slack commands that end users will need. In your browser, open up Slack (if you haven't already). As an example, my Slack team URL is https://sforce-slack-demo.slack.com. We are going to add another integration, however this time we will add Slash Command (remember our first one was a Webhook)

![Add Integration](8.0 - AddIntegration.png)

![Add Slash Command](8.1 - SlashCommand.png)

Click install and then add the following commands to your Team.


Command | URL | Method | Custom Name
--------|-----|--------|------------
/pipeline | https://app_name.herokuapp.com/pipeline | POST | Top Opportunities
/contact | https://app_name.herokuapp.com/contact | POST | Salesforce Contacts
/case | https://app_name.herokuapp.com/case | POST | Salesforce Cases

You will need to create a separate Slash Command for each use case, configuring a Slash command should look like this. This is also where you will get you Slack Token that needs to be configured in your Heroku application.

![Configure](8.2 - slashConfig.png)

Now we have created our Slash Commands, lets update our Heroku app with the Tokens that were just generated for each command. From the Settings tab in your [Heroku Dashboard](dashboard.heroku.com), reveal the config vars and edit. Oh, and don't forget to generate and append your security token to the SF_PASSWORD variable!!

![Tokens](8.3 - TokensInHeroku.png)

Lastly, remember we said there was a Connected App setting we needed to come back to? Thats right, our callback URL is not a localhost address anymore but should now point to our Heroku app.

![Callback](8.4 - CallbackScope.png)

Congratulations, you should now be able to pull Salesforce data into your Slack stream.

[![Salesforce in Slack](8.5 - Step2Video.png)](https://youtu.be/xB-1SsUoBHk)


## 3 - Its all in the Bot

In the last part of our workshop we are going to create an integration using bots, we are going to monitor Slack channels and respond to Salesforce requests expressed in natural language. 

We are going to create an applciation that opens a Websocket connection to Slack, allowing the bot to listen to the Slack channel as well as any direct messages that users send directly to the bot. We are going to use [Botkit](https://github.com/howdyai/botkit) to abstract some of the low level details of Slack (and [Facebook](https://blog.howdy.ai/botkit-for-facebook-messenger-17627853786b)) bot buildling. 

To finish off today, we are going to :
* Configure another Salesforce Connected App
* Create another Heroku App
* Create a bot user in Slack

### 1 - Create a Salesforce Connected App

Create Connected app.

![Connected App Config](9.1 - ConnectedApp Config.png)


### 2 - Create the Slack Bot User

Lets add a bot to our Slack Team. 

![Add the Bot](9.2 - AddTheBot.png)

### 3 - Create the Heroku App

Go ahead and use your Heroku again (or follow this for deploying locally)

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/ibigfoot/salesforce-bot-slack)

Setup your config vars, again to remember that your security token should be appended to the password.

![Heroku Config](9.3 - HerokuAppConfig.png)


You should now be able to have a conversation with your Salesforce Bot!

[![Salesforce Bot](10 - Step3Video.png)](https://youtu.be/1EPNbHi-3UY)


