# httpcontext-shim
## An abstraction for HttpContext that works in self-hosted ASP.NET Web API

```powershell
PM> Install-Package HttpContextShim
```

### Introduction
If you're building Web API components that are meant to operate in both web host (IIS) or self host (in-process), you already know
that the HTTP models of both stacks are drastically different, and that HttpContext is an IIS concept that's not available in 
self-hosted mode. This makes it difficult to build APIs that can migrate from IIS to self-hosted mode with ease, or to provide
middleware to customers that works on multiple hosting platforms. This shim provides a workaround for this distinction by 
giving you an HttpContext-like interface that works in either mode.

### Features

* Provides a stand-in HttpContext that works independently of the hosting environment
* You can use the HttpContext as if you were in web hosted mode in self host, i.e. `HttpContext.Current.Request.IsLocal` and `HttpContextCurrent.Request.UserHostAddress` are available
* Easy to opt-in by installing a message handler

### Usage

Add the `HttpContextHandler` to your configuration's message handler collection _as early as possible_. Normally this means as the first entry,
but you can defer it up to the point where any downstream handlers require access to HttpContext.

```csharp
HttpConfiguration configuration = HoweverYouGetThis();
configuration.MessageHandlers.Add(new HttpContextHandler());
configuration.Routes.MapHttpRoute(
    name: "DefaultApi",
    routeTemplate: "api/{controller}/{id}",
    defaults: new { id = RouteParameter.Optional }
);
```

After installing the handler, you just use our `HttpContext` in place of the default:

```csharp
// Direct usage
using HttpContext = HttpContextShim.HttpContext;
var items = HttpContext.Current.Items;

// Handler usage by accessing properties
using HttpContext = HttpContextShim.HttpContext;
public class MyHandler : DelegatingHandler
{
    private const string HttpContextProperty = "MS_HttpContext";
        
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        return base.SendAsync(request, cancellationToken).ContinueWith(task =>
        {
			var result = task.Result;

			// Do stuff with an HttpContext
			var context = request.Properties[HttpContextProperty] as HttpContext;
			
            return result;
        });
    }
}
```