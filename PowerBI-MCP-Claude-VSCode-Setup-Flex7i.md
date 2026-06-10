# Power BI MCP + Claude + VS Code — One-Day Walk-Through Setup
**Target machine:** Lenovo Flex 7i (14", Gen 7) · Intel Core i7-1255U · 16 GB RAM · ~477 GB SSD · Windows 11 · Thunderbolt 4

---

## 0. Hardware verdict

Your Flex 7i is well-matched to this stack. The i7-1255U (10 cores) and 16 GB RAM comfortably run Power BI Desktop + VS Code + Claude together. 16 GB is the *practical floor* for this combination, so for smooth live work:

- Close heavy browser tabs and other RAM-hungry apps during the session.
- Keep at least ~10–15 GB free on disk for Power BI Desktop temp files and model caches.
- No GPU needed — all AI processing happens on Anthropic's servers.

---

## 1. What this stack actually is

Three pieces:

1. **Power BI Modeling MCP Server** — Microsoft's official MCP server. It ships as a **VS Code extension** that downloads a server executable to your machine. It can build/modify semantic models across Power BI Desktop, Fabric, and Power BI Project (PBIP) files. *Modeling only — it cannot edit report pages or visual layouts.*
2. **VS Code** — the delivery vehicle that installs the server executable (and can also host GitHub Copilot as a client).
3. **Claude** — the AI client that "talks to" the server. Either the **Claude Desktop app** or **Claude Code**.

> **Always back up your `.pbix` / PBIP before letting AI modify a model.** The LLM can make unintended changes.

---

## 2. Prerequisites (install in this order)

| # | Component | Notes |
|---|-----------|-------|
| 1 | **Power BI Desktop** | Windows-only — already fine on your Flex 7i. Use a recent version (must support external tools). Install from Microsoft Store or the standalone MSI. |
| 2 | **Visual Studio Code** | https://code.visualstudio.com — standard installer. |
| 3 | **Power BI Modeling MCP extension** | Installed inside VS Code (Step 3 below). |
| 4 | **A Claude client** | Claude Desktop app **or** Claude Code (see Step 4). |
| 5 | **A Claude paid plan** | Required for Claude Code (Pro/Max/Team/Enterprise or API billing). Recommended for Claude Desktop MCP use too — verify your plan before the conference. |

---

## 3. Install the Power BI Modeling MCP Server (both paths share this)

1. Open **VS Code** → Extensions panel (`Ctrl+Shift+X`).
2. Search for **"Power BI Modeling MCP"** (publisher: Analysis Services) and click **Install**.
3. The extension downloads the server executable to your VS Code extensions folder. Note the path — you'll need it:
   ```
   %USERPROFILE%\.vscode\extensions\analysis-services.powerbi-modeling-mcp-<VERSION>-win32-x64\server\powerbi-modeling-mcp.exe
   ```
   - Open File Explorer, paste `%USERPROFILE%\.vscode\extensions` into the address bar, and find the folder beginning with `analysis-services.powerbi-modeling-mcp`.
   - **The `<VERSION>` number changes between releases** — copy the exact folder name on your machine.

---

## 4. Connect a Claude client

### Path A — Claude Desktop (recommended for the conference)

1. Install the **Claude Desktop app** and sign in.
2. In Claude Desktop: **Settings → Developer → Edit Config**. This opens:
   ```
   %APPDATA%\Claude\claude_desktop_config.json
   ```
3. Add this block (replace `<YOUR_USERNAME>` and `<VERSION>` with your real values; note the **double backslashes**):
   ```json
   {
     "mcpServers": {
       "powerbi-modeling-mcp": {
         "command": "C:\\Users\\<YOUR_USERNAME>\\.vscode\\extensions\\analysis-services.powerbi-modeling-mcp-<VERSION>-win32-x64\\server\\powerbi-modeling-mcp.exe",
         "args": ["--start"],
         "type": "stdio"
       }
     }
   }
   ```
4. **Save**, then fully **quit and restart Claude Desktop** (some setups need a full PC restart to initialize the server).
5. You should see a plugin/tools indicator showing the server is connected.

### Path B — Claude Code (backup / power-user option)

1. Install Claude Code with the **native installer** (PowerShell one-liner from the official docs — no Node.js, no admin rights). Then close/reopen the terminal and confirm:
   ```powershell
   claude --version
   ```
2. From a project folder, register the MCP server (note the `--` separates Claude's flags from the server command):
   ```powershell
   claude mcp add powerbi-modeling-mcp -- "C:\Users\<YOUR_USERNAME>\.vscode\extensions\analysis-services.powerbi-modeling-mcp-<VERSION>-win32-x64\server\powerbi-modeling-mcp.exe" --start
   ```
   - Add `--scope user` to make it available across all projects: `claude mcp add --scope user powerbi-modeling-mcp -- "...exe" --start`
3. Verify:
   ```powershell
   claude mcp list
   ```
   The server should show as connected.

---

## 5. Verify the whole chain works

1. Open **Power BI Desktop** and load a `.pbix` (or PBIP) with a real model — **leave it open**. The MCP server connects to the *running* Power BI instance.
2. Open your Claude client.
3. Prompt: **"Analyze the current data model"** or **"What tables are in this model?"**
4. If it lists your tables/measures, you're connected. Try **"Create a measure for Total Sales"** to confirm write-back.

**Model tip:** Microsoft recommends a deep-reasoning model for modeling tasks. With the current lineup, use **Opus 4.8** or **Sonnet 4.6**. Loading a full schema consumes tokens each session — relevant if you're on API billing.

---

## 6. Common errors & fixes

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Server doesn't appear in Claude | Wrong path or unescaped backslashes in JSON | Use `\\` (double backslash) in `claude_desktop_config.json`; verify the exact extension folder name. |
| Connected, but "no model found" | Power BI Desktop not running / no model open | Open Power BI Desktop with a model **before** prompting Claude. |
| Changes appear but reports don't | By design | The server does **modeling only** — it can't touch report pages/visuals. |
| `claude` not recognized (Claude Code) | PATH not updated | Reopen terminal; confirm `%USERPROFILE%\.local\bin` is on PATH. |
| Trailing commas / missing quotes | JSON syntax | Validate the JSON; silent failures are common. |

---

## 7. Pre-session checklist

Do all of this **before** the walk-through starts — ideally the day before, so nothing depends on venue Wi-Fi:

- [ ] Power BI Desktop installed and opens a model cleanly.
- [ ] VS Code installed; **Power BI Modeling MCP** extension installed.
- [ ] Server `.exe` path copied and saved (with exact version number).
- [ ] Chosen Claude client installed and **signed in**; paid plan confirmed.
- [ ] MCP server connected and **verified** with "Analyze the current data model."
- [ ] A backup copy of any model you'll modify.
- [ ] Laptop charged + USB-C/TB4 charger packed.

On the day, just close unneeded apps/tabs to free RAM and re-run the verification prompt once before you begin. Keep the session's own setup sheet handy in case their steps differ slightly.

---

*Built for the one-day "Power BI MCP Server + Claude + VS Code" hands-on walk-through. Verify version numbers and plan requirements against current docs on the day.*
