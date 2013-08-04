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
summary: Want to know what kind of HTTP APIs suck? The ones that break all of their clients ever time a new version is minted. One way to keep ourselves in check is to sling up an integration test suite. In this post I cover an approach I have used to integration test my ASP.NET Web APIs both in memory, and then use that same suite to test the API once its deployed to a test environment.

---
Want to know what kind of HTTP APIs suck? The ones that break all of their clients ever time a new version is minted. 

Everybody shipping an HTTP API, shoud have one question at the top of their mind.

>Is this change going to break my existing clients? How can I keep breaking changes to a minimum?
 
One way to be sure that you arent going to build one of *those* apis, is to set up a series of automated integration tests. In this post I will show you an approach I found helpful to integration testing HTTP APIs built on top of ASP.NET Web API.


## Host Anywhere, Test Anywhere

{% img right /images/posts/intergration-testing-webapi/movableapi.png%}

The utopia of an api integration test suite, would be a suite which could be executed quickly, and in memory, on our dev machines. This would allow us to more quickly become aware of breaking changes at the boundary of our API.

When starting a new ASP.NET Web API project, you have two choices for how to host it. It can be hosted in a self hosted environment, or it can be hosted inside IIS. If your requirements dictate that you will host your API inside IIS, you shouldn't have to wait for a deployment to the test environment before you can run the test suite. What we need is a way to host our API in memory, to allow us to quickly execute the suite before check in.

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
    void Start();
    void Stop();
}

{% endcodeblock %}

We can now create an in memory version of the API server, which is responsible for starting the API applciation, and hosting it inside the web API self host. 

{% codeblock An in memory API server to use in our tests lang:c# %}

public class InMemoryApiServer : IApiServer
{
    private readonly Uri _serverUri;
    private readonly IApiApplication _apiApplication;
    private readonly HttpSelfHostConfiguration _config;
    private HttpSelfHostServer _server;

    public InMemoryApiServer(Uri serverUri)
    {
        _serverUri = serverUri;
        _config = new HttpSelfHostConfiguration(serverUri);
        _apiApplication = new MyApiApplication(_config);
    }

	public void Start()
    {
        try
        {
            _server = new HttpSelfHostServer(_config);

            _apiApplication.Start();

            _server.OpenAsync().Wait();
            Console.WriteLine("Listening on " + _serverUri);
        }
        catch (Exception e)
        {
            Console.WriteLine("Could not start server: {0}", e);
        }
    }

    public void Stop()
    {
        try
        {
            _server.CloseAsync().Wait();
        }
        catch (Exception e)
        {
            Console.WriteLine("Could not stop server: {0}", e);
        }
    }
}


{% endcodeblock %}

From our test project now we can write test fixtures which start the in memory host during the test fixture setup phase, and tear it down when they are done. Once the in memory hosted API is up and running, we are free to use the Http client libraries to call into the API, executing gets, posts puts and deletes until your heart's content. We can assert that when a resource is not found, the appropriate status code is always returned. We can feel confident that when a new API route is added, all the old ones still work. These things will no longer keep you awake at night, now that you can run your integration tests in memory, before you commit your code.  

{% codeblock NUnit Intergration Test of our API in memory lang:c# %}

[TestFixture]
public class ValueApiTests
{
    private Uri _localUri;
    private IApiServer _server;

    [SetUp]
    public void Setup()
    {
        _localUri = new Uri("http://localhost:3098");
        _server = new InMemoryApiServer(_localUri);
        _server.Start();
    }

    [TearDown]
    public void TearDown()
    {
        _server.Stop();
    }

    [Test]
    public void CanSuccessfullyGetAllValues()
    {
        var valuesUri = new Uri(_server.Uri, "api/values");
        using (var client = new HttpClient())
        {
            HttpResponseMessage httpResponseMessage = client.GetAsync(valuesUri).Result;
            Assert.That(httpResponseMessage.IsSuccessStatusCode);

            IList<string> result = httpResponseMessage.Content.ReadAsAsync<IList<string>>().Result;
            Assert.That(result, Is.Not.Null);
        }
    }
}

{% endcodeblock %}

{% img center /images/posts/intergration-testing-webapi/simple-test-results.png%}

##It gets better!


There *are* some difference between an API hosted inside IIS, and an API hosted in a self hosted environment.  So, while running your integration tests in memory gives you great confidence you haven't introduced any breaking changes.There still is a chance some may sneak in, due to these differences.

Is there a way we can take the integration test suite, and execute it against a real server running our api? Perhaps in a test environment. Absolutely. In fact, I would encourage you to do this, and luckily, our test suite supports this.

{% codeblock Level Up Testing lang:c# %}

[TestFixture]
public class InMemoryValueApiTests : ValueApiTest
{
    private readonly static Uri LocalUri = new Uri("http://localhost:3098");

    public InMemoryValueApiTests() : base(new InMemoryApiServer(LocalUri))
    {
    }
}

[TestFixture]
public class TestEnvironmentServerApiTests : ValueApiTest
{
    private readonly static Uri LocalUri = new Uri("http://myapi.org");

    public TestEnvironmentServerApiTests() : base(new AspNetApiServer(LocalUri))
    {
    }
}

public abstract class ValueApiTest
{
    private readonly IApiServer _server;

    protected ValueApiTest(IApiServer apiServer)
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
    public void CanSuccessfullyGetAllValues()
    {
        var  valuesUri = new Uri(_server.Uri,"api/values");
        using (var client = new HttpClient())
        {
            HttpResponseMessage httpResponseMessage = client.GetAsync(valuesUri).Result;
            Assert.That(httpResponseMessage.IsSuccessStatusCode);

            IList<string> result = httpResponseMessage.Content.ReadAsAsync<IList<string>>().Result;
            Assert.That(result,Is.Not.Null);
        }
    }
}

{% endcodeblock %}
{% img right /images/posts/intergration-testing-webapi/testingapi.png%}

Above, I have turned our unit test into abstract class, and created two test classes which inherit from it. One executes the tests against the InMemoryApiServer, and the other, against the Test Server. Allowing the same test suite to be quickly executed in memory, *and* against the test server when we are ready to.

You may not like using abstract classes in unit tests in this way. Great! Feel free to take these ideas and mould them into your own approach. The thing that I personally have found helpful in this approach has been, it has allowed me to build a comprehensive suite of tests, which I can run before I check in. Equally on the build server, I can make up the tests which execute against the test environment, with an nUnit category (or similar in your test framework of choice) and run those tests as part of a longer running build.

I hope you were able to find something in this approach that may help you do the same.

*As an aside many parts of ASP.NET Web API are built in a way that supports *unit testing*. This blog post focuses on a different kind of testing — integration testing. I would encourage you to unit test your components in the first instance.*

