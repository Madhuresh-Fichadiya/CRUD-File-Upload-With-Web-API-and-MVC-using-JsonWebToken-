# Complete CRUD with Jwt authentication and File upload

**Prerequisite**: Basic knowledge of .NET Core and WebAPI
# Part 1: Web API project

## Add following code in Program.cs file

```csharp
// Configures JWT authentication for the application
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme) // Sets JWT Bearer as the default authentication scheme
    .AddJwtBearer(options => // Configures JWT Bearer authentication options
    {
        // Disables the requirement for HTTPS metadata (useful for development; enable in production)
        options.RequireHttpsMetadata = false;
        // Saves the JWT token in the request context for later use
        options.SaveToken = true;
        // Defines how the JWT token should be validated
        options.TokenValidationParameters = new TokenValidationParameters
        {
            // Ensures the token's issuer matches the expected issuer
            ValidateIssuer = true,
            // Ensures the token's audience matches the expected audience
            ValidateAudience = true,
            // Ensures the token has not expired
            ValidateLifetime = true,
            // Ensures the token is signed with the correct key
            ValidateIssuerSigningKey = true,
            // Specifies the expected issuer (e.g., your app's domain)
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            // Specifies the expected audience (e.g., your app's clients)
            ValidAudience = builder.Configuration["Jwt:Audience"],
            // Specifies the key used to verify the token's signature
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

builder.Services.AddAuthorization();
```
Add following code after   var app = builder.Build();
```csharp
Enable static file middleware that will be helpful to access `wwwroot` folder.
app.UseStaticFiles();
```
