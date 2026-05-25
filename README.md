# mcp-atlassian — Intel Internal Edition

Fork of [ravindren-sm/mcp-atlassian-intel](https://github.com/ravindren-sm/mcp-atlassian-intel) with a patch
for the consumption of the Confluence PAT..

## What's Different from Upstream

The Confluence PAT will be obtained from the Authorization token instead of be an environment variable.
Adding some services that will automate the process for install, uninstall, start and stop the server. 

Intel's corporate proxy (`proxy-chain.intel.com:911`) intercepts outbound HTTPS. The standard
`mcp-atlassian` server ignores the `NO_PROXY` environment variable when proxies are explicitly
configured on the session, routing internal traffic through the proxy and failing.

This fork patches `src/mcp_atlassian/utils/ssl.py` to add:

- **`NoProxyAdapter`** — respects `NO_PROXY` even when proxies are explicitly set on the session
- **`configure_proxy_bypass()`** — mounts the adapter for a given service URL
- **`SSLIgnoreAdapter`** now inherits from `NoProxyAdapter` so SSL bypass and proxy bypass work together

---

## Prerequisites

- **Python 3.10+** — verify with `python --version`
- **Git** — verify with `git --version`
- **VS Code** with the **GitHub Copilot** extension installed and active
- Intel network or VPN access to `wiki.ith.intel.com`

---

## Let Copilot Do the Rest

Use **GitHub Copilot agent mode** to complete Steps 2–5 automatically.

> **How to open agent mode**: In VS Code press `Ctrl+Shift+I` (or click the Copilot icon in the Activity Bar), then select **Agent** from the mode dropdown at the top of the chat panel.

Paste the prompt below into the agent chat and press Enter. Copilot will run each command in the terminal and create the required files — approve each step when prompted.

```
Set up the mcp-atlassian-intel server on my Windows machine. Do the following steps in order:

1. Install uv with: pip install uv --proxy="http://proxy-chain.intel.com:911"
2. Configure git proxy: git config --global http.proxy http://proxy-chain.intel.com:911
3. Clone https://github.com/ravindren-sm/mcp-atlassian-intel and cd into it
4. Install the package with: uv tool install --from . mcp-atlassian
5. Find the exe path with: where mcp-atlassian
6. Create C:\Users\<my-username>\start-atlassian-mcp.vbs with CONFLUENCE_URL=https://wiki.ith.intel.com, CONFLUENCE_SSL_VERIFY=false, NO_PROXY=wiki.ith.intel.com, launching the exe with --transport streamable-http --port 9002
7. From the repo directory, run: Set-ExecutionPolicy -Scope CurrentUser RemoteSigned (if needed), then cd service\windows and run .\install.ps1, .\start.ps1, .\status.ps1
8. Add the MCP server to VS Code User Settings (settings.json) under the "mcp.servers" key with type "http" and url "http://localhost:9002/mcp"
```

---

## Step 2 — Install the Server

**Install `uv`** (the package manager used to install and run the server):

```bash
pip install uv --proxy="http://proxy-chain.intel.com:911"
```

**Configure git to use the Intel proxy** (required for cloning from GitHub on corporate network):

```bash
git config --global http.proxy http://proxy-chain.intel.com:911
```

**Clone and install:**

```bash
git clone https://github.com/ravindren-sm/mcp-atlassian-intel.git
cd mcp-atlassian-intel
uv tool install --from . mcp-atlassian
```

> First install downloads ~120 packages and takes 1-2 minutes.

**Find the installed executable path** — you'll need this in Step 3:

```bash
where mcp-atlassian
# Example output: C:\Users\ravindre\.local\bin\mcp-atlassian.exe
```

---

## Step 3 — Create the Startup Script

Create a file called **`start-atlassian-mcp.vbs`** in your home directory
(`C:\Users\<your-username>\start-atlassian-mcp.vbs`).

```vbs
Dim WshShell
Set WshShell = CreateObject("WScript.Shell")

' Set credentials only for this process — not system-wide
WshShell.Environment("Process")("CONFLUENCE_URL") = "https://wiki.ith.intel.com"
WshShell.Environment("Process")("CONFLUENCE_SSL_VERIFY") = "false"
WshShell.Environment("Process")("NO_PROXY") = "wiki.ith.intel.com"

' Replace the path below with the output of: where mcp-atlassian
' Launch the server silently on port 9002 (0 = hidden window, False = don't wait)
WshShell.Run """C:\Users\<your-username>\.local\bin\mcp-atlassian.exe""" --transport streamable-http --port 9002", 0, False

Set WshShell = Nothing
```

Replace the exe path with your actual value from `where mcp-atlassian`.

---

## Step 4 — Register for Auto-Start at Logon

Open **PowerShell** (search "PowerShell" in the Windows Start menu). If you get an execution policy error, run this once first:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Then register, start, and verify the server:

```powershell
cd service\windows

# Register the VBScript as a Task Scheduler job (runs at logon, auto-restarts on failure)
.\install.ps1

# Start it immediately without logging off
.\start.ps1

# Verify it's healthy
.\status.ps1
```

The server will start silently in the background on every Windows logon.

Health check (manual):

```bash
curl http://localhost:9002/healthz
# Expected: {"status":"ok"}
```

---

## Step 5 — Connect GitHub Copilot in VS Code

The server must be running (healthz returns `ok`) before adding it.

Add the following to your VS Code **User Settings** (`Ctrl+Shift+P` → "Open User Settings (JSON)"):

```json
"mcp": {
  "servers": {
    "atlassian-mcp": {
      "type": "http",
      "url": "http://localhost:9002/mcp"
    }
  }
}
```

Verify: Open the **Copilot Chat** panel, click the **Tools** icon (or type `@` in the chat), and confirm `atlassian-mcp` appears in the MCP servers list.

---

---

## Using the Server

Once the MCP server is running and connected, GitHub Copilot has direct access to Intel Confluence
tools in every session — no special commands needed. Just describe what you want in plain English in
the **Copilot Chat** panel (`Ctrl+Alt+I`).

### Example prompts

**Search and read**
```
What does the wiki say about the Atlas Data Reader AGS role?
```
```
Find the setup guide for the ECDW Snowflake prod environment and summarise the connection steps.
```
```
/wiki-query How do I request a faceless account for Atlas?
```

**Read a specific page**
```
Get the page "How to Extract Data from Atlas" from the AtlasApps space and show me the ECA section.
```

**Create or update pages**
```
Create a new wiki page in the MYSPACE space titled "Q3 Review Notes" with the following content: ...
```
```
Update the page at https://wiki.ith.intel.com/pages/viewpage.action?pageId=12345678 to add a new section called "2026 Update" with this content: ...
```

### Available Confluence tools

The MCP exposes these tools directly to Copilot (visible in the Chat panel under **Tools**):

| Tool | Description |
| --- | --- |
| `confluence_search` | Full-text and CQL search across all spaces |
| `confluence_get_page` | Read a page by ID or title + space key |
| `confluence_get_page_children` | List child pages and folders |
| `confluence_create_page` | Create a new page (Markdown, wiki, or storage format) |
| `confluence_update_page` | Edit an existing page |
| `confluence_add_comment` | Add a comment to a page |
| `confluence_get_comments` | Read comments on a page |
| `confluence_upload_attachment` | Attach a file to a page |
| `confluence_get_attachments` | List attachments on a page |
| `confluence_get_page_diff` | See what changed between two page versions |
| `confluence_get_space_page_tree` | Get a space's full page hierarchy |
| `confluence_search_user` | Look up Intel colleagues by name |

> **Read-only mode**: Set `CONFLUENCE_READ_ONLY=true` in the VBScript to prevent Copilot from
> creating or modifying pages. Useful for sharing the server with colleagues who should only have
> read access.

## Troubleshooting

**MCP server not appearing or failing to connect in Copilot Chat**
The server is not running. From the repo directory run `service\windows\start.ps1`, or check Task Scheduler. Then verify with `curl http://localhost:9002/healthz`.

**`Failed to resolve 'wiki.ith.intel.com'`**
You are not on Intel network or VPN.

**SSL certificate errors**
Ensure `CONFLUENCE_SSL_VERIFY=false` is set in the VBScript. Intel's network performs SSL
inspection with an internal CA that Python does not trust by default.

**`git clone` fails with proxy error**
Run `git config --global http.proxy http://proxy-chain.intel.com:911` and retry.

---

## Updating from Upstream

```bash
cd mcp-atlassian-intel
git stash           # stash Intel patch changes
git pull origin main
git stash pop       # re-apply
# If ssl.py has conflicts, re-apply the changes from intel_noproxy.patch manually
uv tool install --from . mcp-atlassian  # reinstall with updated code
```
