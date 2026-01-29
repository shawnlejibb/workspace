# Architecture Overview: Gemini CLI Extension as an MCP Server

## What is a Gemini CLI Extension?

A Gemini CLI extension is a specialized program that extends the capabilities of the Gemini CLI (Command Line Interface) by providing additional tools and functionalities. This extension uses the **Model Context Protocol (MCP)** to communicate with the Gemini CLI agent.

## What is MCP (Model Context Protocol)?

The Model Context Protocol is a standardized communication protocol that allows AI assistants (like Gemini CLI) to interact with external tools and services. Think of it as a "plugin system" for AI agents.

### Key MCP Concepts:

- **MCP Server**: A program that exposes tools/capabilities to AI agents
- **MCP Client**: The AI agent (Gemini CLI) that uses those tools
- **Tools**: Individual functions that the AI can call (e.g., "create a document", "send an email")
- **Standardized Communication**: Uses JSON-RPC over stdin/stdout for reliable communication

## Is This Extension Locally or Remotely Hosted?

**This extension runs LOCALLY on your machine.** Here's the architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                      Your Local Machine                      │
│                                                               │
│  ┌──────────────┐         ┌────────────────────────┐        │
│  │  Gemini CLI  │◄───────►│  MCP Server (Local)    │        │
│  │   (Client)   │  stdio  │  workspace-server/     │        │
│  │              │         │  dist/index.js         │        │
│  └──────────────┘         └────────────────────────┘        │
│                                      │                        │
│                                      │                        │
│                              ┌───────▼────────┐              │
│                              │ Token Storage  │              │
│                              │ (Keychain/File)│              │
│                              └────────────────┘              │
└───────────────────────────────┬───────────────────────────────┘
                                │
                    OAuth Flow & Token Refresh
                                │
                                ▼
        ┌───────────────────────────────────────────┐
        │         Internet (HTTPS Only)              │
        │                                             │
        │  ┌─────────────────────────────────────┐  │
        │  │  Cloud Function (OAuth Handler)     │  │
        │  │  google-workspace-extension.        │  │
        │  │  geminicli.com                      │  │
        │  │  - Handles OAuth token exchange     │  │
        │  │  - Manages client secret securely   │  │
        │  └─────────────────────────────────────┘  │
        │                                             │
        │  ┌─────────────────────────────────────┐  │
        │  │  Google APIs                        │  │
        │  │  - Gmail, Drive, Docs, Calendar     │  │
        │  │  - All Workspace services           │  │
        │  └─────────────────────────────────────┘  │
        └───────────────────────────────────────────┘
```

### Execution Flow:

1. **Local Execution**: The MCP server runs as a Node.js process on your local machine
2. **Process Communication**: Gemini CLI and the MCP server communicate via stdin/stdout (standard input/output)
3. **No Remote Code Execution**: All business logic, data processing, and tool execution happens locally
4. **Direct API Calls**: The MCP server makes direct API calls to Google services using your OAuth tokens

## How the Cloud Function Fits In

The cloud function at `google-workspace-extension.geminicli.com` has a **very limited role**:

### What It Does:
- **OAuth Token Exchange**: Converts authorization codes to access tokens (OAuth 2.0 standard flow)
- **Token Refresh**: Refreshes expired access tokens using refresh tokens
- **Client Secret Management**: Stores the OAuth client secret securely (cannot be stored in client-side code)

### What It Does NOT Do:
- **Does NOT execute any of your commands**
- **Does NOT process your data**
- **Does NOT store your tokens or data**
- **Does NOT act as a proxy for Google API calls**
- **Does NOT see the content of your documents, emails, or calendar events**

### Why Is It Needed?

OAuth 2.0 requires a **client secret** that must be kept confidential. In a desktop application like this extension, we cannot securely embed the secret in the code (it would be visible to anyone). The cloud function:

1. Stores the client secret in Google Secret Manager (encrypted)
2. Performs token exchanges that require the secret
3. Immediately returns tokens to your local machine
4. Does not log or store any sensitive data

## Data Flow Example: Creating a Google Doc

Let's trace what happens when you ask Gemini CLI to "Create a new Google Doc":

1. **You**: "Create a new Google Doc titled 'My Report'"
2. **Gemini CLI**: Decides to use the `docs.create` tool from this extension
3. **MCP Server (Local)**: 
   - Receives the tool call via stdin
   - Checks if OAuth token is valid
   - If expired, calls cloud function to refresh token
   - Makes HTTPS request directly to `docs.googleapis.com` with your token
   - Creates the document
   - Returns document ID and URL to Gemini CLI via stdout
4. **Gemini CLI**: Shows you the result

**Key Point**: Your document content never passes through the cloud function. The flow is:
```
Your Machine → Google APIs (direct, HTTPS)
```

Not:
```
Your Machine → Cloud Function → Google APIs (this does NOT happen)
```

## MCP Server Lifecycle

### Installation
When you install the extension:
```bash
gemini extensions install https://github.com/gemini-cli-extensions/workspace
```

1. Gemini CLI clones the repository to a local directory
2. Runs `npm install` to install dependencies
3. Builds the TypeScript code with `npm run build`
4. Registers the extension in its configuration

### Running
When Gemini CLI starts:
1. Spawns the MCP server as a child process: `node workspace-server/dist/index.js`
2. Establishes stdin/stdout communication
3. The MCP server remains running in the background as long as Gemini CLI is active

### First Use (Authentication)
On first tool call:
1. MCP server detects no OAuth token
2. Generates OAuth URL with CSRF protection
3. Opens your browser to Google's OAuth consent screen
4. You authorize the application
5. Google redirects to the cloud function with an authorization code
6. Cloud function exchanges code for tokens (using the client secret)
7. Cloud function redirects back to `localhost` with tokens
8. MCP server receives tokens and stores them securely
9. MCP server makes the API call to Google

### Subsequent Uses
1. MCP server loads token from local storage
2. Checks if token is expired
3. If expired, calls cloud function to refresh
4. Makes API call to Google services
5. Returns result to Gemini CLI

## Security Architecture

### Threat Model

The architecture is designed to protect against:

1. **Credential Theft**: Tokens stored in OS keychain or encrypted file
2. **Man-in-the-Middle**: All communications use HTTPS with certificate validation
3. **CSRF Attacks**: OAuth flow uses cryptographic state tokens
4. **Open Redirect**: Cloud function validates redirect URIs
5. **Token Exposure**: Tokens never logged, never sent to untrusted parties
6. **Code Injection**: Input validation on all tool parameters

### Security Layers

#### Layer 1: Local Token Storage
- **Primary**: OS-native keychain (Keychain on macOS, Credential Manager on Windows, Secret Service on Linux)
- **Fallback**: Encrypted file with user-specific encryption key
- **Never**: Plain text files or environment variables

#### Layer 2: OAuth Security
- **PKCE Flow**: Not currently implemented (standard OAuth 2.0 used)
- **CSRF Protection**: Random state tokens validated on callback
- **Redirect Validation**: Only `localhost` and `127.0.0.1` allowed
- **Size Limits**: State parameter limited to 4KB to prevent DoS

#### Layer 3: Network Security
- **TLS/HTTPS Only**: All external communications encrypted
- **Certificate Validation**: Full certificate chain validation
- **No HTTP Fallback**: Application fails if HTTPS unavailable

#### Layer 4: API Permissions (Scopes)
The extension requests these Google OAuth scopes:
- `documents`: Read/write Google Docs
- `drive`: Access Google Drive files
- `calendar`: Manage calendar events
- `chat.*`: Send and receive Google Chat messages
- `gmail.modify`: Read and send emails (not full Gmail access)
- `presentations.readonly`: Read Google Slides
- `spreadsheets.readonly`: Read Google Sheets

**Principle of Least Privilege**: Only requests permissions actually needed

## Comparison: Local vs. Remote Architecture

| Aspect | This Extension (Local MCP) | Hypothetical Remote Service |
|--------|---------------------------|---------------------------|
| **Execution** | Runs on your machine | Runs on remote servers |
| **Data Access** | Direct to Google APIs | Via remote service |
| **Latency** | Low (local process) | Higher (network round-trip) |
| **Privacy** | Data never leaves your control | Service sees all data |
| **Trust** | Trust yourself + Google | Trust service provider + Google |
| **Offline Capability** | Partial (cached data) | None (requires connection) |
| **Updates** | Manual (`gemini extensions update`) | Automatic (service updates) |
| **Auditable** | Yes (open source code) | Depends on service |

## Summary

The Google Workspace extension for Gemini CLI is:

1. **A Local MCP Server**: Runs entirely on your machine as a Node.js process
2. **Not a Remote Service**: Does not execute commands or process data remotely
3. **Direct Google API Client**: Makes API calls directly to Google services
4. **Minimal Cloud Dependency**: Cloud function only handles OAuth token exchange (required by OAuth 2.0 specification)
5. **Privacy-Focused**: Your data flows directly between your machine and Google—nowhere else

The architecture prioritizes:
- **User Privacy**: Data stays on your machine and Google's services
- **Security**: Multiple layers of protection for credentials and communications
- **Transparency**: Open source code that you can audit
- **Simplicity**: Minimal external dependencies

For more details on privacy and security practices, see [PRIVACY.md](PRIVACY.md) and [SECURITY.md](SECURITY.md).
