---
layout: post
title: Hangfire & WCF!
---

## Integrating Hangfire with a WCF application
Often times, there could be scenarios where transient errors can affect reliability of applications - especially if your app or service lives in integration space. The third party endpoint could be temporarily unavailable due to outage, unplanned downtime or even a network partition.  
So, how do we build applications which integrate with various third party endpoints, perform complex orchestrations and do all this in a reliable way?  
After some googling, I found that Polly and Hangfire as the top frameworks fit for my use case.  
Polly is a nice library to handle transient errors and provides a good programming model for configuring retries,  however it does not use persistence. Hangfire on the other hand uses SQL /MSMQ or Redis storage. This makes hangfire retries highly reliable (or guaranteed if you will) - application crash or server reboot, hangfire has you covered!

We decided to use hangfire.

This post aims to describe how to integrate hangfire with a WCF application. It addresses Dependency Injection, using custom JobActivator as well as writing a custom AuthorizationFilter for hangfire dashboard.

### Existing WCF App Overview

Existing application is a WCF service which receives policy/claim information and  document path in request message.
The service then does the following:

1. One or more orchestrations to retrieve and augment document metadata (add more policy/claim details)
2. Read all file content and create a HTTP multipart form-data request
3. Invoke third party endpoint with multipart form-data request
4. Delete all files from file-share if multipart request was successful.

### Hangfire Overview

Hangfire is an open source framework to perform fire-and-forget jobs, recurring jobs or delayed jobs. [Hangfire Docs](https://docs.hangfire.io/en/latest/) are excellent to understand key concepts about this framework.

I am just going to describe how the fire and forget paradigm comes in handy for our use case.

#### Implementation

Following are the high-level tasks to integrate Hangfire with existing application:

1. Add hangfire NuGet packages
2. Refactor existing code
3. Configure hangfire job storage, DI and job server
4. Add dashboard and configure [authN](https://acronyms.thefreedictionary.com/AUTHN) / [authZ](https://acronyms.thefreedictionary.com/AuthZ)

##### Add packages: Install the following packages

1. Hangfire.Core
2. Hangfire.SqlServer

##### Refactor existing code

To begin, consider the code below:

    public class MyWebService : IWebService
    {
        private readonly IFileUploader fileUploader;
        
        //ctor
        public MyWebService(IFileUploader fileUploader)
        {
            this.fileUploader = fileUploader;
        }
        
        //webmethod
        public UploadResponse Upload(UploadRequest request)
        {
            var result = fileUploader.UploadFiles(request)); // re-entrant function
            return result;
        }
    }

In the example above, our ‘fileUploader’ is a dependency which encapsulates entire document upload functionality. It is safe to consider ‘fileUploader.UploadFiles()’ as a function which is [re-entrant](https://en.wikipedia.org/wiki/Reentrancy_(computing)). This is kind of an entry point for all orchestrations which happen along the call stack. For my service, in case of any transient errors or failures, this was the point from where I needed to start over.

It is important to identify such class/method/function. **If your code does not have a re-entrant function, refactor to make it so**.

So we refactor MyWebService class and add BackgroundJob.Enqueue() method and the upload web method now looks like this:

    public UploadResponse Upload(UploadRequest request)
    { 
        var jobId = BackgroundJob.Enqueue(() => fileUploader.UploadFiles(request));
        return new UploadResponse(“Job Enqueued”, jobId);
    }

Above refactoring will work just fine. But, for unit testing, you would likely want to use the IBackgroundJobClient instead of the static BackgroundJob class. Using 'IBackgroundJobClient' does require us to register some hangfire dependencies with our DI container - we will cover this topic a bit later.

I used IBackgroundJobClient as a constructor injected dependency:

    public class MyWebService : IWebService
    {
        private readonly IBackgroundJobClient backgroundJobClient;
        private readonly IFileUploader fileUploader;
        
        //ctor
        public MyWebService(IFileUploader fileUploader)
        {
            this.fileUploader = fileUploader;
        }
        
        //webmethod
        public DocumentUploadResponse Upload(UploadRequest request)
        {
            var jobId =  backgroundJobClient.Create(() => fileUploader.UploadFiles(request), new EnqueuedState());
            return new UploadResponse(“Job Enqueued”, jobId);
        }
    }

##### Dependency Injection

By default, Hangfire expects a parameter-less constructor in classes which it needs to instantiate.  
Now, as I mentioned before, our WCF application uses DI, and our IFileUploader implementation has dependencies which are constructor injected.  
Hangfire instantiates objects using its JobActivator class which internally just invokes .Net provided Activator.CreateInstance(). This means that Hangfire will not be able to instantiate fileUploader without having an instance of its dependencies.  
We need to find a way to wire our DI container with hangfire’s JobActivator.  

So let’s configure the Dependency Injection for Hangfire. To do this, we write a custom JobActivator which will override the default activation behavior.  

    public class MyJobActivator : JobActivator
    {
        private readonly IUnityContainer container;

        //ctor
        public MyJobActivator(IUnityContainer container)
        {
            this.container=container;
        }

        public override object ActivateJob(Type jobType)
        {
            return container.Resolve(jobType);
        }
    }

And then add this job activator to the global configuration object as follows:  

    // DI configuration - registrations
    container.RegisterType<IFoo, Bar>();
    container.RegisterType<ISomeCustomType, CustomType>();

    // if using IBackgroundJobClient, register these hangfire types as follows

    container.RegisterType<JobStorage>(new InjectionFactory(c => JobStorage.Current));
    container.RegisterType<IJobFilterProvider, JobFilterAttributeFilterProvider>(new InjectionConstructor(true));
    container.RegisterType<IBackgroundJobFactory, BackgroundJobFactory>();
    container.RegisterType<IRecurringJobManager, RecurringJobManager>();
    container.RegisterType<IBackgroundJobClient, BackgroundJobClient>();
    container.RegisterType<IBackgroundJobStateChanger, BackgroundJobStateChanger>();

    // configure custom job activator
    GlobalConfiguration.Configuration.UseActivator(new MyJobActivator(container));

    // start hangfire server

**Important Note:**
When you use IBackgroundJobClient instead of the static 'BackgroundJob' class, you do have to wire up a all the types which hangfire needs to instantiate the 'BackgroundJobClient' class. Note that the types which must be registered are mentioned in the snippet above.

##### Configure Hangfire Job Storage and DI

These configuration should be done in entry point of application like Global.asax or Startup.cs or Custom ServiceHost factory in my case since its a WCF app:
Note that order is important here - so in our custom ServiceHostFactory, I configure following:

1. Job Storage
2. DI and job activator
3. Any other custom config hangfire confguration like job expiry filter etc.
4. Start hangfire server

Like

    // configure job storage
        GlobalConfiguration.Configuration.UseSqlServerStorage("HangFireDb", new SqlServerStorageOptions  {  QueuePollInterval = TimeSpan.FromSeconds(15) });

##### Job Server

Hangfire server is the component responsible for background job processing, and I decided to have a single instance of the job server per app domain. I also wanted to have graceful hangfire server shutdown.
This effectively means that I want a singleton for hangfire server.

To achieve a singleton, I used factory pattern and kept a static instance of background job server. I am also calling server.Dispose() in my factory destructor:

    public class BackgroundJobServerFactory
    {
        private static BackgroundJobServer server;
        private static bool isConfigured;
        public static void Start(IUnityContainer container)
        {
            if (isConfigured)
            {
                return;
            }     
            server = new BackgroundJobServer();
            isConfigured = true;
        }

        ~BackgroundJobServerFactory()
        {
            server.Dispose();
        }
    }

##### Hangfire Dashboard

Hangfire dashboard is an OWIN middleware, which can be either [self hosted](https://docs.microsoft.com/en-us/aspnet/web-api/overview/hosting-aspnet-web-api/use-owin-to-self-host-web-api) or you can use any other web container which understands OWIN. I hosted dashboard in IIS along with the WCF - i.e. just add a Startup.cs in your WCF service project.  
And, in our Startup.cs, we just add the following line of code:

    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            //1. configure job storage here explicitly
            //2. then configure dashboard
            app.UseHangfireDashboard();
        }
    }

##### Adding Authorization to Hangfire Dashboard

By default, hangfire dashboard allows only local requests. You can change this by passing your own implementation of authorization filter as shown [here](https://docs.hangfire.io/en/latest/configuration/using-dashboard.html#configuring-authorization).  

You can also secure the dashboard using azure active directory federation and cookie based authentication with OAuth/OpenId connect protocol. In this flow user will be redirected to the login page where one needs to enter their credentials. After entering credentials, users are redirected to Hangfire dashboard.

And that is it ! :) I hope you find this post useful !
