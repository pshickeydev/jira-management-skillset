# Setup Guide

This guide walks you through setting up everything needed to manage Jira
issues using AI-powered skills from your terminal or desktop.

## What This Project Does

This project provides a set of **skills** for
[OpenCode](https://opencode.ai), an open-source AI coding agent. Skills
are packaged instructions that teach the AI agent how to perform specific
tasks — in this case, creating, updating, searching, transitioning,
linking, and bulk-managing Jira issues through natural language
conversation.

Under the hood, the agent talks to Jira through the **Atlassian Rovo MCP
server**. MCP (Model Context Protocol) is a standard that lets AI agents
securely connect to external services. The Rovo MCP server is hosted by
Atlassian and gives the agent read/write access to your Jira data using
your existing Atlassian permissions.

To power the AI, OpenCode needs access to a **large language model
(LLM)** — a service like Anthropic Claude, OpenAI GPT, Google Gemini,
or a locally-run model via Ollama. You provide API credentials for one
of these services, and OpenCode handles the rest.

### How it fits together

```
You  ──▶  OpenCode (AI agent)  ──▶  LLM Provider (Anthropic, Ollama, etc.)
                │
                ├── Skills (this repo: jira-create, jira-search, etc.)
                │
                └── Rovo MCP Server  ──▶  Your Jira instance
```

### Key terms

| Term | What it means |
|------|---------------|
| **OpenCode** | The AI coding agent you interact with. Available as a desktop app, terminal app (TUI), or IDE extension. |
| **Skill** | A set of instructions that teaches OpenCode how to perform a task. Skills live in `.agents/skills/` and are loaded automatically when you open the project. |
| **Command** | A shortcut (like `/jira-create`) you type in OpenCode to invoke a skill. Commands live in `.opencode/commands/`. |
| **MCP server** | A service that gives OpenCode access to external data. The Atlassian Rovo MCP server connects OpenCode to Jira. |
| **LLM provider** | The AI model service that powers OpenCode's responses. You need at least one (cloud-based or local). |
| **`opencode.json`** | The configuration file for OpenCode. It specifies which providers and MCP servers to use. Placed in the project root or `~/.config/opencode/`. |

## Prerequisites

Before you begin, make sure you have:

- An Atlassian Cloud site with access to at least one Jira project
- Git

---

## Step 1: Install OpenCode

OpenCode is the AI coding agent that runs the skills in this project.
It is available as a terminal-based interface (TUI), a desktop app, or
an IDE extension. Choose the interface that suits your workflow.

### OpenCode Desktop (desktop app)

Download the desktop app from
[opencode.ai](https://opencode.ai). It is available for macOS, Linux,
and Windows. The desktop app bundles the OpenCode CLI as a sidecar
process — no separate CLI installation is needed.

*(Note: Do not open a project directory in the app just yet. Complete
the configuration steps below first, and you will clone and open the
repository in Step 5).*

### OpenCode CLI (terminal)

Install the CLI if you prefer working in a terminal.

#### Install script (recommended)

```bash
curl -fsSL https://opencode.ai/install | bash
```

#### npm (requires Node.js)

```bash
npm install -g opencode-ai
```

#### Homebrew (macOS / Linux)

```bash
brew install anomalyco/tap/opencode
```

#### Fedora (binary install)

```bash
ARCH=$(uname -m)
case "$ARCH" in
  x86_64)  ARCH="x64" ;;
  aarch64) ARCH="arm64" ;;
esac

VERSION=$(curl -fsSL https://api.github.com/repos/anomalyco/opencode/releases/latest \
  | grep '"tag_name"' | head -1 | sed 's/.*"v\(.*\)".*/\1/')

curl -fsSL "https://github.com/anomalyco/opencode/releases/download/v${VERSION}/opencode-linux-${ARCH}.tar.gz" \
  | tar xz -C /tmp

sudo mv /tmp/opencode /usr/local/bin/opencode
sudo chmod +x /usr/local/bin/opencode
```

### Other methods

| Platform       | Command                                      |
|----------------|----------------------------------------------|
| Arch Linux     | `sudo pacman -S opencode`                    |
| Chocolatey     | `choco install opencode`                     |
| Scoop          | `scoop install opencode`                     |
| Docker         | `docker run -it --rm ghcr.io/anomalyco/opencode` |

Binaries are also available on the
[GitHub Releases page](https://github.com/anomalyco/opencode/releases).

> **Windows users:** For the best experience, use
> [Windows Subsystem for Linux (WSL)](https://opencode.ai/docs/windows-wsl).

Full installation docs: <https://opencode.ai/docs/#install>

---

## Step 2: Configure an LLM Provider

OpenCode needs access to a large language model (LLM) to power its AI
capabilities. An LLM is the AI service that reads your requests and
generates responses — without one, OpenCode has no "brain."

You must configure at least one provider. Choose from the options below
based on what you have access to.

> **How OpenCode commands work:** Inside the OpenCode interface, you
> type `/` followed by a command name. For example, `/connect` opens the
> provider setup wizard, and `/models` shows available AI models. These
> are not shell commands — they are typed in the OpenCode prompt.

### Option A: Anthropic (API key)

Best for users with an Anthropic API key.

1. Sign up at <https://console.anthropic.com/> and create an API key.
2. Launch OpenCode and run `/connect`:

    ```
    /connect
    ```

3. Select **Anthropic**, then choose **Manually enter API Key**.
4. Paste your API key and press Enter.
5. Run `/models` to select a model (Claude Sonnet 4 recommended).

### Option B: Google Cloud / Vertex AI

Best for users with a Google Cloud project. This approach allows using
enterprise credentials rather than a personal API key.

1. **Install the Google Cloud SDK:**

   **macOS:**
   ```bash
   brew install google-cloud-sdk
   ```

   **Fedora:**
   ```bash
   sudo tee /etc/yum.repos.d/google-cloud-sdk.repo > /dev/null <<'EOF'
   [google-cloud-cli]
   name=Google Cloud CLI
   baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64
   enabled=1
   gpgcheck=1
   repo_gpgcheck=0
   gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
   EOF

   sudo dnf install -y libxcrypt-compat google-cloud-cli
   ```

2. **Authenticate:**

   ```bash
   gcloud auth login
   gcloud auth application-default login
   ```

3. **Set your project** (find the ID in
   [Google Cloud Console](https://console.cloud.google.com)):

   ```bash
   gcloud config set project YOUR_GCP_PROJECT_ID
   ```

4. **Add environment variables** to your shell profile (`~/.zshrc` for
   macOS/zsh, or `~/.bashrc`/`~/.bash_profile` for bash):

   ```bash
   # ── Vertex AI provider config ──
   export ANTHROPIC_VERTEX_PROJECT_ID=YOUR_GCP_PROJECT_ID
   export GOOGLE_CLOUD_PROJECT="$ANTHROPIC_VERTEX_PROJECT_ID"
   export VERTEX_LOCATION="global"
   export GOOGLE_CLOUD_LOCATION="$VERTEX_LOCATION"
   ```

   Then reload your profile:

   ```bash
   source ~/.zshrc   # or: source ~/.bashrc
   ```

   > The `global` region improves availability and reduces errors at no
   > extra cost. Use a specific region (e.g. `us-central1`) only if you
   > have data residency requirements.

5. **Configure OpenCode** (optional — only needed if models don't appear
   automatically). Add a `provider` section to your
   `~/.config/opencode/opencode.json`:

   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "provider": {
       "google-vertex": {
         "options": {
           "project": "{env:GOOGLE_CLOUD_PROJECT}",
           "location": "global"
         }
       },
       "google-vertex-anthropic": {
         "options": {
           "project": "{env:GOOGLE_CLOUD_PROJECT}",
           "location": "global"
         }
       }
     }
   }
   ```

6. Launch OpenCode, run `/models`, and select a model.

Full Vertex AI docs: <https://opencode.ai/docs/providers/#google-vertex-ai>

### Option C: OpenAI / ChatGPT Plus

Best for users with a ChatGPT Plus or Pro subscription or an OpenAI API key.

1. Launch OpenCode and run `/connect`:

   ```
   /connect
   ```

2. Select **OpenAI**.
3. Choose **ChatGPT Plus/Pro** (opens browser for authentication) or
   **Manually enter API Key** if you have one.
4. Run `/models` to select a model.

### Option D: Ollama (local, free)

Run models locally on your own hardware with
[Ollama](https://ollama.com). No API key or cloud account required.

1. **Install Ollama:**

   Download from <https://ollama.com/download>, or on Linux:

   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

2. **Pull a model:**

   ```bash
   ollama pull gemma4:26b
   ```

   Browse available models at <https://ollama.com/search>. For coding
   tasks, `gemma4:26b` is a good choice if your hardware can handle it.
   Smaller options like `gemma4:e4b` work on machines with less RAM/VRAM.

3. **Verify Ollama is running:**

   ```bash
   ollama list
   ```

   Ollama serves an OpenAI-compatible API at
   `http://localhost:11434/v1` by default.

4. **Configure OpenCode** by creating or editing `opencode.json` in
   your project root (or `~/.config/opencode/opencode.json` for global
   config):

   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "provider": {
       "ollama": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "Ollama (local)",
         "options": {
           "baseURL": "http://localhost:11434/v1"
         },
         "models": {
           "gemma4:26b": {
             "name": "Gemma 4 26B (local)"
           }
         }
       }
     }
   }
   ```

   Replace `gemma4:26b` with whatever model you pulled. The model ID
   must match what `ollama list` shows.

5. Launch OpenCode, run `/models`, and select your Ollama model.

> **Tip:** If tool calls are not working well, try increasing Ollama's
> context window: set `num_ctx` to 16384 or higher, or choose a model
> with strong tool-calling support.

Full Ollama docs: <https://docs.ollama.com/quickstart>
OpenCode Ollama config: <https://opencode.ai/docs/providers/#ollama>

### Option E: Other Providers

OpenCode supports 75+ providers including Amazon Bedrock, Azure OpenAI,
DeepSeek, Groq, OpenRouter, GitHub Copilot, and many more. See the full
directory at <https://opencode.ai/docs/providers/>.

---

## Step 3: Set Up the Atlassian Rovo MCP Server

As described in [What This Project Does](#what-this-project-does), the
Rovo MCP server is the bridge between OpenCode and your Jira data. It is
hosted by Atlassian — you do not need to install or run anything
locally. You just need to point OpenCode at it and sign in with your
Atlassian account.

### What you need

- An **Atlassian Cloud** site (e.g. `yoursite.atlassian.net`) with Jira
- Permissions to access the Jira projects you want to manage
- A modern browser to complete the OAuth authentication flow

### 3a. Add the MCP server to OpenCode

Create or edit the `opencode.json` file in your project root (or
`~/.config/opencode/opencode.json` for global config):

**Minimal config (Jira MCP only):**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "jira": {
      "type": "remote",
      "url": "https://mcp.atlassian.com/v1/mcp/authv2"
    }
  }
}
```

> **Note:** The MCP server URL is `https://mcp.atlassian.com/v1/mcp/authv2`.
> The older `/v1/sse` endpoint is deprecated. Always use the
> Atlassian-hosted Rovo MCP server — do not use local `uvx`/`mcp-atlassian`
> packages (that approach is no longer supported).

### 3b. Authenticate

Launch OpenCode in this project directory:

```bash
opencode
```

OpenCode will detect the MCP server configuration. When the agent first
tries to use a Jira tool (or when you manually trigger auth), it will
start an OAuth 2.1 flow:

1. A browser window will open to Atlassian's consent screen.
2. Sign in with your Atlassian account.
3. Grant the requested permissions (Jira, Confluence, etc.).
4. The browser will redirect back and OpenCode will store the token.

You can also manually trigger authentication:

```bash
opencode mcp auth jira
```

And check authentication status:

```bash
opencode mcp list
```

### 3c. Verify the connection

Once authenticated, ask OpenCode a question that uses Jira:

```
What Jira projects do I have access to?
```

If the MCP server is working, you will see a response listing your
projects.

### Troubleshooting MCP authentication

| Issue                  | Cause                              | Fix                                              |
|------------------------|------------------------------------|--------------------------------------------------|
| OAuth flow won't start | Pop-up blocker or CLI error        | Disable pop-up blockers, re-run `opencode mcp auth jira` |
| Redirect fails         | Blocked localhost or network issue | Check firewall/proxy settings                    |
| Access denied          | Insufficient Jira permissions      | Verify product access with your Atlassian admin  |
| No data returned       | Token expired or wrong scopes     | Re-authenticate with `opencode mcp auth jira`    |
| `Unauthorized` with no sign-in prompt | Client may not support OAuth dynamic client registration | Check OpenCode version is up to date; see the [Rovo MCP troubleshooting guide](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/troubleshooting-and-verifying-your-setup/) |

> **Important:** Always use the Atlassian-hosted Rovo MCP server URL
> (`https://mcp.atlassian.com/v1/mcp/authv2`). Do not use local
> `uvx`/`mcp-atlassian` packages — that approach is no longer supported.

Full Rovo MCP docs:
- Getting started: <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/getting-started-with-the-atlassian-remote-mcp-server/>
- OAuth configuration: <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/configuring-oauth-2-1/>
- Setting up clients: <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/setting-up-clients/>
- Troubleshooting: <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/troubleshooting-and-verifying-your-setup/>

---

## Step 4: Disable Session Sharing and the OpenCode Zen Provider

By default, OpenCode allows sharing conversations as public links (via
the `/share` command), and automatically loads OpenCode's own hosted LLM
provider ("Zen") if credentials are available. If you are working with
sensitive or proprietary data, you should disable both to ensure no
conversation content leaves your chosen provider.

Add the following to your `opencode.json` (project-level) or
`~/.config/opencode/opencode.json` (global). If the file already has
content, make sure you merge these keys into the existing `{ }` braces
rather than pasting the whole block at the end:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "share": "disabled",
  "disabled_providers": ["opencode"]
  // ... other configuration like "mcp" goes here
}
```

- **`"share": "disabled"`** prevents all conversation sharing. The
  `/share` command will not work. No conversation data is sent to
  OpenCode's servers. This is recommended for enterprise environments
  and any project handling confidential data.

    > **Note on sharing:** If you need to share a conversation for
    > debugging or collaboration but cannot use the public link feature,
    > you should export the session locally. Do **not** use the in-session
    > `/export` command, as it simply dumps the raw markdown. Instead,
    > exit OpenCode and use the CLI to export and automatically redact
    > sensitive transcript/file data:
    >
    > ```bash
    > opencode export --sanitize
    > ```

- **`"disabled_providers": ["opencode"]`** prevents the OpenCode Zen
  provider from loading. Zen models will not appear in the `/models` picker. This
  ensures all LLM traffic goes exclusively through your chosen provider
  (Vertex AI, Anthropic, Ollama, etc.).

> **Tip:** To restrict OpenCode to *only* your chosen providers (instead
> of selectively disabling), use `"enabled_providers"` instead. Inside your
> `opencode.json`, add:
>
> ```json
> "enabled_providers": ["google-vertex", "google-vertex-anthropic"]
> ```
>
> This ignores all other providers regardless of configured credentials.
> Note that `disabled_providers` takes priority over `enabled_providers`
> if both are set.

Full docs:
- Sharing: <https://opencode.ai/docs/share/>
- Config reference: <https://opencode.ai/docs/config/>
- Provider management: <https://opencode.ai/docs/config/#disabled-providers>

---

## Step 5: Clone This Repository

Clone this repository. It contains all the skills, commands, templates,
and configuration — everything OpenCode needs to manage your Jira
issues.

```bash
git clone https://github.com/pshickeydev/jira-management-skillset.git
cd jira-management-skillset
```

When you open this directory in OpenCode, it automatically discovers:
- **Skills** in `.agents/skills/` — the instructions that teach OpenCode
  how to create issues, search, transition, etc.
- **Commands** in `.opencode/commands/` — the `/jira-create`,
  `/jira-search`, etc. shortcuts you type in the OpenCode prompt.
- **`opencode.json`** — your provider and MCP server configuration.

There is nothing to symlink or install separately.

- **OpenCode Desktop:** Launch the app and use the directory picker to
  open this cloned directory as your project.
- **OpenCode TUI:** Run `opencode` from inside the cloned directory.

```bash
opencode
```

---

## Step 6: Configure the Skillset

Now that OpenCode is installed, connected to an LLM, authenticated with
Jira, and pointed at this repository, the last step is to run the
configuration wizard. This teaches the skills about *your* specific Jira
setup — which projects you work on, what issue types are available, and
what defaults to use.

Type this command in the OpenCode prompt:

```
/configure-jira-skillset
```

Or simply ask OpenCode in plain language:

```
Set up Jira for this project.
```

The wizard will walk you through:

1. Finding your Jira site automatically via the MCP server connection.
2. Selecting which Jira projects you want to manage.
3. Discovering the issue types, security levels, priorities, and custom
   fields available in those projects.
4. Setting your preferred defaults (e.g. default priority, labels,
   whether to auto-assign issues to yourself).
5. Generating description templates for each issue type in `templates/`.
6. Saving everything to `config.json` (this file is gitignored since it
   contains instance-specific data).

After configuration, you can use all the skills:

| Command                     | What it does                           |
|-----------------------------|----------------------------------------|
| `/jira-create`              | Create a new issue                     |
| `/jira-update`              | Edit an existing issue                 |
| `/jira-transition`          | Change issue status                    |
| `/jira-search`              | Search issues with JQL or plain text   |
| `/jira-link`                | Link two issues together               |
| `/jira-bulk`                | Batch operations across multiple issues|

---

## Quick Reference

### Verify everything is working

```bash
# Check OpenCode is installed
opencode --version

# Check Ollama is running (if using local models)
ollama list

# Check MCP auth status
opencode mcp list

# Launch OpenCode and test
opencode
```

Inside the OpenCode prompt (these are OpenCode commands, not shell
commands):

```
# Show available AI models (verifies your provider is connected)
/models

# Test the Jira MCP connection by asking a plain-language question
What are my open Jira issues?

# Run the skillset configuration wizard
/configure-jira-skillset
```

### Key files

| File                  | Purpose                                       |
|-----------------------|-----------------------------------------------|
| `opencode.json`       | OpenCode configuration (providers, MCP servers) |
| `config.json`         | Skillset configuration (generated, gitignored) |
| `config.example.json` | Schema reference for `config.json`             |
| `AGENTS.md`           | Operational guide for the agent skills         |
| `templates/`          | Issue description templates                    |

### Documentation links

- **OpenCode:** <https://opencode.ai/docs/>
- **OpenCode providers:** <https://opencode.ai/docs/providers/>
- **OpenCode MCP servers:** <https://opencode.ai/docs/mcp-servers/>
- **Atlassian Rovo MCP server:** <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/use-atlassian-rovo-mcp-server/>
- **Rovo MCP OAuth setup:** <https://support.atlassian.com/atlassian-rovo-mcp-server/docs/configuring-oauth-2-1/>
