---
sidebar_label: "Developer's Guide: Providers & Deployments"
sidebar_position: 99
title: "Developer's Guide: Setting Up AI Providers and Deployments"
description: A comprehensive developer reference for configuring AI providers, connections, deployments, and chat profiles within the CrestApps OrchardCore architecture.
---

# Developer's Guide: Setting Up AI Providers and Deployments

This guide walks you through every step needed to configure AI providers and deployments in a local clone of the CrestApps.OrchardCore repository. It covers all four supported providers, multi-type deployments, recipe-based provisioning, and practical use-case scenarios with code samples drawn directly from the codebase.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Core Concepts](#core-concepts)
4. [Provider Setup: OpenAI](#provider-setup-openai)
5. [Provider Setup: Azure OpenAI](#provider-setup-azure-openai)
6. [Provider Setup: Ollama (Local LLMs)](#provider-setup-ollama-local-llms)
7. [Provider Setup: Azure AI Inference (GitHub Models)](#provider-setup-azure-ai-inference-github-models)
8. [Multi-Type Deployments](#multi-type-deployments)
9. [Creating Chat Profiles](#creating-chat-profiles)
10. [Recipe-Based Provisioning](#recipe-based-provisioning)
11. [Aspire-Based Local Development](#aspire-based-local-development)
12. [End-to-End Use-Case Walkthroughs](#end-to-end-use-case-walkthroughs)
13. [Troubleshooting](#troubleshooting)
14. [Quick Reference](#quick-reference)

---

## Prerequisites

### 1. Install .NET 10.0 SDK

The repository targets `net10.0`. Verify with `dotnet --version` (should show `10.0.x`).

```bash
# Ubuntu/Debian
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb && rm packages-microsoft-prod.deb
sudo apt-get update && sudo apt-get install -y dotnet-sdk-10.0
```

The `global.json` in the repository root pins the SDK:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestMajor"
  }
}
```

### 2. Install Node.js (15+)

Required for building frontend assets:

```bash
npm install      # Install dependencies
npm run rebuild  # Build all frontend assets
```

### 3. Clone and Build

```bash
git clone https://github.com/CrestApps/CrestApps.OrchardCore.git
cd CrestApps.OrchardCore

npm install && npm run rebuild
dotnet build
```

### 4. NuGet Package Sources

The `NuGet.config` includes the Orchard Core preview feed:

```xml
<packageSources>
  <add key="NuGet" value="https://api.nuget.org/v3/index.json" />
  <add key="OrchardCorePreview"
       value="https://nuget.cloudsmith.io/orchardcore/preview/v3/index.json" />
</packageSources>
```

### 5. Run the CMS

```bash
cd src/Startup/CrestApps.OrchardCore.Cms.Web
dotnet run
# Visit http://localhost:5000 and complete the Orchard Core setup wizard
```

---

## Architecture Overview

The AI system is organized in three layers:

```
┌─────────────────────────────────────────────────┐
│                  AI Profiles                     │
│   (Chat, Utility, TemplatePrompt, Agent)         │
│   References: ChatDeploymentId, UtilityDeploymentId
├─────────────────────────────────────────────────┤
│                AI Deployments                    │
│   (Typed: Chat, Utility, Embedding, Image, etc.) │
│   References: ConnectionName, ClientName         │
├─────────────────────────────────────────────────┤
│            AI Provider Connections               │
│   (OpenAI, Azure, Ollama, AzureAIInference)      │
│   Holds: Endpoint, ApiKey, AuthenticationType    │
└─────────────────────────────────────────────────┘
```

**Key relationships:**

- A **Connection** holds provider credentials (API key, endpoint).
- A **Deployment** maps a model name (e.g., `gpt-4o`) to a connection and declares what types it supports (Chat, Embedding, etc.).
- A **Profile** references specific deployments and adds chat behavior (system prompt, welcome message, temperature).

---

## Core Concepts

### AIDeploymentType (Flags Enum)

```csharp
// src/Abstractions/CrestApps.OrchardCore.AI.Abstractions/Models/AIDeploymentType.cs

[Flags]
public enum AIDeploymentType
{
    None        = 0,
    Chat        = 1 << 0,   // 1
    Utility     = 1 << 1,   // 2
    Embedding   = 1 << 2,   // 4
    Image       = 1 << 3,   // 8
    SpeechToText = 1 << 4,  // 16
    TextToSpeech = 1 << 5,  // 32
}
```

Because it's a `[Flags]` enum, a single deployment can support multiple types simultaneously using bitwise OR. The extension methods in `AIDeploymentTypeExtensions.cs` provide helpers:

```csharp
// src/Abstractions/CrestApps.OrchardCore.AI.Abstractions/Models/AIDeploymentTypeExtensions.cs

public static bool Supports(this AIDeploymentType value, AIDeploymentType type)
    => type != AIDeploymentType.None && (value & type) == type;

public static bool IsValidSelection(this AIDeploymentType value)
    => value != AIDeploymentType.None && (value & ~_allSupportedTypes) == 0;

public static IEnumerable<AIDeploymentType> GetSupportedTypes(this AIDeploymentType value)
    => Enum.GetValues<AIDeploymentType>().Where(type => value.Supports(type));
```

### Provider Client Names (Constants)

Each provider is identified by a constant `ClientName`:

| Provider | ClientName Constant | Source File |
|----------|-------------------|-------------|
| OpenAI | `"OpenAI"` | `src/Core/CrestApps.OrchardCore.OpenAI.Core/OpenAIConstants.cs` |
| Azure OpenAI | `"Azure"` | `src/Core/CrestApps.OrchardCore.OpenAI.Azure.Core/AzureOpenAIConstants.cs` |
| Ollama | `"Ollama"` | `src/Modules/CrestApps.OrchardCore.Ollama/OllamaConstants.cs` |
| Azure AI Inference | `"AzureAIInference"` | `src/Modules/CrestApps.OrchardCore.AzureAIInference/AzureAIInferenceConstants.cs` |

### Feature IDs

Enable these features in the Orchard Core admin dashboard:

| Feature | Feature ID |
|---------|-----------|
| AI Services (base) | `CrestApps.OrchardCore.AI` |
| AI Connection Management | `CrestApps.OrchardCore.AI.ConnectionManagement` |
| AI Chat Services | `CrestApps.OrchardCore.AI.Chat.Core` |
| AI Chat UI | `CrestApps.OrchardCore.AI.Chat` |
| AI Chat WebAPI | `CrestApps.OrchardCore.AI.Chat.Api` |
| OpenAI Provider | `CrestApps.OrchardCore.OpenAI` |
| Azure OpenAI Provider | `CrestApps.OrchardCore.OpenAI.Azure` |
| Ollama Provider | `CrestApps.OrchardCore.Ollama` |
| Azure AI Inference | `CrestApps.OrchardCore.AzureAIInference` |

---

## Provider Setup: OpenAI

### Feature ID

`CrestApps.OrchardCore.OpenAI`

### Configuration via `appsettings.json`

Add to `src/Startup/CrestApps.OrchardCore.Cms.Web/appsettings.json`:

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "OpenAI": {
          "DefaultConnectionName": "openai-cloud",
          "Connections": {
            "openai-cloud": {
              "ApiKey": "sk-your-api-key-here",
              "Deployments": [
                { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
                { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true },
                { "Name": "text-embedding-3-large", "Type": "Embedding", "IsDefault": true },
                { "Name": "dall-e-3", "Type": "Image", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

### Configuration via Recipe (AI Connection Management)

```json
{
  "steps": [
    {
      "name": "AIProviderConnections",
      "connections": [
        {
          "Source": "OpenAI",
          "Name": "openai-cloud",
          "IsDefault": true,
          "DisplayText": "OpenAI Cloud",
          "Deployments": [
            { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
            { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true },
            { "Name": "text-embedding-3-large", "Type": "Embedding", "IsDefault": true },
            { "Name": "dall-e-3", "Type": "Image", "IsDefault": true }
          ],
          "Properties": {
            "OpenAIConnectionMetadata": {
              "Endpoint": "https://api.openai.com/v1",
              "ApiKey": "sk-your-api-key-here"
            }
          }
        }
      ]
    }
  ]
}
```

### OpenAI-Compatible Providers

The OpenAI module supports any provider with an OpenAI-compatible API. Configure by changing the `Endpoint`:

#### DeepSeek

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "OpenAI": {
          "DefaultConnectionName": "deepseek",
          "Connections": {
            "deepseek": {
              "Endpoint": "https://api.deepseek.com/v1",
              "ApiKey": "your-deepseek-api-key",
              "Deployments": [
                { "Name": "deepseek-chat", "Type": "Chat", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

#### Google Gemini

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "OpenAI": {
          "DefaultConnectionName": "google-gemini",
          "Connections": {
            "google-gemini": {
              "Endpoint": "https://generativelanguage.googleapis.com/v1",
              "ApiKey": "your-google-api-key",
              "Deployments": [
                { "Name": "gemini-pro", "Type": "Chat", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

### How It's Registered (Startup.cs)

```csharp
// src/Modules/CrestApps.OrchardCore.OpenAI/Startup.cs

public sealed class Startup : StartupBase
{
    public override void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<IAIClientProvider, OpenAIClientProvider>();
        services.AddAIProfile<OpenAICompletionClient>(
            OpenAIConstants.ImplementationName,
            OpenAIConstants.ClientName, o =>
        {
            o.DisplayName = S["OpenAI"];
            o.Description = S["Provides AI profiles using OpenAI."];
        });

        services.AddAIDeploymentProvider(OpenAIConstants.ClientName, o =>
        {
            o.DisplayName = S["OpenAI"];
            o.Description = S["OpenAI model deployments."];
        });
    }
}
```

---

## Provider Setup: Azure OpenAI

### Feature ID

`CrestApps.OrchardCore.OpenAI.Azure`

### Configuration via `appsettings.json`

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "Azure": {
          "DefaultConnectionName": "my-azure-account",
          "Connections": {
            "my-azure-account": {
              "Endpoint": "https://my-account.openai.azure.com/",
              "AuthenticationType": "ApiKey",
              "ApiKey": "your-azure-api-key",
              "Deployments": [
                { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
                { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true },
                { "Name": "text-embedding-ada-002", "Type": "Embedding", "IsDefault": true },
                { "Name": "dall-e-3", "Type": "Image", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

### Authentication Types

Azure OpenAI supports two authentication types:

| Type | Description |
|------|-------------|
| `ApiKey` | Traditional API key authentication |
| `ManagedIdentity` | Azure Managed Identity (no API key needed) |

For Managed Identity, omit the `ApiKey` and set:

```json
{
  "AuthenticationType": "ManagedIdentity",
  "IdentityId": "optional-user-assigned-managed-identity-client-id"
}
```

### Connection Metadata Model

```csharp
// src/Core/CrestApps.OrchardCore.OpenAI.Azure.Core/Models/AzureOpenAIConnectionMetadata.cs

public class AzureOpenAIConnectionMetadata : AzureConnectionMetadata
{
    public bool EnableLogging { get; set; }
}

// src/Core/CrestApps.Azure.Core/Models/AzureConnectionMetadata.cs
public class AzureConnectionMetadata
{
    public Uri Endpoint { get; set; }
    public AzureAuthenticationType AuthenticationType { get; set; }
    public string ApiKey { get; set; }
    public string IdentityId { get; set; }
}
```

### Configuration via Recipe

```json
{
  "steps": [
    {
      "name": "AIProviderConnections",
      "connections": [
        {
          "Source": "Azure",
          "Name": "my-azure-account",
          "IsDefault": true,
          "DisplayText": "Azure OpenAI",
          "Deployments": [
            { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
            { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true }
          ],
          "Properties": {
            "AzureOpenAIConnectionMetadata": {
              "Endpoint": "https://my-account.openai.azure.com/",
              "AuthenticationType": "ApiKey",
              "ApiKey": "your-azure-api-key",
              "EnableLogging": false
            }
          }
        }
      ]
    }
  ]
}
```

---

## Provider Setup: Ollama (Local LLMs)

### Feature ID

`CrestApps.OrchardCore.Ollama`

### Prerequisites

Install [Ollama](https://ollama.com/) and pull a model:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model (e.g., deepseek-v2)
ollama pull deepseek-v2:16b

# Verify it's running
ollama list
```

### Configuration via `appsettings.json`

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "Ollama": {
          "DefaultConnectionName": "Default",
          "DefaultDeploymentName": "deepseek-v2:16b",
          "Connections": {
            "Default": {
              "Endpoint": "http://localhost:11434",
              "ChatDeploymentName": "deepseek-v2:16b"
            }
          }
        }
      }
    }
  }
}
```

### Configuration via Environment Variables (Aspire)

This is how the Aspire AppHost configures Ollama (from `src/Startup/CrestApps.Aspire.AppHost/Program.cs`):

```csharp
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_AI__Providers__Ollama__DefaultConnectionName", "Default");
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_AI__Providers__Ollama__DefaultDeploymentName", ollamaModelName);
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_AI__Providers__Ollama__Connections__Default__Endpoint",
    "http://localhost:11434");
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_AI__Providers__Ollama__Connections__Default__ChatDeploymentName",
    ollamaModelName);
```

---

## Provider Setup: Azure AI Inference (GitHub Models)

### Feature ID

`CrestApps.OrchardCore.AzureAIInference`

### Configuration via `appsettings.json`

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "AzureAIInference": {
          "DefaultConnectionName": "github-models",
          "Connections": {
            "github-models": {
              "Endpoint": "https://models.inference.ai.azure.com",
              "AuthenticationType": "ApiKey",
              "ApiKey": "your-github-token",
              "Deployments": [
                { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

### Connection Metadata Model

```csharp
// src/Modules/CrestApps.OrchardCore.AzureAIInference/Models/AzureAIInferenceConnectionMetadata.cs

public sealed class AzureAIInferenceConnectionMetadata : AzureConnectionMetadata
{
}
```

### Configuration via Recipe

```json
{
  "steps": [
    {
      "name": "AIProviderConnections",
      "connections": [
        {
          "Source": "AzureAIInference",
          "Name": "github-models",
          "IsDefault": true,
          "DisplayText": "GitHub Models",
          "Deployments": [
            { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true }
          ],
          "Properties": {
            "AzureAIInferenceConnectionMetadata": {
              "Endpoint": "https://models.inference.ai.azure.com",
              "AuthenticationType": "ApiKey",
              "ApiKey": "your-github-token"
            }
          }
        }
      ]
    }
  ]
}
```

---

## Multi-Type Deployments

A key feature of the deployment system is that a single deployment can support multiple capability types. This eliminates duplication when one model handles both chat and embeddings.

### Configuration via `appsettings.json`

Use a JSON array for the `Type` field:

```json
{
  "Deployments": [
    {
      "Name": "gpt-4o",
      "Type": ["Chat", "Utility"],
      "IsDefault": true
    },
    {
      "Name": "text-embedding-3-large",
      "Type": "Embedding",
      "IsDefault": true
    }
  ]
}
```

### Configuration via Recipe

The `AIDeployment` recipe step supports both string and array formats for `Type`:

```json
{
  "steps": [
    {
      "name": "AIDeployment",
      "deployments": [
        {
          "Name": "gpt-4o",
          "Type": ["Chat", "Utility"],
          "IsDefault": true,
          "ClientName": "OpenAI",
          "ConnectionName": "openai-cloud"
        }
      ]
    }
  ]
}
```

### How It Works in Code

The recipe step handler parses both formats:

```csharp
// src/Modules/CrestApps.OrchardCore.AI/Recipes/AIDeploymentStep.cs

private static bool TryGetDeploymentType(JsonNode typeNode, out AIDeploymentType type)
{
    type = AIDeploymentType.None;

    if (typeNode is JsonArray array)
    {
        foreach (var item in array)
        {
            if (Enum.TryParse<AIDeploymentType>(item.GetValue<string>(),
                ignoreCase: true, out var parsedType))
            {
                type |= parsedType;  // Bitwise OR to combine types
            }
        }
        return type.IsValidSelection();
    }

    // Single string format
    var typeValue = typeNode.GetValue<string>();
    return Enum.TryParse(typeValue, ignoreCase: true, out type) &&
           type.IsValidSelection();
}
```

### Checking Type Support at Runtime

```csharp
// Query if a deployment supports a specific type
if (deployment.SupportsType(AIDeploymentType.Chat))
{
    // This deployment can handle chat completions
}

// Get all types a deployment supports
var types = deployment.Type.GetSupportedTypes();
// e.g., [Chat, Utility] for a combined deployment
```

---

## Creating Chat Profiles

Profiles define the chat behavior and reference specific deployments.

### Profile Model

```csharp
// src/Abstractions/CrestApps.OrchardCore.AI.Abstractions/Models/AIProfile.cs

public sealed class AIProfile : CatalogItem
{
    public string Name { get; set; }
    public string DisplayText { get; set; }
    public AIProfileType Type { get; set; }          // Chat, Utility, TemplatePrompt, Agent
    public string ChatDeploymentId { get; set; }     // References an AIDeployment
    public string UtilityDeploymentId { get; set; }  // Optional fallback deployment
    public string WelcomeMessage { get; set; }
    public string PromptTemplate { get; set; }
    public AISessionTitleType? TitleType { get; set; }
    public JsonObject Settings { get; init; } = [];
}
```

### Creating a Profile via Recipe

```json
{
  "steps": [
    {
      "name": "AIProfile",
      "profiles": [
        {
          "Name": "my-assistant",
          "DisplayText": "My AI Assistant",
          "Type": "Chat",
          "WelcomeMessage": "Hello! How can I help you today?",
          "TitleType": "InitialPrompt",
          "ChatDeploymentId": "gpt-4o",
          "UtilityDeploymentId": "gpt-4o-mini",
          "Properties": {
            "AIProfileMetadata": {
              "SystemMessage": "You are a helpful AI assistant for our CMS platform.",
              "Temperature": 0.7,
              "TopP": null,
              "MaxTokens": 2048
            }
          }
        }
      ]
    }
  ]
}
```

---

## Recipe-Based Provisioning

Recipes allow you to automate the entire AI stack setup. Execute them via:

- **Admin UI**: Configuration → Recipes → Execute
- **Setup**: Include in the setup recipe during initial site provisioning

### Complete Stack Recipe

This recipe sets up a connection, deployments, and a chat profile in one step:

```json
{
  "steps": [
    {
      "name": "feature",
      "enable": [
        "CrestApps.OrchardCore.OpenAI",
        "CrestApps.OrchardCore.AI.ConnectionManagement",
        "CrestApps.OrchardCore.AI.Chat"
      ]
    },
    {
      "name": "AIProviderConnections",
      "connections": [
        {
          "Source": "OpenAI",
          "Name": "openai-cloud",
          "IsDefault": true,
          "DisplayText": "OpenAI Cloud",
          "Properties": {
            "OpenAIConnectionMetadata": {
              "Endpoint": "https://api.openai.com/v1",
              "ApiKey": "sk-your-api-key"
            }
          }
        }
      ]
    },
    {
      "name": "AIDeployment",
      "deployments": [
        {
          "Name": "gpt-4o",
          "Type": ["Chat", "Utility"],
          "IsDefault": true,
          "ClientName": "OpenAI",
          "ConnectionName": "openai-cloud"
        },
        {
          "Name": "text-embedding-3-large",
          "Type": "Embedding",
          "IsDefault": true,
          "ClientName": "OpenAI",
          "ConnectionName": "openai-cloud"
        }
      ]
    },
    {
      "name": "AIProfile",
      "profiles": [
        {
          "Name": "default-chat",
          "DisplayText": "Default Chat Assistant",
          "Type": "Chat",
          "WelcomeMessage": "How can I assist you?",
          "TitleType": "InitialPrompt",
          "ChatDeploymentId": "gpt-4o",
          "Properties": {
            "AIProfileMetadata": {
              "SystemMessage": "You are a helpful assistant.",
              "Temperature": 0.7
            }
          }
        }
      ]
    }
  ]
}
```

### Recipe Step Reference

| Step Name | Purpose | Key Fields |
|-----------|---------|------------|
| `AIProviderConnections` | Create/update provider connections | `Source`, `Name`, `Properties` |
| `AIDeployment` | Create/update typed deployments | `ClientName`, `ConnectionName`, `Type`, `IsDefault` |
| `AIProfile` | Create/update chat profiles | `ChatDeploymentId`, `UtilityDeploymentId`, `Type` |

---

## Aspire-Based Local Development

The repository includes an Aspire AppHost for orchestrating local development with Ollama, Redis, and the CMS.

### Running with Aspire

```bash
cd src/Startup/CrestApps.Aspire.AppHost
dotnet run
```

### What Aspire Configures

From `src/Startup/CrestApps.Aspire.AppHost/Program.cs`:

```csharp
const string ollamaModelName = "deepseek-v2:16b";

// Ollama container with GPU support
var ollama = builder.AddOllama("Ollama")
    .WithDataVolume()
    .WithGPUSupport()
    .WithHttpEndpoint(port: 11434, targetPort: 11434, name: "HttpOllama");
ollama.AddModel(ollamaModelName);

// Redis for caching
var redis = builder.AddRedis("Redis");

// Main CMS with AI configuration injected via environment variables
var orchardCore = builder.AddProject<Projects.CrestApps_OrchardCore_Cms_Web>("OrchardCoreCMS")
    .WithHttpsEndpoint(5001, name: "HttpsOrchardCore")
    .WithEnvironment((options) =>
    {
        // Ollama provider configuration
        options.EnvironmentVariables.Add(
            "OrchardCore__CrestApps_AI__Providers__Ollama__DefaultConnectionName", "Default");
        options.EnvironmentVariables.Add(
            "OrchardCore__CrestApps_AI__Providers__Ollama__Connections__Default__Endpoint",
            "http://localhost:11434");
        options.EnvironmentVariables.Add(
            "OrchardCore__CrestApps_AI__Providers__Ollama__Connections__Default__ChatDeploymentName",
            ollamaModelName);

        // MCP and A2A auth disabled for local development
        options.EnvironmentVariables.Add(
            "OrchardCore__CrestApps_AI__McpServer__AuthenticationType", "None");
        options.EnvironmentVariables.Add(
            "OrchardCore__CrestApps_AI__A2AHost__AuthenticationType", "None");
    });
```

### Copilot BYOK (Bring Your Own Key) with Aspire

The Aspire setup also supports configuring the Copilot orchestrator with a local Ollama model:

```csharp
// Uncomment these lines in Program.cs to use Ollama for Copilot
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_OrchardCore_AI_Chat_Copilot__AuthenticationType", "ApiKey");
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_OrchardCore_AI_Chat_Copilot__ProviderType", "openai");
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_OrchardCore_AI_Chat_Copilot__BaseUrl",
    "http://localhost:11434/v1");
options.EnvironmentVariables.Add(
    "OrchardCore__CrestApps_OrchardCore_AI_Chat_Copilot__DefaultModel", ollamaModelName);
```

---

## End-to-End Use-Case Walkthroughs

### Use Case 1: RAG Chatbot with OpenAI

**Goal:** Build a chatbot that can search through uploaded documents using embeddings and answer questions with GPT-4o.

**Step 1:** Configure `appsettings.json`

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "OpenAI": {
          "DefaultConnectionName": "openai-cloud",
          "Connections": {
            "openai-cloud": {
              "ApiKey": "sk-your-api-key",
              "Deployments": [
                { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
                { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true },
                { "Name": "text-embedding-3-large", "Type": "Embedding", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

**Step 2:** Enable features in Admin → Features:
- `CrestApps.OrchardCore.OpenAI`
- `CrestApps.OrchardCore.AI.Chat`
- `CrestApps.OrchardCore.AI.ConnectionManagement`
- `CrestApps.OrchardCore.AI.Documents` (for document indexing)

**Step 3:** Create a chat profile via Admin → AI → Profiles → New, or via recipe as shown above.

The Embedding deployment handles document vectorization. The Chat deployment handles user queries. The Utility deployment handles auxiliary tasks like query rewriting and title generation.

---

### Use Case 2: Cost-Optimized Multi-Type Deployment

**Goal:** Use a single `gpt-4o-mini` model for both Chat and Utility tasks to reduce costs and configuration overhead.

```json
{
  "Deployments": [
    {
      "Name": "gpt-4o-mini",
      "Type": ["Chat", "Utility"],
      "IsDefault": true
    }
  ]
}
```

Or via recipe:

```json
{
  "steps": [
    {
      "name": "AIDeployment",
      "deployments": [
        {
          "Name": "gpt-4o-mini",
          "Type": ["Chat", "Utility"],
          "IsDefault": true,
          "ClientName": "OpenAI",
          "ConnectionName": "openai-cloud"
        }
      ]
    }
  ]
}
```

One deployment, one API key, one model — handles both chat completions and auxiliary tasks. When the profile needs a Chat deployment, it resolves `gpt-4o-mini`. When it needs a Utility deployment (e.g., for title generation), it also resolves `gpt-4o-mini`.

---

### Use Case 3: Local Development with Ollama

**Goal:** Run AI chat locally without cloud API keys using Ollama.

**Step 1:** Install and start Ollama, pull a model:

```bash
ollama pull llama3.2
ollama serve  # Runs on http://localhost:11434
```

**Step 2:** Configure `appsettings.json`:

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "Ollama": {
          "DefaultConnectionName": "Default",
          "DefaultDeploymentName": "llama3.2",
          "Connections": {
            "Default": {
              "Endpoint": "http://localhost:11434",
              "ChatDeploymentName": "llama3.2"
            }
          }
        }
      }
    }
  }
}
```

**Step 3:** Enable the Ollama feature in Admin → Features → `CrestApps.OrchardCore.Ollama`.

---

### Use Case 4: Multi-Tenant CMS with Azure OpenAI

**Goal:** Share one Azure OpenAI resource across multiple Orchard Core tenants, each with their own connection configuration.

Each tenant gets its own `appsettings.json` override (or tenant-scoped configuration):

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "Azure": {
          "DefaultConnectionName": "tenant-a",
          "Connections": {
            "tenant-a": {
              "Endpoint": "https://shared-resource.openai.azure.com/",
              "AuthenticationType": "ManagedIdentity",
              "Deployments": [
                { "Name": "gpt-4o", "Type": ["Chat", "Utility"], "IsDefault": true },
                { "Name": "text-embedding-ada-002", "Type": "Embedding", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

With multi-type deployments, each tenant only needs two deployment entries instead of four, simplifying per-tenant provisioning.

---

### Use Case 5: Hybrid Provider Setup

**Goal:** Use Azure OpenAI for production chat and OpenAI for image generation.

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "Providers": {
        "Azure": {
          "DefaultConnectionName": "production-azure",
          "Connections": {
            "production-azure": {
              "Endpoint": "https://prod.openai.azure.com/",
              "AuthenticationType": "ApiKey",
              "ApiKey": "azure-api-key",
              "Deployments": [
                { "Name": "gpt-4o", "Type": "Chat", "IsDefault": true },
                { "Name": "gpt-4o-mini", "Type": "Utility", "IsDefault": true },
                { "Name": "text-embedding-ada-002", "Type": "Embedding", "IsDefault": true }
              ]
            }
          }
        },
        "OpenAI": {
          "DefaultConnectionName": "openai-images",
          "Connections": {
            "openai-images": {
              "ApiKey": "sk-openai-key",
              "Deployments": [
                { "Name": "dall-e-3", "Type": "Image", "IsDefault": true }
              ]
            }
          }
        }
      }
    }
  }
}
```

This lets you use Azure for secured, enterprise-grade chat and embeddings while using OpenAI directly for image generation (which may not be available in your Azure region).

---

## Troubleshooting

### Build Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `NU1301` about `nuget.cloudsmith.io` | Network access restricted | Expected in sandboxed environments; focus on asset builds |
| SDK version mismatch | Wrong .NET SDK | Install .NET 10.0.100+ |
| `npm install` failures | Wrong Node.js version | Use Node.js 15+ |

### Runtime Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Provider not appearing in dropdowns | Feature not enabled | Enable the provider feature in Admin → Features |
| Connection management UI missing | Feature not enabled | Enable `CrestApps.OrchardCore.AI.ConnectionManagement` |
| "Unable to find a provider-source" in recipes | Wrong `Source` / `ClientName` | Use exact constant: `OpenAI`, `Azure`, `Ollama`, `AzureAIInference` |
| Deployment type defaults to Chat | `Type` field missing in recipe | Always include `Type` in deployment recipes |

### Configuration Key Format

When using environment variables, convert JSON paths with double underscores:

```
appsettings.json path:
OrchardCore:CrestApps_AI:Providers:OpenAI:DefaultConnectionName

Environment variable:
OrchardCore__CrestApps_AI__Providers__OpenAI__DefaultConnectionName
```

---

## Quick Reference

### Deployment Type Values

| Type | Value | Use For |
|------|-------|---------|
| `Chat` | `1` | Primary conversational AI |
| `Utility` | `2` | Lightweight tasks (title generation, query rewriting) |
| `Embedding` | `4` | Vector embeddings for RAG / semantic search |
| `Image` | `8` | Image generation (DALL-E, etc.) |
| `SpeechToText` | `16` | Audio transcription |
| `TextToSpeech` | `32` | Speech synthesis |

### Provider Source Names for Recipes

| Provider | `Source` / `ClientName` value |
|----------|-------------------------------|
| OpenAI (and compatibles) | `OpenAI` |
| Azure OpenAI | `Azure` |
| Ollama | `Ollama` |
| Azure AI Inference | `AzureAIInference` |

### Connection Metadata Property Names

| Provider | Properties Key |
|----------|---------------|
| OpenAI | `OpenAIConnectionMetadata` |
| Azure OpenAI | `AzureOpenAIConnectionMetadata` |
| Azure AI Inference | `AzureAIInferenceConnectionMetadata` |

### Default AI Parameters

You can set global defaults in `appsettings.json`:

```json
{
  "OrchardCore": {
    "CrestApps_AI": {
      "DefaultParameters": {
        "Temperature": 0,
        "MaxOutputTokens": 800
      }
    }
  }
}
```

---

*This guide references code from the CrestApps.OrchardCore repository. For the latest API changes, consult the source files and the [changelog](../changelog/v2.0.0.md).*
