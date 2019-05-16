# aspnet-core-docker-ssl
Example Docker Compose configuration for ASP.NET Core and Let's Encrypt.

Copy _Dockerfile_ and _docker-compose.yml_ to your own project before continuing.

## Configuration

### Docker

Create a file called _.env_ in the same folder as _docker-compose.yml_ and set the required environment variables:

```
# Change these
HOST=example.org,www.example.org
EMAIL=admin@example.org
POSTGRES_PASSWORD=Password1.
```

Make sure to include both the bare domain and the www sub-domain in HOST as shown above.

### Application

Update the last line of _Dockerfile_ with the name of your ASP.NET Core project. The name is case sensitive. 

`ENTRYPOINT ["dotnet", "Example.dll"]`

## Build your application

You should build your application first to make sure everything is working.

`docker-compose build`

This will pull all the build requirements and build your application.

## Start containers

I recommend starting PostgreSQL first as the first startup takes a while.

`docker-compose up -d postgres` 

Wait 5 seconds or so and start the rest of the containers:

`docker-compose up -d`

You can check the logs to see if everything is working:

`docker-compose logs` 

Press Ctrl+C to close logs.

## Redirect to www

If you want your bare domain to redirect to the www sub-domain you must add a line to the configuration file for the bare domain, like this:

```
HOST=example.org # Change this
VOLUME=$(docker volume inspect --format '{{ .Mountpoint }}' aspnet-core-docker-ssl_nginx-vhost)
echo "return 301 \$scheme://www.$HOST\$request_uri;" >> $VOLUME/$HOST
```
