# Running SQL Server in Docker and Building an ASP.NET Core Web API

This guide walks you through setting up a Microsoft SQL Server in Docker, creating an ASP.NET Core Web API, connecting it to the database, and testing the setup. It also includes optional enhancements like using Docker Compose and seeding the database.

---

## Prerequisites
- Docker Desktop installed
- .NET SDK installed (version 8.0 or later recommended)
- Basic knowledge of Docker, ASP.NET Core, and Entity Framework Core

---

## Step 1: Run SQL Server in Docker

### Pull MSSQL Server Image from Docker Hub
```bash
docker pull mcr.microsoft.com/mssql/server:2022-latest
```

### Run SQL Server Container
Run the following command to start a SQL Server instance in Docker. Replace `YourStrong!Passw0rd` with a secure password that meets SQL Server's requirements (at least 8 characters, including uppercase, lowercase, numbers, and symbols).

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrong!Passw0rd" \
   -p 1433:1433 --name sqlserver2022 \
   -v sqlvolume:/var/opt/mssql \
   -d mcr.microsoft.com/mssql/server:2022-latest
```

- `-e "ACCEPT_EULA=Y"`: Accepts the SQL Server End-User License Agreement.
- `-e "MSSQL_SA_PASSWORD=YourStrong!Passw0rd"`: Sets the SA user password.
- `-p 1433:1433`: Maps port 1433 on the host to the container.
- `--name sqlserver2022`: Names the container.
- `-v sqlvolume:/var/opt/mssql`: Persists SQL Server data using a Docker volume.
- `-d`: Runs the container in detached mode.

### Test SQL Server Connection
Verify the SQL Server is running by connecting to it using `sqlcmd`:

```bash
docker exec -it sqlserver2022 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P YourStrong!Passw0rd
```

- Inside the `sqlcmd` prompt, you can run SQL queries (e.g., `SELECT @@VERSION;` to check the SQL Server version).
- Exit the prompt with `QUIT`.

---

## Step 2: Create ASP.NET Core Web API Project

### Create the Project
Run the following commands to create a new ASP.NET Core Web API project and add required Entity Framework Core packages:

```bash
dotnet new webapi -n DockerSqlDemo
cd DockerSqlDemo
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

---

## Step 3: Define Model and DbContext

### Create User Model
Create a `Models` folder and add a `User.cs` file with the following content:

```csharp
namespace DockerSqlDemo.Models
{
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
    }
}
```

### Create DbContext
Create a `Data` folder and add an `AppDbContext.cs` file with the following content:

```csharp
using Microsoft.EntityFrameworkCore;
using DockerSqlDemo.Models;

namespace DockerSqlDemo.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

        public DbSet<User> Users { get; set; }
    }
}
```

---

## Step 4: Configure Connection String

Update the `appsettings.json` file to include the SQL Server connection string:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=TestDb;User Id=sa;Password=YourStrong!Passw0rd;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

- `TrustServerCertificate=True`: Bypasses certificate validation for development. Use proper certificates in production.

---

## Step 5: Configure DbContext in Program.cs

Update the `Program.cs` file to configure the `AppDbContext` and set up the Web API:

```csharp
using DockerSqlDemo.Data;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

## Step 6: Create a Sample Controller

Create a `Controllers` folder (if not already present) and add a `UsersController.cs` file with the following content:

```csharp
using DockerSqlDemo.Data;
using DockerSqlDemo.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace DockerSqlDemo.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class UsersController : ControllerBase
    {
        private readonly AppDbContext _context;
        public UsersController(AppDbContext context)
        {
            _context = context;
        }

        [HttpGet]
        public async Task<IActionResult> GetUsers()
        {
            var users = await _context.Users.ToListAsync();
            return Ok(users);
        }

        [HttpPost]
        public async Task<IActionResult> AddUser(User user)
        {
            _context.Users.Add(user);
            await _context.SaveChangesAsync();
            return Ok(user);
        }
    }
}
```

---

## Step 7: Apply Migrations

Run the following commands to create and apply database migrations:

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

- `InitialCreate`: Creates the initial migration to set up the `Users` table.
- `database update`: Applies the migration to create the `TestDb` database and `Users` table in SQL Server.

---

## Step 8: Run and Test the API

### Run the Application
```bash
dotnet run
```

- The API will be available at `https://localhost:5001` or `http://localhost:5000` (check the terminal output for the exact URL).
- Access the Swagger UI at `/swagger` (e.g., `https://localhost:5001/swagger`) to test the API endpoints.

### Test the API
- **GET** `api/users`: Retrieves all users.
- **POST** `api/users`: Adds a new user. Example payload:
  ```json
  {
    "name": "John Doe",
    "email": "john@example.com"
  }
  ```

---

## Optional Enhancements

### 1. Use Docker Compose
To run the SQL Server and the ASP.NET Core Web API together, create a `docker-compose.yml` file:

```yaml
version: '3.8'
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - MSSQL_SA_PASSWORD=YourStrong!Passw0rd
    ports:
      - "1433:1433"
    volumes:
      - sqlvolume:/var/opt/mssql
    networks:
      - app-network

  webapi:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:80"
    depends_on:
      - sqlserver
    environment:
      - ConnectionStrings__DefaultConnection=Server=sqlserver,1433;Database=TestDb;User Id=sa;Password=YourStrong!Passw0rd;TrustServerCertificate=True;
    networks:
      - app-network

volumes:
  sqlvolume:

networks:
  app-network:
    driver: bridge
```

Create a `Dockerfile` in the project root:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["DockerSqlDemo.csproj", "./"]
RUN dotnet restore "DockerSqlDemo.csproj"
COPY . .
WORKDIR "/src"
RUN dotnet build "DockerSqlDemo.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "DockerSqlDemo.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "DockerSqlDemo.dll"]
```

Run the application using Docker Compose:

```bash
docker-compose up --build
```

### 2. Seed Database Automatically
Create a `.sql` file (e.g., `seed-data.sql`) to seed the database:

```sql
INSERT INTO Users (Name, Email) VALUES ('Alice', 'alice@example.com');
INSERT INTO Users (Name, Email) VALUES ('Bob', 'bob@example.com');
```

Execute the SQL file in the SQL Server container:

```bash
docker exec -i sqlserver2022 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P YourStrong!Passw0rd -d TestDb < seed-data.sql
```

Alternatively, modify `AppDbContext` to seed data programmatically by overriding `OnModelCreating`:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>().HasData(
        new User { Id = 1, Name = "Alice", Email = "alice@example.com" },
        new User { Id = 2, Name = "Bob", Email = "bob@example.com" }
    );
}
```

Re-run migrations and update the database:

```bash
dotnet ef migrations add SeedData
dotnet ef database update
```

### 3. Security Note
- In development, `TrustServerCertificate=True` is used to bypass certificate validation. For production, configure proper SSL certificates for secure connections to SQL Server.

---

## Troubleshooting
- **SQL Server Connection Issues**: Ensure the container is running (`docker ps`) and the password meets complexity requirements.
- **Migration Errors**: Verify the connection string in `appsettings.json` and ensure the SQL Server container is accessible.
- **API Not Responding**: Check the port mappings and ensure the API is running (`dotnet run` or `docker-compose up`).

---

## Next Steps
- Add authentication/authorization to the API.
- Implement additional CRUD operations in `UsersController`.
- Deploy the application to a cloud provider (e.g., Azure, AWS).