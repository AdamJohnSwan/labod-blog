---
layout: post
title:  "Creating OpenAPI Docs With Swashbuckle"
date:   2019-09-05
categories: post
---

## Generating OpenAPI spec as part of a build process

```
cd /the/project-directory

dotnet new tool-manifest
dotnet tool install SwashBuckle.AspNetCore.Cli

dotnet build

dotnet swagger tofile --output path/to/folder/v1/swagger.json bin\Debug\net8.0\AssemblyName.dll v1
```

This method caused an error. My project does not use a Startup type and I didn't want to change the layout for this tool. 
```
Unhandled exception. System.InvalidOperationException: A type named 'StartupProduction' or 'Startup' could not be found in assembly 'TrackLaw.Api'.
```

Alternatively, it can be generated at runtime

## Generating OpenAPI spec as part of a project runtime

```
ISwaggerProvider sw = app.Services.GetRequiredService<ISwaggerProvider>();
Microsoft.OpenApi.Models.OpenApiDocument doc = sw.GetSwagger(documentName: "v1", host: null, basePath: "/");
string swaggerString = JsonSerializer.Serialize(doc);
File.WriteAllText($"path/to/folder/v1/swagger.json", swaggerString);
```


## Display the documentation
The file has been created. Many projects can handle displaying OpenAPI documents. Here is an example using [Redoc](https://github.com/Redocly/redoc)

If you're using vanilla javascript in your frontend add this to your page.
```
<body>
	<redoc spec-url='path/to/folder/v1/swagger.json'></redoc>
	<script src="https://cdn.redoc.ly/redoc/latest/bundles/redoc.standalone.js"></script>
</body>
```

Visiting the page will load the documentation in an easy to read format!
