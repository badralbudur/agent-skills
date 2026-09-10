---
name: fulcra-workspaces-cli
description: "CLI command references for executing artifact uploads and workspace inbox operations with Fulcra."
---

# Fulcra Workspaces CLI Reference

This reference dictates the exact shell commands required to execute the `fulcra-workspaces` skill's operations. Ensure all CLI operations run in the agent's workspace.

## Authentication Note
If you need to authenticate to Fulcra before running these commands, you must use the non-blocking two-step login process to prevent the CLI from hanging:
1. `uv tool run fulcra-api auth login --get-auth-url` (present URL and code to user)
2. `uv tool run fulcra-api auth login --device-code <DEVICE_CODE> --poll-timeout=5` (after user finishes flow)

## 1. Checking Recent Workspace File Changes

To quickly check for recent updates across a workspace's namespaces without listing individual directories:

```bash
# Get a summary of files changed in the last 1 day
uv tool run fulcra-api data-updates "1 day"

# Example output:
# {
#   "data_types": {},
#   "file_changes": [
#     {
#       "full_name": "/workspace/first-olympiad/progress.md",
#       "uploaded_at": "2026-07-01T21:23:28.690719Z",
#       "state": "uploaded",
#       "...": "..."
#     },
#     {
#       "full_name": "/workspace/first-olympiad/task/setup-dashboard.md",
#       "uploaded_at": "2026-07-01T21:23:28.690719Z",
#       "state": "uploaded",
#       "...": "..."
#     }
#   ]
# }
```

*Note: The `file_changes` key is a list of file metadata objects. You can extract the `full_name` from each to see the file paths. If the summary shows that specific workspace files were changed, you can then read those specific files to update your context.*

## 2. Uploading User Artifacts

When an agent generates a file (like an HTML dashboard, an image, or a report), and the user explicitly approves saving it to their Fulcra account, upload it to the `artifact/` subdirectory.

```bash
# Replace <agent_name> with the agent's name, and <artifact_name> with the file's name
uv tool run fulcra-api file upload /path/to/local/file "agent/<agent_name>/artifact/<artifact_name>"
```

## 2. Workspace Coordination (Inbox & Archive)

Agents can coordinate by writing to and reading from workspace namespaces.

**Message Naming Convention:**
Messages must follow the format `YYYYMMDD-HHMMSS_<sender-name>_<short-topic>.md`. Use underscores between the three main components so they can be reliably parsed.
*Note: When replying to a message or providing a status update, always reuse the exact same `<short-topic>` as the original message to maintain thread continuity.*

**Step A: Sending a message to a workspacemate's inbox**
```bash
# Upload a local markdown file to the target agent's inbox
uv tool run fulcra-api file upload /tmp/message.md "workspace/<workspace_name>/member/<target_agent_name>/inbox/20260608-232500_wazir_status-update.md"
```

**Step B: Checking your inbox**
```bash
# List files in your agent's inbox
uv tool run fulcra-api file list "workspace/<workspace_name>/member/<your_agent_name>/inbox/"
```

**Step C: Processing and Archiving a message**
Once you have downloaded and read a message from your inbox, move it to the archive. If the file was manually dropped and lacks a timestamp, **you must prepend one** (`YYYYMMDD-HHMMSS_`) when saving it to `archive/`.

```bash
# 1. Download to read (if you haven't already)
uv tool run fulcra-api file download "workspace/<workspace_name>/member/<your_agent_name>/inbox/20260608-232500_wazir_status-update.md" /tmp/20260608-232500_wazir_status-update.md

# 2. Upload it to your archive directory
uv tool run fulcra-api file upload /tmp/20260608-232500_wazir_status-update.md "workspace/<workspace_name>/member/<your_agent_name>/archive/20260608-232500_wazir_status-update.md"

# 3. Verify archival succeeded before deletion!
uv tool run fulcra-api file stat "workspace/<workspace_name>/member/<your_agent_name>/archive/20260608-232500_wazir_status-update.md"

# 4. Delete it from the inbox to clear it (only if step 3 succeeded)
uv tool run fulcra-api file delete "workspace/<workspace_name>/member/<your_agent_name>/inbox/20260608-232500_wazir_status-update.md"
```

## 3. Workspace Activity Tracking (OKF Compliant)

Agents can update shared files to track the workspace's high-level progress and completed objectives. Ensure all markdown files contain OKF YAML frontmatter, and that `log.md` and `index.md` are updated when appropriate.

**Step A: Updating Workspace Progress**
To update the `progress.md` file (which stores what the workspace members have recently done and what they plan to do next):
```bash
# 1. Download the current progress file
uv tool run fulcra-api file download "workspace/<workspace_name>/progress.md" /tmp/workspace_progress.md || touch /tmp/workspace_progress.md

# 2. Edit /tmp/workspace_progress.md locally to reflect the latest plans and recent work. 
# Make sure it has OKF frontmatter:
# ---
# type: Progress Report
# title: Workspace Progress
# ---

# 3. Upload the updated file back to Fulcra
uv tool run fulcra-api file upload /tmp/workspace_progress.md "workspace/<workspace_name>/progress.md"

# 4. Also append an update entry to log.md
DATE=$(date -u +"%Y-%m-%d")
echo "## $DATE" > /tmp/log_update.md
echo "* **Update**: <agent_name> updated workspace progress." >> /tmp/log_update.md
# (In practice, download log.md, append the update under the correct date, and re-upload)
```

**Step B: Recording Completed Objectives**
To add a newly completed high-level objective to `completed.md` (which should generally only grow):
```bash
# 1. Download the current completed file
uv tool run fulcra-api file download "workspace/<workspace_name>/completed.md" /tmp/workspace_completed.md || touch /tmp/workspace_completed.md

# 2. Append the new objective (ensure OKF frontmatter exists at the top of the file)
echo "- [$(date +%Y-%m-%d)] <Objective summary>" >> /tmp/workspace_completed.md

# 3. Upload the updated file back to Fulcra
uv tool run fulcra-api file upload /tmp/workspace_completed.md "workspace/<workspace_name>/completed.md"
```

**Step C: Syncing Workspace and Member Roles & Progress**
To ensure the workspace and its members understand their purpose and current context, maintain `role.md` and member `progress.md` files (with proper OKF frontmatter).
```bash
# Update the overall workspace role
uv tool run fulcra-api file upload /tmp/workspace-role.md "workspace/<workspace_name>/role.md"

# Update your specific agent's role within the workspace
uv tool run fulcra-api file upload /tmp/member-role.md "workspace/<workspace_name>/member/<your_agent_name>/role.md"

# Update your specific agent's progress (critical for isolated background jobs)
uv tool run fulcra-api file upload /tmp/member-progress.md "workspace/<workspace_name>/member/<your_agent_name>/progress.md"
```

## 4. Workspace Session and Task Tracking

When completing a discrete block of work or tracking a long-running project within the workspace, upload summaries to the workspace namespace rather than your personal memory namespace.

**Step A: Uploading a Session Summary**
When a workspace session concludes, create a concise markdown summary (with `type: Session Summary` frontmatter) and upload it.
```bash
# Filename convention: YYYYMMDD-HHMMSS_<agent-name>_<subject>.md
uv tool run fulcra-api file upload /tmp/session-summary.md "workspace/<workspace_name>/session/20260623-180530_treecle_setup-dashboard.md"
```

**Step B: Updating a Task Tracker**
For ongoing workspace objectives, update a task tracker (with `type: Task` frontmatter) and its index.
```bash
# Filename convention: <task-name>.md (No timestamp)
uv tool run fulcra-api file upload /tmp/task-status.md "workspace/<workspace_name>/task/setup-dashboard.md"
uv tool run fulcra-api file upload /tmp/task-index.md "workspace/<workspace_name>/task/index.md"
```
