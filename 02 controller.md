# ASP.NET MVC: Controllers, Actions & Routing - Zusammenfassung

## Das Grundprinzip: Request/Response-Zyklus

Stell dir vor, ein User tippt eine URL ein. Was passiert dann?

```
Request kommt rein → Route wird gematcht → Controller wird erstellt → Action wird aufgerufen → Result wird ausgeführt → Response geht raus
```

Die roten Punkte im Diagramm zeigen, wo **Filter** eingreifen können (dazu später mehr).

---

## Controllers: Das Herzstück

Ein Controller ist einfach eine Klasse, die HTTP-Requests entgegennimmt und verarbeitet.

**Die Konventionen:**

```csharp
// Datei: Controllers/ProductController.cs
public class ProductController : Controller  // ← Muss von Controller erben
{                                            // ← Muss "...Controller" heissen
    public IActionResult Index()
    {
        return View();
    }
}
```

**Wichtig:** Für jeden Request wird eine **neue Instanz** des Controllers erstellt. Das Framework setzt dabei automatisch nützliche Properties wie `Request`, `Response`, `HttpContext`, `User`, etc.

---

## Dependency Injection

Wie bekommt ein Controller Zugriff auf Services (z.B. Datenbank, Logger)?

```csharp
public class ProductController : Controller
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductController> _logger;

    // Constructor Injection - das Framework übergibt die Services automatisch
    public ProductController(IProductService productService, ILogger<ProductController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    public IActionResult Index()
    {
        var products = _productService.GetAll();  // Service nutzen
        _logger.LogInformation("Products loaded");
        return View(products);
    }
}
```

**Services registrieren** (in `Program.cs`):

```csharp
// Drei verschiedene Lifetimes:

// TRANSIENT: Jedes Mal neue Instanz
builder.Services.AddTransient<IEmailService, EmailService>();

// SCOPED: Eine Instanz pro HTTP-Request (typisch für DB-Contexts!)
builder.Services.AddScoped<IProductService, ProductService>();

// SINGLETON: Eine einzige Instanz für die ganze App-Lebenszeit
builder.Services.AddSingleton<ICacheService, CacheService>();
```

**Wann welchen Scope?**

| Scope | Wann verwenden | Beispiel |
|-------|----------------|----------|
| Transient | Stateless, leichtgewichtig | Validator, Formatter |
| Scoped | Zustand pro Request | DbContext, UnitOfWork |
| Singleton | Teurer Aufbau, globaler State | Cache, Konfiguration |

---

## Action Methods: Was der Controller macht

Jede **public** Methode in einem Controller ist eine Action, die per Request aufgerufen werden kann.

```csharp
public class ProductController : Controller
{
    // GET /Product/Index oder GET /Product
    public IActionResult Index()
    {
        return View();
    }

    // GET /Product/Details/5
    public IActionResult Details(int id)  // ← Parameter aus URL
    {
        var product = _service.GetById(id);
        return View(product);
    }

    // GET /Product/Search?query=laptop&maxPrice=1000
    public IActionResult Search(string query, decimal? maxPrice)  // ← Query-Parameter
    {
        var results = _service.Search(query, maxPrice);
        return View(results);
    }

    // POST /Product/Create
    [HttpPost]
    public IActionResult Create(ProductViewModel model)  // ← Form-Daten
    {
        if (!ModelState.IsValid)
            return View(model);
        
        _service.Create(model);
        return RedirectToAction("Index");
    }
}
```

---

## Action Results: Was zurückgegeben wird

Das Framework bietet viele fertige Result-Typen:

```csharp
public class ExamplesController : Controller
{
    // Eine View rendern
    public IActionResult ShowView()
    {
        return View();                    // Views/Examples/ShowView.cshtml
        return View("CustomName");        // Views/Examples/CustomName.cshtml
        return View(myModel);             // View mit Model
    }

    // JSON für APIs
    public IActionResult GetData()
    {
        var data = new { Name = "Test", Value = 42 };
        return Json(data);                // {"name":"Test","value":42}
    }

    // Weiterleitung
    public IActionResult GoSomewhere()
    {
        return RedirectToAction("Index");                     // Gleicher Controller
        return RedirectToAction("Index", "Home");             // Anderer Controller
        return RedirectToAction("Details", new { id = 5 });   // Mit Parameter
        return Redirect("https://google.com");                // Externe URL
    }

    // Dateien
    public IActionResult Download()
    {
        byte[] fileBytes = System.IO.File.ReadAllBytes("path/to/file.pdf");
        return File(fileBytes, "application/pdf", "download.pdf");
    }

    // HTTP Status Codes
    public IActionResult StatusExamples()
    {
        return Ok();                      // 200
        return NotFound();                // 404
        return BadRequest("Invalid");     // 400
        return Unauthorized();            // 401
        return StatusCode(418);           // I'm a teapot
    }

    // Einfacher Text/HTML
    public IActionResult PlainText()
    {
        return Content("Hello World");
        return Content("<h1>HTML</h1>", "text/html");
    }
}
```

---

## Async Actions

Für I/O-Operationen (DB, API-Calls, Dateien) solltest du async verwenden:

```csharp
public class ProductController : Controller
{
    private readonly IProductService _service;

    // SYNCHRON (blockiert Thread während DB-Zugriff)
    public IActionResult GetProductSync(int id)
    {
        var product = _service.GetById(id);  // Thread wartet...
        return View(product);
    }

    // ASYNCHRON (Thread wird freigegeben während Warten)
    public async Task<IActionResult> GetProductAsync(int id)
    {
        var product = await _service.GetByIdAsync(id);  // Thread frei für andere Requests
        return View(product);
    }
}
```

---

## Routing: URL → Controller/Action

### Convention-Based Routing (in Program.cs)

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}"
);
//         └── Optional (?)    └── Default: Index    └── Default: Home
```

**Wie URLs gematcht werden:**

| URL | Controller | Action | id |
|-----|------------|--------|-----|
| `/` | Home | Index | null |
| `/Product` | Product | Index | null |
| `/Product/Details` | Product | Details | null |
| `/Product/Details/5` | Product | Details | 5 |

### Attribute-Based Routing (direkt an der Action)

```csharp
[Route("api/products")]  // ← Prefix für alle Actions im Controller
public class ProductApiController : Controller
{
    [Route("")]           // GET api/products
    [Route("all")]        // GET api/products/all
    public IActionResult GetAll() { ... }

    [Route("{id:int}")]   // GET api/products/5  (nur integers!)
    public IActionResult GetById(int id) { ... }

    [Route("search/{term}")]  // GET api/products/search/laptop
    public IActionResult Search(string term) { ... }

    [Route("category/{category}/page/{page:int?}")]  // GET api/products/category/electronics/page/2
    public IActionResult ByCategory(string category, int page = 1) { ... }
}
```

**Route Constraints** (Typ-Einschränkungen):

```csharp
[Route("user/{id:int}")]              // Nur Zahlen
[Route("user/{name:alpha}")]          // Nur Buchstaben
[Route("user/{id:min(1)}")]           // Minimum 1
[Route("date/{date:datetime}")]       // Gültiges Datum
[Route("file/{filename:regex(.*\\.pdf$)}")]  // Regex
```

---

## Filter: Cross-Cutting Concerns

Filter führen Code vor/nach Actions aus - perfekt für Logging, Auth, Caching, etc.

### Eingebaute Filter

```csharp
public class AdminController : Controller
{
    [Authorize]                    // Nur eingeloggte User
    [Authorize(Roles = "Admin")]   // Nur Admins
    public IActionResult Dashboard() { ... }

    [RequireHttps]                 // Erzwingt HTTPS
    public IActionResult Secure() { ... }

    [ResponseCache(Duration = 60)] // 60 Sekunden cachen
    public IActionResult Cached() { ... }
}
```

### Custom Filter erstellen

```csharp
public class LoggingFilter : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        // VOR der Action
        Console.WriteLine($"Starting: {context.ActionDescriptor.DisplayName}");
    }

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        // NACH der Action
        Console.WriteLine($"Finished: {context.ActionDescriptor.DisplayName}");
    }
}

// Verwendung:
[LoggingFilter]  // ← Auf ganzen Controller
public class ProductController : Controller
{
    [LoggingFilter]  // ← Oder nur auf einzelne Action
    public IActionResult Index() { ... }
}
```

### Filter-Reihenfolge

```
Authorization Filter  →  Action Filter (Before)  →  ACTION  →  Action Filter (After)  →  Result Filter  →  Exception Filter (bei Fehler)
```

---

## Vollständiges Beispiel

```csharp
// Models/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Services/IProductService.cs
public interface IProductService
{
    Task<IEnumerable<Product>> GetAllAsync();
    Task<Product?> GetByIdAsync(int id);
    Task CreateAsync(Product product);
}

// Controllers/ProductController.cs
[Route("products")]
public class ProductController : Controller
{
    private readonly IProductService _service;
    private readonly ILogger<ProductController> _logger;

    public ProductController(IProductService service, ILogger<ProductController> logger)
    {
        _service = service;
        _logger = logger;
    }

    [Route("")]
    public async Task<IActionResult> Index()
    {
        var products = await _service.GetAllAsync();
        return View(products);
    }

    [Route("{id:int}")]
    public async Task<IActionResult> Details(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product == null)
            return NotFound();
        return View(product);
    }

    [Route("create")]
    [HttpGet]
    public IActionResult Create()
    {
        return View();
    }

    [Route("create")]
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(Product product)
    {
        if (!ModelState.IsValid)
            return View(product);

        await _service.CreateAsync(product);
        _logger.LogInformation("Product {Name} created", product.Name);
        return RedirectToAction(nameof(Index));
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

---

## Cheat Sheet für die Prüfung

| Konzept | Merksatz |
|---------|----------|
| Controller | Klasse die von `Controller` erbt, heisst `XyzController` |
| Action | Public Methode im Controller |
| DI Scopes | **T**ransient = immer neu, **S**coped = pro Request, **S**ingleton = einmal |
| Routing | Convention: `{controller}/{action}/{id?}` oder Attribute: `[Route("...")]` |
| Results | `View()`, `Json()`, `RedirectToAction()`, `NotFound()`, `File()` |
| Filter | Authorization → Action → Result → Exception |
| Async | `async Task<IActionResult>` + `await` für I/O |