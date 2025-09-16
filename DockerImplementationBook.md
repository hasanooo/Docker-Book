#At first write Command on cmd to Run SQL Server in Docker

##For pull MSSQL from Dockerhub To Desktop Docker and Username and Password

docker pull mcr.microsoft.com/mssql/server:2022-latest

docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrong!Passw0rd" \
   -p 1433:1433 --name sqlserver2022 \
   -v sqlvolume:/var/opt/mssql \
   -d mcr.microsoft.com/mssql/server:2022-latest


#Test SQL Server Connection

docker exec -it sqlserver2022 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P YourStrong!Passw0rd

#Create ASP.NET Core Web API Project
dotnet new webapi -n DockerSqlDemo
cd DockerSqlDemo
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools

#Add DbContext & Model
namespace DockerSqlDemo.Models
{
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
    }
}
##Data/AppDbContext.cs:

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

#Configure Connection String
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

#Configure DbContext in Program.cs

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


#Create Sample Controller

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

#Apply Migrations

dotnet ef migrations add InitialCreate
dotnet ef database update

#Test API
##Run the app:

dotnet run

#Optional Enhancements:

##Use Docker Compose to run SQL Server + your API together.

##Seed database automatically using a .sql file.

##Set TrustServerCertificate=True in dev; for prod use proper certificates.
