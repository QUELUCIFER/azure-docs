---
title: Stream live with Azure Media Services v3 | Microsoft Docs
description: This tutorial walks you through the steps of streaming live with Media Services v3 using .NET Core.
services: media-services
documentationcenter: ''
author: juliako
manager: femila
editor: ''

ms.service: media-services
ms.workload: media
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: tutorial
ms.custom: mvc
ms.date: 01/22/2019
ms.author: juliako

---

# Tutorial: Stream live with Media Services v3 using APIs

In Azure Media Services, [Live Events](https://docs.microsoft.com/rest/api/media/liveevents) are responsible for processing live streaming content. A Live Event provides an input endpoint (ingest URL) that you then provide to a live encoder. The Live Event receives live input streams from the live encoder and makes it available for streaming through one or more [Streaming Endpoints](https://docs.microsoft.com/rest/api/media/streamingendpoints). LiveEvents also provide a preview endpoint (preview URL) that you use to preview and validate your stream before further processing and delivery. This tutorial shows how to use .NET Core to create a **pass-through** type of a live event. 

> [!NOTE]
> Make sure to review [Live streaming with Media Services v3](live-streaming-overview.md) before proceeding. 

The tutorial shows you how to:    

> [!div class="checklist"]
> * Access the Media Services API
> * Configure the sample app
> * Examine the code that performs live streaming
> * Watch the event with [Azure Media Player](http://amp.azure.net/libs/amp/latest/docs/index.html) at http://ampdemo.azureedge.net
> * Clean up resources

[!INCLUDE [quickstarts-free-trial-note](../../../includes/quickstarts-free-trial-note.md)]

## Prerequisites

The following are required to complete the tutorial.

- Install Visual Studio Code or Visual Studio.
- Install and use the CLI locally, this article requires the Azure CLI version 2.0 or later. Run `az --version` to find the version you have. If you need to install or upgrade, see [Install the Azure CLI](/cli/azure/install-azure-cli). 

    Currently, not all [Media Services v3 CLI](https://aka.ms/ams-v3-cli-ref) commands work in the Azure Cloud Shell. It is recommended to use the CLI locally.

- [Create a Media Services account](create-account-cli-how-to.md).

    Make sure to remember the values that you used for the resource group name and Media Services account name

- A camera or a device (like laptop) that is used to broadcast an event.
- An on-premises live encoder that converts signals from the camera to streams that are sent to the Media Services live streaming service. The stream has to be in **RTMP** or **Smooth Streaming** format.

## Download the sample

Clone a GitHub repository that contains the streaming .NET sample to your machine using the following command:  

 ```bash
 git clone https://github.com/Azure-Samples/media-services-v3-dotnet-core-tutorials.git
 ```

The live streaming sample is located in the [Live](https://github.com/Azure-Samples/media-services-v3-dotnet-core-tutorials/tree/master/NETCore/Live/MediaV3LiveApp) folder.

> [!IMPORTANT]
> This sample uses unique suffix for each resource. If you cancel the debugging or terminate the app without running it through, you will end up with multiple LiveEvents in your account. <br/>
> Make sure to stop the running Live Events. Otherwise, you will be **billed**!

[!INCLUDE [media-services-v3-cli-access-api-include](../../../includes/media-services-v3-cli-access-api-include.md)]

## Examine the code that performs live streaming

This section examines functions defined in the [Program.cs](https://github.com/Azure-Samples/media-services-v3-dotnet-core-tutorials/blob/master/NETCore/Live/MediaV3LiveApp/Program.cs) file of the *MediaV3LiveApp* project.

The sample creates a unique suffix for each resource so that we don't have name collisions if you run the sample multiple times without cleaning up.

> [!IMPORTANT]
> This sample uses unique suffix for each resource. If you cancel the debugging or terminate the app without running it through, you will end up with multiple Live Events in your account. <br/>
> Make sure to stop the running Live Events. Otherwise, you will be **billed**!
 
### Start using Media Services APIs with .NET SDK

To start using Media Services APIs with .NET, you need to create an **AzureMediaServicesClient** object. To create the object, you need to supply credentials needed for the client to connect to Azure using Azure AD. In the code you cloned at the beginning of the article, the **GetCredentialsAsync** function creates the ServiceClientCredentials object based on the credentials supplied in local configuration file. 

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CreateMediaServicesClient)]

### Create a live event

This section shows how to create a **pass-through** type of Live Event (LiveEventEncodingType set to None). If you want to create a Live Event that is enabled for live encoding set LiveEventEncodingType to **Standard**. 

Some other things that you might want to specify when creating the live event are:

* Media Services location 
* The streaming protocol for the Live Event (currently, the RTMP and Smooth Streaming protocols are supported).<br/>You cannot change the protocol option while the Live Event or its associated Live Outputs are running. If you require different protocols, you should create separate Live Event for each streaming protocol.  
* IP restrictions on the ingest and preview. You can define the IP addresses that are allowed to ingest a video to this Live Event. Allowed IP addresses can be specified as either a single IP address (for example '10.0.0.1'), an IP range using an IP address and a CIDR subnet mask (for example, '10.0.0.1/22'), or an IP range using an IP address and a dotted decimal subnet mask (for example, '10.0.0.1(255.255.252.0)').<br/>If no IP addresses are specified and there is no rule definition, then no IP address will be allowed. To allow any IP address, create a rule and set 0.0.0.0/0.<br/>The IP addresses have to be in one of the following formats: IpV4 address with 4 numbers, CIDR address range.
* When creating the event, you can specify to auto start it. <br/>When autostart is set to true, the Live Event will be started after creation. That means, the billing starts as soon as the Live Event startsrunning. You must explicitly call Stop on the Live Event resource to halt further billing. For more information, see [Live Event states and billing](live-event-states-billing.md).

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CreateLiveEvent)]

### Get ingest URLs

Once the Live Event is created, you can get ingest URLs that you will provide to the live encoder. The encoder uses these URLs to input a live stream.

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#GetIngestURL)]

### Get the preview URL

Use the previewEndpoint to preview and verify that the input from the encoder is actually being received.

> [!IMPORTANT]
> Make sure that the video is flowing to the Preview URL before continuing!

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#GetPreviewURLs)]

### Create and manage Live Events and Live Outputs

Once you have the stream flowing into the Live Event, you can begin the streaming event by creating an Asset, Live Output, and Streaming Locator. This will archive the stream and make it available to viewers through the Streaming Endpoint. 

#### Create an Asset

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CreateAsset)]

Create an Asset for the Live Output to use.

#### Create a Live Output

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CreateLiveOutput)]

#### Create a Streaming Locator

> [!NOTE]
> When your Media Services account is created a **default** streaming endpoint is added to your account in the **Stopped** state. To start streaming your content and take advantage of dynamic packaging and dynamic encryption, the streaming endpoint from which you want to stream content has to be in the **Running** state. 

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CreateStreamingLocator)]

```csharp

// Get the url to stream the output
ListPathsResponse paths = await client.StreamingLocators.ListPathsAsync(resourceGroupName, accountName, locatorName);

foreach (StreamingPath path in paths.StreamingPaths)
{
    UriBuilder uriBuilder = new UriBuilder();
    uriBuilder.Scheme = "https";
    uriBuilder.Host = streamingEndpoint.HostName;

    uriBuilder.Path = path.Paths[0];
    // Get the URL from the uriBuilder: uriBuilder.ToString()
}
```

### Cleaning up resources in your Media Services account

If you are done streaming events and want to clean up the resources provisioned earlier, follow the following procedure.

* Stop pushing the stream from the encoder.
* Stop the Live Event. Once the Live Event is stopped, it will not incur any charges. When you need to start it again, it will have the same ingest URL so you won't need to reconfigure your encoder.
* You can stop your Streaming Endpoint, unless you want to continue to provide the archive of your live event as an on-demand stream. If the Live Event is in stopped state, it will not incur any charges.

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CleanupLiveEventAndOutput)]

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CleanupLocatorAssetAndStreamingEndpoint)]

[!code-csharp[Main](../../../media-services-v3-dotnet-core-tutorials/NETCore/Live/MediaV3LiveApp/Program.cs#CleanupAccount)]   

## Watch the event

To watch the event, copy the streaming URL that you got when you ran code described in [Create a Streaming Locator](#create-a-streaminglocator) and use a player of your choice. You can use [Azure Media Player](http://amp.azure.net/libs/amp/latest/docs/index.html) to test your stream at http://ampdemo.azureedge.net. 

Live event automatically converts events to on-demand content when stopped. Even after you stop and delete the event, the users would be able to stream your archived content as a video on demand, for as long as you do not delete the asset. An asset cannot be deleted if it is used by an event; the event must be deleted first. 

## Clean up resources

If you no longer need any of the resources in your resource group, including the Media Services and storage accounts you created for this tutorial, delete the resource group you created earlier.

Execute the following CLI command:

```azurecli-interactive
az group delete --name amsResourceGroup
```

> [!IMPORTANT]
> Leaving the Live Event running incurs billing costs. Be aware, if the project/program crashes or is closed out for any reason, it could leave the Live Event running in a billing state.

## Next steps

[Stream files](stream-files-tutorial-with-api.md)
 
