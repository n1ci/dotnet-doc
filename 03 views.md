# ASP.NET MVC Zusammenfassung

> Für BTI4304 - Modern Web Technologies with .NET

---

## Teil 1: Controllers, Actions & Routing

### Request/Response-Zyklus

```text
Request → Route matchen → Controller erstellen → Action aufrufen → Result ausführen → Response
```

An den Übergängen können **Filter** eingreifen (Authorization, Action, Result, Exception).

---

### Controllers

Ein Controller ist eine Klasse, die HTTP-Requests verarbeitet.

**Konventionen:**

- Liegt im Ordner `Controllers/`
- Erbt von `Controller`
- Name endet auf `Controller`
- Hat mindestens eine public Methode

```csharp
// Controllers/ProductController.cs
public class ProductController : Controller
{
    public IActionResult Index()
    {
        return View();
    }
}
```

**Wichtig:** Pro Request wird eine neue Controller-Instanz erstellt. Das Framework setzt automatisch Properties wie `Request`, `Response`, `HttpContext`, `User`, etc.

---

### Dependency Injection

Services werden über den Konstruktor injiziert:

```csharp
public class ProductController : Controller
{
    private readonly IProductService _service;
    private readonly ILogger<ProductController> _logger;

    public ProductController(IProductService service, ILogger<ProductController> logger)
    {
        _service = service;
        _logger = logger;
    }

    public IActionResult Index()
    {
        var products = _service.GetAll();
        return View(products);
    }
}
```

**Services registrieren** in `Program.cs`:

```csharp
// TRANSIENT: Jedes Mal neue Instanz
builder.Services.AddTransient<IEmailService, EmailService>();

// SCOPED: Eine Instanz pro HTTP-Request (typisch für DbContext)
builder.Services.AddScoped<IProductService, ProductService>();

// SINGLETON: Eine Instanz für die ganze App
builder.Services.AddSingleton<ICacheService, CacheService>();
```

| Scope | Wann verwenden | Beispiel |
|-------|----------------|----------|
| Transient | Stateless, leichtgewichtig | Validator, Formatter |
| Scoped | Zustand pro Request | DbContext, UnitOfWork |
| Singleton | Teurer Aufbau, globaler State | Cache, Konfiguration |

---

### Action Methods

Jede **public** Methode im Controller ist eine Action:

```csharp
public class ProductController : Controller
{
    // GET /Product/Index
    public IActionResult Index()
    {
        return View();
    }

    // GET /Product/Details/5 - Parameter aus URL
    public IActionResult Details(int id)
    {
        var product = _service.GetById(id);
        return View(product);
    }

    // GET /Product/Search?query=laptop&maxPrice=1000 - Query-Parameter
    public IActionResult Search(string query, decimal? maxPrice)
    {
        var results = _service.Search(query, maxPrice);
        return View(results);
    }

    // POST /Product/Create - Form-Daten
    [HttpPost]
    public IActionResult Create(ProductViewModel model)
    {
        if (!ModelState.IsValid)
            return View(model);
        
        _service.Create(model);
        return RedirectToAction("Index");
    }
}
```

---

### Action Results

```csharp
public class ExamplesController : Controller
{
    // View rendern
    public IActionResult ShowView()
    {
        return View();                    // Views/Examples/ShowView.cshtml
        return View("CustomName");        // Views/Examples/CustomName.cshtml
        return View(myModel);             // View mit Model
    }

    // JSON für APIs
    public IActionResult GetData()
    {
        return Json(new { Name = "Test", Value = 42 });
    }

    // Weiterleitung
    public IActionResult Redirect()
    {
        return RedirectToAction("Index");                     // Gleicher Controller
        return RedirectToAction("Index", "Home");             // Anderer Controller
        return RedirectToAction("Details", new { id = 5 });   // Mit Parameter
        return Redirect("https://google.com");                // Externe URL
    }

    // Dateien
    public IActionResult Download()
    {
        byte[] bytes = System.IO.File.ReadAllBytes("path/to/file.pdf");
        return File(bytes, "application/pdf", "download.pdf");
    }

    // HTTP Status Codes
    public IActionResult StatusCodes()
    {
        return Ok();              // 200
        return NotFound();        // 404
        return BadRequest();      // 400
        return Unauthorized();    // 401
    }

    // Text/HTML
    public IActionResult PlainText()
    {
        return Content("Hello World");
        return Content("<h1>HTML</h1>", "text/html");
    }
}
```

---

### Async Actions

Für I/O-Operationen (DB, API-Calls, Dateien):

```csharp
// SYNCHRON - blockiert Thread
public IActionResult GetProduct(int id)
{
    var product = _service.GetById(id);
    return View(product);
}

// ASYNCHRON - Thread wird freigegeben
public async Task<IActionResult> GetProductAsync(int id)
{
    var product = await _service.GetByIdAsync(id);
    return View(product);
}
```

---

### Routing

#### Convention-Based Routing

In `Program.cs`:

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);
```

| URL | Controller | Action | id |
|-----|------------|--------|-----|
| `/` | Home | Index | null |
| `/Product` | Product | Index | null |
| `/Product/Details/5` | Product | Details | 5 |

#### Attribute-Based Routing

```csharp
[Route("api/products")]
public class ProductApiController : Controller
{
    [Route("")]              // GET api/products
    public IActionResult GetAll() { }

    [Route("{id:int}")]      // GET api/products/5
    public IActionResult GetById(int id) { }

    [Route("search/{term}")] // GET api/products/search/laptop
    public IActionResult Search(string term) { }
}
```

**Route Constraints:**

```csharp
[Route("user/{id:int}")]          // Nur Zahlen
[Route("user/{name:alpha}")]      // Nur Buchstaben
[Route("user/{id:min(1)}")]       // Minimum 1
```

---

### Filter

Filter führen Code vor/nach Actions aus.

#### Eingebaute Filter

```csharp
[Authorize]                    // Nur eingeloggte User
[Authorize(Roles = "Admin")]   // Nur Admins
[RequireHttps]                 // Erzwingt HTTPS
[ResponseCache(Duration = 60)] // 60 Sekunden cachen
```

#### Custom Filter

```csharp
public class LoggingFilter : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine($"Starting: {context.ActionDescriptor.DisplayName}");
    }

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine($"Finished: {context.ActionDescriptor.DisplayName}");
    }
}

// Verwendung:
[LoggingFilter]
public class ProductController : Controller { }
```

#### Filter-Reihenfolge

```text
Authorization → Action (Before) → ACTION → Action (After) → Result → Exception
```

---

## Teil 2: Helper Methods & Tag Helpers

### Überblick

| Ansatz | Syntax | Verfügbar seit |
|--------|--------|----------------|
| Html/Url Helpers | `@Html.ActionLink(...)` | MVC 1 |
| Tag Helpers | `<a asp-action="...">` | ASP.NET Core |

---

### Html Helper

```csharp
// Generiert komplettes <a>-Tag
@Html.ActionLink("Startseite", "Index", "Home")
// Output: <a href="/Home/Index">Startseite</a>

// Mit Route-Parameter
@Html.ActionLink("Details", "Details", "Product", new { id = 5 }, null)
// Output: <a href="/Product/Details/5">Details</a>

// Mit CSS-Klasse
@Html.ActionLink("Löschen", "Delete", "Product", new { id = 5 }, new { @class = "btn" })
```

### Url Helper

```csharp
// Generiert nur die URL (ohne Tag)
<a href="@Url.Action("Index", "Home")">
    <i class="icon"></i> Startseite
</a>

// Für statische Dateien
<img src="@Url.Content("~/images/logo.png")" />
```

### Formular mit Html Helper

```csharp
@model ProductViewModel

@using (Html.BeginForm("Create", "Product", FormMethod.Post))
{
    @Html.AntiForgeryToken()
    
    @Html.LabelFor(m => m.Name)
    @Html.TextBoxFor(m => m.Name, new { @class = "form-control" })
    @Html.ValidationMessageFor(m => m.Name)
    
    @Html.DropDownListFor(m => m.CategoryId, Model.Categories)
    
    <button type="submit">Speichern</button>
}
```

---

### Custom Html Helper

```csharp
// Helpers/CustomHtmlHelpers.cs
using Microsoft.AspNetCore.Html;
using Microsoft.AspNetCore.Mvc.Rendering;

namespace MyApp.Helpers
{
    public static class CustomHtmlHelpers
    {
        public static IHtmlContent Badge(this IHtmlHelper html, string text, string type = "primary")
        {
            return new HtmlString(
                $"<span class=\"badge bg-{type}\">{html.Encode(text)}</span>"
            );
        }
    }
}
```

Registrieren in `_ViewImports.cshtml`:

```csharp
@using MyApp.Helpers
```

Verwendung:

```csharp
@Html.Badge("Neu", "success")
```

---

### Tag Helpers

#### Aktivieren

In `_ViewImports.cshtml`:

```csharp
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

#### Vergleich

```csharp
// Html Helper (alt)
@Html.ActionLink("Details", "Details", "Product", new { id = 5 }, new { @class = "btn" })

// Tag Helper (neu) - viel lesbarer!
<a asp-controller="Product" asp-action="Details" asp-route-id="5" class="btn">Details</a>
```

#### Anchor Tag Helper

```html
<a asp-controller="Home" asp-action="Index">Startseite</a>

<a asp-controller="Product" asp-action="Details" asp-route-id="5">Details</a>

<a asp-controller="Product" 
   asp-action="List" 
   asp-route-category="electronics"
   asp-route-page="2">
    Elektronik
</a>
```

#### Form Tag Helper

```html
@model ProductViewModel

<form asp-controller="Product" asp-action="Create" method="post">
    
    <label asp-for="Name"></label>
    <input asp-for="Name" class="form-control" />
    <span asp-validation-for="Name" class="text-danger"></span>
    
    <select asp-for="CategoryId" asp-items="Model.Categories">
        <option value="">-- Bitte wählen --</option>
    </select>
    
    <button type="submit">Speichern</button>
</form>
```

**Was `asp-for` automatisch macht:**

- Setzt `name` und `id` basierend auf Property-Namen
- Setzt `type` basierend auf Datentyp
- Fügt Validierungs-Attribute hinzu
- Befüllt den Wert aus dem Model
- Anti-Forgery Token wird automatisch eingefügt

#### Weitere Tag Helpers

```html
<!-- Image mit Cache-Busting -->
<img src="~/images/logo.png" asp-append-version="true" />

<!-- Environment -->
<environment include="Development">
    <link href="~/css/site.css" rel="stylesheet" />
</environment>

<!-- Partial View -->
<partial name="_ProductCard" model="product" />
```

---

### Custom Tag Helper

```csharp
// TagHelpers/EmailTagHelper.cs
using Microsoft.AspNetCore.Razor.TagHelpers;

namespace MyApp.TagHelpers
{
    public class EmailTagHelper : TagHelper
    {
        public string Address { get; set; }

        public override void Process(TagHelperContext context, TagHelperOutput output)
        {
            output.TagName = "a";
            output.Attributes.SetAttribute("href", $"mailto:{Address}");
            output.Content.SetContent(Address);
        }
    }
}
```

Verwendung:

```html
<email address="info@example.com"></email>
<!-- Output: <a href="mailto:info@example.com">info@example.com</a> -->
```

#### Tag Helper auf bestehende Elemente

```csharp
[HtmlTargetElement("button", Attributes = "confirm")]
public class ConfirmButtonTagHelper : TagHelper
{
    public string Confirm { get; set; }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        output.Attributes.SetAttribute("onclick", $"return confirm('{Confirm}');");
    }
}
```

Verwendung:

```html
<button type="submit" confirm="Wirklich löschen?">Löschen</button>
```

#### Registrieren

In `_ViewImports.cshtml`:

```csharp
@addTagHelper *, MyApp
```

---

## Cheat Sheet

### Controller & Actions

| Konzept | Merksatz |
|---------|----------|
| Controller | Erbt von `Controller`, heisst `XyzController` |
| Action | Public Methode im Controller |
| DI Scopes | **T**ransient = neu, **S**coped = pro Request, **S**ingleton = einmal |
| Routing | `{controller}/{action}/{id?}` oder `[Route("...")]` |
| Results | `View()`, `Json()`, `RedirectToAction()`, `NotFound()` |
| Filter | Authorization → Action → Result → Exception |
| Async | `async Task<IActionResult>` + `await` |

### Html Helper vs Tag Helper

| Aufgabe | Html Helper | Tag Helper |
|---------|-------------|------------|
| Link | `@Html.ActionLink("Text", "Action")` | `<a asp-action="Action">` |
| Formular | `@using (Html.BeginForm()) { }` | `<form asp-action="...">` |
| Textfeld | `@Html.TextBoxFor(m => m.Name)` | `<input asp-for="Name" />` |
| Label | `@Html.LabelFor(m => m.Name)` | `<label asp-for="Name">` |
| Dropdown | `@Html.DropDownListFor(m => m.Id, items)` | `<select asp-for="Id" asp-items="items">` |
| Validation | `@Html.ValidationMessageFor(m => m.Name)` | `<span asp-validation-for="Name">` |
| Partial | `@Html.Partial("_Name")` | `<partial name="_Name" />` |

---

> **Tipp:** In neuen ASP.NET Core Projekten → Tag Helpers verwenden!