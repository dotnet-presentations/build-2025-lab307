# Deploy to Azure

## In this lab

In this lab, you will learn how to deploy your AI application to Azure using the Azure Developer CLI (`azd`). You'll deploy your PostgreSQL-based application to Azure Container Apps for production use.

> [!TIP]
> If you haven't completed the previous steps in the lab or are having trouble with your code, you can use the `/src/complete` folder which already includes all the necessary changes. The complete code has already been updated with the PostgreSQL configuration and external HTTP endpoints setup described in this section. You can skip directly to the "Set Up the Azure Developer CLI" section and deploy that code instead.

## Configure the web application for external access

  Before the web application is deployed to Azure Container Apps, you will need to configure it so that it is available via web browser. Update **AppHost** `Program.cs` to add the following line just before the call to `builder.Build().Run();` at the end of the file:

  ```csharp
  webApp.WithExternalHttpEndpoints();
  ```

## Set Up the Azure Developer CLI

1. **Install the Azure Developer CLI (azd)**:

   If you don't already have the Azure Developer CLI installed, you can install it with:

   ```powershell
   winget install microsoft.azd
   ```

   Or using PowerShell:

   ```powershell
   irm https://aka.ms/install-azd.ps1 | iex
   ```

1. Close and re-open the terminal to make sure *azd* has been added to the path.

1. **Login to Azure**:

   ```powershell
   azd auth login
   ```

## Deploy to Azure Container Apps

1. Ensure you are in the root directory which contains the solution file.

1. **Initialize your Azure environment**:

   ```powershell
   # Initialize the application for managment with azd
   azd init
   ```

1. When prompted with "How do you wnat to initializer your app?", select the default: "Use code in the current directory"

1. After scanning the directory, `azd` prompts you to confirm that it found the correct .NET Aspire *AppHost* project. Select the **Confirm and continue initializing my app** option.

1. When prompted to "Enter a unique environment name", enter "mygenaiapp" or choose something else if you would like.

> [!WARNING]
> **FOR BUILD 2025 LAB ATTENDEES**: You MUST enter exactly **"mygenaiapp"** as your environment name. This is because azd will automatically add the "rg-" prefix when creating the resource group. The lab environment has already been configured with "rg-mygenaiapp" as the resource group, and permissions are restricted to only allow creating resources in this resource group.

1. **Provision Azure resources**:

   ```powershell
   azd provision
   ```

   This command creates all the necessary Azure resources, including:
   - Resource group
   - Container registry
   - Container apps environment
   - Container apps for your application
   - Log Analytics workspace

> [!NOTE]
> When provisioning resources with `azd`, it will automatically create a resource group with the prefix "rg-" added to your environment name (e.g., "rg-mygenaiapp"). For Build 2025 lab attendees, this is why it's essential to use exactly "mygenaiapp" as your environment name.
  
1. When prompted to "Enter a value for the 'azureAISearch' infrastructure secured parameter, copy and past the value from your `secrets.json` file. It will begin with "Endpoint=" and end with your search key. Make sure that you grab your Azure AI Search connection string and not the Azure OpenAI connection string!

1. When prompted to select a location, select "West US 3" for Build 2025 attendees (or another nearby Azure datacenter if you're following this lab outside of the conference).

> [!WARNING]
> **FOR BUILD 2025 LAB ATTENDEES**: You MUST select "West US 3" as your location. This region has been pre-configured with the necessary quotas for your lab environment.

1. When prompted to "Enter a value for the 'openai' infrastructure secured parameter, copy and past the value from your `secrets.json` file. As before, it will begin with "Endpoint=" and end with your Azure OpenAI key.

1. Press enter and watch as your resources are provisioned! You can either just follow along in the terminal, or you can click on the link to watch the progress in the Azure portal. Provisioning should take roughly 5 minutes, but may take longer during conference events as multiple concurrent deployments can slow things down.

1. **Deploy your application code**:

   ```powershell
   azd deploy
   ```

   This command:
   - Builds your .NET application
   - Creates container images
   - Pushes them to the Azure Container Registry
   - Deploys them to Azure Container Apps
  
   This should take roughly 2 minutes, but may take longer under busy conditions.

1. **Access your deployed application**:

   After deployment completes, you'll receive a URL to access your application in the terminal output. You can also view it using:

   ```powershell
   azd show
   ```

## Manage Your Deployment

Once deployed, you can manage your deployment using various Azure Developer CLI commands:

1. **View deployment information**:

   ```powershell
   azd show
   ```

   This command shows your deployment details, including endpoints and resource information. Launch the link for the *aichatweb-app** service and verify that it is continuing to run as it did locally.

1. **Monitor your application**:

   ```powershell
   azd monitor
   ```

   This opens the Application Insights dashboard for your application, where you can view logs, metrics, and performance data.

1. **Update your deployment**:

   After making changes to your application:

   ```powershell
   azd deploy
   ```

1. **Delete your deployment**:

   To completely clean up all resources when you're done:

   ```powershell
   azd down --purge --force
   ```

## Production Considerations

### Security Best Practices

1. **Secure your API keys**:
   - Use Azure Key Vault for storing API keys and secrets
   - Never hardcode keys in your application code
   - Rotate keys periodically

1. **Implement proper authentication and authorization**:
   - Add authentication to your application
   - Protect API endpoints
   - Consider identity providers like Azure AD

1. **Use HTTPS everywhere**:
   - Enable HTTPS for all endpoints
   - Configure proper CORS policies

### Scaling and Performance

1. **Configure scaling rules in Azure Container Apps**:
   - Set minimum and maximum replicas
   - Configure scaling metrics based on load

1. **Implement caching for AI responses**:
   - Use distributed caching (Redis)
   - Cache common AI-generated content

1. **Optimize network communication**:
   - Use gRPC for internal service communication
   - Configure appropriate timeouts

### Cost Management

1. **Monitor AI service usage**:
   - Track token usage with telemetry
   - Set up cost alerts and budgets

1. **Optimize embedding generation**:
   - Only generate embeddings when necessary
   - Cache embedding results

1. **Configure appropriate instance sizes**:
   - Start with smaller instances and scale up as needed
   - Use autoscaling to optimize costs

## What You've Learned

- How to use the Azure Developer CLI (azd) to deploy your AI application
- How to set up and configure Azure Container Apps for production workloads
- How to manage and monitor your deployed application
- Best practices for security, scaling, and cost management in production

## Conclusion

Congratulations! You've completed all parts of the AI Web Chat template lab. You now have the knowledge to:

1. Create AI applications using the AI Web Chat template
2. Understand and customize the template code structure
3. Migrate from GitHub Models to Azure OpenAI
4. Implement AI-powered features like the Products page using PostgreSQL
5. Deploy your application to production environments using Azure

Continue exploring the possibilities of AI with .NET and build amazing AI-powered applications!
