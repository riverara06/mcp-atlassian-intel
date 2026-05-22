# Windows Service Scripts

PowerShell scripts to manage the `mcp-atlassian` server as a Windows background service via Task Scheduler.

## Prerequisites

- `mcp-atlassian` installed: `uv tool install --from . mcp-atlassian`
- `start-atlassian-mcp.vbs` created in your home directory (see [main README Step 3](../../README.md#step-3--create-the-startup-script))

## Scripts

| Script | What it does |
|---|---|
| `install.ps1` | Registers the VBScript as a **Task Scheduler** job (runs at logon, auto-restarts on failure) |
| `uninstall.ps1` | Removes the scheduled task and stops any running processes |
| `start.ps1` | Runs the VBScript immediately and waits for the health check to pass |
| `stop.ps1` | Kills all running `mcp-atlassian` processes |
| `status.ps1` | Shows process state, health endpoint, and scheduler registration |

## Quick start

```powershell
cd service\windows

# 1. Register for auto-start at logon
.\install.ps1

# 2. Start now (without logging off)
.\start.ps1

# 3. Check it's healthy
.\status.ps1
```

## Parameters

All scripts accept optional parameters:

```powershell
.\install.ps1   -VbsPath "C:\Users\you\start-atlassian-mcp.vbs" -TaskName "Atlassian MCP Server"
.\uninstall.ps1 -TaskName "Atlassian MCP Server"
.\start.ps1     -VbsPath "C:\Users\you\start-atlassian-mcp.vbs" -Port 9002 -HealthCheckTimeout 15
.\stop.ps1      -Port 9002
.\status.ps1    -Port 9002 -TaskName "Atlassian MCP Server"
```

## Execution policy

If you get an execution policy error, run once:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

