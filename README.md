# Google Workspace Extension for Gemini CLI

[![Build Status](https://github.com/gemini-cli-extensions/workspace/actions/workflows/ci.yml/badge.svg)](https://github.com/gemini-cli-extensions/workspace/actions/workflows/ci.yml)

The Google Workspace extension for Gemini CLI brings the power of your Google Workspace apps to your command line. Manage your documents, spreadsheets, presentations, emails, chat, and calendar events without leaving your terminal.

## What is this extension?

This is a **local MCP (Model Context Protocol) server** that runs on your machine and enables Gemini CLI to interact with your Google Workspace data. Your data flows directly between your machine and Google's APIs—no intermediary services process your content.

**Key Points:**
- ✅ **Runs locally** on your computer (not a remote service)
- ✅ **Open source** and auditable code
- ✅ **Direct API calls** to Google services
- ✅ **Secure OAuth** token storage in your OS keychain
- ⚠️ **Privacy matters**: See [PRIVACY.md](PRIVACY.md) for details

**Want to understand more?** Read the [Architecture Overview](ARCHITECTURE.md) to learn how this extension works and how your data is protected.

## Prerequisites

Before using the Google Workspace extension, you need to be logged into your Google account.

## Installation

Install the Google Workspace extension by running the following command from your terminal:

```bash
gemini extensions install https://github.com/gemini-cli-extensions/workspace
```

## Usage

Once the extension is installed, you can use it to interact with your Google Workspace apps. Here are a few examples:

**Create a new Google Doc:**

> "Create a new Google Doc with the title 'My New Doc' and the content '# My New Document\n\nThis is a new document created from the command line.'"

**List your upcoming calendar events:**

> "What's on my calendar for today?"

**Search for a file in Google Drive:**

> "Find the file named 'my-file.txt' in my Google Drive."

## Commands

This extension provides a variety of commands. Here are a few examples:

### Get Schedule

**Command:** `/calendar:get-schedule [date]`

Shows your schedule for today or a specified date.

### Search Drive

**Command:** `/drive:search <query>`

Searches your Google Drive for files matching the given query.

## Resources

- [Documentation](docs/index.md): Detailed documentation on all the available tools.
- [Architecture Overview](ARCHITECTURE.md): **Understanding what this extension is and how it works**
- [Privacy Policy](PRIVACY.md): **How your data is protected and where it goes**
- [Security Policy](SECURITY.md): Security measures and best practices
- [GitHub Issues](https://github.com/gemini-cli-extensions/workspace/issues): Report bugs or request features.

## Important security consideration: Indirect Prompt Injection Risk

When exposing any language model to untrusted data, there's a risk of an [indirect prompt injection attack](https://en.wikipedia.org/wiki/Prompt_injection). Agentic tools like Gemini CLI, connected to MCP servers, have access to a wide array of tools and APIs.

This MCP server grants the agent the ability to read, modify, and delete your Google Account data, as well as other data shared with you.

* Never use this with untrusted tools
* Never include untrusted inputs into the model context. This includes asking Gemini CLI to process mail, documents, or other resources from unverified sources.
* Untrusted inputs may contain hidden instructions that could hijack your CLI session. Attackers can then leverage this to modify, steal, or destroy your data.
* Always carefully review actions taken by Gemini CLI on your behalf to ensure they are correct and align with your intentions.

**For more details**, see the [Security Policy](SECURITY.md) and [Privacy Policy](PRIVACY.md).

## Frequently Asked Questions (FAQ)

### Is this a remote service or does it run locally?

**This extension runs entirely on your local machine.** It's not a cloud service or remote server. When you install it, Gemini CLI downloads the code and runs it as a local Node.js process on your computer. Your data flows directly between your machine and Google's APIs—nowhere else.

See [ARCHITECTURE.md](ARCHITECTURE.md) for a detailed explanation with diagrams.

### Does the cloud function see my data?

**No, the cloud function does not see your documents, emails, or calendar data.** The cloud function at `google-workspace-extension.geminicli.com` has a very limited role: it only handles OAuth token exchange and refresh. This is required because OAuth 2.0 needs a "client secret" that must be kept confidential.

The cloud function:
- ✅ Exchanges OAuth codes for tokens (standard OAuth flow)
- ✅ Refreshes expired tokens
- ❌ Does NOT see your document contents
- ❌ Does NOT see your emails or calendar events
- ❌ Does NOT log or store your tokens
- ❌ Does NOT proxy your API calls

Your data flows directly: `Your Machine ←→ Google APIs`

See [PRIVACY.md](PRIVACY.md) for complete data flow details.

### Can my personal data leak to this extension?

Your data is as secure as your local machine and your Google account. The extension:

- **Stores tokens securely** in your OS keychain (Keychain on macOS, Credential Manager on Windows, Secret Service on Linux)
- **Uses HTTPS** for all communications with Google
- **Does not transmit** your data to any third parties
- **Is open source** so you can audit the code yourself

However, remember:
- ⚠️ If your machine is compromised (malware), tokens could be stolen
- ⚠️ If you process untrusted content (indirect prompt injection), the AI could be tricked
- ⚠️ Google can access your data per their [privacy policy](https://policies.google.com/privacy) (same as any Google Workspace usage)

See [PRIVACY.md](PRIVACY.md) for comprehensive privacy information.

### How do I know this is secure?

Security measures include:
- ✅ **Open source code**: Fully auditable on GitHub
- ✅ **Local execution**: No remote code execution
- ✅ **Secure token storage**: OS keychain or AES-256-GCM encryption
- ✅ **HTTPS only**: All network traffic encrypted
- ✅ **OAuth 2.0**: Industry-standard authentication
- ✅ **CSRF protection**: Random state tokens in OAuth flow
- ✅ **Input validation**: All parameters validated

Known limitations:
- ⚠️ Indirect prompt injection (inherent AI risk—see warning above)
- ⚠️ Cloud function trust (cannot verify deployed code matches source)
- ⚠️ No PKCE yet (standard OAuth flow used)

See [SECURITY.md](SECURITY.md) for the complete security policy.

### What permissions does this extension need?

The extension requests these Google OAuth scopes:
- **Documents**: Read and write your Google Docs
- **Drive**: Access files and folders in your Drive
- **Calendar**: View and manage calendar events
- **Chat**: Send and receive Google Chat messages
- **Gmail**: Read, send, and modify emails (not full Gmail access)
- **Slides**: Read presentations (read-only)
- **Sheets**: Read spreadsheets (read-only)
- **User Profile**: Your name and email

You authorize these when you first use the extension. You can revoke access anytime at [Google Account Permissions](https://myaccount.google.com/permissions).

### How do I revoke access?

You can revoke this extension's access in two ways:

1. **Via the extension**: Run `/auth:clear` command in Gemini CLI
2. **Via Google**: Go to [Google Account Permissions](https://myaccount.google.com/permissions), find "Google Workspace Extension for Gemini CLI", and click "Remove Access"

After revoking, the extension cannot access your data until you re-authenticate.

### Is this compliant with GDPR/privacy regulations?

The extension is designed with privacy in mind:
- **You are the data controller** of your own Google Workspace data
- **Google is the data processor** for data stored in their services
- **The extension acts as your agent**, executing commands on your behalf
- **No third-party data sharing** except with Google (which you already use)
- **Right to access**: All code is open source
- **Right to erasure**: Use `/auth:clear` or revoke in Google settings

For GDPR compliance of your Google Workspace usage, see Google's compliance documentation.

### What happens if the extension is compromised?

If malicious code is added to the extension:
1. **Open source protection**: Community can review and flag malicious changes
2. **Manual updates**: You control when to update (not automatic)
3. **Token revocation**: Revoke access immediately at [Google Account Permissions](https://myaccount.google.com/permissions)
4. **Limited blast radius**: Only affects users who update to the compromised version

**Your response plan**:
1. Revoke access in Google settings
2. Run `/auth:clear` to clear local tokens
3. Review account activity at [My Activity](https://myactivity.google.com/)
4. Report to Google Security: [https://g.co/vulnz](https://g.co/vulnz)

### Can I use this with my work/enterprise Google account?

Yes, but check with your IT administrator first. Some organizations restrict third-party app access. The extension:
- Uses standard Google OAuth
- Requests only necessary API scopes
- Runs locally (not a SaaS service)
- Is open source (can be audited by your security team)

Your IT admin may need to approve the application in Google Workspace Admin Console.

### Does this work offline?

Partially. The extension requires internet access to:
- Authenticate with Google OAuth
- Refresh tokens (every ~1 hour)
- Make API calls to Google services

However:
- The MCP server itself runs locally
- If you have valid, non-expired tokens, you can make API calls
- Google Workspace data is not cached locally (by design for security)

### How do I update the extension?

Update to the latest version:
```bash
gemini extensions update workspace
```

Check for security updates regularly. Critical security patches will be noted in [release notes](docs/release_notes.md).

## Contributing

Contributions are welcome! Please read the [CONTRIBUTING.md](CONTRIBUTING.md) file for details on how to contribute to this project.

## 📄 Legal

- **License**: [Apache License 2.0](LICENSE)
- **Terms of Service**: [Terms of Service](https://policies.google.com/terms)
- **Privacy Policy**: [Privacy Policy](https://policies.google.com/privacy)
- **Security**: [Security Policy](SECURITY.md)

