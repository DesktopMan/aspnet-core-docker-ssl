# Usage

## Copy files

Copy _Dockerfile_, _docker-compose.yml_ and _.dockerignore_ to the root of your ASP.NET Core project.

## Install packages

```
dotnet tool install --global dotnet-ef
dotnet tool install --global dotnet-aspnet-codegenerator
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

# Configuration

## ASP.NET Core

### Add connection strings

You can set up ASP.NET Core so that it uses SQLite during development and PostgreSQL in production. You do this by adding two different connection strings. You only need to add the connection string, leave the rest of the config file as it is.

In _appsettings.Development.json_:

```
{
  "ConnectionStrings": {
    "DefaultConnection": "Filename=Example.db"
  },
  "Logging": {
    ...
  }
}
```

In _appsettings.json_:

```
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=postgres;Port=5432;Username=postgres;Password=Password1.;Database=postgres;"
  },
  "Logging": {
    ...
  },
  "AllowedHosts": "*"
}
```

### Modify Program.cs

The database initializer should behave different in development vs production, and because of this it needs to know what environment we are in:

```
var host = CreateHostBuilder(args).Build();

using (var scope = host.Services.CreateScope())
{
    var services = scope.ServiceProvider;

    // Get our database context from the service provider
    var context = services.GetRequiredService<ExampleDbContext>();

    // Get the environment so we can check if this is running in development or otherwise
    var environment = services.GetService<IHostEnvironment>();

    // Initialise the database using the initializer from Data/ExampleDbInitializer.cs
    DbInitializer.Initialize(context, environment.IsDevelopment());
}

host.Run();
```

### Modify Startup.cs

The _Startup_ class needs to be updated so it also receives the environment in its constructor:

```
// The second parameter has been added so we can get the current environment in ConfigureServices
public Startup(IConfiguration configuration, IHostEnvironment environment)
{
    Configuration = configuration;
    Environment = environment;
}

public IConfiguration Configuration { get; }
public IHostEnvironment Environment { get; }
```

_ConfigureServices()_ needs to be updated to use the appropriate connection string for each environment:

```
if (Environment.IsDevelopment()) // Database used during development
{
    // Register the database context as a service. Use the SQLite for this
    services.AddDbContext<ExampleDbContext>(options =>
        options.UseSqlite(Configuration.GetConnectionString("DefaultConnection")));
}
else // Database used in all other environments (production etc)
{
    // Register the database context as a service. Use PostgreSQL server for this
    services.AddDbContext<ExampleDbContext>(options =>
        options.UseNpgsql(Configuration.GetConnectionString("DefaultConnection")));
}
```

### Add the database initializer

The database initializer will receive the current environment from the change in _Program.cs_, and decide what to do based on that:

```
public static class DbInitializer
{
    public static void Initialize(ExampleDbContext context, bool development)
    {
        // Run migrations if we're not in development mode
        if (!development)
        {
            context.Database.Migrate();
            return;
        }

        // If we are in development mode the code below is run.

        // Delete the database before we initialize it.
        // This is common to do during development.
        context.Database.EnsureDeleted();

        // Make sure the database and tables exist
        context.Database.EnsureCreated();

        // You can now add test data here
    }
}
```

### Create your database and schema

Now make sure you have the lastes migrations files of your database schema.

```
dotnet ef migrations add InitialCreate
```

## Docker Compose

Create a file called _.env_ in the same directory as _docker-compose.yml_ and set the required environment variables:

```
# Change these
HOST=example.org,www.example.org
EMAIL=admin@example.org
POSTGRES_PASSWORD=Password1.
```

Make sure to include both the bare domain and the www sub-domain in HOST as shown above.

## Dockerfile

Update the last line of _Dockerfile_ with the name of your ASP.NET Core project. The name is case sensitive.

`ENTRYPOINT ["dotnet", "Example.dll"]`

# Build your application

You should build your application first to make sure everything is working.

`docker-compose build`

This will pull all the build requirements and build your application.

# Start containers

## 1. PostgreSQL database

I recommend starting PostgreSQL first as the first startup takes a while.

`docker-compose up -d postgres`

## 2. Nginx proxy

First start the container:

`docker-compose up -d nginx-proxy`

Then configure the www-forwarding and restart it:

```
HOST=example.org # Change this
VOLUME=$(docker volume inspect --format '{{ .Mountpoint }}' $(basename $PWD)_nginx-vhost)
echo "return 301 \$scheme://www.$HOST\$request_uri;" >> $VOLUME/$HOST
docker-compose restart nginx-proxy
```

## 3. Start the rest

`docker-compose up -d`

## Checking for errors

You can check the logs to see if everything is working:

`docker-compose logs`

Press Ctrl+C to close logs.

---

Every time you make changes to your models that may change the database scheme, you will have to migrate.

```
# Delete old migrations files
rm -rf Migrations

# Create new migration
dotnet ef migrations add InitialCreate

# Stop and remove application and the database server
docker-compose rm -fs application postgres

# Remove the existing database volume
docker volume ls # Find the name / ID it should end with postgres-data
docker volume rm <name or id>

# Restart the database server (now with empty database)
docker-compose up -d postgres

# Build and start application with the new migration
docker-compose build
docker-compose up -d application
```

For more about migrations see https://docs.microsoft.com/en-us/ef/core/managing-schemas/migrations/?tabs=dotnet-core-cli
