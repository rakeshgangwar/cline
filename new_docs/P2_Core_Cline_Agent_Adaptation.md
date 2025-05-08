# P2: Core Cline Agent Adaptation

## 1. Introduction

This document details the necessary adaptations to the existing Cline core engine (primarily the `Task` class and related components from `src/core/`) to enable its operation as a headless, autonomous agent within the Remote Cline Agent Platform.

## 2. Goals of Adaptation

*   **Headless Operation:** The Cline engine must run without a VSCode instance or its UI (webview).
*   **API-Driven Execution:** Tasks will be initiated and managed via API calls from the Task Orchestrator, not user interactions in a chat interface.
*   **Environment Independence:** Minimize dependencies on VSCode-specific APIs and environment assumptions.
*   **Statelessness (where possible):** Agent instances should be relatively stateless or able to rehydrate state quickly to facilitate scaling and fault tolerance. Task-specific state will be managed.
*   **Secure Operation:** Ensure all operations, especially file system access and tool execution, are secure in a server-side environment.

## 3. Key Components for Adaptation (from `cline-overview.md`)

*   **`Task` Class (`src/core/task/index.ts`):** This is the heart of the Cline engine.
*   **API Providers (`src/api/providers/`):** For interacting with AI models.
*   **Tool Execution Logic:** The mechanisms for `execute_command`, `write_to_file`, `replace_in_file`, browser automation, etc.
*   **Context Management (`ContextManager`):** For managing the AI's conversation history and context window.
*   **MCP Integration (`McpHub`):** If MCP tools are to be supported.

## 4. Adaptation Details

### 4.1. Decoupling from VSCode UI and `WebviewProvider`

*   **Input Mechanism:**
    *   **Current:** Receives user input and task definitions through `WebviewProvider` and `Controller`.
    *   **Needed:** Implement a new input mechanism where task definitions, initial prompts, and subsequent "user" messages (which will be tool results or system directives) are passed via function calls or an internal API from the agent's entry point (invoked by the Task Orchestrator).
*   **Output Mechanism:**
    *   **Current:** Sends messages (`say` method), tool requests, and errors to the `WebviewProvider` for display.
    *   **Needed:** Redirect these outputs.
        *   AI text responses, progress updates, and logs should be streamed/sent to the Task Orchestrator (which then forwards to LogStore or NotificationService).
        *   Tool usage requests: Instead of asking a user for approval via UI, the system might have pre-approved tools for certain tasks, or a mechanism for the Task Orchestrator to programmatically approve/deny based on task parameters or policies. For MVP, many tools might be auto-approved if the task implies their use.
        *   Errors: Report errors to the Task Orchestrator.
*   **State Management (`Controller` dependency):**
    *   **Current:** `Task` relies on `Controller` for global/workspace state, secrets, and task history.
    *   **Needed:**
        *   Secrets (API keys for AI models, VCS) must be securely provided to the agent instance at runtime (e.g., via environment variables, mounted secrets by the Agent Manager).
        *   Task-specific history (`apiConversationHistory`, `clineMessages`) will still be crucial but might be persisted by the agent to a location accessible by the Task Orchestrator (e.g., distributed cache, or passed back upon task segment completion) rather than VSCode's storage.

### 4.2. Tool Execution in a Headless Environment

*   **File System Tools (`write_to_file`, `replace_in_file`, `read_file`, `list_files`):**
    *   **Current:** Operate on the local file system within VSCode's context.
    *   **Needed:** These tools will operate within the agent's isolated workspace (e.g., a container volume). Paths will be relative to this workspace. Ensure robust error handling for file operations.
*   **Command Execution (`execute_command`):**
    *   **Current:** Uses VSCode's integrated terminal.
    *   **Needed:** Execute commands directly using a shell within the agent's container. Output (stdout, stderr) must be captured and relayed to the Task Orchestrator/LogStore. Security is paramount: commands must be sandboxed and restricted if possible. Consider whitelisting commands or using very restricted execution environments.
*   **Browser Automation Tools:**
    *   **Current:** Uses Puppeteer, potentially displaying the browser.
    *   **Needed:** Run Puppeteer in headless mode. Ensure the agent environment has necessary dependencies for headless Chrome/Chromium. Screenshots or extracted data will be sent to Artifact Storage.
*   **`ask_followup_question` Tool:**
    *   **Current:** Asks the user a question via the UI.
    *   **Needed:** This tool is problematic for fully autonomous agents.
        *   For MVP, tasks should be defined clearly enough to minimize the need for follow-up questions.
        *   If unavoidable, the agent could "pause" the task and signal the Task Orchestrator that user input is required. The Task Orchestrator would then notify the user via the Web App. This adds complexity.
        *   Alternatively, the AI could be prompted to make its best judgment or list assumptions if information is missing.
*   **MCP Tool Integration (`use_mcp_tool`):**
    *   **Current:** `McpHub` connects to MCP servers.
    *   **Needed:** If supported, each agent instance would need its own `McpHub` or connect to a shared `McpHub` service. Configuration for MCP servers would need to be provided to the agent.

### 4.3. Context Management (`ContextManager`)

*   **Current:** Manages conversation history for the AI.
*   **Needed:** This logic remains largely the same, but the storage and retrieval of `apiConversationHistory` will change as noted above. The context manager will be crucial for long-running autonomous tasks.

### 4.4. Configuration

*   **Current:** API configurations, model selections, etc., are managed via VSCode settings.
*   **Needed:** These configurations must be passed to the agent instance at startup or as part of the task definition by the Task Orchestrator. This includes AI provider details, model preferences for the task, and any tool-specific settings.

### 4.5. Entry Point and Lifecycle

*   **Needed:** A new entry point for the headless Cline engine. This script/function will:
    1.  Initialize with task definition, configuration, and credentials.
    2.  Set up the workspace (e.g., clone repo if specified in task).
    3.  Instantiate and run the `Task` loop.
    4.  Communicate progress, results, and errors back to the Task Orchestrator.
    5.  Handle shutdown signals gracefully.

### 4.6. Error Handling and Reporting

*   **Current:** Errors are often displayed in the VSCode UI.
*   **Needed:** Robust error handling within the agent. All errors must be:
    *   Logged comprehensively.
    *   Reported to the Task Orchestrator with sufficient detail for diagnosis.
    *   The agent should attempt to clean up its workspace or state in case of critical failure where possible.

## 5. Security Considerations for Adapted Agent

*   **Filesystem Access:** Strictly confined to the isolated workspace.
*   **Command Execution:** Sandboxed, restricted, potentially whitelisted commands. No direct access to host system.
*   **Network Access:** Restricted to necessary endpoints (AI APIs, Git providers, configured MCP servers).
*   **Credential Management:** Secure injection and handling of secrets; avoid storing them plaintext in the workspace.

## 6. Deliverables for Adaptation

*   A runnable package/container image of the Headless Cline Engine.
*   Clear API/interface for the Task Orchestrator to invoke and manage the engine.
*   Documentation on how to configure and run the headless engine.

---
Return to [README](../README.md)
