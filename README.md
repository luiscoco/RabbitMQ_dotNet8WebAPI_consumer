# How to create a .NET8 WebAPI for consuming messages from RabbitMQ

## 1. Pull and run the RabbitMQ Docker container

Install and run **Docker Desktop** on your machine, if you haven't already

You can download Docker from the following link: https://www.docker.com/products/docker-desktop

Start **RabbitMQ** in a Docker container by running the following command:

```
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_producer/assets/32194879/25e200c7-7309-4380-97c3-27d269474e6e)

The **password** is also **guest**

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_producer/assets/32194879/8f3e7ac0-fb1b-4f5f-9229-189d8d611fbd)

## 2. Create a .NET8 WebAPI with VSCode

To create the project, you can use the **dotnet new** command to create a new **Web API project**

```
dotnet new webapi -o RabbitMQWebAPI
```

## 3. Load project dependencies

First, you will need to install the **RabbitMQ.Client** NuGet package in your project

```
dotnet add package RabbitMQ.Client
```

This is the csproj file

```csproj
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.2" />
    <PackageReference Include="RabbitMQ.Client" Version="6.8.1" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.4.0" />
  </ItemGroup>

</Project>
```

This is the project files and folders structure

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_consumer/assets/32194879/e2647f65-3c78-474a-b428-65cd8d33e418)

## 4. Create the Controller

```csharp
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using Microsoft.AspNetCore.Mvc;
using System.Text;

namespace RabbitMQWebAPIConsumer.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class ValuesController : ControllerBase
    {
        private readonly IConnection _connection;

        public ValuesController(IConnection connection)
        {
            _connection = connection;
        }

        [HttpGet]
        public void Get()
        {
            using var channel = _connection.CreateModel();
            channel.QueueDeclare(queue: "hello",
                                 durable: false,
                                 exclusive: false,
                                 autoDelete: false,
                                 arguments: null);

            var consumer = new EventingBasicConsumer(channel);
            consumer.Received += (model, ea) =>
            {
                var body = ea.Body.ToArray();
                var message = Encoding.UTF8.GetString(body);
                Console.WriteLine(" [x] Received {0}", message);
            };
            channel.BasicConsume(queue: "hello",
                                 autoAck: true,
                                 consumer: consumer);
        }
    }
}
```

## 5 . Modify the application middleware(program.cs)

```csharp
using RabbitMQ.Client;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var factory = new ConnectionFactory()
{
    HostName = "localhost",
    UserName = "guest",
    Password = "guest"
};
builder.Services.AddSingleton(factory.CreateConnection());

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseRouting();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

## 6. Run and test the application

We have first to run the producer application and send a messate to RabbitMQ queue "hello"

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_consumer/assets/32194879/4b5d32c1-ae0c-4413-862f-4fd38224d122)

We run the application with this command

```
dotnet run
```

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_consumer/assets/32194879/dcc8279f-827b-45d4-b50b-b32cab6586b8)

We navigate to the application swagger API doc URL: http://localhost:5044/swagger/index.html

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_consumer/assets/32194879/e0ee4b7e-6d90-4a6d-a232-46bc9924a996)

We received a message from RabbitMQ queue "hello"

![image](https://github.com/luiscoco/RabbitMQ_dotNet8WebAPI_consumer/assets/32194879/cb170ee9-dfe2-45a9-9380-ff41780d29e0)

