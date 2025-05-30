# Write a New Products Page

## In this lab

In this lab, you'll enhance your application by creating a Products page that leverages AI to automatically generate product descriptions and categorize items based on their content. This lab demonstrates several key concepts in modern AI application development:

🧩 **What we're building:**

- A product catalog service that uses AI to analyze product documentation
- A database system to store and retrieve product information
- A user interface that displays and filters AI-processed products

🔍 **Key technical concepts you'll learn:**

- **AI Service Abstraction**: Work with `IChatClient` interface that allows you to interact with AI models without being tied to a specific provider (like GitHub Models or Azure OpenAI)
  
- **Vector Database Integration**: Learn how vector embeddings enable semantic search and information retrieval from your product documentation

- **Cross-Service Data Flow**: See how data flows between your application components:
  
  ```mermaid
  %%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#f4f4f4', 'primaryTextColor': '#000', 'primaryBorderColor': '#333', 'lineColor': '#333', 'secondaryColor': '#e1f5fe', 'tertiaryColor': '#f3e5f5' }}}%%
  flowchart LR
    PDF[Product PDFs] -->|Ingestion| VDB[(Vector DB)]
    User(User Query) -->|Generate Embeddings| VQ[Vector Query]
    VQ --> VDB
    VDB -->|Relevant Chunks| AI{AI Model}
    User -->|Original Question| AI
    AI -->|Structured JSON| DB[(Product DB)]
    DB -->|Filtered Results| UI[UI Display]
    
    style PDF fill:#f9d5e5
    style VDB fill:#e1f5fe
    style AI fill:#d5e8d4
    style DB fill:#f3e5f5
  ```

- **Prompt Engineering for Structured Data**: Design AI prompts that return structured JSON responses for reliable data processing

- **Entity Framework Integration**: Connect AI-generated content with a traditional database for efficient querying and filtering

The Products feature showcases how AI can enhance traditional web applications by automatically analyzing content, generating descriptions, and organizing information - all while using provider-agnostic interfaces that would allow you to easily switch between AI services.

## Create the Product Models

First, we need to define the database models that will store our AI-generated product information. These models will allow us to save product descriptions and categories for later retrieval and filtering.

1. Add a new folder named `Models` to the project `src/start/GenAiLab.Web`, by right-clicking on the project and selecting "Add" > "New Folder".

1. In this new folder, create a new file `ProductInfo.cs` and replace the content with the following code:

```csharp
using System;
using System.Collections.Generic;

namespace GenAiLab.Web.Models;

public class ProductInfo
{
    public Guid Id { get; set; }
    public required string Name { get; set; }
    public required string ShortDescription { get; set; }
    public required string Category { get; set; }
    public required string FileName { get; set; }

    // For filtering
    public static List<string> AvailableCategories { get; set; } = new List<string>();
}
```

1. In the folder `Services` of the project `src/start/GenAiLab.Web`, create a database context for products in `ProductDbContext.cs`. This database context will manage the Entity Framework Core interactions with our PostgreSQL database, allowing us to query and save product information:

```csharp
using Microsoft.EntityFrameworkCore;
using GenAiLab.Web.Models;

namespace GenAiLab.Web.Services;

public class ProductDbContext : DbContext
{
    public ProductDbContext(DbContextOptions<ProductDbContext> options) : base(options)
    {
    }

    public DbSet<ProductInfo> Products { get; set; }
    public DbSet<ProductCategory> Categories { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Configure ProductInfo entity
        modelBuilder.Entity<ProductInfo>()
            .HasKey(p => p.Id);

        modelBuilder.Entity<ProductInfo>()
            .Property(p => p.Name)
            .IsRequired();

        modelBuilder.Entity<ProductInfo>()
            .Property(p => p.FileName)
            .IsRequired();

        // Configure ProductCategory entity
        modelBuilder.Entity<ProductCategory>()
            .HasKey(c => c.Id);

        modelBuilder.Entity<ProductCategory>()
            .Property(c => c.Name)
            .IsRequired();

        modelBuilder.Entity<ProductCategory>()
            .HasIndex(c => c.Name)
            .IsUnique();    }
    
    // Helper method to initialize the database
    public static void Initialize(IServiceProvider serviceProvider)
    {
        using var scope = serviceProvider.CreateScope();
        using var context = scope.ServiceProvider.GetRequiredService<ProductDbContext>();
        context.Database.EnsureCreated();
    }
}

// New entity for storing categories
public class ProductCategory
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

## Create the Product Service

Now we'll create a service that coordinates between the AI model, vector database, and our product database. This service will:

1. Extract product information from the vector database
2. Use AI to generate descriptions and categorize products
3. Store the results in our PostgreSQL database

In the folder `Services` of the project `src/start/GenAiLab.Web`, create a new file `ProductService.cs` to generate product information using AI. Replace the content with the following code:

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Extensions.VectorData;
using GenAiLab.Web.Models;
using System.Text;
using OpenAI;
using Microsoft.EntityFrameworkCore;

namespace GenAiLab.Web.Services;

public class ProductService(
        IEmbeddingGenerator<string, Embedding<float>> _embeddingGenerator,
        IVectorStore _vectorStore,
        ProductDbContext _dbContext,
        IChatClient _chatClient,
        ILogger<ProductService> _logger)
{
    public async Task<IEnumerable<ProductInfo>> GetProductsAsync(string? categoryFilter = null)
    {
        // Make sure we have products
        await EnsureProductsExistAsync();

        // Simple filtering by category if specified
        var query = string.IsNullOrEmpty(categoryFilter)
            ? _dbContext.Products
            : _dbContext.Products.Where(p => p.Category == categoryFilter);

        return await query.ToListAsync();
    }    public async Task<List<string>> GetCategoriesAsync()
    {
        await EnsureProductsExistAsync();
        return await _dbContext.Categories.Select(c => c.Name).ToListAsync();
    }
    
    private async Task EnsureProductsExistAsync()
    {
        if (!await _dbContext.Products.AnyAsync())
        {
            await GenerateAndSaveProductsAsync();
        }
    }
}
```

## Implement Product Generation with AI

Next, we'll add methods to generate product information by retrieving document content from the vector database. These methods will:

1. Find unique product files in the vector database
2. Extract relevant content chunks for each product
3. Process each document to create product entries

Add the following methods to the `ProductService` class:

```csharp
private async Task GenerateAndSaveProductsAsync()
{
    // Get documents from vector store
    var fileNames = await GetUniqueFileNamesAsync();
      if (fileNames.Count == 0)
    {
        _logger.LogWarning("No documents found in vector store");
        return;
    }

    var categories = new HashSet<string>();

    // Process each file
    foreach (var fileName in fileNames)
    {
        var productName = Path.GetFileNameWithoutExtension(fileName)
            .Replace("Example_", "")
            .Replace("_", " ");        // Get document content
        var content = await GetDocumentContentAsync(fileName, productName);
        
        if (string.IsNullOrWhiteSpace(content))
        {
            continue;
        }// Get product description and category from AI
        var (description, category) = await AskAIForProductInfoAsync(content, productName);
        categories.Add(category);
        
        // Save to database
        _dbContext.Products.Add(new ProductInfo
        {
            Name = productName,
            ShortDescription = description,
            Category = category,
            FileName = fileName
        });
    }

    // Save categories
    foreach (var category in categories)
    {
        if (!await _dbContext.Categories.AnyAsync(c => c.Name == category))
        {
            _dbContext.Categories.Add(new ProductCategory { Name = category });
        }
    }

    ProductInfo.AvailableCategories = categories.ToList();
    await _dbContext.SaveChangesAsync();
}

private async Task<List<string>> GetUniqueFileNamesAsync()
{
    var vectorCollection = _vectorStore.GetCollection<Guid, SemanticSearchRecord>("data-genailab-ingested");
    
    try
    {
        var dummyEmbedding = await _embeddingGenerator.GenerateEmbeddingVectorAsync("all documents");
        var searchResults = await vectorCollection.VectorizedSearchAsync(
            dummyEmbedding,
            new VectorSearchOptions<SemanticSearchRecord> { Top = 1000 }
        );

        var uniqueFileNames = new HashSet<string>();
        await foreach (var result in searchResults.Results)
        {
            uniqueFileNames.Add(result.Record.FileName);
        }

        return uniqueFileNames.ToList();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error retrieving documents from vector store");
        return new List<string>();
    }
}

private async Task<string> GetDocumentContentAsync(string fileName, string productName)
{
    var vectorCollection = _vectorStore.GetCollection<Guid, SemanticSearchRecord>("data-genailab-ingested");

    try
    {
        var contentEmbedding = await _embeddingGenerator.GenerateEmbeddingVectorAsync($"Information about {productName}");
        var contentResults = await vectorCollection.VectorizedSearchAsync(
            contentEmbedding,
            new VectorSearchOptions<SemanticSearchRecord> 
            { 
                Top = 5,
                Filter = record => record.FileName == fileName
            });

        var contentBuilder = new StringBuilder();
        await foreach (var item in contentResults.Results)
        {
            contentBuilder.AppendLine(item.Record.Text);
        }

        return contentBuilder.ToString();
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Error getting content for {FileName}", fileName);
        return string.Empty;
    }
}
```

## Implement AI-Based Product Description Generation

This is the core of our AI integration - where we prompt the AI model to analyze product documentation and return structured data. This method:

1. Creates a carefully designed prompt that asks for JSON-formatted output
2. Sends the prompt to the AI model through the abstract IChatClient interface
3. Parses the response and extracts description and category information

Add the following methods to use the AI service for generating product descriptions and categories:

```csharp
// Simple record for JSON deserialization
private record ProductResponse(string Description, string Category);

// This is the key method that uses IChatClient
private async Task<(string Description, string Category)> AskAIForProductInfoAsync(string content, string productName)
{
    try
    {
        // Create a simple prompt requesting JSON response
        var prompt = $@"Based on this content about '{productName}', provide a JSON object with these properties:
1. description: A concise product description (max 200 characters)
2. category: One of: 'Electronics', 'Safety Equipment', 'GPS', 'Backpack', 'Outdoor Gear', or 'General'

Return ONLY the raw JSON object without any markdown formatting, code blocks, or backticks.

Content: {content}";

        // Get response from the chat client
        var chatResponse = await _chatClient.GetResponseAsync(
            new[] {
                new ChatMessage(ChatRole.System, "You are a product information assistant. Respond with valid JSON only, no markdown formatting or backticks."),
                new ChatMessage(ChatRole.User, prompt)
            });
        
        // Remove any markdown code block indicators (```json and ```)
        string cleanedResponse = chatResponse.Text
            .Replace("```json", "")
            .Replace("```", "")
            .Trim();
        
        // Try to parse the cleaned JSON response
        var options = new System.Text.Json.JsonSerializerOptions { PropertyNameCaseInsensitive = true };
        var responseJson = System.Text.Json.JsonSerializer.Deserialize<ProductResponse>(cleanedResponse, options);

        if (responseJson != null)
        {
            return (responseJson.Description, responseJson.Category);
        }
    }
    catch (Exception ex)
    {
        _logger.LogWarning("AI processing error for {ProductName}: {Error}", productName, ex.Message);
    }

    // Simple fallback
    return ($"A high-quality {productName}", "General");
}
```

## Add the PostgreSQL Database in AppHost

Now we need to create the actual PostgreSQL database resource in our Aspire application. Using PostgreSQL from the start ensures our application is production-ready without requiring database migrations later. The `productDb` resource will:

1. Create a PostgreSQL database container for storing product information
2. Make the database accessible to our web application through connection strings
3. Ensure our web app waits for the database to be ready before starting

First, you need to add the PostgreSQL NuGet package to your AppHost project. You can do this in multiple ways:

**Using Visual Studio's .NET Aspire tooling**:

- Right-click on the `GenAiLab.AppHost` project in Solution Explorer
- Select "Add" > ".NET Aspire package..."
- In the package manager that opens (with pre-filtered .NET Aspire packages), search for "Aspire.Hosting.PostgreSQL"
- Select "9.1.0" for the package version *Important, don't skip this!*
- Click "Install"

**Using Terminal**:

```powershell
dotnet add GenAiLab.AppHost/GenAiLab.AppHost.csproj package Aspire.Hosting.PostgreSQL -v 9.1.0
```

Next, you need to add the `productDb` PostgreSQL resource to your AppHost project. Open the file `src/start/GenAiLab.AppHost/Program.cs` and add the following lines after the vector database declaration:

```csharp
var vectorDB = builder.AddQdrant("vectordb")
    .WithDataVolume()
    .WithLifetime(ContainerLifetime.Persistent);

var postgres = builder.AddPostgres("postgres")
    .WithLifetime(ContainerLifetime.Persistent);
var productDb = postgres.AddDatabase("productDb");
```

Then, make sure your web application references this resource by adding these lines after the existing vector database reference:

```csharp
webApp
    .WithReference(productDb)
    .WaitFor(productDb);
```

Your AppHost's `Program.cs` should now look similar to this:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// You will need to set the connection string to your own value
// You can do this using Visual Studio's "Manage User Secrets" UI, or on the command line:
//   cd this-project-directory
//   dotnet user-secrets set ConnectionStrings:openai "Endpoint=https://models.inference.ai.azure.com;Key=YOUR-API-KEY"
var openai = builder.AddConnectionString("openai");

var vectorDB = builder.AddQdrant("vectordb")
    .WithDataVolume()
    .WithLifetime(ContainerLifetime.Persistent);

var postgres = builder.AddPostgres("postgres")
    .WithLifetime(ContainerLifetime.Persistent);
var productDb = postgres.AddDatabase("productDb");

var webApp = builder.AddProject<Projects.GenAiLab_Web>("aichatweb-app");
webApp.WithReference(openai);
webApp
    .WithReference(vectorDB)
    .WaitFor(vectorDB);
webApp
    .WithReference(productDb)
    .WaitFor(productDb);

builder.Build().Run();
```

## Register the Services

With our database resource created, we now need to register our services with the dependency injection system. This step:

1. Connects our `ProductDbContext` to the PostgreSQL database we created
2. Makes our `ProductService` available throughout the application
3. Initializes the database schema when the application starts

First, you need to add the PostgreSQL Entity Framework NuGet package to your Web project:

**Using Visual Studio's NuGet Package Manager**:

- Right-click on the `GenAiLab.Web` project in Solution Explorer
- Select "Manage NuGet Packages..."
- Click on the "Browse" tab
- Search for "Aspire.Npgsql.EntityFrameworkCore.PostgreSQL"
- Select "9.1.0" for the package version *Important, don't skip this!*
- Select the package and click "Install"

**Using Terminal**:

```powershell
dotnet add GenAiLab.Web/GenAiLab.Web.csproj package Aspire.Npgsql.EntityFrameworkCore.PostgreSQL -v 9.1.0
```

In the project `src/start/GenAiLab.Web`, update your `Program.cs` file to register the new services. First, add the using statement at the top of the file:

```csharp
using Aspire.Npgsql.EntityFrameworkCore.PostgreSQL;
```

Then, just before the `var app = builder.Build();` line, add the following code:

```csharp
// Add database support
builder.AddNpgsqlDbContext<ProductDbContext>("productDb");

// Register product service
builder.Services.AddScoped<ProductService>();
```

And after the `var app = builder.Build();` line, add initialization for the ProductDbContext:

```csharp
var app = builder.Build();
ProductDbContext.Initialize(app.Services); // Add this line
```

## Create the Products Page

Finally, let's create a user interface to display and filter our AI-generated product information. This page will:

1. Fetch products and categories from our ProductService
2. Display products in a filterable grid using QuickGrid
3. Allow users to filter products by their AI-determined categories

Let's use the new AspNetCore QuickGrid component to display the products. First, we need to add the Nuget package `Microsoft.AspNetCore.Components.QuickGrid`.

There are multiple ways to do this:

- Open the GenAiLab.Web project file and add at the end of the packages `<ItemGroup>`

    ```xml
    <PackageReference Include="Microsoft.AspNetCore.Components.QuickGrid" Version="9.0.4" />
    ```

or

- Type the following command in the Package Manager Console:

    ```powershell
    NuGet\Install-Package Microsoft.AspNetCore.Components.QuickGrid -Version 9.0.4
    ```

Create a new file `Components/Pages/Products.razor`:

```csharp
@page "/products"
@using GenAiLab.Web.Models
@using GenAiLab.Web.Services
@using Microsoft.AspNetCore.Components.QuickGrid
@using System.Linq.Expressions
@inject ProductService ProductService

<PageTitle>Products - GenAI Lab</PageTitle>

<h1>📦 Our Products</h1>

@if (AllProducts == null)
{
    <div class="message-box">
        <span>🔄 Loading products...</span>
    </div>
}
else if (!FilteredProducts.Any())
{
    <div class="message-box">
        <span>📦 No products found</span>
    </div>
}
else
{
    <div>
        <select @bind="CategoryFilter" @bind:after="StateHasChanged">
            <option value="">✨ All Categories</option>
            @foreach (var category in Categories)
            {
                <option value="@category">📁 @category</option>
            }
        </select>

        <div class="product-table-container">
            <QuickGrid Items="@FilteredProducts">
                <PropertyColumn Property="@(p => p.Name)" Title="📦 Product Name" Sortable="true" />
                <PropertyColumn Property="@(p => p.ShortDescription)" Title="📝 Description" />
                <PropertyColumn Property="@(p => p.Category)" Title="🏷️ Category" Sortable="true" />
            </QuickGrid>
        </div>
    </div>
}

<style>
    h1 {
        margin-bottom: 1.5rem;
    }

    select {
        padding: 0.5rem;
        border: 1px solid #ccc;
        border-radius: 0.25rem;
        margin-bottom: 1rem;
        align-self: flex-end;
    }

    .message-box {
        padding: 1rem;
        margin-bottom: 1.5rem;
        border-left: 4px solid #3a4ed5;
        background-color: #f0f4ff;
        border-radius: 0.25rem;
    }

    .product-table-container {
        margin-bottom: 2rem;
        border: 1px solid #e0e0e0;
        border-radius: 0.25rem;
        overflow: hidden;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    }

    ::deep table {
        width: 100%;
        border-collapse: collapse;
    }

    ::deep th {
        background-color: #f5f5f5;
        font-weight: 600;
        text-align: left;
        padding: 0.75rem 1rem;
        border-bottom: 2px solid #ddd;
    }

    ::deep td {
        padding: 0.75rem 1rem;
        border-bottom: 1px solid #eee;
    }

    ::deep tr:nth-child(even) {
        background-color: #f9f9f9;
    }

    ::deep tr:hover {
        background-color: #f0f4ff;
    }

    ::deep .col-options-button {
        color: #3a4ed5;
    }

    ::deep .col-options-menu {
        padding: 0.75rem;
        border-radius: 0.25rem;
    }
</style>

@code {
    private IQueryable<ProductInfo>? AllProducts;
    private List<string> Categories { get; set; } = new List<string>();
    private string CategoryFilter { get; set; } = string.Empty;

    private IQueryable<ProductInfo> FilteredProducts
    {
        get
        {
            if (AllProducts == null)
                return Enumerable.Empty<ProductInfo>().AsQueryable();

            if (string.IsNullOrEmpty(CategoryFilter))
                return AllProducts;

            return AllProducts.Where(p => p.Category == CategoryFilter);
        }
    }

    protected override async Task OnInitializedAsync()
    {
        await LoadData();
    }

    private async Task LoadData()
    {
        Categories = await ProductService.GetCategoriesAsync();
        var products = await ProductService.GetProductsAsync();
        AllProducts = products.AsQueryable();
    }
}
```

## Update the Navigation

To make our new Products page accessible to users, we need to add a navigation link to the main layout. This step:

1. Adds a button to the header navigation that links to our Products page
2. Provides visual consistency with the rest of the application
3. Makes the feature discoverable to users

In the GenAiLab.Web project, locate the file `Components/Layout/MainLayout.razor` and update it to include a link to the Products page.

If your project uses a different navigation structure, find the appropriate file (such as `NavMenu.razor` or `ChatHeader.razor`) and add a navigation link to the Products page:

```csharp
<div class="chat-header-container main-background-gradient">
    <div class="chat-header-controls page-width" style="display: flex; gap: 8px; align-items: center;">
        <button class="btn-default" @onclick="@OnNewChat">
            <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5"
                stroke="currentColor" class="new-chat-icon">
                <path stroke-linecap="round" stroke-linejoin="round" d="M12 4.5v15m7.5-7.5h-15" />
            </svg>
            New chat
        </button>
        @* 👇👇👇 Add the button here 👇👇👇 *@
        <button class="btn-subtle" onclick="location.href='/products'" style="display: inline-flex; align-items: center;">
            📦 Products
        </button>
        @* 👆👆👆 Add the button here 👆👆👆 *@
    </div>

    <h1 class="page-width">GenAiLab.Web</h1>
</div>
```

## Testing the Products Feature

Now it's time to see our AI-powered product catalog in action! This testing phase:

1. Verifies that the vector database is correctly supplying product content
2. Confirms that the AI model is generating appropriate descriptions and categories
3. Ensures the user interface properly displays and filters the products

Follow these steps:

1. Run your application and navigate to the Products page
2. You should see a list of products with AI-generated descriptions
3. Try filtering products by category using the dropdown

## What You've Learned

In this lab, you've built a complete AI-powered product catalog system that demonstrates several key aspects of modern AI-integrated applications:

- How to create models and database contexts for a new feature
- How to build a service that interacts with AI to generate product information
- How to use vector embeddings to find relevant document content
- How to prompt AI models for structured JSON responses
- How to handle and parse JSON responses from AI models
- How to create a user interface that displays and filters AI-generated content

## Next Steps

Now that you've implemented the Products feature, proceed to [Deploy to Azure](part5-deploy-azure.md) to learn how to prepare your application for production deployment to Azure using the Azure Developer CLI.

🏗️ **Architecture Overview:**

This Products feature demonstrates a practical implementation of the AI architecture we've been building throughout these labs:

1. **Data Source Layer**: Product documentation stored as PDF files
2. **Vector Storage Layer**: Embedding vectors stored in Qdrant for semantic search  
3. **AI Processing Layer**: IChatClient interface providing access to AI models
4. **Application Layer**: Product service coordinating between database and AI
5. **Presentation Layer**: Blazor UI with filtering capabilities

This implementation highlights the power of combining traditional database capabilities with AI features. The AI does the heavy lifting of understanding document content, while the database provides efficient querying and filtering that would be hard to achieve using only vector search.

> **Note:** This lab uses PostgreSQL from the start to ensure production readiness. By using PostgreSQL directly in development, we avoid the need for database migrations when deploying to Azure, making the deployment process smoother and more reliable.
