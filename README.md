<p align="center">
<img src="img/banner.jpg" alt="decorative banner" width="1200"/>
</p>

# BUILD25 LAB307 - Building GenAI Apps in C#: AI Templates, GitHub Models, Azure OpenAI & More

Get up to speed quickly with AI app building in .NET! Explore the new .NET AI project templates integrated with Microsoft Extensions for AI (MEAI), GitHub Models, and vector data stores. Learn how to take advantage of free GitHub Models in development, then deploy with global scale and enterprise support using Azure OpenAI. Gain hands-on experience building cutting-edge intelligent solutions with state-of-the-art frameworks and best practices.

## Prerequisites

- Visual Studio 2022 with .NET Aspire workload installed
- .NET AI Web Chatbot template installed
- .NET 9.0 SDK or later
- Azure OpenAI subscription (optional, but recommended for full experience)
- GitHub Copilot subscription (optional, but recommended for full experience)

## Lab Overview

The lab consists of a series of hands-on exercises where you'll build an AI-powered web application using the new .NET AI project templates. The application includes:

- **AI Chatbot**: A conversational interface that can answer questions about products
- **Product Catalog**: AI-generated product descriptions and categories
- **Semantic Search**: Vector-based search using document embeddings
- **Integration with GitHub Models and Azure OpenAI**: Use free models for development and enterprise-grade models for production

## Key Technologies

- **.NET 9**: The latest version of .NET
- **Microsoft Extensions for AI (MEAI)**: Libraries for integrating AI capabilities into .NET applications
- **Blazor**: For building interactive web UIs
- **.NET Aspire**: For orchestrating cloud-native distributed applications
- **GitHub Models**: Free AI models for development
- **Azure OpenAI**: Enterprise-grade AI models for production
- **Qdrant Vector Database**: For storing and searching vector embeddings

## Getting Started

Follow the [setup instructions](lab/part0-setup.md) to get started with the lab.

## Lab Modules

The lab is divided into five modules:

1. [**Create a Project with AI Web Chat Template**](lab/part1-create-project.md): Learn how to create an AI-powered web application using the .NET AI Web Chat template.

1. [**Explore the Template Code**](lab/part2-explore-template.md): Explore the structure of an AI Web Chat project, including core components like vector embeddings, semantic search, and chat interfaces.

1. [**Convert from GitHub Models to Azure OpenAI**](lab/part3-azure-openai.md): Migrate your application from GitHub Models to Azure OpenAI for production-ready AI capabilities.

1. [**Write a Products Page**](lab/part4-products-page.md): Enhance your application with a new feature that uses AI to generate product information.

1. [**Deploy to Azure**](lab/part5-deploy-azure.md): Deploy your application to Azure using Azure Developer CLI.

## Lab Structure

The repository is structured as follows:

- `/lab`: Contains all the lab instructions and documentation
- `/src/start`: Contains the starting code for the lab exercises
- `/src/complete`: Contains the completed solution after all lab exercises


## Session Resources 

| Resources          | Links                             | Description        |
|:-------------------|:----------------------------------|:-------------------|
| Build session page | https://build.microsoft.com/sessions/lab307 | Event session page with downloadable recording, slides, resources, and speaker bio |
|Microsoft Learn|https://aka.ms/build25/plan/ADAI_DevStartPlan|Official Collection or Plan with skilling resources to learn at your own pace|
|Microsoft Learn|https://learn.microsoft.com/en-us/dotnet/machine-learning/ai-overview|.NET AI Documentation|
|Microsoft Learn|https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview|.NET Aspire Documentation|
|Microsoft Learn|https://learn.microsoft.com/en-us/dotnet/machine-learning/extensions-ai/|Microsoft Extensions for AI Documentation|
|Microsoft Learn|https://learn.microsoft.com/en-us/azure/ai-services/openai/|Azure OpenAI Documentation|


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
