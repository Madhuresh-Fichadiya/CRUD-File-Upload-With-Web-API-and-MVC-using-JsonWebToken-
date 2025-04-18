# Complete CRUD with Jwt authentication and File upload

**Prerequisite**: Part 1 - Web API
# Part 2: MVC project with Session Management

## Step 1: Add following package(s)
- Newtonsoft.json

## Step 2: Add Authentication service class
Create a Services folder and add AuthService class inside folder
```csharp
public class AuthService
{
    // Declare a private field to hold the HttpClient instance.
    private readonly HttpClient _httpClient;

    // Constructor that accepts an HttpClient instance via dependency injection.
    public AuthService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<string?> AuthenticateUserAsync(string username, string password)
    {
        // Create an anonymous object to hold the login credentials.
        var requestData = new { Username = username, Password = password };

        // Convert the request data to a JSON string and wrap it in a StringContent object with appropriate media type.
        var content = new StringContent(
            JsonConvert.SerializeObject(requestData), // Serialize object to JSON
            Encoding.UTF8,                            // Use UTF-8 encoding
            "application/json"                        // Set content type to application/json
        );

        // Send a POST request to the login endpoint of the API.
        HttpResponseMessage response = await _httpClient.PostAsync("https://localhost:7228/api/Auth/Login", content);

        // Check if the response indicates success (HTTP status code 200-299).
        if (response.IsSuccessStatusCode)
        {
            // Read the response content (expected to be a JWT token) as a string.
            var result = await response.Content.ReadAsStringAsync();
            return result; // Return the token
        }

        // If authentication failed, return null.
        return null;
    }
}
```

## Step 3: Add following in Program.cs

```csharp
// Register AuthService with the built-in HttpClient factory.
// This allows AuthService to receive an HttpClient instance via dependency injection.
builder.Services.AddHttpClient<AuthService>();

// Register IHttpContextAccessor so services can access the current HttpContext.
// Useful for accessing session data, user identity, request information, etc.
builder.Services.AddHttpContextAccessor();

// Enable session support in the application.
// This allows the app to store and retrieve user-specific data (e.g., login state) across multiple requests.
builder.Services.AddSession();
```
Add following code after var app = builder.Build();

//Enable static file middleware that will be helpful to access `wwwroot` folder.
app.UseAuthorization();
app.UseSession();

## Step 4: Add model classes as required
```csharp
public class UserModel
{
    public string? Username { get; set; }
    public string? Password { get; set; }
}
```

## Step 5: Add Custom Action Filter as below
Add CheckAccess class inside Services folder

```csharp
/// <summary>
/// Custom authorization filter to check if a user is authenticated based on JWT token stored in session.
/// If the token is missing, it redirects the user to the login page.
/// </summary>
public class CheckAccess : ActionFilterAttribute, IAuthorizationFilter
{
    /// <summary>
    /// Called early in the request pipeline to check user authorization.
    /// If JWT token is not found in session, user is redirected to login page.
    /// </summary>
    /// <param name="filterContext">Authorization context containing session and HTTP request data.</param>
    public void OnAuthorization(AuthorizationFilterContext filterContext)
    {
        // If JWT token is not found in session, redirect to login page
        if (filterContext.HttpContext.Session.GetString("JWTToken") == null)
        {
            filterContext.Result = new RedirectResult("~/Auth/Login");
        }
    }

    /// <summary>
    /// Called before the result is executed (like rendering a view).
    /// Prevents caching of protected pages once the session (JWT) is cleared (e.g., after logout).
    /// </summary>
    /// <param name="context">Result context to modify HTTP headers.</param>
    public override void OnResultExecuting(ResultExecutingContext context)
    {
        // If JWT token is not found, add headers to prevent browser from caching the previous page
        if (context.HttpContext.Session.GetString("JWTToken") == null)
        {
            context.HttpContext.Response.Headers["Cache-Control"] = "no-cache, no-store, must-revalidate";
            context.HttpContext.Response.Headers["Expires"] = "-1";
            context.HttpContext.Response.Headers["Pragma"] = "no-cache";
        }

        base.OnResultExecuting(context);
    }
}
```

## Step 6: Create CommonVariables class
Create CommonVariables class inside Services folder, to retrieve session data easily.
```csharp
/// <summary>
/// A static helper class used to access session variables like JWT token and UserName from anywhere in the application.
/// </summary>
public static class CommonVariables
{
    // Provides access to the current HttpContext (request-specific data like session, user, etc.)
    private static IHttpContextAccessor _httpContextAccessor;

    // Static constructor initializes the IHttpContextAccessor
    static CommonVariables()
    {
        _httpContextAccessor = new HttpContextAccessor();
    }

    /// <summary>
    /// Retrieves the JWT token from the session, if available.
    /// </summary>
    /// <returns>JWT token as string if exists, otherwise null.</returns>
    public static string? Token()
    {
        string? Token = null;

        // Check if the session contains the "JWTToken" key
        if (_httpContextAccessor.HttpContext?.Session.GetString("JWTToken") != null)
        {
            // If it exists, get the token value from session
            Token = _httpContextAccessor.HttpContext.Session.GetString("JWTToken");
        }

        return Token;
    }

    /// <summary>
    /// Retrieves the user's username from the session, if available.
    /// </summary>
    /// <returns>Username as string if exists, otherwise null.</returns>
    public static string? UserName()
    {
        string? UserName = null;

        // Check if the session contains the "UserName" key
        if (_httpContextAccessor.HttpContext?.Session.GetString("UserName") != null)
        {
            // If it exists, get the username value from session
            UserName = _httpContextAccessor.HttpContext.Session.GetString("UserName");
        }

        return UserName;
    }
}
```

## Step 7: Add Authentication Controller for Login and Logout
```csharp
/// <summary>
    /// Controller responsible for handling login and logout operations.
    /// </summary>
    public class AuthController : Controller
    {
        private readonly AuthService _authService;
        private readonly IHttpContextAccessor _httpContextAccessor;

        // Constructor with dependency injection for AuthService and HttpContextAccessor
        public AuthController(AuthService authService, IHttpContextAccessor httpContextAccessor)
        {
            _authService = authService;
            _httpContextAccessor = httpContextAccessor;
        }

        /// <summary>
        /// Displays the login page (GET request).
        /// </summary>
        [HttpGet]
        public IActionResult Login()
        {
            return View();
        }

        /// <summary>
        /// Handles login form submission (POST request).
        /// Authenticates the user using AuthService and stores the JWT token in session.
        /// </summary>
        /// <param name="Username">The username entered by the user.</param>
        /// <param name="Password">The password entered by the user.</param>
        /// <returns>Redirects to home page on success, otherwise reloads login view with error.</returns>
        [HttpPost]
        public async Task<IActionResult> Login(string Username, string Password)
        {
            // Call the authentication service to get the JWT token
            var jsonData = await _authService.AuthenticateUserAsync(Username, Password);

            // If no data is returned, show error message
            if (jsonData == null)
            {
                ViewBag.Error = "Invalid credentials.";
                return View();
            }

            // Deserialize the JSON response into a dictionary
            var data = JsonConvert.DeserializeObject<Dictionary<string, dynamic>>(jsonData);
            string token = data["token"]; // Extract the token from the response

            // If token is null, show error
            if (token == null)
            {
                ViewBag.Error = "Invalid credentials.";
                return View();
            }

            // Store the token in session for future use (e.g., authorization)
            _httpContextAccessor.HttpContext.Session.SetString("JWTToken", token);

            // Redirect to the home page after successful login
            return RedirectToAction("Index", "Home");
        }

        /// <summary>
        /// Logs the user out by clearing the session and redirecting to login page.
        /// </summary>
        public IActionResult Logout()
        {
            // Clear all session data including the JWT token
            _httpContextAccessor.HttpContext.Session.Clear();

            // Redirect the user to the login page
            return RedirectToAction("Login");
        }
    }
```
## Step 8: Add View Pages as required
Here we have added Login Page
```csharp
@model UserModel;

<h2>Login</h2>

@if (ViewBag.Error != null)
{
    <div class="alert alert-danger">@ViewBag.Error</div>
}

<form method="post" class="col-4">
    <div class="form-group">
        <label>Username</label>
        <input type="text" class="form-control" name="Username" required />
    </div>
    <div class="form-group">
        <label>Password</label>
        <input type="password" class="form-control" name="Password" required />
    </div><br/>
    <div class="form-group">
        <button type="submit" class="btn btn-primary">Login</button>
    </div>
</form>
```
