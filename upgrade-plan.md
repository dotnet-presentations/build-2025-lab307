# Upgrade Plan for New Template Version

This document outlines the changes needed to update the lab instructions to match the new template version that removes the SQLite database dependency for document ingestion cache. The ingestion cache is now handled directly in the vector database using Microsoft Extensions for Vector Data.

## Part 0: Setup

- [x] No changes required

## Part 1: Create Project

- [x] No changes required (template creation process remains the same)

## Part 2: Explore Template

- [x] Remove SQLite ingestion cache references from AppHost Program.cs code examples
- [x] Update AppHost Program.cs to show only Qdrant vector database and OpenAI connection
- [x] Remove `var ingestionCache = builder.AddSqlite("ingestionCache");` line
- [x] Remove ingestionCache references and WaitFor dependencies from webApp configuration
- [x] Update Web Program.cs code examples to remove `IngestionCacheDbContext` registration
- [x] Remove `builder.AddSqliteDbContext<IngestionCacheDbContext>("ingestionCache");` line
- [x] Remove `IngestionCacheDbContext.Initialize(app.Services);` line
- [x] Update DataIngestor service explanation to reflect new vector-based approach
- [x] Update DataIngestor.cs code examples to show new vector collection-based implementation
- [x] Remove references to Entity Framework and SQLite in the ingestion process explanation
- [x] Update vector data section to explain new `VectorStoreCollection` usage pattern
- [x] Add explanation of how `IngestedChunk` and `IngestedDocument` are now stored directly in vector database
- [x] Update service registration to show new vector collection services:
  - `builder.Services.AddQdrantCollection<Guid, IngestedChunk>("data-genailab-chunks")`
  - `builder.Services.AddQdrantCollection<Guid, IngestedDocument>("data-genailab-documents")`

## Part 3: Azure OpenAI

- [x] No changes required (AI service configuration remains the same)

## Part 4: Products Page

- [x] Remove SQLite ingestion cache references from AppHost configuration section
- [x] Update AppHost Program.cs example to remove `ingestionCache` variable and references
- [x] Update final AppHost Program.cs to show only productDb database (no ingestionCache)
- [x] Remove references to `IngestionCacheDbContext.Initialize(app.Services);` from Web Program.cs
- [x] Update the "What you've accomplished" section to remove mentions of ingestion cache database
- [x] Update database configuration explanation to focus only on productDb for the products feature
- [x] Switch from SQLite to PostgreSQL directly in Part 4 to avoid migration in Part 5
- [x] Update AppHost to use `builder.AddPostgres("postgres")` and `postgres.AddDatabase("productDb")`
- [x] Update Web Program.cs to use `builder.AddNpgsqlDbContext<ProductDbContext>("productDb")`
- [x] Add PostgreSQL NuGet package installation instructions
- [x] Update ProductService.cs to include improved JSON parsing that handles AI responses with markdown formatting
- [x] Add explanation of JSON cleaning logic that removes markdown code blocks (`\`\`\`json` and `\`\`\``)
- [x] Include updated system prompt that explicitly requests "valid JSON only, no markdown formatting or backticks"
- [x] Update error handling examples to show robust JSON deserialization with fallback values

## Part 5: Deploy to Azure

- [x] Remove SQLite to PostgreSQL migration explanation (no longer needed since Part 4 uses PostgreSQL)
- [x] Remove database migration section entirely
- [x] Update deployment process to focus on Azure Container Apps configuration
- [x] Simplify deployment considerations since database setup is already PostgreSQL
- [x] Update cost estimation section to remove migration-related costs

## Cross-cutting Changes

- [x] Update all Mermaid diagrams to remove ingestion cache database components
- [x] Update architecture explanations to clarify that document tracking is now handled in vector database
- [x] Remove any troubleshooting sections related to SQLite or ingestion cache database connectivity
- [x] Update performance considerations to reflect vector-only storage approach
- [x] Review and update any code comments that reference the old ingestion cache approach

## New Concepts to Add

- [x] Explain Microsoft Extensions for Vector Data collection pattern
- [x] Document how `VectorStoreCollection<TKey, TRecord>` replaces Entity Framework for ingestion tracking
- [x] Explain benefits of vector-native storage for document ingestion cache
- [x] Add explanation of `IngestedChunk` and `IngestedDocument` models as vector records
- [x] Document the simplified architecture without separate cache database

## Technical Details to Address

- [x] Update constructor injection examples for DataIngestor to show new dependencies
- [x] Replace Entity Framework queries with vector collection operations
- [x] Update async patterns to use vector collection methods (`GetAsync`, `UpsertAsync`, `DeleteAsync`)
- [x] Show new service registration pattern for vector collections
- [x] Update error handling examples to reflect vector operations instead of database operations
