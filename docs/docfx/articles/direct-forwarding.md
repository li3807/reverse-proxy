---
uid: direct-forwarding
title: Direct Forwarding
---

# Direct Forwarding

Some applications only need the ability to take a specific request and forward it to a specific destination. These applications do not need, or have addressed in other ways, the other features of the proxy like configuration discovery, routing, load balancing, etc..

## IHttpForwarder

[IHttpForwarder](xref:Yarp.ReverseProxy.Forwarder.IHttpForwarder) serves as the core proxy adapter between incoming AspNetCore and outgoing System.Net.Http requests. It handles the mechanics of creating a HttpRequestMessage from a HttpContext, sending it, and relaying the response.

IHttpForwarder supports:
- Dynamic destination selection, you specify the destination for each request
- Http client customization, you provide the HttpMessageInvoker
- Request and response customization (except bodies)
- Streaming protocols like gRPC and WebSockets
- Error handling

It does not include:
- Routing
- Load balancing
- Affinity
- Retries

## Example

See [ReverseProxy.Direct.Sample](https://github.com/microsoft/reverse-proxy/tree/release/latest/samples/ReverseProxy.Direct.Sample) as a pre-built sample, or use the steps below.

### Create a new project

Follow the [Getting Started](xref:getting-started) guide to create a project and add the Yarp.ReverseProxy nuget dependency.

### Update Startup

In this example the IHttpForwarder is registered in DI, injected into the `Startup.Configure` method, and used to forward requests from a specific route to `https://localhost:10000/prefix/`.

The optional transforms show how to copy all request headers except for the `Host`, it's common that the destination requires its own `Host` from the url.

```C#
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpForwarder();
}

public void Configure(IApplicationBuilder app, IHttpForwarder forwarder)
{
    var httpClient = new HttpMessageInvoker(new SocketsHttpHandler()
    {
        UseProxy = false,
        AllowAutoRedirect = false,
        AutomaticDecompression = DecompressionMethods.None,
        UseCookies = false,
        ActivityHeadersPropagator = new ReverseProxyPropagator(DistributedContextPropagator.Current),
        ConnectTimeout = TimeSpan.FromSeconds(15),
    });
    var transformer = new CustomTransformer(); // or HttpTransformer.Default;
    var requestConfig = new ForwarderRequestConfig { ActivityTimeout = TimeSpan.FromSeconds(100) };

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.Map("/{**catch-all}", async httpContext =>
        {
            var error = await forwarder.SendAsync(httpContext, "https://localhost:10000/",
                httpClient, requestConfig, transformer);
            // Check if the operation was successful
            if (error != ForwarderError.None)
            {
                var errorFeature = httpContext.GetForwarderErrorFeature();
                var exception = errorFeature.Exception;
            }
        });
    });
}
```

```C#
private class CustomTransformer : HttpTransformer
{
    public override async ValueTask TransformRequestAsync(HttpContext httpContext,
        HttpRequestMessage proxyRequest, string destinationPrefix)
    {
        // Copy all request headers
        await base.TransformRequestAsync(httpContext, proxyRequest, destinationPrefix);

        // Customize the query string:
        var queryContext = new QueryTransformContext(httpContext.Request);
        queryContext.Collection.Remove("param1");
        queryContext.Collection["area"] = "xx2";

        // Assign the custom uri. Be careful about extra slashes when concatenating here. RequestUtilities.MakeDestinationAddress is a safe default.
        proxyRequest.RequestUri = RequestUtilities.MakeDestinationAddress("https://example.com", httpContext.Request.Path, queryContext.QueryString);

        // Suppress the original request header, use the one from the destination Uri.
        proxyRequest.Headers.Host = null;
    }
}
```

There are also [extension methods](xref:Microsoft.AspNetCore.Builder.DirectForwardingIEndpointRouteBuilderExtensions) available that simplify the mapping of IHttpForwarder to endpoints.

```C#
...

public void Configure(IApplicationBuilder app, IHttpForwarder forwarder)
{
    ...

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapForwarder("/{**catch-all}", "https://localhost:10000/", requestConfig, transformer, httpClient);
    });
}
```

### The HTTP Client

The http client may be customized, but the above example is recommended for common proxy scenarios.

Always use HttpMessageInvoker rather than HttpClient, HttpClient buffers responses by default. Buffering breaks streaming scenarios and increases memory usage and latency.

Re-using a client for requests to the same destination is recommended for performance reasons as it allows you to re-use pooled connections. A client may also be re-used for requests to different destinations if the configuration is the same.

### Transforms

The request and response can be modified by providing a derived [HttpTransformer](xref:Yarp.ReverseProxy.Forwarder.HttpTransformer) as a parameter to [`SendAsync`](xref:Yarp.ReverseProxy.Forwarder.IHttpForwarder) method.

### Error handling

IHttpForwarder catches exceptions and timeouts from the HTTP client, logs them, and converts them to 5xx status codes or aborts the response. An error code is returned from `SendAsync`, and the error details can be accessed from the [IForwarderErrorFeature](xref:Yarp.ReverseProxy.Forwarder.IForwarderErrorFeature) as shown above.
