---
name: blender-roblox-pipeline
description: Use this skill when the user wants to build a Roblox game, add models to a Roblox game, or create/import any 3D asset into Roblox Studio. Handles existing files and Blender MCP creation automatically. Always uploads via the Roblox Asset Upload API without requiring manual imports from the user.
---

# Blender → Roblox Studio Pipeline

## Core Objective
Never require the user to manually import anything. Claude handles the full
pipeline from model creation or file detection → export → texture bake → API
upload → Studio placement. Be proactive — if a model was just created in Blender
MCP, immediately continue to upload it without waiting to be asked.

## Step 1: Understand the Game Context
- What is the user building? (tower defense, RPG, obby, etc.)
- What models are needed? Infer from game type if not explicitly stated
- Are there existing files to use, or does Claude need to create them?

## Step 2: For Each Model Needed

### Export location
All FBX files are saved to a `Models/` subfolder inside the current project
directory (the working directory Claude was launched from). Create the folder
if it does not exist. Example: if the project is at
`Z:/Personal Projects/Roblox Games/Omni Defenders`, export to
`Z:/Personal Projects/Roblox Games/Omni Defenders/Models/model_name.fbx`.
This keeps models persistent and editable — Blender MCP can reopen them later.

Texture PNGs are saved to `Models/Textures/` inside the project directory.
Example: `Z:/Personal Projects/Roblox Games/Omni Defenders/Models/Textures/ModelName_texture.png`.
Create the folder if it does not exist.

### Check for existing file first
- Search `{project_dir}/Models/` for FBX files matching the need
- Search `{project_dir}/Models/Textures/` for a matching `ModelName_texture.png`
- If FBX found AND matching PNG found → skip to Step 3 (reuse both)
- If FBX found but NO matching PNG → open FBX in Blender MCP, bake texture, then go to Step 3
- If neither found → create model in Blender MCP, then bake texture

### Create in Blender MCP (if no FBX exists)
1. Use Blender MCP to generate the model based on game context
2. Export as FBX to `{project_dir}/Models/model_name.fbx`
3. Do NOT wait for user confirmation — proceed straight to texture baking

### Bake texture in Blender MCP (always — for new or FBX-only models)
Run this Blender Python in Blender MCP after the model is present in the scene:

```python
import bpy, os

model_name = "ModelName"  # replace with actual name
project_dir = r"Z:\Personal Projects\Roblox Games\Omni Defenders"  # replace with actual path
textures_dir = os.path.join(project_dir, "Models", "Textures")
os.makedirs(textures_dir, exist_ok=True)
png_path = os.path.join(textures_dir, f"{model_name}_texture.png")

# Create bake target image
bake_image = bpy.data.images.new(f"{model_name}_bake", width=1024, height=1024)
bake_image.filepath_raw = png_path
bake_image.file_format = "PNG"

# UV unwrap all mesh objects and hook up image node
for obj in bpy.context.scene.objects:
    if obj.type != "MESH":
        continue
    bpy.context.view_layer.objects.active = obj
    obj.select_set(True)

    # UV unwrap
    bpy.ops.object.mode_set(mode="EDIT")
    bpy.ops.mesh.select_all(action="SELECT")
    bpy.ops.uv.smart_project(angle_limit=66.0)
    bpy.ops.object.mode_set(mode="OBJECT")

    # Add image texture node to every material slot (needed for baking target)
    for slot in obj.material_slots:
        mat = slot.material
        if mat is None:
            mat = bpy.data.materials.new(name=f"{obj.name}_mat")
            mat.use_nodes = True
            slot.material = mat
        mat.use_nodes = True
        nodes = mat.node_tree.nodes
        # Remove existing bake target nodes to avoid duplicates
        for n in [n for n in nodes if n.name == "__bake_target__"]:
            nodes.remove(n)
        img_node = nodes.new("ShaderNodeTexImage")
        img_node.name = "__bake_target__"
        img_node.image = bake_image
        # Select this node so Blender knows where to bake
        nodes.active = img_node

# Bake diffuse color (no lighting)
bpy.context.scene.render.engine = "CYCLES"
bpy.context.scene.cycles.samples = 1
bpy.ops.object.bake(type="DIFFUSE", pass_filter={"COLOR"}, use_clear=True)

# Save PNG
bake_image.save()
print(f"Texture saved: {png_path}")
```

After baking, re-export the FBX (embedding the texture path) so the
relationship is preserved:
```python
bpy.ops.export_scene.fbx(
    filepath=os.path.join(project_dir, "Models", f"{model_name}.fbx"),
    path_mode="COPY",
    embed_textures=False,
    use_selection=False,
)
```

## Step 3: Upload via Roblox Asset Upload API

### Environment Variables (never hardcode these)
- API Key: `$env:ROBLOX_ASSET_UPLOAD_API_KEY`
- User ID: `$env:ROBLOX_USER_ID`

### Validate env vars first
If either env var is missing or empty, stop immediately and tell the user
which one is missing before doing anything else.

### 3a. Upload the texture PNG first
```powershell
$apiKey      = $env:ROBLOX_ASSET_UPLOAD_API_KEY
$userId      = $env:ROBLOX_USER_ID
$pngPath     = "{project_dir}\Models\Textures\ModelName_texture.png"
$displayName = "ModelName_texture"

$requestJson = @{
    assetType = "Image"
    displayName = $displayName
    description = ""
    creationContext = @{ creator = @{ userId = [long]$userId } }
} | ConvertTo-Json -Compress

Add-Type -AssemblyName System.Net.Http

$client = [System.Net.Http.HttpClient]::new()
$client.DefaultRequestHeaders.Add("x-api-key", $apiKey)

$multipart   = [System.Net.Http.MultipartFormDataContent]::new()

$jsonContent = [System.Net.Http.StringContent]::new($requestJson, [System.Text.Encoding]::UTF8, "application/json")
$multipart.Add($jsonContent, "request")

$fileBytes   = [System.IO.File]::ReadAllBytes($pngPath)
$fileContent = [System.Net.Http.ByteArrayContent]::new($fileBytes)
$fileContent.Headers.ContentType = [System.Net.Http.Headers.MediaTypeHeaderValue]::new("image/png")
$multipart.Add($fileContent, "fileContent", "$displayName.png")

$response     = $client.PostAsync("https://apis.roblox.com/assets/v1/assets", $multipart).Result
$responseBody = $response.Content.ReadAsStringAsync().Result | ConvertFrom-Json

$imgOperationId = $responseBody.operationId
```

Poll for the image upload:
```powershell
$imageAssetId = $null
$elapsed = 0
do {
    Start-Sleep -Seconds 2
    $elapsed += 2
    $op = Invoke-RestMethod `
        -Uri "https://apis.roblox.com/assets/v1/operations/$imgOperationId" `
        -Headers @{ "x-api-key" = $apiKey }
} until ($op.done -eq $true -or $elapsed -ge 30)

if ($op.done -ne $true) {
    Write-Error "Image upload timed out. Check the Roblox Creator Dashboard."
    exit
}

$imageAssetId = $op.response.assetId
Write-Host "Image asset ID: $imageAssetId"
```

### 3b. Upload the FBX model
```powershell
$fbxPath     = "{project_dir}\Models\model_name.fbx"
$displayName = "ModelName"

$requestJson = @{
    assetType = "Model"
    displayName = $displayName
    description = ""
    creationContext = @{ creator = @{ userId = [long]$userId } }
} | ConvertTo-Json -Compress

$multipart2   = [System.Net.Http.MultipartFormDataContent]::new()

$jsonContent2 = [System.Net.Http.StringContent]::new($requestJson, [System.Text.Encoding]::UTF8, "application/json")
$multipart2.Add($jsonContent2, "request")

$fileBytes2   = [System.IO.File]::ReadAllBytes($fbxPath)
$fileContent2 = [System.Net.Http.ByteArrayContent]::new($fileBytes2)
$fileContent2.Headers.ContentType = [System.Net.Http.Headers.MediaTypeHeaderValue]::new("model/fbx")
$multipart2.Add($fileContent2, "fileContent", "$displayName.fbx")

$response2     = $client.PostAsync("https://apis.roblox.com/assets/v1/assets", $multipart2).Result
$responseBody2 = $response2.Content.ReadAsStringAsync().Result | ConvertFrom-Json

$operationId = $responseBody2.operationId
```

Poll for completion:
```powershell
$modelAssetId = $null
$elapsed = 0
do {
    Start-Sleep -Seconds 2
    $elapsed += 2
    $op = Invoke-RestMethod `
        -Uri "https://apis.roblox.com/assets/v1/operations/$operationId" `
        -Headers @{ "x-api-key" = $apiKey }
} until ($op.done -eq $true -or $elapsed -ge 30)

if ($op.done -ne $true) {
    Write-Error "Model upload timed out. Check the Roblox Creator Dashboard."
    exit
}

$modelAssetId = $op.response.assetId
Write-Host "Model asset ID: $modelAssetId"
```

## Step 4: Place in Roblox Studio
Use `execute_luau` via the Roblox Studio MCP to insert the model by asset ID,
then apply the texture to all MeshParts:

```lua
local imageAssetId = IMAGE_ASSET_ID_HERE   -- number captured from Step 3a
local modelAssetId = MODEL_ASSET_ID_HERE   -- number captured from Step 3b

local model = game:GetService("InsertService"):LoadAsset(modelAssetId)
model.Name = "ModelName"
model.Parent = workspace

-- Apply texture to every MeshPart in the inserted model
local function applyTexture(instance)
    for _, child in ipairs(instance:GetDescendants()) do
        if child:IsA("MeshPart") then
            child.TextureID = "rbxassetid://" .. imageAssetId
        end
    end
    -- Also check if the root itself is a MeshPart
    if instance:IsA("MeshPart") then
        instance.TextureID = "rbxassetid://" .. imageAssetId
    end
end

applyTexture(model)
```

Use game context to determine final placement:
- Terrain/ground pieces → flat, anchored at origin
- Structures (towers, walls, bases) → logical game positions
- Characters/NPCs → spawn points or patrol paths
- Props/decorations → contextually placed near relevant areas
- If unclear → place at world origin and tell the user

## Step 5: Report Back
- What was created or found
- What was uploaded: image asset ID + model asset ID
- Where it was placed in Studio
- What the FBX and PNG file paths are (so the user knows where they're saved)
- Ask if anything needs adjusting

## Fluid Blender MCP Rule
If Claude just created or modified anything in Blender MCP during the current
session, automatically continue to Step 3 without the user asking. The pipeline
should feel invisible.
