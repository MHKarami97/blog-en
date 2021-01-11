---
title: "Create http request with flurl and polly"
date: 2021-01-11T09:46:00-00:00
categories:
  - Core
tags:
  - core
  - polly
  - flurl
---

Making http request is not easy to handle. for example dns changing, just create single HttpClient and ...
<br />
also it's need some default cofig and inject HttpClient to call api
<br />
Flurl is a library to make http request in simplest way
<br />
Polly is a library to automatic retry method if it riase expection
<br />
<br />
repository links:
<br />
[polly github](https://github.com/App-vNext/Polly)  
[flurl github](https://github.com/tmenier/Flurl)  
<br />
working with flurl is very easy and it's not need any config or dependency injection
<br />
now you can use it like this:
<br />

```c#
var result = await "https://api.com"
    .SetQueryParams(new { api_key = "abc" })
    .WithOAuthBearerToken("my_token")
    .PostJsonAsync(new { first_name = firstName, last_name = lastName })
    .ReceiveJson<T>();
```

thre is no need to inject any HttpClient, you need just your url as string to call eat
<br />
we use these items:
<br />

- SetQueryParams : add custom parameter to url
- WithOAuthBearerToken : add auth token to header
- PostJsonAsync : data that send as post method
- ReceiveJson : result as json

also there are more method that you can see in [this link](https://flurl.dev/docs/fluent-http)
<br />
or use it like ReceiveJson() to make result type dynamic
or add custom header to your request
<br />
and any thing else that HttpClient has and you need to work with rest api
<br />
<br />
this library has very good exception handling.
<br />
we can define some status code to prevent exception or make it default to raise  exception in result status code is not 200:

```c#
url.AllowHttpStatus("400-404,6xx").GetAsync();
url.AllowAnyHttpStatus().GetAsync();
```

if we impact error, this exceptions raise:

- FlurlHttpTimeoutException
- FlurlHttpException

for example it modes that api return some info when exception happend.

```c#
return await ex.GetResponseJsonAsync() ??
             ex.GetBaseException().Message;
```

one of most important item that you should pat attention on it, is not create multi HttpClient in single call. this problem is handle currently by [this library](https://flurl.dev/docs/client-lifetime)
<br />
this library is familier with test, also has fake implementation that you can see it in [this link](https://flurl.dev/docs/testable-http/)

```c#
[Test]
public void Test_Some_Http_Calling_Method() {
    using (var httpTest = new HttpTest()) {
        sut.CallThingThatUsesFlurlHttp();
    }
}
```

<br />
one of best library to retry code for example when time out happend, is Polly.
<br />
we can use it like this:
<br />

```c#
private readonly AsyncRetryPolicy _polly;

public MyConstructor()
{
    _polly = Policy
        .Handle<FlurlHttpTimeoutException>()
        .WaitAndRetryAsync(new[]
        {
            TimeSpan.FromSeconds(1),
            TimeSpan.FromSeconds(2)
        });
}
```

in Handle we can say that witch exceptin we want to retry, for example if FlurlHttpTimeoutException happen, we retry code
<br />
after that we define retry count in WaitAndRetryAsync
<br />
in this example we say that retry 2 times and in first time, wait 1 second and then wait 2 second
<br />
know we can use this code in every where we want:

```c#
var result = await _polly
    .ExecuteAsync(async ct =>
            await MyMethod(),
        cancellationToken);
```

for example if we want use it by flurl, we can do this:


```c#
var result = await _polly
    .ExecuteAsync(async ct =>
            await BaseInfo.WebApiAddress
                .AppendPathSegment("auth")
                .WithHeader(abc,
                    "in header")
                .PostJsonAsync(new
                {
                    Username = userUserName,
                    Password = userPassword
                }, ct).ReceiveJson<ResultInfo>(),
        cancellationToken);
```
