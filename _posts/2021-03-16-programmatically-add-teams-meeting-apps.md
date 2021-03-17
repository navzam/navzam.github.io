---
layout: post
title: "Programmatically adding Teams apps to meetings"
date: 2021-03-16
---

In addition to surfacing Teams apps within teams and chats, [you can now add apps to meetings](https://www.microsoft.com/en-us/microsoft-365/blog/2020/11/16/enhancing-your-microsoft-teams-experience-with-the-apps-you-need/) to enhance the meeting experience for participants. Meeting organizers can add apps to meeting using the Teams client, but there are cases where you may want to add these apps programmatically.

<!--more-->

For example, if you're programmatically creating meetings for a specific purpose, you may want to pre-load an app into those meetings so that organizers don't have to do it manually for every meeting. It can also help with discoverability since the app will automatically show up in the meeting experience.

[Microsoft Graph](https://developer.microsoft.com/en-us/graph) provides APIs for interacting with Teams apps, such as [adding an app to a team](https://docs.microsoft.com/en-us/graph/api/team-post-installedapps?view=graph-rest-1.0&tabs=http), but it isn't obvious how to add apps to meetings.

> TL;DR
>
> 1. Get the Teams app ID for [your app in the app catalog](https://docs.microsoft.com/en-us/graph/api/resources/teamsapp?view=graph-rest-1.0). Note that this is different than the app ID in your manifest.
> 1. Get the chat thread ID for the [online meeting](https://docs.microsoft.com/en-us/graph/api/resources/onlinemeeting?view=graph-rest-1.0) (see the `chatInfo` property for the `threadId`).
> 1. [Add your app to the Teams chat](https://docs.microsoft.com/en-us/graph/api/chat-post-installedapps?view=graph-rest-1.0) associated with the meeting.
> 1. [Add your tab to the Teams chat](https://docs.microsoft.com/en-us/graph/api/chat-post-tabs?view=graph-rest-1.0) associated with the meeting.

# Setup

You'll need a Teams meeting app added to your tenant app catalog. If you don't have one, check out the [docs for apps in Teams meetings](https://docs.microsoft.com/en-us/microsoftteams/platform/apps-in-teams-meetings/teams-apps-in-meetings) to get started.

We'll also be using several APIs from Microsoft Graph. For testing, the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer) is a great way to quickly try these APIs without having to register an AAD app. In practice, you'll need a [Graph access token](https://developer.microsoft.com/en-us/graph/graph-explorer) with these scopes:

- `AppCatalog.Read.All` (for finding your app in the app catalog)
- `OnlineMeetings.ReadWrite` (for creating a test meeting)
- `TeamsAppInstallation.ReadWriteSelfForChat` (for installing the app to the meeting chat)
- `TeamsTab.ReadWriteForChat` (for adding the app tab to the meeting chat)

If you're using Graph Explorer, you'll have to consent to the scopes above before using the APIs.

The last two scopes require admin consent. If you need to install your Teams app from a backend service without user context, you'll use [application permissions](https://docs.microsoft.com/en-us/graph/auth-v2-service?view=graph-rest-1.0), which require admin consent anyway. 

# Getting your Teams app ID

The first step is to get the Teams app ID. This is NOT the same as the app ID you specify in your app's `manifest.json`. Instead, it's an ID that's generated specifically for that instance of your app in the app catalog. In other words, if you remove your app from the app catalog and re-add it, this ID will change (even if the ID in your manifest is the same).

This generated ID is the `id` property of the [teamsApp resource in Graph](https://docs.microsoft.com/en-us/graph/api/resources/teamsapp?view=graph-rest-1.0). (There's also an `externalId` property, which is the ID from your app manifest.) You can use the [list teamsApp](https://docs.microsoft.com/en-us/graph/api/appcatalogs-list-teamsapps?view=graph-rest-1.0&tabs=http) API to list all apps in your app catalog:

```
GET /appCatalogs/teamsApps
```

which will return a list of Teams apps:

```json
{
  "value": [
    {
      "id": "b1c5353a-7aca-41b3-830f-27d5218fe0e5",
      "externalId": "f31b1263-ba99-435a-a679-911d24850d7c",
      "name": "Test App",
      "version": "1.0.1",
      "distributionMethod": "Organization"
    }
  ]
}
```

In this case, the ID you want is `b1c5353a-7aca-41b3-830f-27d5218fe0e5`.

If you have a lot of apps in your tenant, it might be easier to filter the list using the ID from your manifest (i.e. the `externalId`):

```
GET  https://graph.microsoft.com/v1.0/appCatalogs/teamsApps?$filter=externalId eq 'f31b1263-ba99-435a-a679-911d24850d7c'
```

# Creating an online meeting

We also need a Teams meeting for testing, which is represented by the [onlineMeeting resource in Graph](https://docs.microsoft.com/en-us/graph/api/resources/onlinemeeting?view=graph-rest-1.0). You can use the [create onlineMeeting](https://docs.microsoft.com/en-us/graph/api/application-post-onlinemeetings?view=graph-rest-1.0&tabs=http) API to create one (request body can be empty):

```
POST /me/onlineMeetings
```

The response will contain a few important properties to note down:
- `id`: The ID of the online meeting
- `joinWebUrl`: The URL you can use to join the meeting
- `chatInfo.threadId`: The ID of the chat thread associated with the meeting.

# Installing your app to the meeting

Now that we have a meeting, let's install the app to it. Really, we're going to install the app to the associated chat thread using the [add app to chat](https://docs.microsoft.com/en-us/graph/api/chat-post-installedapps?view=graph-rest-1.0) API:

```
POST /chats/{chat-id}/installedApps

{
    "teamsApp@odata.bind":"https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/{app-id}"
}
```

where `chat-id` is the thread ID from the online meeting above, and `app-id` is your app ID that we retrieved above. If all goes well, the response should be `201 Created`.

# Adding your tab to the meeting

With your app installed, you can finally add your tab to the meeting. Even if you're developing an app for the meeting side panel, it's defined as a "tab." Again, what we're really doing is adding the tab to the associated chat thread using the [add tab to chat](https://docs.microsoft.com/en-us/graph/api/chat-post-tabs?view=graph-rest-1.0) API:

```
POST /chats/{chat-id}/tabs

{
  "displayName": "My Contoso Tab",
  "teamsApp@odata.bind": "https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/{app-id}",
  "configuration": {
    "entityId": "some_entity_id",
    "contentUrl": "https://www.contoso.com/MeetingApp"
  }
}
```

with the same `chat-id` and `app-id` as above. The `contentUrl` should point to the content you want to show in your meeting tab or side panel. If all goes well, the response should be `201 Created`.

# Testing it out

Your app should now be added to the meeting! To try it out, use the join URL for the meeting to join the meeting. If your app is enabled for the meeting side panel, you should see your app icon at the top that you can click to open the side panel.