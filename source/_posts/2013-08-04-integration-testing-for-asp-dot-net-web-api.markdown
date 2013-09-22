---
layout: post
title: "Integration Testing ASP.NET Web API"
date: 2013-08-04 13:23
comments: true
categories:
  - Code
  - ASP.NET Web Api
  - Testing
  - REST
tags:
  - Code
  - REST
  - ASP.NET Web Api
image: images/featured/intergration-testing-apis.png
summary: Want to know what kind of HTTP APIs suck? The ones that break all of their clients every time a new version is minted. One way to keep ourselves in check is to sling up an integration test suite. In this post I cover an approach I have used to integration test my ASP.NET Web APIs both in memory, and then use that same suite to test the API once its deployed to a test environment.

---
Want to know what kind of HTTP APIs suck? The ones that break all of their clients every time a new version is minted. 
Everybody shipping an HTTP API, shoud have one question at the top of their mind.

>Is this change going to break my existing clients? How can I keep breaking changes to a minimum?
 
One way to be sure that you arent going to build one of *those* APIs, is to set up a series of automated integration tests. In this post I will show you an approach I found helpful to integration testing HTTP APIs built on top of ASP.NET Web API.


## Test With Zero HTTP Traffic

The utopia of an API integration test suite, would be a suite which could be executed quickly, and in memory, on our dev machines. Without *any* HTTP traffic. This would allow us to more quickly become aware of breaking changes at the boundary of our API. Wouldnt it also be great if we could then take those *same* tests, and run them against our test deployment?

To do this, we need to design our API in a way that supports a host anywhere — test anywhere mentality. Our application must first be designed in a way that can be started in both the IIS hosted environmnt, and the self hosted environment. 

Encapsulating the root of our API application is an important first step. When either the ASP.NET web host, or the self hosted API kicks off, it is able to call into our API application and call *Start()*. At this point all the configuration required for API can be carried out. 

{% codeblock API Application Interface lang:c# %}

public interface IApiApplication
{
	void Start();
}

{% endcodeblock %}

When our hosting environment starts, we are able to carry out all the configuration required for our API application to run. We can attach all of our custom message handlers, add any custom media type formatters, and configure our routes. What this achieves is a way for us to bootstrap our API, independent of the host infrastructure.

{% codeblock Api Application Implementation lang:c# %}

public class MyApiApplication : IApiApplication
{
    private readonly HttpConfiguration _configuration;

    public MyApiApplication(HttpConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void Start()
    {
        // composition root activites
        // set up DI container
        // set seralisation settings on configuratin object
        // set up routes
        _configuration.Routes.MapHttpRoute(
            name: "API Default",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
    }
}

{% endcodeblock %}

Now we can take our IApiApplication, and place it in a different host. Each host is able to call *Start()*, and magically our API is bootstrapped in the way we expect.

## Testing the API application

This is great news for our integration test suite. We can now take our API, which we may have previously hosted inside IIS, and host it in memory — inside a test session.

Start out by creating abstractions in your API server

{% codeblock API Server Interface lang:c# %}

public interface IApiServer
{
    HttpMessageHandler ServerHandler { get; }
    void Start();
    void Stop();
}

{% endcodeblock %}

We can now create a version of the API server, which is responsible for starting the API applciation, and hosting it inside our unit tests. 

{% codeblock An in memory API server to use in our tests lang:c# %}

public class InMemoryApiServer : IApiServer
{
    private HttpServer _server;

    public Uri BaseAddress { get { return new Uri("http://localhost"); }}

    public ApiServerHost Kind
    {
        get { return ApiServerHost.InMemory; }
    }

    public HttpMessageHandler ServerHandler { get { return _server; } }

    public void Start()
    {
        try
        {
            var httpConfig = new HttpConfiguration();
            var apiConfig = new ApiServiceConfiguration(httpConfig);
            apiConfig.Configure();
            _server = new HttpServer(httpConfig);
        }
        catch (Exception e)
        {
            Console.WriteLine("Could not create server: {0}", e);
            Assert.Fail("Could not create server: {0}", e);
        }
    }

    public void Stop()
    {
        try
        {
            _server.Dispose();
        }
        catch (Exception e)
        {
            Console.WriteLine("Could not stop server: {0}", e);
        }
    }
}

{% endcodeblock %}

From our test project now we can write test fixtures which start the in memory server during the test fixture setup phase, and tear it down when they are done. 

The IApiServer exposes a ServerHandler. We are going to use this handler to take advantage of a neat trick 
exposed by the HttpClinet, which will allow us to pass the *server* code directly to the client. In this way *ZERO* HTTP traffic will be generated. 

{% codeblock API Server Interface lang:c# %}

public interface IApiServer
{
    HttpMessageHandler ServerHandler { get; }
    // other bits
}

// in our test

using (var client = new HttpClient(_server.ServerHandler))
{
    //do stuff
}

{% endcodeblock %}


{% codeblock NUnit Intergration Test of our API in memory lang:c# %}

[TestFixture]
public abstract class BookApiTests
{
    private readonly IApiServer _server;

    private const string BooksRelativeUri = "api/books/1";

    protected BookApiTests(IApiServer apiServer)
    {
        _server = apiServer;
    }

    [SetUp]
    public void Setup()
    {
        _server.Start();
    }

    [TearDown]
    public void TearDown()
    {
        _server.Stop();
    }

    [Test]
    public void GetOneBookReturnsSuccessfulStatusCode()
    {
        var valuesUri = new Uri(_server.BaseAddress, BooksRelativeUri);
        using (var client = new HttpClient(_server.ServerHandler))
        {
            HttpResponseMessage httpResponseMessage = client.GetAsync(valuesUri).Result;
            Assert.That(httpResponseMessage.IsSuccessStatusCode);
            Assert.That(httpResponseMessage.StatusCode, Is.EqualTo(HttpStatusCode.OK));
        }
    }
}

{% endcodeblock %}

{% img center /images/posts/intergration-testing-webapi/simple-test-results.png%}

##It gets better!

Is there a way we can take the integration test suite, and execute it against a real server running our api? Perhaps in a test environment. Absolutely. In fact, I would encourage you to do this, and luckily, our test suite supports this.

{% codeblock Level Up Testing lang:c# %}

[TestFixture]
public class InMemoryBooksApiTests : BooksApiTests
{
    public InMemoryBooksApiTests() : base(new InMemoryApiServer())
    {
    }
}

[TestFixture]
public class AgainstServerBooksApiTests : BooksApiTests
{
    public AgainstServerBooksApiTests(): base(new AspNetApiServer(ApiHost.URI))
    {
    }
}

public abstract class BookApiTests
{
    private readonly IApiServer _server;

    private const string BooksRelativeUri = "api/books/1";

    protected BookApiTests(IApiServer apiServer)
    {
        _server = apiServer;
    }

    [SetUp]
    public void Setup()
    {
        _server.Start();
    }

    [TearDown]
    public void TearDown()
    {
        _server.Stop();
    }

    [Test]
    public void GetOneBookReturnsSuccessfulStatusCode()
    {
        var valuesUri = new Uri(_server.BaseAddress, BooksRelativeUri);
        using (var client = new HttpClient(_server.ServerHandler))
        {
            HttpResponseMessage httpResponseMessage = client.GetAsync(valuesUri).Result;
            Assert.That(httpResponseMessage.IsSuccessStatusCode);
            Assert.That(httpResponseMessage.StatusCode, Is.EqualTo(HttpStatusCode.OK));
        }
    }
}

{% endcodeblock %}
{% img right /images/posts/intergration-testing-webapi/testingapi.png%}

Above, I have turned our unit test into abstract class, and created two test classes which inherit from it. One executes the tests against the InMemoryApiServer, and the other, against the Test Server. Allowing the same test suite to be quickly executed in memory, *and* against the test server when we are ready to.

You may not like using abstract classes in unit tests in this way. Great! Feel free to take these ideas and mould them into your own approach. The thing that I personally have found helpful in this approach has been, it has allowed me to build a comprehensive suite of tests, which I can run before I check in. Equally on the build server, I can make up the tests which execute against the test environment, with an nUnit category (or similar in your test framework of choice) and run those tests as part of a longer running build.

I hope you were able to find something in this approach that may help you do the same.

*As an aside many parts of ASP.NET Web API are built in a way that supports *unit testing*. This blog post focuses on a different kind of testing — integration testing. I would encourage you to unit test your components in the first instance.*

Checkout the tests in this sample project as an example [Hyper Library](https://github.com/ammeep/hyper-library/)

