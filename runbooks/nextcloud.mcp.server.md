# NextCloud MCP Server Integration Guide

## Overview & Goal
This guide documents the process of integrating a NextCloud instance with an LLM interface (Open WebUI) using the `nextcloud-mcp-server`. The primary goal is to enable the AI agent to manage **Calendar**, **Notes**, and **Files** without risking accidental data deletion or tool confusion.

*   **Project Source:** [https://github.com/cbcoutinho/nextcloud-mcp-server](https://github.com/cbcoutinho/nextcloud-mcp-server)
*   **Target Interface:** Open WebUI (via MCP protocol)
*   **Scope:** Basic functionality only (Calendar, Notes, File Upload/Read). 

---

## 1. Prerequisites
Before beginning, ensure the following are available:
*   A running NextCloud instance with the Calendar, Notes, and Files apps installed.
*   Docker installed on your host machine.
*   Open WebUI configured to accept MCP tool calls.
*   Network connectivity between the host and the NextCloud server (no firewall blocks).

---

## 2. Installation (Docker)

Deploy the `nextcloud-mcp-server` container using a standard Docker Compose configuration.

### docker-compose.yml
Create or update your `docker-compose.yml` with the following structure. Ensure you do not expose sensitive internal IPs in this file.

```yaml
services:
  nextcloud-mcp-server:
    image: ghcr.io/cbcoutinho/nextcloud-mcp-server:latest
    container_name: nextcloud-mcp-server
    restart: unless-stopped
    env_file:
      - .env
    environment:
      # Disable semantic search for this baseline setup.
      ENABLE_SEMANTIC_SEARCH: "false" 
    ports:
      - "8181:8000"
    networks:
      - nextcloud_default

networks:
  nextcloud_default:
    external: true
```

**Notes:**
*   The `ENABLE_SEMANTIC_SEARCH` flag is set to `"false"` for this guide. If you wish to enable semantic search later, change this value to `"true"`.
*   Ensure the container can reach your NextCloud instance via its internal network name or IP (configured in `.env`).

---

## 3. Configuration (Open WebUI)

Once the MCP server is running, you must configure Open WebUI to expose only the necessary tools. By default, the server exposes all available tools (Mail, Calendar, Notes, Files, etc.). This can lead to tool confusion and errors if the model attempts to use a tool that fails or isn't needed.

### Step 1: Whitelist Tools
In Open WebUI, navigate to **Admin Settings → Integrations**. Click the gear icon next to `NextCloudMCP` to edit the connection settings.

You must populate the **Function Name Filter List** field with specific tool names or prefixes. This prevents the model from attempting to call tools it shouldn't use (e.g., Mail tools) and reduces error rates.

**Recommended Tool List:**
To enable Calendar, Notes, and File management while excluding deletion capabilities:

```text
calendar_list_calendars, calendar_list_events, calendar_get_event, calendar_create_event, calendar_update_event, calendar_search_events, nc_notes_list_notes, nc_notes_get_note, nc_notes_create_note, nc_notes_update_note, webdav_list_files, webdav_read_file, webdav_upload_file, webdav_make_directory, webdav_search_files
```

**Why exclude deletes?**
The model may attempt to delete files or events if not explicitly restricted. This list excludes `calendar_delete_event`, `nc_notes_delete_note`, and `webdav_delete_file` to ensure safety during initial testing.

### Step 2: Verify Connection
1.  Save the settings in Open WebUI.
2.  Start a new chat session.
3.  Ask the model to perform a simple task, such as listing calendar events or reading a note.
4.  Check server logs (`docker logs nextcloud-mcp-server`) to ensure requests are being processed without `tool_error` warnings.

---

## 4. Troubleshooting Common Issues

### Issue: Infinite Retry Loop & Tool Errors
**Symptoms:** The model repeatedly calls the same tool (e.g., `nc_mail_list_accounts`) and receives a `tool_error`. Logs show repeated requests with no progress.

**Cause:** Open WebUI exposes too many tools by default. If the model attempts to use a Mail tool that fails or isn't configured for the user, it retries indefinitely rather than switching to available tools (Calendar/Notes).

**Resolution:**
1.  **Filter Tools:** Apply the whitelist configuration described in Section 3. This restricts the model's options to only those that are likely to succeed and match your intent.
2.  **Restart Chat:** Open a fresh chat session after applying filters. The model will not have access to the previously exposed Mail tools.

### Issue: Weak Search Quality
**Symptoms:** When asked to find a document by description, the model struggles to locate the file correctly. It may guess filenames incorrectly or crawl folder trees inefficiently.

**Cause:** Without semantic search enabled (Qdrant), the server relies on literal filename matching (`webdav_search_files`). If the model guesses a filename that doesn't match exactly, it returns no results and retries.

**Resolution for Baseline Setup:**
*   **Prompt Engineering:** Instruct the model to list directory contents first before guessing filenames.
    *   *Example Prompt:* "List files in my root directory first, then read the one matching 'gift ideas'."
*   **Future Enhancement:** Enable semantic search (`ENABLE_SEMANTIC_SEARCH: "true"`) if you require content-based searching later.

---

## 5. Outcome & Verification

After applying the tool filters and configuration steps above, the integration should function as follows:

| Feature | Status | Capabilities |
| :--- | :--- | :--- |
| **Calendar** | ✅ Active | List calendars, list events, create/update/search events. |
| **Notes** | ✅ Active | List notes, read content, create/update notes. |
| **Files** | ✅ Active | List files/folders, upload/read text files, search by name. |
| **Deletion** | ⛔ Disabled | No delete tools exposed in the filter list. |

The model is now capable of drafting documents and viewing calendar data without risking accidental deletion or getting stuck in error loops.
