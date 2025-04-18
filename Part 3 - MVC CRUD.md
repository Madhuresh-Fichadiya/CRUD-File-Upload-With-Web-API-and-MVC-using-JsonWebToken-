# Complete CRUD with Jwt authentication and File upload

**Prerequisite**: Prerequisite: Part 1 - Web API, Part 2 - MVC Session Management
# Part 3: MVC CRUD Operation

## Step 1: Create Model Class

```csharp
public class Student
{
    public int StudentID { get; set; }
    public string Name { get; set; }
    public string? FilePath { get; set; }
    public IFormFile? FormFile { get; set; }
    //public byte[] FileData { get; set; }
}
```
## Step 2: Add Controller
```csharp
[CheckAccess] // Custom filter to check if the user is authenticated via session token
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly HttpClient _httpClient;

    // Fetches the JWT token from the session using CommonVariables
    string token = CommonVariables.Token();

    public HomeController(ILogger<HomeController> logger, IHttpContextAccessor httpContextAccessor, HttpClient httpClient)
    {
        _logger = logger;
        _httpContextAccessor = httpContextAccessor;
        _httpClient = httpClient;
    }

    // Displays list of students retrieved from API
    public async Task<IActionResult> Index()
    {
        // Prepare request with token in the Authorization header
        var request = new HttpRequestMessage(HttpMethod.Get, $"https://localhost:7228/api/Home/Index");
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

        var response = await _httpClient.SendAsync(request);
        if (response.IsSuccessStatusCode)
        {
            var result = response.Content.ReadAsStringAsync().Result;

            // Deserialize JSON result into a list of Student objects
            var students = JsonConvert.DeserializeObject<List<Student>>(result);

            // Pass data to the view
            return View("Index", students);
        }
        else
        {
            TempData["ErrorMessage"] = "Unable to fetch customer data. Please try again later.";
        }

        // If error, redirect to Dashboard view
        return View("Dashboard");
    }

    // Displays the Add/Edit student form
    [HttpGet]
    public async Task<IActionResult> AddStudent(int? StudentID)
    {
        // If StudentID is present, fetch student details from API
        if (StudentID != null && StudentID > 0)
        {
            Student student = new Student();
            var request = new HttpRequestMessage(HttpMethod.Get, $"https://localhost:7228/api/Home/GetStudentByID?studentID={StudentID}");
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

            var response = await _httpClient.SendAsync(request);
            if (response.IsSuccessStatusCode)
            {
                var result = response.Content.ReadAsStringAsync().Result;

                // Deserialize the result into a Student object
                student = JsonConvert.DeserializeObject<Student>(result);

                return View("AddStudent", student);
            }
        }

        // If new student or fetch failed, return empty form
        return View("AddStudent");
    }

    // Deletes a student by ID
    [HttpPost]
    public async Task<IActionResult> Delete(int StudentID)
    {
        if (StudentID > 0)
        {
            // Send DELETE request to API
            var request = new HttpRequestMessage(HttpMethod.Delete, $"https://localhost:7228/api/Home/DeleteByStudentID?studentID={StudentID}");
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

            var response = await _httpClient.SendAsync(request);
            if (response.IsSuccessStatusCode)
            {
                TempData["SuccessMessage"] = "Record Deleted";
            }
            else
            {
                TempData["ErrorMessage"] = "Error Occured";
            }
        }

        // Redirect back to the Index page after deletion
        return RedirectToAction("Index");
    }

    // Saves or updates student record
    [HttpPost]
    public async Task<IActionResult> Save(Student student)
    {
        // Convert student model to multipart form-data for API post
        var formData = ConvertToMultipartFormData(student);

        // Prepare POST request with JWT token
        var request = new HttpRequestMessage(HttpMethod.Post, $"https://localhost:7228/api/Home/Save");
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        request.Content = formData;

        var response = await _httpClient.SendAsync(request);
        Console.Write(response);

        if (response.IsSuccessStatusCode)
        {
            // If it's an update (existing student)
            if (student.StudentID > 0)
            {
                TempData["SuccessMessage"] = "Record Updated Successfully";
                return RedirectToAction("Index");
            }
            else
            {
                // If it's a new student
                TempData["SuccessMessage"] = "Record Saved Successfully";
                return RedirectToAction("AddStudent");
            }
        }
        else
        {
            TempData["ErrorMessage"] = "Error Occured";

            // Reload the form with current student data
            return View("AddStudent", student);
        }
    }

    // Converts the Student model object to multipart form-data for file upload via API
    public MultipartFormDataContent ConvertToMultipartFormData(Student student)
    {
        var formData = new MultipartFormDataContent();

        // Add basic properties to form-data
        formData.Add(new StringContent(student.StudentID.ToString() ?? ""), "StudentID");
        formData.Add(new StringContent(student.Name ?? ""), "Name");
        formData.Add(new StringContent(student.FilePath ?? ""), "FilePath");

        // Add uploaded file if available
        if (student.FormFile != null && student.FormFile.Length > 0)
        {
            var streamContent = new StreamContent(student.FormFile.OpenReadStream());
            streamContent.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue(student.FormFile.ContentType);

            formData.Add(streamContent, "File", student.FormFile.FileName);
        }

        return formData;
    }
}
```
## Step 3: Add _Layout design page

```csharp
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - JwtAuthMVC</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" />
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/">JwtAuthMVC</a>
            <div class="collapse navbar-collapse">
                <ul class="navbar-nav ms-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="/Home/Index">Student List</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="/Auth/Logout">Logout</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        @RenderBody()
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

## Step 4: Add List Page (Index.cshtml) Design
```csharp
@{
    ViewData["Title"] = "Home Page";
}
@model List<Student>

<div class="text-center">
    <h1 class="display-4">Student List</h1>
    @if (TempData["ErrorMessage"] != null)
    {
        <div class="alert alert-danger">@TempData["ErrorMessage"]</div>
    }

    @if (TempData["SuccessMessage"] != null)
    {
        <div class="alert alert-success">@TempData["SuccessMessage"]</div>
    }
    <a asp-action="AddStudent" class="btn btn-success">Add Student</a>
    <div class="container mt-4">
        <h3 class="mb-3">Student List</h3>
        <table class="table table-bordered table-hover align-middle">
            <thead class="table-dark">
                <tr>
                    <th scope="col">Student ID</th>
                    <th scope="col">Name</th>
                    <th scope="col">Image</th>
                    <th colspan="2"> Action</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var student in Model)
                {
                    <tr>
                        <td>@student.StudentID</td>
                        <td>@student.Name</td>
                        <td>
                            <img src="@student.FilePath" alt="Student Image" class="img-thumbnail" style="max-height: 100px;">
                        </td>
                        <td><a asp-action="AddStudent" class="btn btn-primary" asp-route-studentID="@student.StudentID">Edit</a></td>
                        <td>
                            <form asp-action="Delete" method="post">
                                <input type="hidden" name="StudentID" value="@student.StudentID" />
                                <input type="submit" class="btn btn-danger" value="Delete"/>
                            </form>
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>
```

## Step 4: Add Add/Edit Page (StudentAddEdit.cshtml) Design
```csharp
@model Student
@if (TempData["ErrorMessage"] != null)
{
    <div class="alert alert-danger">@TempData["ErrorMessage"]</div>
}

@if (TempData["SuccessMessage"] != null)
{
    <div class="alert alert-success">@TempData["SuccessMessage"]</div>
}

<form asp-action="Save" asp-controller="Home" enctype="multipart/form-data">
    <input type="hidden" asp-for="FilePath" />
    <div class="form-group">
        <label for="exampleInputEmail1">Student ID</label>
        <input type="text" class="form-control" asp-for="StudentID" placeholder="Enter Student ID">
        <small id="emailHelp" class="form-text text-muted">We'll never share your email with anyone else.</small>
    </div>
    <div class="form-group">
        <label for="exampleInputPassword1">Student Name</label>
        <input type="text" class="form-control" asp-for="Name" placeholder="Student Name">
    </div>
    <br />
    <div class="form-group">
        <label for="exampleFormControlFile1">file input</label>
        <input type="file" class="form-control-file" asp-for="FormFile">
    </div>
    @if (Model != null && Model.FilePath != null)
    {
        <div class="form-group">
            <img src="@Model.FilePath" alt="Student Image" class="img-thumbnail" style="max-height: 100px;" />
        </div>
    }
    <br />
    <button type="submit" class="btn btn-primary">Save</button>
    <a asp-action="Index" asp-controller="Home" class="btn btn-danger">Cancle</a>
</form>
```
