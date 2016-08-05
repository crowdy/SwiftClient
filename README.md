# OpenStack Swift client for .NET

[![Build status](https://ci.appveyor.com/api/projects/status/77ts9n1a6w5xnrjw?svg=true)](https://ci.appveyor.com/project/stefanprodan/swiftclient)
[![NuGet version](https://img.shields.io/nuget/vpre/SwiftClient.svg)](https://www.nuget.org/packages/SwiftClient/)

***SwiftClient*** is an async HTTP wrapper over OpenStack Swift REST API and follows the [Object Storage API Reference](http://developer.openstack.org/api-ref-objectstorage-v1.html). 
It can be installed via NuGet from [nuget.org/packages/SwiftClient](https://www.nuget.org/packages/SwiftClient/) and it's compatible with .NET Framework 4.5 and .NET Core 1.0.

***SwiftClient.AspNetCore*** is a SwiftClient implementation for ASP.NET Core MVC that comes with utilities for buffered upload/download and video streaming. 
It can be installed via NuGet from [nuget.org/packages/SwiftClient.AspNetCore](https://www.nuget.org/packages/SwiftClient.AspNetCore/).

### SwiftClient usage for .NET 4.5 or .NET Core

Install NuGet package:
```
Install-Package SwiftClient
```

Configuration:

```cs
var swiftClient = new SwiftClient.Client()
.WithCredentials(new SwiftCredentials
{
     Username = "test:tester",
     Password = "testing",
     Endpoints = new List<string> { 
		"http://192.168.3.31:8080",
		"http://192.168.3.32:8080"
		}
})
.SetRetryCount(6)
.SetRetryPerEndpointCount(2)
.SetLogger(new SwiftLogger());
```

The client implements a configurable retry mechanism, so you don't have to worry about the token expiration date or a temporary request failure. 
It also supports multiple endpoints (Swift proxy address), it will iterate through each endpoint till it finds one that's available, if the maximum retry count is reached an exception will be thrown.
If you want to log failure events, just pass the client your implementation of the `ISwiftLogger` interface. In the CLI project there is a [stdout log example](https://github.com/vtfuture/SwiftClient/blob/master/samples/SwiftClient.Cli/SwiftConsoleLog.cs).

You have to supply your own implementation of `ISwiftAuthManager` class and provide a caching mechanism for the ***authentication token*** so that each Swift request is not being preceded by an authentication request. It is recommended to use a dedicated cache storage like Redis so multiple instances of your app can reuse the authentication token. In the AspNetCore project there is a `ISwiftAuthManager` [implementation](https://github.com/vtfuture/SwiftClient/blob/master/src/SwiftClient.AspNetCore/SwiftAuthManagerMemoryCache.cs) that uses ASP.NET Core in memory cache.

###  SwiftClient.AspNetCore usage for ASP.NET Core MVC

Install NuGet package:

```
Install-Package SwiftClient.AspNetCore
```

You can load Swift credentials from an json file in ASP.NET Core projects. Add an [`appsettings.json`](https://github.com/vtfuture/SwiftClient/blob/master/samples/SwiftClient.AspNetCore.Demo/appsettings.json) file in the root your project and load the settings in [`Startup.cs`](https://github.com/vtfuture/SwiftClient/blob/master/samples/SwiftClient.AspNetCore.Demo/Startup.cs).

Configure SwiftClient service in Startup.cs:

```cs
public void ConfigureServices(IServiceCollection services)
{
	services.AddOptions();
	services.AddMemoryCache();

	services.AddMvc();

	services.Configure<SwiftServiceOptions>(Configuration.GetSection("SwiftCluster"));
	services.AddSingleton<ISwiftLogger, SwiftServiceLogger>();
	services.AddSingleton<ISwiftAuthManager, SwiftAuthManagerMemoryCache>();
	services.AddTransient<ISwiftClient, SwiftService>();
}
```

Use the `SwiftService` in your controller:

```cs
private readonly ISwiftClient _swiftService;

public HomeController(ISwiftClient swiftService)
{
	_swiftService = swiftService;
}
```

Simple upload/download example for small files:

```cs
public async Task<IActionResult> UploadFile(IFormFile file)
{ 
    using (var fileStream = file.OpenReadStream())
    {
        using (var memoryStream = new MemoryStream())
        {
            await fileStream.CopyToAsync(memoryStream);

            var resp = await _swiftService.PutObject(containerId, fileId, memoryStream);

            return new JsonResult(new
            {
                Success = resp.IsSuccess
            });
        }
    }
}

public async Task<IActionResult> DownloadFile(string fileId)
{
    var rsp = await _swiftService.GetObject("containerId", fileId);

    if (rsp.IsSuccess)
    {
        return new FileStreamResult(rsp.Stream, "application/octet-stream");
    }

    return new NotFoundResult();
}
```

Chunked upload example for large files:

```cs
public async Task<IActionResult> UploadChunk(int segment)
{
	if (Request.Form.Files != null && Request.Form.Files.Count > 0)
	{
		var file = Request.Form.Files[0];
		
		using (var fileStream = file.OpenReadStream())
		{
			using (var memoryStream = new MemoryStream())
			{
				var fileName = file.GetFileName();

				await fileStream.CopyToAsync(memoryStream);

				// upload file chunk
				await _swiftService.PutChunkedObject(containerTempId, fileName, memoryStream.ToArray(), segment);
			}
		}
	}

	return new JsonResult(new
	{
		Success = false
	});
}

public async Task<IActionResult> UploadDone(string fileName, string contentType)
{
	// use manifest to merge chunks
        await _swiftService.PutManifest(containerTempId, fileName);

        // copy chunks to new file and set some meta data info about the file (filename, contentype)
        await _swiftService.CopyObject(containerTempId, fileName, containerDemoId, fileName, new Dictionary<string, string>
        {
            { $"X-Object-Meta-{metaFileName}", fileName },
            { $"X-Object-Meta-{metaContentType}", contentType }
        });

        // cleanup temp
        await _swiftService.DeleteContainerContents(containerTempId);
}
```

Buffered download example using `BufferedHTTPStream`:

```cs
public async Task<IActionResult> DownloadFile(string fileId)
{
	var headObject = await _swiftService.HeadObject(containerId, fileId);

	if (headObject.IsSuccess && headObject.ContentLength > 0)
	{
		var fileName = headObject.GetMeta("Filename");
		var contentType = headObject.GetMeta("Contenttype");

		Response.Headers.Add("Content-Disposition", $"attachment; filename={fileName}");

		var stream = new BufferedHTTPStream((start, end) =>
		{
			using (var response = _swiftService.GetObjectRange(containerId, objectId, start, end).Result)
           {
	                var ms = new MemoryStream();
	
	                response.Stream.CopyTo(ms);
	
	                return ms;
           }

		}, () => headObject.ContentLength);

		return new FileStreamResult(stream, contentType);
	}

	return new NotFoundResult();
}
```

### Running the ASP.NET Core MVC demo

The [SwiftClient.AspNetCore.Demo](https://github.com/vtfuture/SwiftClient/tree/master/samples/SwiftClient.AspNetCore.Demo) project is an example of how to authenticate against Swift, do chunked upload for a large file and download it and also video streaming. 

You will need at least one Ubuntu 14.04 box to host OpenStack Swift proxy and storage. For dev/test environments we provide a docker image with a single Swift proxy and storage, follow the setup instruction from [docker-swift](https://github.com/vtfuture/SwiftClient/tree/master/tools/docker-swift) to build and run the Swift container. After you've started the Swift all-in-one container, put your Ubuntu box IP in the `appsettings.json` from the demo project and your good to go. You can also use Docker for Windows to host the Swift dev container.

If you want to setup Swift for production on a Ubuntu cluster check out the [documentation](https://github.com/vtfuture/SwiftClient/wiki) from our wiki.
