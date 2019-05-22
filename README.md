# REST API'S WITH ASP.NET CORE 2 & EF CORE 2

## Table of contents
* [ 1. ASP.NET Core](#1-aspnet-core)
* [ 2. MVC](#2-mvc)
* [ 3. HTTP status code levels](#3-http-status-code-levels)
* [ 4. HTTP actions and proper responses](#4-http-actions-and-proper-responses)
* [ 5. Global error handling](#5-global-error-handling)
* [ 6. Logging with NLog](#6-logging-with-nlog)
* [ 7. Content negotiation](#7-content-negotiation)
* [ 8. IOC and dependency injection](#8-ioc-and-dependency-injection)
* [ 9. Configuration files and environment variables](#9-configuration-files-and-environment-variables)
* [ 10. Entity Framework Core 2](#10-entity-framework-core-2)
* [ 11. DTO's and AutoMapper](#11-dtos-and-automapper)

## 1. ASP.NET Core
- ASP.NET Core can run on both the full .NET framework and the .NET Core framework (.NET Standard is not a framework, it is a standard which the frameworks comply with)
- ASP.NET Core can run on both Windows and Linux, but the full .NET framework does not support Linux
- Inside the Startup class ConfigureServices() method, we wire up the dependency injection system, by adding dependencies to the IOC container
- Inside the Startup class Configure() method, we wire up the HTTP request chain, by adding middleware, like MVC, EF, logging, etc.
- ASP.NET Core supports different environments, Development, Staging, and Production are builtin, but we can add more
- Environments are independent of Debug/Release build configuration settings
- If we change the environment, we should restart the web server (Kestrel, IIS, Apache, etc.) for the changes to take effect

## 2. MVC:
- MVC middleware covers both MVC (Razor views) and WebAPI (Web services) applications
- Microsoft.AspNetCore.All is a meta-package that includes ASP.NET Core packages, including MVC, Authentication, EF Core, and others
- Runtime Store is a special common folder on the machine where the packages in the meta-packages sit, and where they can be shared by all apps
- In ASP.NET Core 2, packages in the Runtime Store folder are not copied to the output folder of the app by default, and need to be deployed separately
- We can define a convention based routing in app.UseMVC() or use attribute based routing on controllers or action methods, which is the recommended way for API's
- Route attribute works at the controller level, HttpGet, HttpPost, HttpPut, HttpPatch, and HttpDelete attributes work at the action level, and they all accept a string URI parameter to define the routes
- We can put parameters in curly braces inside the route URI, which will also be passed to the action method as parameters
- We should return the correct HTTP status code and payload in an action method

## 3. HTTP status code levels:
- Level 200 status codes mean success, like 200 OK, 201 Created, 204 No Content
- Level 400 status codes mean client error, like 400 Bad Request, 401 Unauthorized (user is not authorized), 403 Forbidden (user is authorized but lacks permission), 404 Not Found, 409 Conflict (conflicting updates)
- Level 500 status codes mean server error, like 500 Internal Server Error

## 4. HTTP actions and proper responses:
The correct REST status codes and payloads are listed as follows:
- GET without an id returns:
- 200 Ok with the collection data in the payload, whether the collection data is empty or not
```cs
[HttpGet]
public async Task<ActionResult<IEnumerable<Models.Movie>>> GetMovies()
{
    var movieEntities = await _moviesRepository.GetMoviesAsync();
    return Ok(_mapper.Map<IEnumerable<Models.Movie>>(movieEntities));
}
```
- GET with an id returns:
- 200 OK with the data in the payload on success, 
- 404 Not Found if the resource does not exist
```cs
[HttpGet("{movieId}", Name = "GetMovie")]
public async Task<ActionResult<Models.Movie>> GetMovie(Guid movieId)
{
    var movieEntity = await _moviesRepository.GetMovieAsync(movieId);
    if (movieEntity == null)
    {
        return NotFound();
    }

    return Ok(_mapper.Map<Models.Movie>(movieEntity));
} 
```
- POST returns:
- 201 Created with the URI to the newly created resource in the payload on success, 
- 400 Bad Request if the input payload is empty, 
- 422 Unprocessable Entity with the validation errors in the payload if there are validation errors
```cs
[HttpPost]
public async Task<IActionResult> CreateMovie(
    [FromBody] Models.MovieForCreation movieForCreation)
{
    // model validation 
    if (movieForCreation == null)
    {
        return BadRequest();
    } 

    if (!ModelState.IsValid)
    {
        // return 422 - Unprocessable Entity when validation fails
        return new UnprocessableEntityObjectResult(ModelState);
    }

    var movieEntity = _mapper.Map<Movie>(movieForCreation);
    _moviesRepository.AddMovie(movieEntity);
    
    // save the changes
    await _moviesRepository.SaveChangesAsync();

    // Fetch the movie from the data store so the director is included
    await _moviesRepository.GetMovieAsync(movieEntity.Id);

    return CreatedAtRoute("GetMovie",
        new { movieId = movieEntity.Id },
        _mapper.Map<Models.Movie>(movieEntity));
}
```
- PUT returns:
- 200 OK with either the data in the payload, or empty payload on success, 
- 400 Bad Request if the input payload is empty, 
- 422 Unprocessable Entity with the validation errors in the payload if there are validation errors,
- 404 Not Found if the resource does not exist
```cs
[HttpPut("{movieId}")]
public async Task<IActionResult> UpdateMovie(Guid movieId, 
    [FromBody] Models.MovieForUpdate movieForUpdate)
{
    // model validation 
    if (movieForUpdate == null)
    {
        //return BadRequest();
    }

    if (!ModelState.IsValid)
    {
        // return 422 - Unprocessable Entity when validation fails
        return new UnprocessableEntityObjectResult(ModelState);
    }

    var movieEntity = await _moviesRepository.GetMovieAsync(movieId);
    if (movieEntity == null)
    {
        return NotFound();
    }

    // map the inputted object into the movie entity
    // this ensures properties will get updated
    _mapper.Map(movieForUpdate, movieEntity);

    // call into UpdateMovie even though in our implementation 
    // this doesn't contain code - doing this ensures the code stays
    // reliable when other repository implemenations (eg: a mock 
    // repository) are used.
    _moviesRepository.UpdateMovie(movieEntity);

    await _moviesRepository.SaveChangesAsync();

    // return the updated movie, after mapping it
    return Ok(_mapper.Map<Models.Movie>(movieEntity));
}
```
- PATCH returns:
- 200 with either the data in the payload, or empty payload on success,
- 422 Unprocessable Entity with the validation errors in the payload if there are validation errors,
- 404 Not Found if the resource does not exist
```cs
[HttpPatch("{movieId}")]
public async Task<IActionResult> PartiallyUpdateMovie(Guid movieId, 
    [FromBody] JsonPatchDocument<Models.MovieForUpdate> patchDoc)
{
    var movieEntity = await _moviesRepository.GetMovieAsync(movieId);
    if (movieEntity == null)
    {
        return NotFound();
    }

    // the patch is on a DTO, not on the movie entity
    var movieToPatch = Mapper.Map<Models.MovieForUpdate>(movieEntity);

    patchDoc.ApplyTo(movieToPatch, ModelState);
      
    if (!ModelState.IsValid)
    {
        return new UnprocessableEntityObjectResult(ModelState);
    }

    // map back to the entity, and save
    Mapper.Map(movieToPatch, movieEntity);

    // call into UpdateMovie even though in our implementation 
    // this doesn't contain code - doing this ensures the code stays
    // reliable when other repository implemenations (eg: a mock 
    // repository) are used.
    _moviesRepository.UpdateMovie(movieEntity);

    await _moviesRepository.SaveChangesAsync();

    // return the updated movie, after mapping it
    return Ok(_mapper.Map<Models.Movie>(movieEntity));
}
```
- DELETE returns:
- 204 No Content with an empty payload on success,
- 404 Not Found if the resource does not exist
```cs
[HttpDelete("{movieid}")]
public async Task<IActionResult> DeleteMovie(Guid movieId)
{
    var movieEntity = await _moviesRepository.GetMovieAsync(movieId);
    if (movieEntity == null)
    {
        return NotFound();
    }

    _moviesRepository.DeleteMovie(movieEntity);
    await _moviesRepository.SaveChangesAsync();

    return NoContent();
}
```
- These also apply to actions involving child resources, but additionally, it should return 404 Not Found if the parent resource does not exist.
- For example, actions on a URI like cities/1/districts or cities/1/districts/1, will return 404 Not Found if the city with id 1 does not exist.

## 5. Global error handling:
- In Development environment, we can show the detailed exception page on any uncatched exception, by using UseDeveloperExceptionPage() inside the Startup class Configure() method
- In other environments, we can return 500 Server Error with a generic explanation, by using UseExceptionHandler() inside the Startup class Configure() method, we can also handle logging here
- This way, there is no need to use try/catch blocks to catch exceptions inside action methods and return 500 Server Error in each catch block
- Inside UseExceptionHandler(), we can log these global errors (see next section, "Logging with NLog")

## 6. Logging with NLog:
- We add the NLog package, and add NLog in the Startup class Configure() method, using AddNLog()
- We can even use AddNLog() earlier, in Program class of the hosting console application, to log things during the hosting process
- We also configure with the nlog.config file, where and at which level (Debug, Error, Fatal, Info, Trace, Warn, etc.) it will create the logs
- To use the logger in MyController, we inject "ILogger<MyController> logger" in the constructor of MyController, and use logger.LogInformation() or other logger methods to log information
- Inside UseExceptionHandler() in the Startup class Configure() method, we can give a lambda parameter to do things when an error occurs. To log global errors inside this lambda parameter, we can use 

```cs
if (env.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler(appBuilder =>
    {
        appBuilder.Run(async context =>
        {
            var exceptionHandlerFeature = context.Features.Get<IExceptionHandlerFeature>();
            if (exceptionHandlerFeature != null)
            {
                var logger = loggerFactory.CreateLogger("Global exception logger");
                logger.LogError(500,
                    exceptionHandlerFeature.Error,
                    exceptionHandlerFeature.Error.Message);
            }

            context.Response.StatusCode = 500;
            await context.Response.WriteAsync("An unexpected fault happened. Try again later.");

        });                      
    });
}
```
    
## 7. Content negotiation:
- We can add output formatters at the Startup class ConfigureServices() method to support different response (returned output) media types determined by the accept header
- We can return 406 Not Acceptable for an accept header that we do not support, by using ReturnHttpNotAcceptable() at the Startup class ConfigureServices() method
- We can add input formatters at the Startup class ConfigureServices() method to support different request (parameter input) media types determined by the content-type header
- JSON formatters come already added by default, so it supports JSON input and output, unless the formatter is removed

## 8. IOC and dependency injection:
- IOC container and constructor injection is supported by default
- It is advised to use constructor injection, but we can also use HttpContext.RequestServices.GetService() to get an instance from the IOC container
- We add and configure the lifetime of our dependencies in the Startup class ConfigureServices() method
- AddScoped() uses per request lifetime (uses the same instance during an HTTP request)
- AddSingleton() uses application lifetime (always uses the same static instance)
- AddTransient() re-instantiates the dependency each time it is requested (injected), this is recommended for stateless lightweight dependencies

## 9. Configuration files and environment variables:
- We can use the appSettings.json file to store and access configuration information
- We can also scope this file for different environments, by naming the file like appSettings.Production.json
- The scoped file, when running in that environment, overrides the regular appSettings.json file
- When we define an environment variable and assign it a value at the OS level, it will override the setting with the same key in the appSettings.json or the scoped appSettings files
- It is advisable to store secrets such as connection strings or API keys in environment variables, instead of in appSettings files
- We can also define an environment variable and assign it a value in the launchSettings.json file inside Visual Studio, for development purposes only

## 10. Entity Framework Core 2:
- EF Core 2 works similar to EF 6, but with some features missing and some with slight changes
- We can use the same naming convention or data annotations as in EF 6, to define primary keys, foreign keys, child collection, and navigational fields
- We define DbSet table mapping in our DbContext derived custom DB context class, and add it as a scoped dependency using AddDbContext() in Startup class ConfigureServices() method
- We can give the connection string using AddSqlServer() in Startup class ConfigureServices() method
- We can also use manual mapping in OnModelCreating() method of our DbContext based class, and even dismiss using navigational properties, as advised for DDD applications
- See this link: https://stackoverflow.com/questions/20886049/ef-code-first-foreign-key-without-navigation-property
- We can use Database.EnsureCreated() in our DB context constructor to create the DB if it does not exist
- We can also use Database.Migrate() in our DB context constructor to run DB migrations if they exist
- EF Core 2 uses the __EFMigrationHistory table to track which migrations have been applied to the database
- On the package manager console, we can use the Add-Migration to create a new migration class with Up() and Down() methods, and use the Update-Database command to apply pending migrations to the DB
- We can insert seed data in Startup class Configure() method, if we want to do so
- It is advisable to use the repository pattern, with methods returning IEnumerable for collections, instead of directly working with DB context in the action methods

## 11. DTO's and AutoMapper:
- It is advisable to use DTO model classes for API input and output, which are different than the entity model classes, and map data between these classes, either manually or with AutoMapper
- We can use Mapper.Initialize() and also configure mappings using CreateMap() inside the Startup class Configure() method
- Default configuration maps fields with the same name to each other and ignores null values, and is enough for most of the time
- We can use Mapper.Map() to map data from one class to another
