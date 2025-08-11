
## Dockerizing Weather with Docker Compose
We will now containerize a simple default weather forecast API and a client for it, and run them together using Docker Compose. 

### Prerequisites
1. Docker Desktop must be installed and running.
2. VSCode Docker Extension is recommended.
3. Your project folder structure should look like this:

```
/DockerCompose
├── API/
│   └── API.csproj
├── Client/
│   └── Client.csproj
├── docker-compose.yml
└── .gitignore
```

### Step 1: Create Dockerfiles
In both the `API/` and `Client/` folders, create a `Dockerfile` with no file extension.

Paste the following into each:

**For API/Dockerfile**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "API.dll"]
```

**For Client/Dockerfile**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "Client.dll"]
```

### Step 2: Create the `docker-compose.yml` file

At the root (next to API and Client), create a file named `docker-compose.yml`:

```yaml
version: '3.8'

services:
  api:
    build:
      context: ./API
      dockerfile: Dockerfile
    container_name: api
    environment:
      ASPNETCORE_ENVIRONMENT: Development
    ports:
      - "5000:80"
    networks:
      - net

  client:
    build:
      context: ./Client
      dockerfile: Dockerfile
    container_name: client
    depends_on:
      - api
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      API_BASE_URL: http://api
    ports:
      - "5001:80"
    networks:
      - net

networks:
  net:
```

### Step 3: Fix Program.cs if needed

In both API and Client, **remove `UseHttpsRedirection()`** to avoid HTTPS conflicts in containers.

**Optional (recommended):** Add this line in `API/Program.cs` before `app.Run();`:
```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(80);
});
```
### Step 4: Build and Run

From your root folder (where `docker-compose.yml` is):

```bash
docker compose build
docker compose up
```

To stop containers:

```bash
docker compose down
```

Step 5: Test

- `http://localhost:5000/weatherforecast` → API response
- `http://localhost:5001/` → Your client consuming the API


### Troubleshooting Tips

| Problem | Solution |
|--------|----------|
| `ERR_EMPTY_RESPONSE` | Remove `UseHttpsRedirection()`. Make sure your app binds to port 80 |
| API not reachable | Check if the container is running using `docker ps`. Look at logs with `docker compose logs mathapi` |
| Client can’t call API | Ensure `API_BASE_URL=http://mathapi` is set in Docker Compose |

---

To deploy your Dockerized app to Render, follow these additional steps after you've set up your Docker Compose project as per the instructions above:

### Step 6: Prepare for Deployment on Render
**Create a Render Account**:

   * Sign up or log in to Render - would reccommend signing up with GitHub ([https://render.com](https://render.com)).

**Create a New Web Service on Render**:

   * On your Render dashboard, click **New** and choose **Web Service**.
   * Connect your GitHub repository where the Dockerized project is hosted.
   * Set the environment to **Docker** for a custom Docker build.

**Configure Dockerfile for Render**:

   * Render automatically detects the `Dockerfile` in your repository. Ensure that both the API and Client Dockerfiles are included in the repository.

**Set Docker Build Command (optional)**:
   If Render doesn't automatically detect the Docker Compose configuration, you can add a custom **Build Command**:

   ```bash
   docker-compose build
   ```

   This will instruct Render to use Docker Compose to build your containers.

**Set the Start Command**:
   In the **Start Command** section, you can specify:

   ```bash
   docker-compose up
   ```

**Environment Variables Setup**:

   * In Render's Web Service setup, add necessary environment variables like:

     * `API_BASE_URL=http://api`
     * `ASPNETCORE_ENVIRONMENT=Production`

### Step 7: Push Your Code to GitHub

Make sure your project is pushed to GitHub (or another Git repository provider). If you haven’t pushed the code yet:

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin <YOUR_REPOSITORY_URL>
git push -u origin main
```

### Step 8: Link Your GitHub Repository to Render

In Render, follow the prompts to **connect your GitHub repository** to your Render project.

* Choose the **main branch** for deployment.
* Set the environment to **Docker** and ensure Docker Compose is used for building and running your containers.

### Step 9: Deploy the Application

Once everything is connected and the environment is set:

* Render will automatically build the Docker containers.
* It will then deploy both the API and the Client on separate subdomains (or ports).

### Step 10: Access Your Deployed Application

After deployment, Render will provide URLs for both the API and Client.

* **API URL**: `https://your-api-name.onrender.com` (or similar)
* **Client URL**: `https://your-client-name.onrender.com` (or similar)

Test your deployed services:

* **API**: Visit `https://your-api-name.onrender.com/weatherforecast` to ensure the API works.
* **Client**: Visit `https://your-client-name.onrender.com` to see the client consuming the API.

### Step 11: Monitor and Debug

* **Logs**: Use Render's logging system to monitor your app's logs and debug if necessary.
* **Scaling**: If required, scale your service by upgrading or adding more instances from the Render dashboard.

---

### Additional Tips:

1. **Database Connection**:

   * If you want to use a database like PostgreSQL or MySQL with Docker on Render, add a database service to your Docker Compose file and set up the necessary environment variables for database credentials in Render.

2. **Persistent Storage**:

   * For any data that needs to persist, consider using Render’s managed databases or configuring Docker volumes for persistent storage.


### Next Steps

1. Add SQL Server as a third service in `docker-compose.yml`.
2. Update your connection string using `host.docker.internal`.
3. Add volume mounting and bind keys if using session-based features.
4. Dockerize **MathAPI** and ***MathAPIClient**
