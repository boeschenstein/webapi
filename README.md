# ASP.NET WebApi

ASP.NET see: <https://github.com/boeschenstein/aspnet>

## Routing

### Add relative sub routes

```cs
[ApiController]
[Route('api/mainroute')]
public class MyControllerClass ...

    [HttpPost('my/additional/subroute1')]
    public async Task<IActionResult> MyEndpoint1Async ...

    [Authorize(Policy = PfwPolicyRequirement.EditorOrAdmin)]
    //[ProducesDefaultResponseType] todo what is this ?
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(typeof(IReadOnlyCollection<string>), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(IReadOnlyCollection<string>), StatusCodes.Status404NotFound)]
    [Produces("application/octet-stream")]
    [HttpGet('my/additional/subroute2/{id:Guid}', Name = 'anothername')]
    public async Task<IActionResult> MyEndpoint2Async ...
```

## Move Controllers to other project/assembly

### Option 1)

Use ASP.NET Core APIs in a class library. Add this to your csproj:

```xml
  <!-- Use ASP.NET Core APIs in a class library -->
  <!-- https://learn.microsoft.com/en-us/aspnet/core/fundamentals/target-aspnetcore -->
  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>
```

### Option b)

program.cs:

```cs
var presentationAssembly = typeof(PresentationProj.AssemblyReference).Assembly;

// AddControllers() needs <FrameworkReference Include="Microsoft.AspNetCore.App" />
services.AddControllers().AddApplicationPart(presentationAssembly);
```

- Source: Clean Architecture With .NET 6 And CQRS - Project Setup by Milan Jovanović <https://youtu.be/tLk4pZZtiDY> Min 11:20
- Docs: <https://learn.microsoft.com/en-us/aspnet/core/mvc/advanced/app-parts>

This approach might need some nugets:

```cs
<PackageReference Include="Microsoft.Extensions.Hosting.Abstractions" Version="8.0.0" />
<PackageReference Include="Microsoft.AspNetCore.Components" Version="8.0.5" />
```

## Order of Middleware

> The order that middleware components are added in the Program.cs file defines the order in which the middleware components are invoked on requests (and the reverse order for the response).
> The order is critical for security, performance, and functionality.

Use this recommended order:

- Start: Request
- ExceptionHandler
- HSTS
- HttpsRedirection (avoid this anyway: can do harm in OIDC scenario: does not forward security headers)
- Static Files
- Routing
- CORS
- Authentication
- Authorization
- Custom middlewares
- Endpoint

Source: <https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-8.0#middleware-order>

## File Download

### Zip, using efficiant streams

<https://swimburger.net/blog/dotnet/create-zip-files-on-http-request-without-intermediate-files-using-aspdotnet-mvc-razor-pages-and-endpoints>

```cs
[HttpGet]
[ProducesResponseType(StatusCodes.Status200OK)]
[Produces("application/octet-stream")]
[Route("download0", Name = "zipfile")]
public async Task DownloadBots()
{
    Response.ContentType = "application/octet-stream";
    Response.Headers.Add("Content-Disposition", "attachment; filename=\"Bots.zip\"");

    var botsFolderPath = Path.Combine("c:\\temp", "bots");
    var botFilePaths = Directory.GetFiles(botsFolderPath);
    using (ZipArchive archive = new ZipArchive(Response.BodyWriter.AsStream(), ZipArchiveMode.Create))
    {
        foreach (var botFilePath in botFilePaths)
        {
            var botFileName = Path.GetFileName(botFilePath);
            var entry = archive.CreateEntry(botFileName);
            using (var entryStream = entry.Open())
            using (var fileStream = System.IO.File.OpenRead(botFilePath))
            {
                await fileStream.CopyToAsync(entryStream);
            }
        }
    }
}
```

### CSV using CSVHelper (stream only, no files)

Inspired from <https://stackoverflow.com/questions/13646105/csvhelper-not-writing-anything-to-memory-stream/22997765#22997765>

```cs
[HttpGet]
[ProducesResponseType(StatusCodes.Status200OK)]
[Produces("application/octet-stream")]
[Route("download1", Name = "mycsv")]
public void DownloadCsv()
{
    Response.ContentType = "application/octet-stream";
    Response.Headers.Add("Content-Disposition", "attachment; filename=\"mycsv.csv\"");

    using (var streamWriter = new StreamWriter(Response.BodyWriter.AsStream()))
    using (var csvWriter = new CsvWriter(streamWriter, CultureInfo.InvariantCulture))
    {
        csvWriter.WriteRecords<User>(users);
        csvWriter.Flush();
    } // StreamWriter gets flushed here.
}
```

### No CSVHelper

from <https://arminzia.com/blog/export-data-to-excel-with-aspnet-core/>

```cs
[HttpGet]
[ProducesResponseType(StatusCodes.Status200OK)]
[Produces("application/octet-stream")]
[Route("download2", Name = "usercsvfile")]
public IActionResult Csv()
{
    var builder = new StringBuilder();
    builder.AppendLine("Id,Username,Email,JoinedOn,SerialNumber");
    foreach (var user in users)
    {
        builder.AppendLine($"{user.Id},{user.Username},{user.Email},{user.JoinedOn.ToShortDateString()},{user.SerialNumber}");
    }
    return File(Encoding.UTF8.GetBytes(builder.ToString()), "text/csv", "users.csv");
}
```

### Data classes:

```cs
    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
        public string Email { get; set; }
        public string SerialNumber { get; set; }
        public DateTime JoinedOn { get; set; }
    }

    private readonly List<User> users = new()
    {
      new User
      {
          Id = 1,
          Username = "ArminZia",
          Email = "armin.zia@gmail.com",
          SerialNumber = "NX33-AZ47",
          JoinedOn = new DateTime(1988, 04, 20)
      },
      new User
      {
          Id = 2,
          Username = "DoloresAbernathy",
          Email = "dolores.abernathy@gmail.com",
          SerialNumber = "CH1D-4AK7",
          JoinedOn = new DateTime(2021, 03, 24)
      }
    };
```

## Information

- Handle errors in ASP.NET Core controller-based web APIs
  - UseExceptionHandler, UseStatusCodePages: <https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors?view=aspnetcore-8.0>
  - Translating Exceptions into Problem Details Responses <https://timdeschryver.dev/blog/translating-exceptions-into-problem-details-responses#>
- Result Pattern <https://www.milanjovanovic.tech/blog/functional-error-handling-in-dotnet-with-the-result-pattern>
- IMiddleware <https://www.milanjovanovic.tech/blog/3-ways-to-create-middleware-in-asp-net-core>
