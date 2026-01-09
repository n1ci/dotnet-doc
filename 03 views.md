# ASP.NET MVC: Helper Methods & Tag Helpers - Zusammenfassung

## Überblick: Zwei Wege zum HTML generieren

In Razor Views hast du zwei Haupt-Ansätze, um dynamisches HTML zu erzeugen:

| Ansatz | Syntax | Verfügbar seit |
|--------|--------|----------------|
| **HTML/URL Helpers** | `@Html.ActionLink(...)` | MVC 1 (.NET Framework) |
| **Tag Helpers** | `<a asp-action="...">` | ASP.NET Core |

Tag Helpers sind der modernere Weg und fühlen sich wie normales HTML an.

---

## Html und Url Helper

Jede View hat zwei eingebaute Properties: `Html` und `Url`

### Navigation mit Html Helper

```cshtml
@* Html.ActionLink generiert komplette <a>-Tags *@

@* Einfacher Link *@
@Html.ActionLink("Zur Startseite", "Index", "Home")
@* Output: <a href="/Home/Index">Zur Startseite</a> *@

@* Mit Route-Parameter *@
@Html.ActionLink("Details anzeigen", "Details", "Product", new { id = 5 }, null)
@* Output: <a href="/Product/Details/5">Details anzeigen</a> *@

@* Mit HTML-Attributen (CSS-Klasse) *@
@Html.ActionLink("Löschen", "Delete", "Product", new { id = 5 }, new { @class = "btn btn-danger" })
@* Output: <a href="/Product/Details/5" class="btn btn-danger">Löschen</a> *@
```

### Navigation mit Url Helper

```cshtml
@* Url.Action generiert nur die URL (ohne <a>-Tag) *@
@* Nützlich wenn du mehr Kontrolle über das HTML brauchst *@

<a href="@Url.Action("Index", "Home")" class="nav-link">
    <i class="icon-home"></i> Startseite
</a>

<form action="@Url.Action("Search", "Product")" method="get">
    <input type="text" name="query" />
    <button type="submit">Suchen</button>
</form>

@* Für statische Dateien (CSS, JS, Bilder) *@
<img src="@Url.Content("~/images/logo.png")" alt="Logo" />
<link href="@Url.Content("~/css/style.css")" rel="stylesheet" />
```

### Formular-Helper

```cshtml
@model ProductViewModel

@using (Html.BeginForm("Create", "Product", FormMethod.Post))
{
    @Html.AntiForgeryToken()
    
    <div class="form-group">
        @Html.LabelFor(m => m.Name)
        @Html.TextBoxFor(m => m.Name, new { @class = "form-control" })
        @Html.ValidationMessageFor(m => m.Name)
    </div>
    
    <div class="form-group">
        @Html.LabelFor(m => m.Price)
        @Html.TextBoxFor(m => m.Price, new { @class = "form-control", type = "number" })
        @Html.ValidationMessageFor(m => m.Price)
    </div>
    
    <div class="form-group">
        @Html.LabelFor(m => m.CategoryId)
        @Html.DropDownListFor(m => m.CategoryId, Model.Categories, "-- Auswählen --")
    </div>
    
    <button type="submit" class="btn btn-primary">Speichern</button>
}
```

---

## Custom Html Helper (Eigene Helper-Methoden)

Du kannst eigene Helper-Methoden als Extension Methods schreiben.

### .NET Core Version

```csharp
// Helpers/CustomHtmlHelpers.cs
using Microsoft.AspNetCore.Html;
using Microsoft.AspNetCore.Mvc.Rendering;

namespace MyApp.Helpers
{
    public static class CustomHtmlHelpers
    {
        // Einfacher Helper: Badge anzeigen
        public static IHtmlContent Badge(this IHtmlHelper html, string text, string type = "primary")
        {
            return new HtmlString($"<span class=\"badge bg-{type}\">{html.Encode(text)}</span>");
        }

        // Helper mit komplexerem Objekt
        public static IHtmlContent ProductCard(this IHtmlHelper html, Product product)
        {
            return new HtmlString($@"
                <div class=""card"">
                    <div class=""card-body"">
                        <h5 class=""card-title"">{html.Encode(product.Name)}</h5>
                        <p class=""card-text"">CHF {product.Price:F2}</p>
                    </div>
                </div>");
        }

        // Helper für Datum-Formatierung
        public static IHtmlContent FormattedDate(this IHtmlHelper html, DateTime date)
        {
            var formatted = date.ToString("dd. MMMM yyyy");
            return new HtmlString($"<time datetime=\"{date:yyyy-MM-dd}\">{formatted}</time>");
        }
    }
}
```

### Helper verfügbar machen

```cshtml
@* Option 1: In einzelner View *@
@using MyApp.Helpers

@* Option 2: Global in _ViewImports.cshtml (empfohlen) *@
@using MyApp.Helpers
```

### Verwendung in der View

```cshtml
@* Jetzt kannst du die Helper benutzen *@

<p>Status: @Html.Badge("Neu", "success")</p>
<p>Status: @Html.Badge("Ausverkauft", "danger")</p>

@foreach (var product in Model.Products)
{
    @Html.ProductCard(product)
}

<p>Erstellt am: @Html.FormattedDate(Model.CreatedAt)</p>
```

---

## Tag Helpers (Der moderne Weg)

Tag Helpers sehen aus wie normales HTML mit speziellen Attributen (`asp-*`).

### Tag Helpers aktivieren

```cshtml
@* In _ViewImports.cshtml *@
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

### Vergleich: Html Helper vs Tag Helper

```cshtml
@* ===== ALTE SYNTAX (Html Helper) ===== *@
@Html.ActionLink("Details", "Details", "Product", new { id = Model.Id }, new { @class = "btn" })

@* ===== NEUE SYNTAX (Tag Helper) ===== *@
<a asp-controller="Product" asp-action="Details" asp-route-id="@Model.Id" class="btn">Details</a>

@* Beide erzeugen: <a href="/Product/Details/5" class="btn">Details</a> *@
```

Der Tag Helper ist viel lesbarer!

### Anchor Tag Helper (Links)

```cshtml
@* Einfacher Link *@
<a asp-controller="Home" asp-action="Index">Startseite</a>

@* Mit Route-Parameter *@
<a asp-controller="Product" asp-action="Details" asp-route-id="5">Produkt #5</a>

@* Mehrere Parameter *@
<a asp-controller="Product" 
   asp-action="List" 
   asp-route-category="electronics"
   asp-route-page="2">
    Elektronik Seite 2
</a>
@* Output: /Product/List?category=electronics&page=2 *@

@* Mit benannter Route *@
<a asp-route="ProductDetails" asp-route-id="5">Details</a>
```

### Form Tag Helper

```cshtml
@model ProductViewModel

<form asp-controller="Product" asp-action="Create" method="post">
    @* Anti-Forgery Token wird automatisch eingefügt! *@
    
    <div class="mb-3">
        <label asp-for="Name" class="form-label"></label>
        <input asp-for="Name" class="form-control" />
        <span asp-validation-for="Name" class="text-danger"></span>
    </div>
    
    <div class="mb-3">
        <label asp-for="Price" class="form-label"></label>
        <input asp-for="Price" class="form-control" />
        <span asp-validation-for="Price" class="text-danger"></span>
    </div>
    
    <div class="mb-3">
        <label asp-for="CategoryId" class="form-label"></label>
        <select asp-for="CategoryId" asp-items="Model.Categories" class="form-select">
            <option value="">-- Bitte wählen --</option>
        </select>
    </div>
    
    <div class="mb-3">
        <label asp-for="Description" class="form-label"></label>
        <textarea asp-for="Description" class="form-control" rows="4"></textarea>
    </div>
    
    <button type="submit" class="btn btn-primary">Speichern</button>
</form>
```

**Was `asp-for` automatisch macht:**
- Setzt `name` und `id` Attribute basierend auf Property-Namen
- Setzt `type` basierend auf Datentyp (z.B. `type="number"` für `decimal`)
- Fügt Validierungs-Attribute hinzu (aus Data Annotations)
- Befüllt den Wert aus dem Model

### Image Tag Helper

```cshtml
@* Cache-Busting: Fügt automatisch einen Hash hinzu, damit Browser-Cache bei Änderungen invalidiert wird *@
<img src="~/images/logo.png" asp-append-version="true" alt="Logo" />
@* Output: <img src="/images/logo.png?v=abc123..." alt="Logo" /> *@
```

### Environment Tag Helper

```cshtml
@* Unterschiedliche Inhalte je nach Environment *@
<environment include="Development">
    <link rel="stylesheet" href="~/css/site.css" />
    <script src="~/js/site.js"></script>
</environment>

<environment exclude="Development">
    <link rel="stylesheet" href="~/css/site.min.css" asp-append-version="true" />
    <script src="~/js/site.min.js" asp-append-version="true"></script>
</environment>
```

### Partial Tag Helper

```cshtml
@* Partial View einbinden *@
<partial name="_ProductCard" model="product" />

@* Ist equivalent zu: *@
@await Html.PartialAsync("_ProductCard", product)
```

---

## Custom Tag Helper erstellen

### Einfaches Beispiel: Email-Link

```csharp
// TagHelpers/EmailTagHelper.cs
using Microsoft.AspNetCore.Razor.TagHelpers;

namespace MyApp.TagHelpers
{
    // Aktiviert sich bei <email>-Tags
    public class EmailTagHelper : TagHelper
    {
        public string Address { get; set; }

        public override void Process(TagHelperContext context, TagHelperOutput output)
        {
            // Ändere <email> zu <a>
            output.TagName = "a";
            output.Attributes.SetAttribute("href", $"mailto:{Address}");
            output.Content.SetContent(Address);
        }
    }
}
```

**Verwendung:**
```cshtml
<email address="info@example.com"></email>
@* Output: <a href="mailto:info@example.com">info@example.com</a> *@
```

### Komplexeres Beispiel: Alert-Box

```csharp
// TagHelpers/AlertTagHelper.cs
using Microsoft.AspNetCore.Razor.TagHelpers;

namespace MyApp.TagHelpers
{
    // Aktiviert sich bei <alert>-Tags
    [HtmlTargetElement("alert")]
    public class AlertTagHelper : TagHelper
    {
        public string Type { get; set; } = "info";  // info, success, warning, danger
        public string Title { get; set; }
        public bool Dismissible { get; set; } = false;

        public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
        {
            output.TagName = "div";
            output.Attributes.SetAttribute("class", $"alert alert-{Type}" + (Dismissible ? " alert-dismissible fade show" : ""));
            output.Attributes.SetAttribute("role", "alert");

            // Inneren Content holen (was zwischen <alert>...</alert> steht)
            var childContent = await output.GetChildContentAsync();

            var content = "";
            
            if (!string.IsNullOrEmpty(Title))
            {
                content += $"<strong>{Title}</strong> ";
            }
            
            content += childContent.GetContent();

            if (Dismissible)
            {
                content += @"<button type=""button"" class=""btn-close"" data-bs-dismiss=""alert""></button>";
            }

            output.Content.SetHtmlContent(content);
        }
    }
}
```

**Verwendung:**
```cshtml
<alert type="success" title="Erfolg!">Das Produkt wurde gespeichert.</alert>

<alert type="danger" title="Fehler!" dismissible="true">
    Es ist ein Fehler aufgetreten. Bitte versuchen Sie es erneut.
</alert>

@* Output: 
<div class="alert alert-success" role="alert">
    <strong>Erfolg!</strong> Das Produkt wurde gespeichert.
</div>
*@
```

### Tag Helper auf bestehende HTML-Elemente anwenden

```csharp
// Aktiviert sich bei <button>-Tags mit dem Attribut "confirm"
[HtmlTargetElement("button", Attributes = "confirm")]
public class ConfirmButtonTagHelper : TagHelper
{
    public string Confirm { get; set; }

    public override void Process(TagHelperContext context, TagHelperOutput output)
    {
        // Füge onclick-Handler hinzu
        output.Attributes.SetAttribute("onclick", $"return confirm('{Confirm}');");
    }
}
```

**Verwendung:**
```cshtml
<button type="submit" confirm="Wirklich löschen?" class="btn btn-danger">
    Löschen
</button>
@* Output: <button type="submit" onclick="return confirm('Wirklich löschen?');" class="btn btn-danger">Löschen</button> *@
```

### Tag Helper registrieren

```cshtml
@* In _ViewImports.cshtml *@

@* Alle Tag Helpers aus deinem Projekt *@
@addTagHelper *, MyApp

@* Oder einzelne Tag Helper *@
@addTagHelper MyApp.TagHelpers.EmailTagHelper, MyApp
@addTagHelper MyApp.TagHelpers.AlertTagHelper, MyApp
```

---

## Vollständiges Formular-Beispiel

### Model mit Validierung

```csharp
// Models/ProductViewModel.cs
public class ProductViewModel
{
    public int Id { get; set; }
    
    [Required(ErrorMessage = "Name ist erforderlich")]
    [StringLength(100, MinimumLength = 3)]
    [Display(Name = "Produktname")]
    public string Name { get; set; }
    
    [Required]
    [Range(0.01, 10000)]
    [Display(Name = "Preis (CHF)")]
    public decimal Price { get; set; }
    
    [Display(Name = "Kategorie")]
    public int CategoryId { get; set; }
    
    [Display(Name = "Beschreibung")]
    [DataType(DataType.MultilineText)]
    public string Description { get; set; }
    
    // Für Dropdown
    public SelectList Categories { get; set; }
}
```

### View mit Tag Helpers

```cshtml
@model ProductViewModel

<h1>Produkt @(Model.Id == 0 ? "erstellen" : "bearbeiten")</h1>

<form asp-action="Save" method="post" class="needs-validation" novalidate>
    <input type="hidden" asp-for="Id" />
    
    <div class="row">
        <div class="col-md-6 mb-3">
            <label asp-for="Name" class="form-label"></label>
            <input asp-for="Name" class="form-control" />
            <span asp-validation-for="Name" class="invalid-feedback"></span>
        </div>
        
        <div class="col-md-6 mb-3">
            <label asp-for="Price" class="form-label"></label>
            <div class="input-group">
                <span class="input-group-text">CHF</span>
                <input asp-for="Price" class="form-control" />
            </div>
            <span asp-validation-for="Price" class="invalid-feedback"></span>
        </div>
    </div>
    
    <div class="mb-3">
        <label asp-for="CategoryId" class="form-label"></label>
        <select asp-for="CategoryId" asp-items="Model.Categories" class="form-select">
            <option value="">-- Kategorie wählen --</option>
        </select>
        <span asp-validation-for="CategoryId" class="invalid-feedback"></span>
    </div>
    
    <div class="mb-3">
        <label asp-for="Description" class="form-label"></label>
        <textarea asp-for="Description" class="form-control" rows="4"></textarea>
    </div>
    
    <div class="d-flex gap-2">
        <button type="submit" class="btn btn-primary">Speichern</button>
        <a asp-action="Index" class="btn btn-secondary">Abbrechen</a>
    </div>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## Cheat Sheet: Html Helper vs Tag Helper

| Aufgabe | Html Helper | Tag Helper |
|---------|-------------|------------|
| Link | `@Html.ActionLink("Text", "Action", "Controller")` | `<a asp-action="Action" asp-controller="Controller">Text</a>` |
| URL generieren | `@Url.Action("Action", "Controller")` | - |
| Formular | `@using (Html.BeginForm(...)) { }` | `<form asp-action="...">` |
| Textfeld | `@Html.TextBoxFor(m => m.Name)` | `<input asp-for="Name" />` |
| Label | `@Html.LabelFor(m => m.Name)` | `<label asp-for="Name"></label>` |
| Dropdown | `@Html.DropDownListFor(m => m.Id, items)` | `<select asp-for="Id" asp-items="items">` |
| Validation | `@Html.ValidationMessageFor(m => m.Name)` | `<span asp-validation-for="Name">` |
| Partial | `@Html.Partial("_Name")` | `<partial name="_Name" />` |

**Faustregel:** In neuen ASP.NET Core Projekten → Tag Helpers verwenden!