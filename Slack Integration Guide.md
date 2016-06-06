# Automate Instructor and Teaching Assistant Management

## Introduction

In this project you will combine three tools to automate the management of Instructors and Teaching Assistants for an organization that delivers coding classes in various Chapters. 

You will use:
* Process Builder - recruit Teaching Assistants via Chatter, and add the Instructor to a Chatter Group 
* Flow - prep for emailing potential Teaching Assistants by creating Campaign Member child records
* Invokable Methods in Apex - thank the Instructor by posting a Thanks badge

You will combine these actions in Process Builder, enabling you to easily maintain the processes and collaborate with other members of your team with the process diagram.

Let's get started! 

## 0 - Defining the Data Model
There is some data model pre-work before you get to automating these processes. Class sessions are modeled as Campaigns, and the Chapter (Account) and Instructor (User) are added to the Campaign. Teaching Assistants are modeled as Contacts.

![Data Model](0.1 - data model2.png)

### Pre-Work Steps
No sense waiting, let's get started...

1. Customize Campaign: add field named Chapter (Lookup to Account); add field named Instructor (Lookup to User); edit Type picklist to include 'Dev Class' value.
2. Create sample data: at least one Account record for a Chapter (e.g., NYC Chapter).
3. Create sample data: at least two Teaching Assistants, create as Contacts with a lookup to the Chapter (Account) you created.
4. Create sample data: one Instructor, create as User with Salesforce license and Standard User profile.
5. Create Women in Technology Chatter Group: set Group Access to Private, Allow Customers.
7. From Classic Setup UI, enable Thanks on Global Publisher: Build | Customize | Work.com | Work.com Settings | enable Thanks Setting 'Turn on Thanks action on the Global Publisher layout.'

## 1 - New Class Sessions: Automating with Process Builder
As the Chapter Leader, you've done the legwork to identify the starting point for a new class session (modeled as a Campaign). Now you need to get the class session into the system and start recruiting Teaching Assistant volunteers. You've been doing this manually, but it is always the same thing: post to the Women in Technology Chatter Group, provide the details of the class, and ask volunteers to email you. Also, you make sure the Instructor is in the Women in Technology Chatter Group. Currently, you do that manually - first adding the Instructor to the Campaign and then heading over to the Women in Technology Chatter Group and adding the Instructor there. Save yourself some time and automate that part, too.

### What you will do
1. Create a process in Process Builder for the Campaign object
2. Add the process Criteria
3. Add an Action to post to Chatter to ask for volunteers
4. Add an Action to call a Quick Create that adds a member to a group
5. Activate and test the process

### Create Process and Define Criteria
These first two automations can be done right within the Process Builder, completely declaratively! 

Lets fire up our Process Builder and create this process.

![Create Process](5.0 - create process.png)

Click New and populate the details of your new process.

![New Process](1.1 - new process window.png)

Select the Campaign object, then add the selection criteria. i.e. When does the action need to fire? In our case, Campaigns are used for many things, so we only want to fire these actions when it is a Dev Class type of Campaign, and we only want to do this for new Campaigns. Notice here we could add plenty of different conditions if required. 

![Process Criteria](1.2 - dev class process criteria.png)

Next add an action that posts to Chatter, using merge fields to fill in the details.
Chatter message should read:
```
HELP WANTED!
I'm looking for Teaching Assistant volunteers for a {![Campaign].Name} class taught by {![Campaign].Instructor__r.FirstName} {![Campaign].Instructor__r.LastName} on {![Campaign].StartDate}!
Please email me at {![Campaign].Owner.Email} if you are interested.
```

![Set the Action](1.3 - chatter post action2.png)

Now add an action of type 'Quick Action'. This lets you call any Global or Object-Specific Quick Action and map in the values it needs to do it's job. 

In the Select and Define Action settings, select Action Type = Quick Actions, and then provide an Action Name. Next, you need to find the action you are looking for...which is the Object-Specific Quick Action to add New Members to a group. Filter Search By 'Object', select the Object labeled 'Group' (API Name CollaborationGroup), and the Action 'NewGroupMember'. 

Now you should see the available values you need to provide. For the 'Related Record ID' field, the Quick Action wants the ID of the Chatter Group, and in this case you will need to hardcode it. Navigate to the Women in Technology Chatter Group and copy the ID from the URL.

Set the Quick Action Field Values as follows:
* Related Record ID | ID | (ID You Copied From The Chatter Group URL, not the one in the screen shot!)
* Member ID | Reference | [Campaign.Instructor__c] 

![Set the Action](1.4 - create record action2.png)

Finally, Activate the process.

### Test
Your functioning process should now be ready to test. Enter data as you would as the Chapter Leader, following these steps:

1. Enter a new Campaign record, setting Campaign Name = the name of a coding class; Campaign Type = Dev Class; Start Date = any date; Chapter = your sample account; Instructor = your sample instructor. Save.
2. Check the Women in Technology Chatter Group. Did your Teaching Assistant recruitment post make it there? Are the merge fields correct in the post? Did the Instructor get added as a group member? If yes, you just made all of the Chapter Leaders more productive!

## 2 - New Class Sessions: Automating with Flow
The next automation requires querying Contacts to create one or more Campaign Members. This is something you cannot do with just the Process Builder capabilities. This is where Flow comes in.

### What you will do
1. Create a new Flow
2. Add a Fast Lookup to find Contacts for the Chapter
3. Add a Loop to iterate through the collection of Contacts
4. Add an Assignment to map the Contact ID and Campaign ID into a Campaign Member collection
5. Add a Fast Create to create Campaign Members from the collection
6. Activate and test the process

Fire up Flow to start building.

![Create Flow](2.1 - flow in setup.png)

### Create Flow and Define Input Variables
Process Builder will pass in variables to the Flow for Campaign ID (aka Class Session) and Account ID (aka Chapter). You will create these variables, and variables needed to pass data between elements in your Flow, in the Resources tab. After you create them, you can view and edit them in the Explorer tab.

Click New Flow and select the Resources tab.

![New Resource](2.2 - resources tab.png)

In the Resources tab, double-click Variable and fill in the information for the Campaign ID variable. Do it again for the Account ID variable. Remember to set the Input/Output Type to Input Only. This will expose the variable in the Process Builder.

![Campaign Variable](2.3 - campaign variable.png)

![Account Variable](2.4 - account variable.png)

### Define Other Variables
Within the Flow you need variables to pass data between the elements. You'll use SObject Variables and SObject Collection Variables because they automatically know the fields in the object you select and will change if you add or remove fields.

Double-click SObject Collection Variable and fill in the information for the Contact collection variable. This is where you will store the Teaching Assistant records retrieved from the Contact object. Input/Output type should be Private.

![Contact Collection Variable](2.5 - contact collection variable.png)

Double-click SObject Variable (not SObject Collection Variable, because for this part we only need to store a single record) and fill in the information for the Contact single record variable. This is used as temporary storage when looping through the Contact collection variable. Input/Output type should be Private.

![Contact Variable](2.6 - contact variable.png)

Double-click SObject Variable and fill in the information for a Campaign single record variable. This is used as temporary storage when assigning values for the new Campaign Members. Input/Output type should be Private.

![Campaign Member Variable](2.7 - campaign member variable.png)

Double-click SObject Collection Variable and fill in the information for the Campaign Member collection variable. This is used to create new Campaign Member records. Input/Output type should be Private.

![Campaign Member Collection Variable](2.8 - campaign member collection variable.png)

### Define Fast Lookup
With the Account ID variable passed in from the Process Builder process, you can select all of Contacts for that Account with a Fast Lookup on the Contact object. In real life, there might be other criteria in this query (like a "volunteered to be a teaching assistant" checkbox), but that's for another day.

Select the Palette tab. Drag the Fast Lookup element onto the canvas. In General Settings, set the Name to 'Find Chapter TAs'. The Unique Name will default to 'Find_Chapter_TAs'. Then fill in the Filters and Assignments information.

![Find Chapter TAs](2.9 - find chapter tas2.png)

### Define Loop and Assignment
Now you need to iterate through the ChapterTAContacts collection to map the values to the TACampaignMembers collection.

From the Pallete tab, drag the Loop element onto the canvas. Fill in the information. 

![Loop Contacts](2.10 - loop contacts.png)

From the Pallete tab, drag the Assignment element onto the canvas. Fill in the information. This is where you use the Campaign ID we've passed from the Process Builder process.

![Assignment](2.11 - assign contacts to campaign members.png)

### Define Fast Create
Now you have what you need to create the new Campaign Member records for the potential Teaching Assistants! You do that by provide the Fast Create element with the SObject Collection variable you've populated. Boom, that's it!

Select the Palette tab. Drag the Fast Create element onto the canvas. Fill in the information.

![Fast Create Campaign Members](2.12 - fast create campaign members.png)

### Tell the Flow Engine What to Do
Right now your four Flow elements are disconnected and the Flow engine doesn't know where to start. To connect the elements, click the diamond at the bottom of an element and drag to another element to draw a connector line. 

Connect these elements, in this order:
1. Fast Lookup and Loop
2. Loop and Assignment
3. Loop and Fast Create 

Hover over the Fast Loopkup element and click the little green arrow that appears. That marks the Fast Lookup as the starting element.

You should now have a Flow that looks like this...

![Final Flow](2.13 - final flow.png)

### Save and Activate
Finally, you need to save the Flow and then Activate it before you can test.

Click Save and fill in the Flow Properties. Make sure you set the Type to 'Autolaunched Flow'. This makes it available in the Process Builder.

![Save Flow](2.14 - save flow.png)

Click Close. Now you should be on the Flow Detail page. Your Flow is listed in the Flow Versions list. Click the Activate link.

![Activate Flow](2.15 - activate flow.png)

### Add the Flow to your Process
We want the potential Teaching Assistants added as Campaign Members for all new Campaigns for Dev Classes, so we should add this to the process we built in Step 1 because that is firing for all new Campaigns of Type = Dev Class. Calling a Flow from a process in Process Builder gives us the extra power we need for this automation, while keeping it in one place to make it easy to maintain and to share with our team.

Fire up Process Builder and edit your existing process: New Class Sessions. Oh no, you can't! Flows cannot be modified once they have been activated. To modify a Flow, Clone it first. Save the clone as a version of the current process (this is the default).

Now you can add an action that calls your brand new Flow, and provide values for the two input variables you defined in the Flow.
* AccountIDFromPB | Reference | [Campaign.Chapter__c]
* CampaignIDFromPB | Reference | [Campaign.Id] 

![Set the Action](2.16 - call flow from pb.png)

Finally, Activate the process. Activating this cloned Flow will deactivate the Flow you created earlier. Only one version of a Flow can be active at a time.

### Test
Your functioning process should now be ready to test. Once again, enter data as you would as the Chapter Leader, but this time, keep an eye on the Campaign Member related list.

1. Enter a new Campaign record, setting Campaign Name = the name of a coding class; Campaign Type = Dev Class; Start Date = any date; Chapter = your sample account; Instructor = your sample instructor. Save.
2. Look at the Campaign Member list. Are there new Campaign Member records now? There should be, because that's what your Flow should be doing (assuming you created some Contacts for the same Account that you set the Chapter field to on the Campaign).

## 3 - Class Session Completion: Automating with Apex Invokable Methods
The final automation requires giving a Thanks badge to the Instructor. This is something you cannot do with just the Process Builder or Flow capabilities. This is where Apex invokable methods come in. You are going to write a small piece of Apex code that will be fired from a process you will define in the Process Builder.

###Apex Class
Copy the Apex code below to create the Apex Class that is capable of posting a Thanks badge to your awesome Instructor's profile page. The methods in this class will allow Badge Name, Giver ID, Receiver ID, and Thanks Message to be passed in from the process you define in Process Builder. 

```
global without sharing class GiveWorkThanksAction {

    @InvocableMethod(label='Give a Thanks Badge')
    global static void giveWorkBadgeActionsBatch(List<GiveWorkThanksRequest> requests) {
        for(GiveWorkThanksRequest request: requests){
            giveWorkBadgeAction(request);
        }
    }

    public static void giveWorkBadgeAction(GiveWorkThanksRequest request) {
        WorkThanks newWorkThanks = new WorkThanks();

                newWorkThanks.GiverId = request.giverId;
                newWorkThanks.Message = request.thanksMessage;
                newWorkThanks.OwnerId = request.giverId;

        insert newWorkThanks;


        WorkBadge newWorkBadge = new WorkBadge();

                // newWorkBadge.DefinitionId should be set to the ID for the Competitor Badge within this Org
                WorkBadgeDefinition workBadgeDef = [SELECT Id,Name FROM WorkBadgeDefinition WHERE Name = :request.badgeName Limit 1];

                newWorkBadge.DefinitionId = workBadgeDef.Id;
                newWorkBadge.RecipientId = request.receiverId;
                newWorkBadge.SourceId = newWorkThanks.Id ;
                //newWorkBadge.GiverId = request.giverId;

        insert newWorkBadge;

        WorkThanksShare newWorkThanksShare = new WorkThanksShare();

                newWorkThanksShare.ParentId = newWorkThanks.Id ;
                newWorkThanksShare.UserOrGroupId = request.receiverId;

                newWorkThanksShare.AccessLevel = 'Edit';
                insert newWorkThanksShare;

        FeedItem post = new FeedItem();

                post.ParentId = request.receiverId;
                post.CreatedById = request.giverId;
                post.Body = request.thanksMessage;
                post.RelatedRecordId = newWorkThanks.Id ;
                post.Type = 'RypplePost';

        insert post;

    }

    global class GiveWorkThanksRequest {
        @InvocableVariable(label='Giver Id' required=true)
        global Id giverId;

        @InvocableVariable(label='Receiver Id' required=true)
        global Id receiverId;

        @InvocableVariable(label='Thanks Message' required=true)
        global String thanksMessage;

        @InvocableVariable(label='Badge Name' required=true)
        global String badgeName;
    }
}
```
Notice the [@InvocableVariable](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableVariable.htm) and [@InvocableMethod](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableMethod.htm) annotations on this class that allow these methods to be exposed to the configuration tools in the Salesforce system, as well as to other authenticated clients via the REST API. 

### Build the Process
Now you can define the business process that will cause the Thanks automation to fire and post a badge to the Instructor. Here we get to see one of the really powerful features of Salesforce when you are writing code, the ability to bridge from clicks to code seamlessly! 

Fire up Process Builder and create this rule.

Click New and populate the details of your new Process.

![New Process](3.1 - completed class sessions.png)

Select the Campaign object and specify to start the process when a record is created or edited. Then add the selection criteria. i.e. When does the action need to fire? In our case, you want to thank your Instructors only when a class is complete and also only for Dev Class campaigns (not ALL campaigns!). 

![Process Criteria](3.2 - complete class session criteria.png)

Now you can add an action that calls your freshly minted Apex class, ready to accept the four parameters that you annotated with the @InnvocableVariable annotation. Note you are referencing the Badge by Name, which in this example happens to be 'Thanks', but you can chose another badge if you prefer.
* Badge Name | String | Thanks
* Giver ID | Reference | [Campaign.OwnerId] 
* Receiver ID | Reference | [Campaign.Instructor__c] 
* Thanks Message | String | You are awesome!

![Set the Action](3.3 - class apex method.png)

Finally, Activate the process.

### Test
Your functioning process should now be ready to test. Enter data as you would as the Chapter Leader, following these steps:

1. Edit an existing Campaign of Type = Dev Class. Set the Status to Completed. Save.
2. Go to your Chatter Feed. Is there a new post, thanking the Instructor? Does it have the Thanks badge?
 
![Thanks Badge](3.4 - thanks badge.png)

What else will you need to do before you can deploy this process in a production org?



