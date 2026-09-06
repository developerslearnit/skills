# YARP API Gateway Reference

The API Gateway acts as the single entry point for all client traffic (Web, Mobile, Third-party Webhooks). It uses **YARP (Yet Another Reverse Proxy)** to route HTTP requests to internal microservices, enforce rate limiting, handle CORS, forward authorization tokens, and aggregate Swagger/OpenAPI documentation.

---

## 1. Project Directory Layout

```text
SolutionName.ApiGateway/
├── Properties/
│   └── launchSettings.json
├── Configuration/
│   ├── SwaggerAggregationExtensions.cs
│   └── RateLimitingExtensions.cs
├── appsettings.json
├── appsettings.Development.json
├── Dockerfile
├── Program.cs
└── SolutionName.ApiGateway.csproj
```

---

## 2. Gateway `Program.cs`

```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;
using Yarp.ReverseProxy.Configuration;

var builder = WebApplication.CreateBuilder(args);

// 1. Add YARP Reverse Proxy from Configuration
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// 2. Add Rate Limiting Policies
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    
    // Fixed Window Policy for General APIs
    options.AddFixedWindowLimiter("fixed-policy", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromSeconds(60);
        opt.QueueLimit = 10;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });

    // Token Bucket for High-Traffic Ingestion
    options.AddTokenBucketLimiter("token-bucket-policy", opt =>
    {
        opt.TokenLimit = 200;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opt.TokensPerPeriod = 50;
        opt.AutoReplenishment = true;
    });
});

// 3. Add CORS Policies
builder.Services.AddCors(options =>
{
    options.AddPolicy("GatewayCorsPolicy", policy =>
    {
        policy.WithOrigins(builder.Configuration.GetSection("AllowedOrigins").Get<string[]>() ?? ["http://localhost:3000"])
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

// 4. Health Checks
builder.Services.AddHealthChecks();

// 5. OpenAPI / Swagger Aggregation
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "API Gateway v1");
        options.SwaggerEndpoint("/api/v1/tickets/swagger/v1/swagger.json", "Tickets Service");
        options.SwaggerEndpoint("/api/v1/customers/swagger/v1/swagger.json", "Customers Service");
        options.SwaggerEndpoint("/api/v1/knowledge/swagger/v1/swagger.json", "Knowledge Base Service");
        options.RoutePrefix = "swagger";
    });
}

app.UseCors("GatewayCorsPolicy");
app.UseRateLimiter();

app.MapHealthChecks("/health");

// Map YARP Reverse Proxy Pipeline
app.MapReverseProxy(proxyPipeline =>
{
    // Custom transforms or telemetry can be added here
});

app.Run();
```

---

## 3. `appsettings.json` Configuration

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Yarp": "Information"
    }
  },
  "AllowedOrigins": [
    "http://localhost:3000",
    "https://app.yourdomain.com"
  ],
  "ReverseProxy": {
    "Routes": {
      "tickets-route": {
        "ClusterId": "tickets-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": {
          "Path": "/api/v1/tickets/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/v1/tickets/{**catch-all}" },
          { "RequestHeader": "X-Forwarded-Gateway", "Set": "YarpGateway" }
        ]
      },
      "customers-route": {
        "ClusterId": "customers-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": {
          "Path": "/api/v1/customers/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/v1/customers/{**catch-all}" }
        ]
      },
      "knowledge-route": {
        "ClusterId": "knowledge-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": {
          "Path": "/api/v1/knowledge/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/v1/knowledge/{**catch-all}" }
        ]
      },
      "identity-route": {
        "ClusterId": "identity-cluster",
        "RateLimiterPolicy": "fixed-policy",
        "Match": {
          "Path": "/api/v1/identity/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "/api/v1/identity/{**catch-all}" }
        ]
      }
    },
    "Clusters": {
      "tickets-cluster": {
        "Destinations": {
          "tickets-service-1": {
            "Address": "http://tickets-service:8080"
          }
        },
        "HttpRequest": {
          "Timeout": "00:00:30"
        }
      },
      "customers-cluster": {
        "Destinations": {
          "customers-service-1": {
            "Address": "http://customers-service:8080"
          }
        },
        "HttpRequest": {
          "Timeout": "00:00:30"
        }
      },
      "knowledge-cluster": {
        "Destinations": {
          "knowledge-service-1": {
            "Address": "http://knowledge-service:8080"
          }
        },
        "HttpRequest": {
          "Timeout": "00:00:45"
        }
      },
      "identity-cluster": {
        "Destinations": {
          "identity-service-1": {
            "Address": "http://identity-service:8080"
          }
        },
        "HttpRequest": {
          "Timeout": "00:00:30"
        }
      }
    }
  }
}
```

---

## 4. Local Development Port Mapping

When running locally without Docker:
* `ApiGateway`: `http://localhost:5000` (Clients call this exclusively)
* `Identity.Api`: `http://localhost:5001`
* `Tickets.Api`: `http://localhost:5002`
* `Customers.Api`: `http://localhost:5003`
* `Knowledge.Api`: `http://localhost:5004`

When running under Docker Compose:
* The Gateway listens on port `8080` / `5000` on the host, while internal microservices communicate across the private Docker network (`http://tickets-service:8080`, `http://customers-service:8080`, etc.).
