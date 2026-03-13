# Blender → Roblox Studio Automation Pipeline

A Claude Code skill that fully automates the pipeline from 3D model creation to Roblox Studio placement — no manual importing required.

## What It Does

When building a Roblox game, this skill eliminates the manual steps between Blender and Roblox Studio. Claude handles the entire chain automatically:

```
Game context described by user
        ↓
Claude infers what models are needed
        ↓
Check for existing FBX file
        ↓
  ┌─────┴─────┐
File         No file
exists       exists
  ↓               ↓
Reuse        Create in
file         Blender MCP
  └─────┬─────┘
        ↓
Bake texture (Blender MCP)
        ↓
Upload PNG texture → Roblox Asset Upload API
        ↓
Upload FBX model → Roblox Asset Upload API
        ↓
Place model in Roblox Studio workspace (via Luau)
```

## Features

- **Zero manual imports** — Claude detects, creates, exports, uploads, and places models without user intervention
- **Blender MCP integration** — generates models from scratch when no file exists
- **Texture baking** — auto-bakes diffuse textures to PNG using Cycles before upload
- **Roblox Asset Upload API** — uploads both the texture and FBX model, polls for completion
- **Smart placement** — uses game context to determine where models go in the workspace
- **Persistent file structure** — saves FBX and PNG files to `Models/` and `Models/Textures/` in your project directory for reuse
- **Fluid pipeline** — if Blender MCP was just used, the upload triggers automatically

## Project File Structure

The skill organizes exported assets relative to your project directory:

```
Your Roblox Project/
└── Models/
    ├── EnemyModel.fbx
    ├── Tower.fbx
    └── Textures/
        ├── EnemyModel_texture.png
        └── Tower_texture.png
```

## Tech Stack

- Claude Code + Claude MCP skill system
- Blender MCP (model generation + texture baking via Blender Python API)
- Roblox Asset Upload API (Open Cloud)
- Roblox Studio MCP (Luau script execution for placement)
- PowerShell (API calls, multipart upload, polling)

---

## Getting Started

### Prerequisites
Before installing, make sure you have the following set up:
- [Claude Code](https://claude.ai/code)
- [Blender MCP](https://github.com/ahujasid/blender-mcp) connected to Claude Code
- Roblox Studio with MCP plugin installed
- A Roblox Open Cloud API key with asset upload permissions

### 1. Clone the repo
```powershell
git clone https://github.com/RonnieJackson507/blender-roblox-pipeline.git
cd blender-roblox-pipeline
```

### 2. Set your environment variables
Never hardcode your credentials — store them as Windows environment variables:

| Variable | Description |
|---|---|
| `ROBLOX_ASSET_UPLOAD_API_KEY` | Your Roblox Open Cloud API key |
| `ROBLOX_USER_ID` | Your Roblox user ID |

```powershell
[System.Environment]::SetEnvironmentVariable("ROBLOX_ASSET_UPLOAD_API_KEY", "your-key-here", "User")
[System.Environment]::SetEnvironmentVariable("ROBLOX_USER_ID", "your-id-here", "User")
```

### 3. Install the skill
```powershell
# Copy the skill to your Claude commands folder
Copy-Item blender-roblox-pipeline.md ~/.claude/commands/

# Add the auto-trigger to your CLAUDE.md
Add-Content ~/.claude/CLAUDE.md "When the user mentions building a Roblox game or asks for models/assets to be put into a Roblox game, proactively run /blender-roblox-pipeline without being asked."
```

### 4. Start a new Claude Code session
The auto-trigger activates on the next session. Just describe the game you want to build and Claude handles the rest:

```
"Make me a tower defense Roblox game"
"Add a castle wall to my game"
"I need an enemy character model for the level"
```
