---
title: "Httpclient testing: The right way"
tags: 
- dotnet
- testing
- c#
- httpclient
---

# HttpClient Testing: The Right Way

For several years I've managed to test HttpClient using a wrapper class to can control what it return.
It worked and still my first go to option.

Recently I've developed a small application that just call one endpoint with one method and creating a wrapper was overkill.

As my habit I've started to search if **"There is another way?"**

Searching around I've found there are 3 main ways to Test and Mock HttpClient.

- [Flurl](https://flurl.dev)
- [MockHttp](https://github.com/richardszalay/mockhttp)
- [Moq](https://github.com/devlooped/moq)

## Some Considerations

Sometimes, having Interfaces for everything, or "just for testing it" like what happens with HttpClient is not so good but consider the following:

**hiding** of source of the informations behind a wrapper, like for example `IUserService` can be helpful and the real implementation of how and where this information relies on the class implementation.

For example the method LoginUser should return "OK" or "KO"  for the status of the login, the caller should not be aware if there is adatabase or and HTTP server behind it.
In this case for example my suggestion is wrap the HttpClient inside the **UserServiceImplementation** class. You can test everithing ignoring, from the unit testing perspective what is the source of your information, and maybe it's enough for your application.

But sometimes you need to test what is exactly in the headers or in the response headers where you need to be sure some headers are sent in a specific format and the wrapper will not help me this time, then I've started to search some alternatives.

## Moq

First found was obviously Moq where I can just mock everything and check the headers but I have some concerns using it due is "SponsorLink" past, then I've skipped it.

## Flurl

Second finding in my search is Flurl that has testability in mind.

Flurl is great and is testable but for me it was really hard to make it work because in my case [this](https://github.com/tmenier/Flurl/blob/363416c4e60911ccce4f82a5dc9012eab7e833a9/src/Flurl.Http/Content/CapturedJsonContent.cs#L15) line was causing an issue, after several debugging session that even I was adding `WithContentType("application/json")` it was ignored/overriden by that line, because I was posting a json, I needed it to be `"application/json"` only and not `"application/json; charset=UTF-8"` and that will mismatch my requirements. This was hard to find I've also investigated the traffic with a local proxy and track all requests and find out that something was adding something else to my request.
In general it is safe to have the charset mentioned as above but in my case it was not so helpful the server give me only a generic error 400 without any real reason. After investigating it figured out to be the addition to the header that changed the signature of the entire message and invalidate it.

Then after a while analizing how to make it work it become clear to me that I needed to come back in control of my HttpClient and find a way to really test it.

## MockHttp

After searching (again) around I've found a [video](https://youtu.be/7OFZZAHGv9o?si=Mt-QCSPudQ_dP5k4) of Nick Chapsas talking about how to mock HttpClient in the right way, watching it I've figured out a good way to test HttpClient then I've decided to give it a try.

Then I can start to inject the mocked client into my SUT like the following

``` csharp
        _mock = new MockHttpMessageHandler();
        _client = _mock.ToHttpClient();

        _sut = new MySystemUnderTest(_client);
```

and right after I can start asserting and make my "test case" in the right way

``` csharp

        _mock.When(_serviceUrl)
            .Respond(HttpStatusCode.NotFound,"application/json","some json content here");

        var response = await _sut.FetData( "bodyContent");

        response.Should().BeOfType<NotFoundResponse>().Which.Status.Should().Be("NotFound");

```

In this way I've created all scenario I needed and tested everything I needed.

## Conclusions

So from now beside my Wrapper I can finally create some test on the Wrapper itself using ***MockHttp*** library and be sure the code work in isolate way with a good feeling that it will work as expected with the "real" implementation.
