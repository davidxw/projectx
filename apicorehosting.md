# Hosting Options for .Net Core APIs

So, you are a startup with a big idea and you want to create some code to demonstrate it. Your team is exerpeinced wtih .Net Core, so that's your preferred development platform, and you are looking at Azure as a cloud provider. Initially though cost is going to be major factor so ideally you'd like some flexibiliy with your hosting platform - you think Azure PaaS services will be part of your long term plans but you aren't ready to become completely dependent on them just yet.  Given the amount of choice available to you, even with these constraints, where do you start when it comes to creating and hosting an API solution?  Below I've discussed a few different solutions, and while they have a strong Azure bias I've tried to highlight how alternative hosting option could be accomodated.

It also goes without saying that by demanding some flexibility in your future hosting arrangements you are going to have to give up some of the benefits of Azure PaaS services - an early decision to commit to Azure would make some things a lot more straightforward, and there are still methods you can use to allow you more latitude to migrate to an alternative platform if you choose to so down the track.

## API Options using the Azure Functions runtime

The first option is a full Azure PaaS option (at least in it's initial incarnation), and that is to build your API using Azure Functions, and host it on the Azure Functions service. The key advantage of this option (and why I've called it out first) is that the Azure Functions service offers a purely consumption based plan - if your API is not being called, the hosting cost is zero. When cost is critical there are few other alternatives that can offer this type of scalabilty. 

There are a few drawbacks with a consumption plan however - you can't connect the function backend to a Virtual Network (which may or may not be an issue), but more critically your function may incur a start up cost as your app may scale to zero worker instances when idle. This is less of an issue for asychronous processing (e.g. a function that repsonds to mesasages on a queue), but it can be a real issue for APIs where a consistent response time is important.

(You also can't use a consumption plan for functions packaged in containers, but more on this a bit later)

The good news regarding the Azure Function service however is that the consumption plan isn't the only option. The premium functions plan avoids the cold start problem by having at least one worker always available to serve API requests, and has the full suite of network integration options.  You will pay to keep that single instance available however, and the scaling isn't as liner as a consumption plan (as your workload scales the functions controller will add additional instances to handle the load, and the cost of the premium plan is per instance)

A full comparison of all of the various function plans is [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-scale).

Another potential issue with Azure Functions (regardless of the hosting plan) is the way the function code needs to be structured.  Each API operation needs to be represented by a separate function (technically a function with a HTTP trigger), and this can get cumbersome if your API is a large one.  This can be mitigated somewhat via the use of Function proxies (which can for example expose the set of separate functions in the REST API for a single entity using the normal REST API conventions), and also by ensuring you have clean separation in your code between the API implementation (the functions code), and the API logic. Function Proxies add additional costs however.

An additional option to consider for Azure Functions hosting is containerisation. Microsoft makes available a container image that contains the Azure Function runtime, and building a container based on this image to host your functions unlocks additional options. As well as using the Azure Function service, your function containers could also be deployed to a Kubernetes cluster, which would provide a whole additional level of capability (and complexity!)  (When deployed to a Kubernetes cluster you will also need to use another technology called Keda to scale your container instances based on load, which has its own complexities (especially when setting up scalling for HTTP triggers)).

(Hypothetically you also host your Azure function containers on any container orchestrator or cloud service, but you will need to BYO scaling)

While there are issues to consider going down the Azure Functions route for an API it could be a good starter option, with plenty of headroom to scale in future. If you structure your code carefully you could also avoid lock in to the Azure Functions runtime, although migration would always require some level of refactoring.  The other benefit of using Azure Functions is that functions are great for asynchronous process (which we haven't talked about too much here), and there is something to be said for having a consistent approach.

A functions based API roadmap could look like this:

* Create your API using Azure Functions, and host in a consumption plan. Structure your code so that the functions are just passing requests through to your core business logic, and use Function Proxies to expose your functions as "proper" REST APIs.

* If (when) startup cost for your functions become an issue, upgrade to a premium functions plan. This will give you more consistent performance, and a range of other features.

* When your application becomes more complex and you want to move to a more powerfull platform (like Kubernetes), you can containerise your functions and deploy to AKS. This will work for both API (HTTP Trigger) functions, and asynchronous functions, but for both you will need to BYO scaling (either via the Kubernetes HPA or Keda).

* If at any stage you want to simplify your code and remove your dependency on the Azure Functions runtime and Function Proxies, you can migrate to an ASP.Net Core API.  You can still host this in an Azure App Service, or on AKS, or a range of other platforms (more on this below), but you'll no longer have access to the zero to X functions plan. Your asynchronous code can continue to use the Functions runtime

## API Options using ASP.Net Core

[API.Net Core](https://docs.microsoft.com/en-us/aspnet/core) is a powerful, flexible platform for creating web or API applications, with a large range of features and capabilities. Because it based on .Net Core it also runs on both Windows and Linux, which greatly increases the available hosting platorms for your API, both in Azure and elsewhere. Some of this options are described below.

### Azure App Service

Azure App Service is the go to Azure service for hosting web and API workloads, and would be a great fit for hosting your .Net Core API. Like Azure Functions there are a range of pricing tiers offering an increasing number of features, and App Service will scale to suit elastic workloads, but unlike Functions there is no pure consumption tier - there will always be a minium fixed cost for a single worker instance. As well as scaling out however it's also trivial to scale up and down between different performance tiers so you aren't locked in to a particular level of worker instance performance.

Azure App Service also supports deploying your code in a container, so if ultimate portabilty was your goal you could still create a containerised version of your app and host it on Azure App Service.

### Other Web Server Implementations

Outside of Azure an ASP.Net Core can be hosted natively on a range of other web services.  ASP.Core ships with an in-process HTTP server called Kestrel, which runs on both Windows and Linux, which can be used to either directly receive API requests, or, more commonly in production scenarios, paired with a reverse proxy server such as Nginx or Apache.  The ASP.Net Core documentation describes there scenarios [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers)

### Containers

A very popular way to package an ASP.Net Core API is in a Linux container, which then allows you a large amount of flexibility in how and where to host your application. Listing all the possible options for hosting a container based API is will out of the scope of this article, but we've already touched on some of the Azure specific ones - Azure App Service will support code packaged as a container, and when you are ready for more complexity Azure Kubernetes Services will be waiting for you.

## Other Considerations

* Using Azure Static websites to host your SPA files and Azure Functions to host an API is a quick way to get started with your web application and is documented by Microsoft [here](https://docs.microsoft.com/en-us/azure/static-web-apps/add-api)
* Mixing an Azure App Service Hosted API with asynchronous Azure Functions is a very common pattern, and allows you to build a complex solution that is flexible and scalable.
* Regardless of the platform you chose, you should instrument you application right from the very first version.  All of the Azure options presented here integrate seamlessly with Azure Application Insights.
* Similarily a CI/CD pipeline for continuous build and deployment is also something you should consider right from verison 1.0, and while Microsoft currently offers two similar services to meet this need (Azure DevOps Pipelines, and GitHub Actions), you should expect to see increasing investment in GitHub Actions so that's what I woudl use if I was starting from scratch (and it goes without saying you are using GitHub...)

