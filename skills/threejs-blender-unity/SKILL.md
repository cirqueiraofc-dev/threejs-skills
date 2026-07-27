---
name: threejs-blender-unity
description: Blender and Unity integration with Three.js - glTF export pipelines, coordinate system conversion, material/animation round-trips, asset optimization. Use when exporting models from Blender or Unity to Three.js, automating asset pipelines, or debugging imported assets.
---

# Blender & Unity Integration with Three.js

## Quick Start

The pipeline in both cases is: **DCC tool → glTF/GLB → Three.js**. glTF is the only format that preserves PBR materials, animations, and hierarchy reliably across all three tools.

```
Blender ──(built-in glTF exporter)──► model.glb ──► GLTFLoader ──► Three.js
Unity   ──(UnityGLTF / glTFast)────► model.glb ──► GLTFLoader ──► Three.js
```

```javascript
// The Three.js side is identical regardless of source
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("exported-from-blender.glb", (gltf) => {
  scene.add(gltf.scene);
});
```

## Coordinate Systems & Units

The #1 source of import bugs. Know all three conventions:

| Tool     | Up axis | Forward       | Handedness   | Unit    |
| -------- | ------- | ------------- | ------------ | ------- |
| Blender  | +Z      | -Y            | Right-handed | Meters  |
| Three.js | +Y      | -Z (camera)   | Right-handed | Meters  |
| Unity    | +Y      | +Z            | Left-handed  | Meters  |
| glTF     | +Y      | +Z            | Right-handed | Meters  |

**Key facts:**

- Blender's glTF exporter converts Z-up → Y-up automatically (keep "+Y Up" checked, it's the default). Never manually rotate the model -90° on X to compensate — fix the export instead.
- Unity is left-handed; glTF exporters from Unity handle the handedness flip by negating the X axis (or Z, depending on the exporter). If a model appears mirrored, this conversion went wrong — check for negative scale on the root node.
- All three use meters. If a Unity asset arrives 100x too large, it came through FBX (which uses centimeters) somewhere in the chain — re-export as glTF directly.

```javascript
// Diagnose scale/orientation issues on import
loader.load("model.glb", (gltf) => {
  const box = new THREE.Box3().setFromObject(gltf.scene);
  const size = box.getSize(new THREE.Vector3());
  console.log("Model size (meters):", size);

  gltf.scene.traverse((obj) => {
    if (obj.scale.x < 0 || obj.scale.y < 0 || obj.scale.z < 0) {
      console.warn("Negative scale (mirroring) on:", obj.name);
    }
  });
});
```

## Blender → Three.js

### Export Settings (glTF 2.0)

File → Export → glTF 2.0. Recommended settings for web:

- **Format**: `glTF Binary (.glb)` - single file, no loose textures
- **Include → Limit to**: `Selected Objects` when exporting one asset from a working file
- **Transform**: `+Y Up` (default, keep enabled)
- **Data → Mesh**: enable `Apply Modifiers`; enable `UVs`, `Normals`; `Vertex Colors` only if used
- **Data → Material**: `Export`, image format `Automatic` (or `JPEG` if no alpha needed)
- **Data → Compression**: enable Draco for large meshes (remember to configure `DRACOLoader` on the Three.js side)
- **Animation**: enable; use `Group by NLA Track` so each action becomes a separately named `AnimationClip`

### What Survives Export (and What Doesn't)

| Blender feature                          | Exports to glTF?                          |
| ---------------------------------------- | ----------------------------------------- |
| Principled BSDF + image textures         | ✅ → `MeshStandardMaterial`               |
| Base color, metallic, roughness, normal  | ✅ (correct texture slots)                |
| Emission, alpha blend/clip               | ✅                                        |
| Procedural textures (noise, gradients)   | ❌ must be **baked** to image textures    |
| Node setups other than Principled BSDF   | ❌ only Principled BSDF is understood     |
| Armature/bone animation                  | ✅ → skeletal animation                   |
| Shape keys                               | ✅ → morph targets                        |
| Lights, cameras                          | ✅ (punctual lights via extension)        |
| Modifiers                                | Only if "Apply Modifiers" is on          |
| Physics, constraints, drivers            | ❌ bake to keyframes first                |
| Custom properties                        | ✅ → `userData` (enable "Custom Properties") |

### Materials: Principled BSDF → MeshStandardMaterial

Blender's Principled BSDF maps 1:1 to glTF PBR, which Three.js loads as `MeshStandardMaterial` (or `MeshPhysicalMaterial` when transmission/clearcoat extensions are present):

```
Base Color   → material.map / material.color   (sRGB)
Metallic     → material.metalnessMap (B channel) / material.metalness
Roughness    → material.roughnessMap (G channel) / material.roughness
Normal       → material.normalMap   (linear, OpenGL orientation)
Emission     → material.emissiveMap / material.emissive
Alpha        → material.transparent + material.opacity / alphaTest
```

Imported models look dark without environment lighting — glTF assumes IBL:

```javascript
import { RGBELoader } from "three/addons/loaders/RGBELoader.js";

new RGBELoader().load("studio.hdr", (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture; // PBR materials now have light to reflect
});
```

### Custom Properties → userData

Blender custom properties (on objects, meshes, materials) export as glTF `extras` and arrive in `userData`. Use them to drive game/app logic:

```javascript
loader.load("level.glb", (gltf) => {
  gltf.scene.traverse((obj) => {
    if (obj.userData.interactive) makeClickable(obj);
    if (obj.userData.collider === "box") addBoxCollider(obj);
    if (obj.userData.spawnPoint) spawnPoints.push(obj.position.clone());
  });
});
```

### Animations from Blender

Each Blender **action** becomes an `AnimationClip`. To export multiple actions on one armature, stash them as NLA tracks (Action Editor → "Push Down"):

```javascript
loader.load("character.glb", (gltf) => {
  const mixer = new THREE.AnimationMixer(gltf.scene);

  // Clips are found by the Blender action name
  const idle = THREE.AnimationClip.findByName(gltf.animations, "Idle");
  const walk = THREE.AnimationClip.findByName(gltf.animations, "Walk");

  mixer.clipAction(idle).play();
});
```

If `gltf.animations` is empty or missing clips: the actions weren't pushed to NLA tracks, or the exporter's Animation section was set to export only the active action.

### Headless / Automated Export

Automate exports with Blender's CLI for asset pipelines and CI:

```python
# export_glb.py - run with: blender -b scene.blend --python export_glb.py -- output.glb
import bpy, sys

output = sys.argv[sys.argv.index("--") + 1]

bpy.ops.export_scene.gltf(
    filepath=output,
    export_format="GLB",
    export_apply=True,          # apply modifiers
    export_yup=True,            # +Y up for Three.js
    export_animations=True,
    export_extras=True,         # custom properties -> userData
    export_draco_mesh_compression_enable=True,
)
```

```bash
# Batch-export every .blend in a folder
for f in assets/*.blend; do
  blender -b "$f" --python export_glb.py -- "public/models/$(basename "${f%.blend}").glb"
done
```

### Post-Export Optimization

Run exported files through `gltf-transform` before shipping:

```bash
npm install -g @gltf-transform/cli

# Draco geometry + WebP textures + dedupe/prune in one pass
gltf-transform optimize model.glb model-opt.glb --texture-compress webp

# Or individual steps
gltf-transform draco model.glb model-draco.glb
gltf-transform etc1s model.glb model-ktx2.glb   # KTX2/Basis textures (needs KTX2Loader)
```

## Unity → Three.js

Unity has no built-in glTF exporter — use one of these packages:

- **UnityGLTF** (`com.unity.cloud.gltfast` companion / KhronosGroup): editor + runtime export, best glTF fidelity
- **glTFast**: primarily an importer, but supports runtime export of simple scenes

### Editor Export with UnityGLTF

1. Install via Package Manager: `https://github.com/KhronosGroup/UnityGLTF.git`
2. Select objects in the Hierarchy
3. `Assets → UnityGLTF → Export selected as GLB`

Materials on Standard/URP Lit shaders convert to glTF PBR. Custom shaders export as plain PBR with whatever properties map (`_BaseMap`, `_BumpMap`, `_MetallicGlossMap`) — anything else must be rebuilt in Three.js.

### Runtime Export from Unity (C#)

```csharp
using UnityGLTF;

public class GltfExport : MonoBehaviour
{
    public void ExportSelection(Transform root, string path)
    {
        var settings = GLTFSettings.GetOrCreateSettings();
        var context = new ExportContext(settings);
        var exporter = new GLTFSceneExporter(new[] { root }, context);
        exporter.SaveGLB(path, "scene");
    }
}
```

### Unity Gotchas

- **Mirrored geometry**: Unity's left-handed → glTF right-handed conversion negates an axis. If UVs or normals look flipped, update the exporter package — older versions had winding-order bugs.
- **Lightmaps don't export**: glTF has no lightmap slot. Either bake lighting into base color textures, re-light in Three.js with an environment map, or export lightmaps manually and apply as `material.lightMap` (requires a second UV set — `uv2`).
- **Terrain, particles, ProBuilder**: convert to regular meshes before export.
- **Animator Controllers don't export**: only the raw clips do; rebuild state machine logic in JS with `AnimationMixer` and crossfades.
- **Prefer glTF over FBX**: exporting FBX from Unity and loading with `FBXLoader` loses PBR materials and multiplies scale by 100. If you're stuck with FBX, convert offline with `FBX2glTF` instead of loading it directly.

## Three.js → Blender / Unity (Reverse Direction)

Export scenes built in Three.js back to DCC tools with `GLTFExporter`:

```javascript
import { GLTFExporter } from "three/addons/exporters/GLTFExporter.js";

const exporter = new GLTFExporter();
exporter.parse(
  scene,
  (result) => {
    // result is an ArrayBuffer when binary: true
    const blob = new Blob([result], { type: "model/gltf-binary" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "scene.glb";
    a.click();
  },
  (error) => console.error(error),
  { binary: true, animations: mixerClips },
);
```

- **Blender**: File → Import → glTF 2.0 (built-in, lossless round-trip for PBR + animations)
- **Unity**: import with glTFast (`com.unity.cloud.gltfast`) — drag the `.glb` into Assets

Only export what glTF supports: `ShaderMaterial` and post-processing effects won't survive; bake them or re-author on the other side.

## Recommended Asset Pipeline

```
1. Author in Blender/Unity (source of truth: .blend / Unity project)
2. Export .glb          → headless blender script or UnityGLTF
3. Optimize             → gltf-transform optimize (Draco/meshopt + KTX2/WebP)
4. Validate             → https://github.khronos.org/glTF-Validator/ or gltf-transform inspect
5. Load in Three.js     → GLTFLoader (+ DRACOLoader/KTX2Loader as needed)
```

Check exported files before debugging Three.js code:

```bash
gltf-transform inspect model.glb   # meshes, materials, textures, animations, sizes
```

## Troubleshooting Imported Assets

| Symptom                          | Cause                                    | Fix                                                  |
| -------------------------------- | ---------------------------------------- | ---------------------------------------------------- |
| Model is black/dark              | No environment light for PBR             | Set `scene.environment` (HDR + PMREM)                |
| Model lying on its side          | Y-up conversion disabled at export       | Re-export with "+Y Up"; don't rotate in code         |
| Model mirrored/inside-out        | Handedness flip failed (Unity)           | Update exporter; check negative scales               |
| 100x too large or tiny           | FBX cm units in the chain                | Export glTF directly; avoid FBX                      |
| Textures washed out or too dark  | Wrong color space                        | Color maps sRGB, data maps linear (GLTFLoader does this — don't override) |
| Materials are plain gray         | Procedural/custom shaders didn't export  | Bake to image textures in Blender                    |
| `gltf.animations` empty          | Actions not in NLA / animation export off | Push Down actions to NLA tracks, re-export           |
| Seams/faceted shading            | Normals not exported or split by modifier | Enable Normals in export; Shade Smooth in Blender    |
| Jagged mesh after Draco          | Compression too aggressive               | Raise quantization bits or disable Draco for hero assets |

## Performance Tips

1. **One material per asset when possible** - each glTF material becomes a draw call per mesh using it
2. **Join static meshes in Blender** (Ctrl+J) - fewer objects = fewer draw calls
3. **Decimate before export** - Blender's Decimate modifier; web targets rarely need film-res meshes
4. **Resize textures at bake time** - 1K-2K for most assets; 4K only for hero close-ups
5. **Draco + KTX2 for delivery** - 5-10x smaller downloads; decode cost is worth it beyond ~1MB
6. **Strip unused data** - `gltf-transform prune` removes orphaned nodes, unused UVs and vertex colors

## See Also

- `threejs-loaders` - GLTFLoader, DRACOLoader, KTX2Loader configuration
- `threejs-animation` - Playing and blending exported animation clips
- `threejs-materials` - MeshStandardMaterial / MeshPhysicalMaterial details
- `threejs-lighting` - Environment lighting (IBL) for PBR assets
