# DDD Backend

[![Build and Deploy](https://github.com/dddadelaide/ddd-backend/actions/workflows/build-and-deploy.yml/badge.svg)](https://github.com/dddadelaide/ddd-backend/actions/workflows/build-and-deploy.yml)

This project contains backend functionality to run the DDD conferences, including:

* Syncing data from [Sessionize](https://sessionize.com/) to Azure Table Storage (tenanted by conference year) for submitted sessions (and submitters) and separate to that, selected sessions (and presenters)
* APIs that return submission and session (agenda) information during allowed times
* APIs to facilitate voting by the community against (optionally anonymous) submitted sessions (notes stored to Azure Table Storage tenanted by conference year) including various mechanisms to detect fraud
* Syncing Tito order IDs and Azure App Insights voting user IDs to assist with voting fraud detection and validation
* API to return analysed voting information
* Ability to trigger an Azure Logic App when a new session is detected from Sessionize (which can then be used to create Microsoft Teams / Slack notifications for visibility to the organising committee and/or to trigger moderation actions)
* Tito webhook to take order notifications, de-duplicate them and place them in queue storage so they can be picked up by a Logic App (or similar) to do things like create Microsoft Teams / Slack notifications for visibility to the organising committee
* Getting feedback information and prize draw names

## Run locally

### Prerequisites

* VSCode (<https://code.visualstudio.com/>) or your preferred IDE
* Dotnet Core 3.1 (<https://dotnet.microsoft.com/en-us/download/dotnet/3.1>)
* Azure Functions Core Tools(<https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local>)~~~~

### Commands

* `cd DDD.Functions && func host start --build --debug --verbose`

## Structure

* `DDD.Core`: Cross-cutting logic and core domain model
* `DDD.Functions`: Azure Functions project that contains:
  * `AppInsightsSync`: C# Azure Function that syncs app insights user IDs to Azure Table Storage for users that submitted a vote
  * `TitoNotification`: Node.js Azure Function that exposes a web hook URL that can be added to Tito (via Account Settings > Webhooks) for the `order.placed` action that will then de-duplicate webhook events and push them to queue storage for further processing (e.g. via a Logic App)
  * `TitoSync`: C# Azure Function that syncs Tito order IDs to Azure Table Storage for a configured event
  * `GetAgenda`: C# Azure Function that returns sessions and presenters that have been approved for agenda
  * `GetAgendaSchedule`: C# Azure Function that returns agenda schedule from sessionize
  * `GetSubmissions`: C# Azure Function that returns submissions and submitters for use with either voting or showing submitted sessions
  * `GetVotes`: C# Azure Function that returns analysed vote information; can be piped into Microsoft Power BI or similar for further processing and visualisation
  * `NewSessionNotification`: C# Azure Function that responds to new submissions in Azure Table Storage and then calls a Logic App Web Hook URL (from config) with the session and presenter information (marking that session as notified to avoid duplicate notifications)
  * `SessionizeReadModelSync`: C# Azure Function triggered by a cron schedule defined in config that performs a sync from Sessionize to Azure Table Storage for submissions
  * `SessionizeAgendaSync`: C# Azure Function triggered by a cron schedule defined in config that performs a sync from Sessionize to Azure Table Storage for approved sessions
  * `SubmitVote`: : C# Azure Function that allows a vote for submissions to be submitted, where it is validated and persisted to Azure Table Storage
* `DDD.Sessionize`: Syncing logic to sync data from sessionize to Azure Table Storage
* `DDD.Sessionize.Tests`: Unit tests for the Sessionize Syncing code
* `infrastructure`: Azure ARM deployment scripts to provision the backend environment
  * `Deploy-Local.ps1`: Run locally to debug or develop the scripts using your user context in Azure
  * `Deploy.ps1`: Main deployment script that you need to call from CD pipeline
  * `azuredeploy.json`: Azure ARM template
* `.vsts-ci.yml`: VSTS Continuous Integration definition for this project

## Backend date parameters and usage

* `StopSyncingSessionsFrom`: this is when we should stop syncing sessions from Sessionize, usually CFP close date
* `StopSyncingAgendaFrom`: this is when we should stop syncing agenda from Sessionize, usually conference date
* `StopSyncingTitoFrom`: this is when we should stop syncing tickets holder information from Tito usually the date before conference date
* `VotingAvailableFrom`: voting start date
* `VotingAvailableTo`: voting end date
* `SubmissionsAvailableFrom`: this is when we can retrieve submitted submissions for internal usage and voting, usually it is the when voting opens
* `SubmissionsAvailableTo`: this is when we cannot retrieve submitted submissions, usually it is when Agenda is published
* `StartSyncingAppInsightsFrom`: this is when we start collecting insights for voting and submissions, usually when CFP opens
* `StopSyncingAppInsightsFrom`: this is when we stop collecting insights for voting and submissions, usually when voting closes
* `FeedbackAvailableFrom`: this is when we start accepting feedback, usually the conference start date at 8:00am
* `FeedbackAvailableTo`: this is when we stop accepting feedback, usually the conference start date at 5:00pm

## Infrastructure Prerequisites

### Initial Infrastructure

Before deploying the application, a small amount of Azure infrastructure needs to be created. Create a resource group containing:

* A storage account and container to host the build artifacts
* An app service plan (Needs to be 'Basic' or above to run the functions app)
* An EntraID app registration with three associated federated credentials (substitute {org/repo} for your org/repo):
  * repo:{org/repo}:ref:refs/heads/master
  * repo:{org/repo}:pull_request
  * repo:{org/repo}:environment:test
* A role assignemnt of `Contributor` to your subscription (or associated granular role) for your app registration's service principal

### AppInsights Voting Behaviour

The backend application depends on programmatic access to the [Frontend Website's](https://github.com/dddwa/dddperth-website) Application Insights to pull and store information on voting behavior.

To supply this access, create an API key with `Read telemetry` permissions within the frontend website's Application Insights instance in the Azure Portal, and enter the Application ID and Key presented into the `AppInsightsApplicationId` and `AppInsightsApplicationKey` parameters.

## Setting up Continuous Delivery in Github Actions

The build and deploy workflow can be found in `./github/workflows/build-and-deploy.yml`.

For this to run, you'll need to:

* [Define variables and secrets in your GitHub project](https://docs.github.com/en/actions/learn-github-actions/variables#defining-configuration-variables-for-multiple-workflows) to match the variables required by `./github/workflows/build-and-deploy.yml`. `Ctrl+f vars.` and `Ctrl + f secrets.` your way to success to discover the necessary variables and provide appropriate values.

Hot tip: your app service plan needs to be in the same azure region as your azure web app, or the deployment will fail with a less-than-helpful error message.

## New Session Notification Logic App

The `NewSessionNotificationLogicAppUrl` value is gotten by creating a logic app and copying the webhook URL from it. The logic app would roughly have:

* `When a HTTP request is received` trigger with json schema of:

    ```json
    {
        "properties": {
            "Presenters": {
                "items": {
                    "properties": {
                        "Bio": {
                            "type": "string"
                        },
                        "ExternalId": {
                            "type": "string"
                        },
                        "Id": {
                            "type": "string"
                        },
                        "Name": {
                            "type": "string"
                        },
                        "ProfilePhotoUrl": {
                            "type": "string"
                        },
                        "Tagline": {
                            "type": "string"
                        },
                        "TwitterHandle": {
                            "type": "string"
                        },
                        "WebsiteUrl": {
                            "type": "string"
                        }
                    },
                    "required": [
                        "Id",
                        "ExternalId",
                        "Name",
                        "Tagline",
                        "Bio",
                        "ProfilePhotoUrl",
                        "WebsiteUrl",
                        "TwitterHandle"
                    ],
                    "type": "object"
                },
                "type": "array"
            },
            "Session": {
                "properties": {
                    "Abstract": {
                        "type": "string"
                    },
                    "CreatedDate": {
                        "type": "string"
                    },
                    "ExternalId": {
                        "type": "string"
                    },
                    "Format": {
                        "type": "number"
                    },
                    "Id": {
                        "type": "string"
                    },
                    "Level": {},
                    "MobilePhoneContact": {},
                    "PresenterIds": {
                        "items": {
                            "type": "string"
                        },
                        "type": "array"
                    },
                    "Tags": {
                        "type": "array"
                    },
                    "Title": {
                        "type": "string"
                    }
                },
                "type": "object"
            }
        },
        "type": "object"
    }
    ```

* `For each` action against `Presenters` with a nested `Compose` action against `Name`
* `Post message` action (for Teams/Slack) with something like `@{join(actionOutputs('Compose'), ', ')} submitted a talk '@{triggerBody()?['Session']['Title']}' as @{triggerBody()?['Session']['Format']} / @{triggerBody()?['Session']['Level']} with tags @{join(triggerBody()?['Session']['Tags'], ', ')}.`
* `Send an email` action (for O365/GMail/Outlook.com depending on what you have) that sends an email if the previous step failed (via `Configure run after`)

## Tito notification logic app

The logic app would roughly have:

* `When there are messages in a queue` trigger to the `attendees` queue of the `{conferencename}functions{environment}` storage account
* `Post message` action (for Teams/Slack) with something like `@{json(trigger().outputs.body.MessageText).name} is attending @{json(trigger().outputs.body.MessageText).event} as @{json(trigger().outputs.body.MessageText).ticketClass} (orderid: @{json(trigger().outputs.body.MessageText).orderId}). @{json(trigger().outputs.body.MessageText).qtySold}/@{json(trigger().outputs.body.MessageText).totalQty} @{json(trigger().outputs.body.MessageText).ticketClass} tickets taken.`
* `Delete message` action for the `attendees` queue with the Message ID and Pop Receipt from the trigger
