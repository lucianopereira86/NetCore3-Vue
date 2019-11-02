# NetCore3-Vue

Creating a .Net Core 3 web application integrated with Vue.

## Technologies

- [.Net Core 3](https://dotnet.microsoft.com/download/dotnet-core/3.0)
- [Vue](https://vuejs.org/)

## Instructions

Create a new .Net Core web application project:

![print01](/docs/print01.JPG)

Select the "API" model and press on "Create":

![print05](/docs/print05.JPG)

Inside the project folder, open a terminal and run the following command to create a Vue project:

```powershell
vue create client-app
```

Edit manually the _.csproj_ file with the code below to build the Vue project with the .NET Core project:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <RootNamespace>YOUR-PROJECT-NAME-HERE</RootNamespace>
    <TargetFramework>netcoreapp3.0</TargetFramework>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
    <TypeScriptToolsVersion>Latest</TypeScriptToolsVersion>
    <IsPackable>false</IsPackable>
    <SpaRoot>client-app\</SpaRoot>
    <DefaultItemExcludes>$(DefaultItemExcludes);$(SpaRoot)node_modules\**</DefaultItemExcludes>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.SpaServices.Extensions" Version="3.0.0-preview6.19307.2" />
  </ItemGroup>

  <ItemGroup>
    <!-- Don't publish the SPA source files, but do show them in the project files list -->
    <Content Remove="$(SpaRoot)**" />
    <None Remove="$(SpaRoot)**" />
    <None Include="$(SpaRoot)**" Exclude="$(SpaRoot)node_modules\**" />
  </ItemGroup>

  <Target Name="DebugEnsureNodeEnv" BeforeTargets="Build" Condition=" '$(Configuration)' == 'Debug' And !Exists('$(SpaRoot)node_modules') ">
    <!-- Ensure Node.js is installed -->
    <Exec Command="node --version" ContinueOnError="true">
      <Output TaskParameter="ExitCode" PropertyName="ErrorCode" />
    </Exec>
    <Error Condition="'$(ErrorCode)' != '0'" Text="Node.js is required to build and run this project. To continue, please install Node.js from https://nodejs.org/, and then restart your command prompt or IDE." />
    <Message Importance="high" Text="Restoring dependencies using 'npm'. This may take several minutes..." />
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm install" />
  </Target>

  <Target Name="PublishRunWebpack" AfterTargets="ComputeFilesToPublish">
    <!-- As part of publishing, ensure the JS resources are freshly built in production mode -->
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm install" />
    <Exec WorkingDirectory="$(SpaRoot)" Command="npm run build" />

    <!-- Include the newly-built files in the publish output -->
    <ItemGroup>
      <DistFiles Include="$(SpaRoot)dist\**" />
      <ResolvedFileToPublish Include="@(DistFiles->'%(FullPath)')" Exclude="@(ResolvedFileToPublish)">
        <RelativePath>%(DistFiles.Identity)</RelativePath>
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
      </ResolvedFileToPublish>
    </ItemGroup>
  </Target>

</Project>
```

Create a class named _VueHelper_ to configure the server for the Vue application:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.SpaServices;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using System;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Net.NetworkInformation;
using System.Runtime.InteropServices;
using System.Threading.Tasks;

namespace NetCore3_Vue
{
    public static class VueHelper
    {
        // default port number of 'npm run serve'
        private static int Port { get; } = 8080;
        private static Uri DevelopmentServerEndpoint { get; } = new Uri($"http://localhost:{Port}");
        private static TimeSpan Timeout { get; } = TimeSpan.FromSeconds(30);
        // done message of 'npm run serve' command.
        private static string DoneMessage { get; } = "DONE  Compiled successfully in";

        public static void UseVueDevelopmentServer(this ISpaBuilder spa)
        {
            spa.UseProxyToSpaDevelopmentServer(async () =>
            {
                var loggerFactory = spa.ApplicationBuilder.ApplicationServices.GetService<ILoggerFactory>();
                var logger = loggerFactory.CreateLogger("Vue");
                // if 'npm run serve' command was executed yourself, then just return the endpoint.
                if (IsRunning())
                {
                    return DevelopmentServerEndpoint;
                }

                // launch vue.js development server
                var isWindows = RuntimeInformation.IsOSPlatform(OSPlatform.Windows);
                var processInfo = new ProcessStartInfo
                {
                    FileName = isWindows ? "cmd" : "npm",
                    Arguments = $"{(isWindows ? "/c npm " : "")}run serve",
                    WorkingDirectory = "client-app",
                    RedirectStandardError = true,
                    RedirectStandardInput = true,
                    RedirectStandardOutput = true,
                    UseShellExecute = false,
                };
                var process = Process.Start(processInfo);
                var tcs = new TaskCompletionSource<int>();
                _ = Task.Run(() =>
                {
                    try
                    {
                        string line;
                        while ((line = process.StandardOutput.ReadLine()) != null)
                        {
                            logger.LogInformation(line);
                            if (!tcs.Task.IsCompleted && line.Contains(DoneMessage))
                            {
                                tcs.SetResult(1);
                            }
                        }
                    }
                    catch (EndOfStreamException ex)
                    {
                        logger.LogError(ex.ToString());
                        tcs.SetException(new InvalidOperationException("'npm run serve' failed.", ex));
                    }
                });
                _ = Task.Run(() =>
                {
                    try
                    {
                        string line;
                        while ((line = process.StandardError.ReadLine()) != null)
                        {
                            logger.LogError(line);
                        }
                    }
                    catch (EndOfStreamException ex)
                    {
                        logger.LogError(ex.ToString());
                        tcs.SetException(new InvalidOperationException("'npm run serve' failed.", ex));
                    }
                });

                var timeout = Task.Delay(Timeout);
                if (await Task.WhenAny(timeout, tcs.Task) == timeout)
                {
                    throw new TimeoutException();
                }

                return DevelopmentServerEndpoint;
            });

        }

        private static bool IsRunning() => IPGlobalProperties.GetIPGlobalProperties()
                .GetActiveTcpListeners()
                .Select(x => x.Port)
                .Contains(Port);
    }
}

```

Inside the _Startup_ class, change the _ConfigureServices_ method to identify the Vue publishing directory:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddSpaStaticFiles(options => options.RootPath = "client-app/dist");
}
```

Modify the _Configure_ method to allow static files:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Other code
    (...)

    // add following statements
    app.UseSpaStaticFiles();
    app.UseSpa(spa =>
    {
        spa.Options.SourcePath = "client-app";
        if (env.IsDevelopment())
        {
            // Launch development server for Vue.js
            spa.UseVueDevelopmentServer();
        }
    });
}
```

Before launching the app, open the project's property page and edit the _Debug_ section by removing the _Launch browser_ field:

![print06](/docs/print06.JPG)

Run the app and this page will open:

![print07](/docs/print07.JPG)

It's time to connect the app with a web API. Create a controller named _ValuesController_ with a GET method:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace NetCore3_Vue.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ValuesController : ControllerBase
    {
        [HttpGet]
        public IActionResult Get()
        {
            return Ok(new string[] { "value1", "value2" });
        }
    }
}
```

Change the _HelloWorld.vue_ file inside the Vue project to display data obtained from the web API:

```html
<template>
	<div>
		<div :key="r" v-for="r in this.results">{{ r }}</div>
	</div>
</template>

<script>
	export default {
		name: 'HelloWorld',
		data() {
			return {
				results: []
			};
		},
		async created() {
			const r = await fetch('/api/values');
			this.results = await r.json();
		}
	};
</script>
```

Run the app again and this will be the result:

![print09](/docs/print09.JPG)

## Conclusion

In order to make a SPA page running along side a web API in the same solution, it was needed to make changes manually in the .Net Core project config file. A helper class was created to manage the Vue application as well.

## Reference

[How to integrate Vue.js and ASP.NET Core using SPA Extension](https://techcommunity.microsoft.com/t5/Windows-Dev-AppConsult/How-to-integrate-Vue-js-and-ASP-NET-Core-using-SPA-Extension/ba-p/698648)
