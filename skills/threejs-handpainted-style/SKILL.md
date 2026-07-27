---
name: threejs-handpainted-style
description: Hand-painted stylized art style in Three.js - Warcraft 3 / classic Blizzard look, baked-lighting diffuse textures, low-poly meshes, toon shading, team colors, painted texture workflow. Use when creating stylized/hand-painted games, converting PBR assets to a painterly look, or building RTS/MMO-style visuals.
---

# Hand-Painted Style (Warcraft 3 Look)

## The Style, Defined

The 2002 Warcraft 3 / classic Blizzard look is a set of technical constraints turned into an art style:

1. **All lighting is painted into the diffuse texture** - highlights, shadows, ambient occlusion, even fake rim light are brush strokes, not computed lighting
2. **One texture per asset** - a single diffuse map (WC3 used 128-256px; use 512-1024px today), no normal/roughness/metalness maps
3. **Low-poly with strong silhouettes** - WC3 units were 300-1500 triangles; the texture does the detail work, the mesh does the shape work
4. **Saturated, value-contrasted colors** - readable at RTS camera distance; every unit pops against the ground
5. **Exaggerated proportions** - big hands, shoulders, weapons; thick straps and trim (thin details vanish at low resolution)

The rendering consequence: **you don't want PBR**. `MeshStandardMaterial` recomputing light over a texture that already contains light gives double-shaded mud.

## Quick Start

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("grunt.glb", (gltf) => {
  // Replace exported PBR materials with unlit - texture already contains lighting
  gltf.scene.traverse((child) => {
    if (child.isMesh) {
      const map = child.material.map; // keep the painted diffuse
      child.material = new THREE.MeshBasicMaterial({ map });
    }
  });
  scene.add(gltf.scene);
});
```

No lights, no environment map, no shadows required — the texture IS the lighting. This is also extremely fast.

## Material Choices

Three tiers, from most to least authentic:

### Fully Baked (authentic WC3)

```javascript
// MeshBasicMaterial = unlit. What you paint is what you see.
const material = new THREE.MeshBasicMaterial({ map: diffuseTexture });
```

### Baked + Scene Tint (day/night cycles, cave interiors)

```javascript
// MeshLambertMaterial: cheap vertex-ish lighting multiplies the painted texture.
// Keep painted lighting subtle-to-moderate so real light can modulate it.
const material = new THREE.MeshLambertMaterial({ map: diffuseTexture });

// Flat, low-contrast rig - hemisphere fill + one directional, NO shadows
const hemi = new THREE.HemisphereLight(0xfff2e0, 0x4a5a70, 1.2); // warm sky, cool ground
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(5, 10, 3);
scene.add(hemi, sun);
```

### Toon-Shaded (modern stylized, WC3 Reforged-ish)

```javascript
// MeshToonMaterial: banded lighting steps over the painted texture
const gradientMap = new THREE.DataTexture(
  new Uint8Array([80, 160, 255]), // 3 bands: shadow, mid, light
  3, 1, THREE.RedFormat,
);
gradientMap.minFilter = THREE.NearestFilter;
gradientMap.magFilter = THREE.NearestFilter;
gradientMap.needsUpdate = true;

const material = new THREE.MeshToonMaterial({ map: diffuseTexture, gradientMap });
```

**Never** use `MeshStandardMaterial` + `scene.environment` for these assets — PBR reflections destroy the painterly read.

## Texture Settings

```javascript
const texture = new THREE.TextureLoader().load("grunt_diffuse.png");
texture.colorSpace = THREE.SRGBColorSpace; // painted colors are sRGB

// Default (soft, like WC3 at high settings):
texture.minFilter = THREE.LinearMipmapLinearFilter;
texture.magFilter = THREE.LinearFilter;

// Optional crunchy retro look (visible texels up close):
// texture.magFilter = THREE.NearestFilter;

// Painted textures blur badly at grazing angles - anisotropy helps ground tiles
texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
```

For alpha-cutout cards (leaves, hair tufts, banners — WC3 used these everywhere):

```javascript
const foliage = new THREE.MeshBasicMaterial({
  map: leafTexture,
  alphaTest: 0.5,        // hard cutout, no sorting problems
  side: THREE.DoubleSide, // cards visible from both sides
});
```

## Team Colors (the classic WC3 feature)

WC3 recolors each unit per player using a mask: pixels marked in the mask take the player color, the rest stay painted. Reproduce it by extending the material with a mask texture (white = team-colorable areas):

```javascript
function makeTeamColorMaterial(diffuse, teamMask, teamColor) {
  const material = new THREE.MeshBasicMaterial({ map: diffuse });

  material.onBeforeCompile = (shader) => {
    shader.uniforms.teamMask = { value: teamMask };
    shader.uniforms.teamColor = { value: new THREE.Color(teamColor) };

    shader.fragmentShader = shader.fragmentShader
      .replace(
        "#include <common>",
        `#include <common>
         uniform sampler2D teamMask;
         uniform vec3 teamColor;`,
      )
      .replace(
        "#include <map_fragment>",
        `#include <map_fragment>
         float mask = texture2D(teamMask, vMapUv).r;
         // Multiply keeps the painted shading, tints the hue - exactly WC3's trick
         diffuseColor.rgb = mix(diffuseColor.rgb, diffuseColor.rgb * teamColor, mask);`,
      );
  };

  return material;
}

const RED_PLAYER = 0xc00000;
const BLUE_PLAYER = 0x0042ff;
grunt.material = makeTeamColorMaterial(diffuse, mask, RED_PLAYER);
```

Author the mask in Blender as a second grayscale image: paint white over tabards, banners, armor trim.

## Painting the Textures (Blender Workflow)

The asset pipeline (pairs with `threejs-blender-unity`):

1. **Model low-poly** - 500-3000 tris for a hero unit today; delete faces the camera never sees
2. **Unwrap generously** - give the face and chest the most texel density; mirror the UVs of symmetric parts to double effective resolution
3. **Bake a lighting base** - bake AO and/or a top-down sun into the diffuse as a starting layer:
   - Cycles → Bake → Ambient Occlusion onto the diffuse image
   - This gives the "light comes from above" foundation WC3 textures all share
4. **Paint over the bake** in Texture Paint mode (or export to Krita/Photoshop):
   - Block in flat local colors first
   - Paint shadows as **saturated, hue-shifted darks** (purple-ish shadows on skin, blue-ish on metal) — never plain black
   - Paint highlights as **desaturated brights**, hottest on upward-facing surfaces
   - Add painted edge highlights on every silhouette edge and trim — this fakes rim light and sells the style
5. **Export GLB** with the diffuse only — no metallic/roughness/normal maps to export

For the export itself (settings, animations, headless CLI), see `threejs-blender-unity`.

### Palette Rules of Thumb

- 3-5 material families per unit (skin, metal, cloth, leather, glow), each with its own hue
- Shift hue between light and shadow, not just value
- Reserve near-white and near-black for tiny accents (eye glints, deepest creases)
- Test at 25% zoom — if the unit is unreadable small, increase value contrast

## Finishing Touches

### Painted-Style Outlines (inverted hull)

```javascript
function addOutline(mesh, thickness = 0.02, color = 0x1a0f0a) {
  const outline = new THREE.Mesh(
    mesh.geometry,
    new THREE.MeshBasicMaterial({ color, side: THREE.BackSide }),
  );
  outline.scale.setScalar(1 + thickness);
  mesh.add(outline);
}
```

Use a very dark warm brown, not black — matches painted linework.

### Blob Shadows Instead of Shadow Maps

WC3 shipped blob shadows; they read perfectly under stylized units and cost nothing:

```javascript
const blob = new THREE.Mesh(
  new THREE.PlaneGeometry(1.4, 1.4),
  new THREE.MeshBasicMaterial({
    map: radialGradientTexture, // soft black circle, transparent edges
    transparent: true,
    opacity: 0.45,
    depthWrite: false,
  }),
);
blob.rotation.x = -Math.PI / 2;
blob.position.y = 0.01; // just above the ground
unit.add(blob);
```

### Glow Cards for Magic Effects

WC3 spell effects are additive-blended painted sprites:

```javascript
const glow = new THREE.Mesh(
  new THREE.PlaneGeometry(1, 1),
  new THREE.MeshBasicMaterial({
    map: paintedGlowTexture,
    blending: THREE.AdditiveBlending,
    transparent: true,
    depthWrite: false,
  }),
);
// Billboard it toward the camera each frame, or use THREE.Sprite
```

### Vertex Color Tinting

Cheap variation without new textures (WC3 tinted creeps this way):

```javascript
// Same mesh + texture, different tint per instance
const eliteGrunt = grunt.clone();
eliteGrunt.material = grunt.material.clone();
eliteGrunt.material.color.set(0xff9999); // reddish elite variant
```

### Scene Atmosphere

```javascript
// Saturated fog sells the painted-world feel - tint it toward the sky color
scene.fog = new THREE.Fog(0x3a4a6b, 30, 120);
scene.background = new THREE.Color(0x3a4a6b);

// Slightly boosted saturation/contrast in tone mapping direction:
renderer.toneMapping = THREE.NoToneMapping; // keep painted colors exact
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

`NoToneMapping` matters: ACES/filmic tone mapping desaturates exactly the punchy colors this style depends on.

## RTS Camera (to complete the look)

```javascript
const camera = new THREE.PerspectiveCamera(30, aspect, 1, 500); // narrow FOV = less distortion
camera.position.set(0, 22, 14); // high, pitched ~55-60 degrees down
camera.lookAt(0, 0, 0);
// Pan by moving position and target together; zoom by dollying along the view vector
```

The narrow FOV + steep angle is why WC3 units could skip detail on soles, chins, and undersides — do the same and spend those texels elsewhere.

## Performance Tips

1. **Unlit materials are nearly free** - no lighting math; hundreds of units are viable
2. **Atlas shared textures** - all doodads of a tileset on one 2048px atlas = one material = batchable
3. **`InstancedMesh` for armies** - identical units with per-instance `setColorAt()` for team tinting
4. **No shadow maps** - blob shadows cost one transparent quad per unit
5. **Small textures ship fast** - a hand-painted 512px diffuse often beats a 2K PBR set by 10-20x in bytes
6. **Merge static doodads** - `BufferGeometryUtils.mergeGeometries()` for rocks/trees sharing an atlas

## See Also

- `threejs-blender-unity` - Exporting the painted assets from Blender
- `threejs-materials` - MeshBasicMaterial/MeshToonMaterial reference
- `threejs-textures` - Filtering, color space, atlas UVs
- `threejs-shaders` - Going further with onBeforeCompile
