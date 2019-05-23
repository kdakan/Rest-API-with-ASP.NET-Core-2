# BUILDING A REST API WITH ASP.NET CORE 2 & EF CORE 2

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
* [10. Entity Framework Core 2](#10-entity-framework-core-2)
* [11. DTO's and AutoMapper](#11-dtos-and-automapper)
* [12. Async actions](#12-async-actions)
* [13. Paging, Filtering and Sorting Resources](#13-paging-filtering-and-sorting-resources)
* [14. HTTP cache](#14-http-cache)
* [15. HTTP cache expiration and validation](#15-http-cache-expiration-and-validation)
* [16. Example HTTP cache flow](#16-example-http-cache-flow)
* [17. Using HTTP cache and concurrency control](#17-using-http-cache-and-concurrency-control)

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
- We can define a convention based routing in app.UseMVC() or use attribute based routing on controllers or action methods, which is the recommended way for API's (using ApiController attribute on the controller, forces us to use attribute based routing)
- Route attribute works at the controller level, HttpGet, HttpPost, HttpPut, HttpPatch, and HttpDelete attributes work at the action level, and they all accept a string URI parameter to define the routes
- We can put parameters in curly braces inside the route URI, which will also be passed to the action method as parameters
- We should return the correct HTTP status code and payload in an action method

## 3. HTTP status code levels:
- Level 200 status codes mean success, like 200 OK, 201 Created, 204 No Content
- Level 400 status codes mean client error, like 400 Bad Request, 401 Unauthorized (user is not authorized), 403 Forbidden (user is authorized but lacks permission), 404 Not Found, 409 Conflict (conflicting updates)
- Level 500 status codes mean server error, like 500 Internal Server Error

## 4. HTTP actions and proper responses:
The correct HTTP status codes and payloads for a REST API, is listed as follows:
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
- For example, actions on a URI like cities/1/districts or cities/1/districts/1 will return 404 Not Found if the city with id 1 does not exist.

## 5. Global error handling:
- In the Development environment, we can show the detailed exception page on any uncatched exception, by using UseDeveloperExceptionPage() inside the Startup class Configure() method
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
- On the package manager console, we can use the add-migration to create a new migration class with Up() and Down() methods, and use the update-database command to apply pending migrations to the DB
- EF Core 2 uses the __EFMigrationHistory table to track which migrations have been applied to the database
- We can use Database.Migrate() in our DB context constructor to run DB migrations when they exist
- We can insert seed data in Startup class Configure() method, or alternatively use modelBuilder.Entity<T>.HasData() inside OnModelCreating() method of our DbContext based class
- It is advisable to use the repository pattern, with methods returning IEnumerable for collections, instead of directly working with DB context in the action methods

## 11. DTO's and AutoMapper:
- DTO (Data Transfer Object) is the name given for a model class that is specifically designed to transfer data between the clients and the service
- A DTO model doesn't have to be shaped after the entity model, it can lack some fields from the entity model, it can have computed fields, it can even be a summary model that stores data coming from multiple entities
- It is advisable to use DTO model classes for API input and output, which are different than the entity model classes, and map data between these classes, either manually or with AutoMapper
- We can use Mapper.Initialize() to add AutoMapper and also configure the mappings using CreateMap() inside the Startup class Configure() method
- Default configuration for AutoMapper, maps between fields with the same name and ignores missing fields, and is enough for most of the time
- We can use Mapper.Map() to map data from one class to another with AutoMapper

## 12. Async actions:
- Using async/await in IO bound operations (file system, database, network, etc.) scales better, because this way, the thread handling the current request is not blocked during such async operations, is returned to the thread pool, and can be reused for handling other concurrent requests
- It is not advisable to use async/await in CPU bound operations
- async methods are not executed directly, instead, the compiler generates a state machine that begins executing it and than returns back to the caller and recursively back to the top main() method and then back to the thread pool, and then continue execution when the IO bound operation unblocks at the OS level
- Async keyword used in a method declaration, means await operations can be used inside this method
- An async method that does not have any await operation executes sequentially as normal methods do, without the compiler generated state machine
- An async method can return void (not recommended), Task (with no return value), or Task<T> (with return value of type T), state of execution is tracked in the Task object
- in C# 7, an async method can return any class that has a GetAwaiter() method, this allows for value types allowed to be returned from async methods (value types are stored on the stack whereas Task, a reference type, is stored on the heap memory, which needs garbage collection for cleaning up)
- Async methods have names ending with "Async" by convention
- Async keyword is not used inside an interface, it is only used inside a class
- Action methods, and even the main() method can be async
- We can use the async ToListAsync() method on the DbSet, and the async SaveChangesAsync() method on the DbContext, instead of the synchronous ToList() and SaveChanges() methods

## 13. Paging, filtering, and sorting resources:
- Paging and filtering can be supported by using query parameters on top of the regular GET URI for the collection resource, like /people?name=John&pageNumber=2&pageSize=10
- We can use Skip() and Take() methods on the IQueryable interface of the DbSet to get the paged data we need
- We need to be able to sort on fields on DTO's that don't exist on the entity models, like sorting on a computed Name field of the PersonDTO class, instead of FirstName and LastName fields (columns) on the Person entity
- We also need to be able to reverse the order on some fields, like sorting by Age field of a DTO can actually mean sorting by DateOfBirth field (column) of an entity model, but in descending order
- To cover this scenario, we can build a property mapper class which maps a DTO field to multiple entity fields, and optionally reverses the sorting order
- To be able to sort using string field names, we should use the System.Linq.Dynamic.Core package
- Using this package, we can use OrderBy(someString), where the "someString" parameter can hold comma-separated column names and even "ascending" or "descending" appended to the end, to specify the sorting order

## 15. HTTP cache:
Static web pages, images, or static data like definitions, cities, countries, currencies, etc. can be served from an HTTP cache to reduce network traffic or reduce server load on the API. 

There are three types of HTTP cache:
- Client cache is a private cache that lives on the client, like localstorage in the web browser or a private cache in a mobile app
- Gateway cache (or reverse proxy or HTTP accelerator) is a shared cache that lives on the server
- Proxy cache is a shared cache that lives on the network
- There may be all three of them on none in a given system

## 16. HTTP cache expiration and validation:
HTTP cache integrates both expiration and validation to reduce network traffic between clients and the API

HTTP cache expiration:
- Uses the Expires header or the Cache-Control header
- Expires header is not recommended because it uses a timestamp and thus the client and the server clocks should be synchronized, also the client has no control
- Cache-Control header is recommended, it uses max-age to determine how many seconds the API response should be cached, whether the data can be cached in a private or shared cache (with the public and private keywords in the response header), and the client can request data with less max-age data using the same header in the request
- The client can also use no-cache in its Cache-Control request header if it needs to bypass HTTP caching and always hit the API (this is also used when testing an API)
- Similarly, the API can use no-cache in its Cache-Control response header if it doesn't want the response being cached in an HTTP cache

HTTP cache validation:
- Is used by the cache to determine if the cached data is stale (changed), and needs to be updated from the server
- Uses the ETag header to determine if the response body or header has changed
- The Etag value (like a hash value) indicates a specific version of a resource, and thus can be used as an optimistic lock mechanism for concurrent updates

## 17. Example HTTP cache flow:
- When a client requests a URI for the first time, the cache is empty and the API responds with the data and the Cache-Control header having max-age like 1800 seconds (30 minutes), and the Etag header having a value like 12345678
- When the same client (or a different client in the case of a shared/public cache) requests the same URI after 10 minutes, the cache responds with the same data and the Cache-Control header having max-age 1800 seconds (30 minutes) and age 600 seconds (10 minutes), and the Etag header having the same value 12345678, without ever hitting the API
- When a client requests the same URI after an hour, since the cache is already expired, the cache sends the request to the API with the Cache-Control header having If-Non-Match the same value 12345678
- If the resource has not changed at the server, the API responds with the status code 304 Not Modified, with no payload, so that the cache can respond to the client with the same data and with the same ETag header having value 12345678
- When a client requests the same URI again and if the resource has not changed at the server, the same thing will happen
- Only if the resource has changed at the server, the API will serve new data with a new Cache-Control header and a new ETag header, to be cached at the cache again

## 18. Using HTTP cache and concurrency control:
- We can use Marvin.Cache.Headers package to support HTTP cache headers with ETags, and the ASP.NET Core ResponseCaching package in an ASP.NET Core application (CacheCow.Server and CacheCow.Client packages can only be used for older ASP.NET applications, not in ASP.NET Core applications)
- We can use UseHttpCacheHeaders() (before UseMVC()) inside the Startup class Configure() method, and AddHttpCacheHeaders() inside the Startup class ConfigureServices() method to support HTTP cache headers, and also provide options like the max-age seconds inside AddHttpCacheHraders()
- We can use UseResponseCaching() (before UseHttpCacheHeaders()) inside the Startup class Configure() method, and AddResponseCaching() inside the Startup class ConfigureServices() method to support an HTTP cache store, so that our application remembers cached responses and does not serve new data and new headers on each request
- This way, we also have an optimistic locking mechanism, a conflicting update (PUT request with an older version ETag value in its request header) will fail and receive the HTTP status code 412 Precondition Failed response from the API
