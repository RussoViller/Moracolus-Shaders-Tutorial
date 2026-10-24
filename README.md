# Moracolus — Shader Development & Porting Guide

Official community documentation for creating native shaders for Moracolus and adapting existing shader packs to its rendering environment.

Moracolus is a client-side Minecraft Java Edition mod with its own shader-pack system, designed with portability across desktop computers and Android devices in mind. This guide covers the native pack format, OptiFine/Iris compatibility, GLSL, rendering backends, optimization, debugging, and automatic quality presets.

- **Documentation Mod version:** 1.6.0+
- **Mod author:** RussoViller
- **Mod license:** [RVCL-1.0](https://github.com/RussoViller/russostyle-saves/blob/main/RVCL-1.0.md)

> This guide documents behavior identified in the Moracolus 1.6.0 code. Available APIs and backend details can vary with the Minecraft environment and rendering path. Future compatibility must be tested; dependency version ranges alone do not guarantee support.

## Contents

1. [Introduction](#1-introduction)
2. [Internal architecture](#2-internal-architecture)
3. [Pack structure and `moracolusjson`](#3-pack-structure-and-moracolusjson)
4. [GLSL development reference](#4-glsl-development-reference)
5. [Creating a native shader pack](#5-creating-a-native-shader-pack)
6. [Porting OptiFine/Iris shaders](#6-porting-optifineiris-shaders)
7. [Performance optimization](#7-performance-optimization)
8. [OpenGL, Vulkan, and translation layers](#8-opengl-vulkan-and-translation-layers)
9. [Debugging](#9-debugging)
10. [Testing and distribution](#10-testing-and-distribution)
11. [Automatic presets](#11-automatic-presets)
12. [Quick reference](#12-quick-reference)

---

## 1. Introduction

### 1.1 What is Moracolus?

Moracolus is a client-side Fabric mod that adds its own shader system to Minecraft Java Edition. It was designed with Android devices and launchers using translation layers such as MobileGlues in mind, while also supporting desktop environments.

Moracolus recognizes two pack formats:

- **Native Moracolus format:** a `moracolus.json` file exists at the pack root. This is the recommended format and exposes the full set of documented features, including declared passes, render targets, terrain stages, End portal shaders, and device-tier option defaults.
- **OptiFine/Iris compatibility format:** a `shaders/` directory exists at the pack root. Only part of this format is supported; packs written specifically for Iris or OptiFine may need to be ported.

A pack can include both `moracolus.json` and `shaders/`. This is a **hybrid pack**: Moracolus uses the native manifest, while the same archive can retain compatibility with Iris/OptiFine, subject to each loader's own support.

### 1.2 Minecraft, Sodium, and compatibility

Moracolus depends on Sodium and does not modify Sodium's source code. Depending on the pack, it can replace Sodium terrain programs, run post-processing passes after the world is drawn, replace the End portal shader, hide vanilla clouds, and change the selection outline color.

The mod declares incompatibilities with Iris, Oculus, OptiFabric, and Canvas in its Fabric metadata. Check the metadata and distribution notes for the particular build you use. A declared dependency range does not guarantee compatibility with future versions that have not been tested.

### 1.3 Rendering paths and backends

The pack format and intended visual behavior are shared, but the implementation path depends on the environment:

- **Direct OpenGL path:** Moracolus creates and binds GLSL programs and manages rendering resources directly. Support depends on the OpenGL context and the capabilities exposed by the driver or translation layer.
- **Blaze3D path:** where available, Moracolus prepares shaders and delegates compilation and execution to Minecraft's rendering abstraction. Depending on the environment, that abstraction may use OpenGL or Vulkan. Moracolus does **not** implement a separate, independent Vulkan renderer.
- On the Blaze3D path, non-sampler uniforms may be packed into the `std140` `MoraUniforms` block. Write ordinary GLSL uniform declarations and do not assume every backend handles uniforms identically.
- Where available, `MORA_BACKEND_OPENGL`, `MORA_BACKEND_VULKAN`, `MORA_IS_VULKAN`, and the `VULKAN`/`OPENGL` conditions can be used for genuinely backend-specific code. Prefer shared code whenever possible.

### 1.4 Useful controls and paths

Default controls (they may be remapped in **Options → Controls**):

- **O:** open the Moracolus menu.
- **R:** reload shader packs from disk.
- **K:** toggle shaders on or off.

Useful paths:

- Shader packs: `<game directory>/shaderpacks`
- Mod configuration: `config/moracolus/moracolus.json`
- Saved pack options: `config/moracolus/packs/<pack-id>.properties`
- Automatic preset state: `config/moracolus/presets.properties`
- Benchmark results: `config/moracolus/benchmark.properties`

## 2. Internal architecture

### 2.1 High-level rendering flow

1. `Moracolus.onInitializeClient()` loads `config/moracolus/moracolus.json` and registers key bindings.
2. On the first world frame, `PipelineManager.ensureInit()` detects the device, scans `shaderpacks/`, and applies the selected pack.
3. `PipelineManager.apply()` loads the pack, reads saved options, applies automatic preset selection when eligible, and builds the required pipelines:
   - `NativePipeline` for native post-processing;
   - `LegacyPipeline` for the supported OptiFine/Iris-style path;
   - `TerrainStage` for Sodium terrain programs;
   - `EndPortalStage` for a native End portal shader.
4. Each frame, `FrameState.begin()` collects camera, matrices, time, weather, and lighting data. Sodium mixins may switch terrain programs and uniforms during chunk rendering. Moracolus captures depth and runs post-processing before writing the result to the main framebuffer.

### 2.2 Main internal packages

The following names are useful when reading implementation details. Internal classes are not necessarily stable public APIs; shader packs should prefer the interfaces documented here.

| Package/class | Purpose |
|---|---|
| `config/MoraConfig` | Mod configuration (JSON) |
| `device/DeviceInfo` | Device, GPU, translation layer, and backend detection |
| `device/TierLogic` | Device tier and `MORA_TIER` |
| `gui/BenchEngine` | CPU, memory, storage, UX, and GPU benchmark |
| `pack/PackRepository` | Discovers packs in `shaderpacks/` |
| `pack/NativeManifest` | Reads `moracolus.json` |
| `pack/LegacyPackInfo` | Reads supported legacy properties and options |
| `option/OptionSet`, `option/ShaderOption` | Option values, screens, profiles, and types |
| `preset/PresetLevel`, `PresetRanker`, `AutoPreset` | Preset recognition, ranking, and automatic selection |
| `glsl/Prelude` | Injected `MORA_*` and `MC_*` definitions |
| `glsl/GlslIncludes` | Resolves `#include` directives |
| `glsl/LegacyTranslator` | Adapts supported legacy GLSL constructs |
| `pipeline/PipelineManager` | Coordinates rendering pipelines |
| `pipeline/NativePipeline`, `LegacyPipeline` | Native and legacy post-processing |
| `pipeline/TerrainStage` | Sodium terrain shader stage |
| `pipeline/FrameState`, `Uniforms` | Per-frame values and known uniforms |
| `pipeline/AdaptiveScaler` | Adaptive render scaling |
| `terrain/BlockIdMap`, `SpriteInfo` | Block and sprite metadata |
| `gl/CustomTextures` | Pack-provided PNG textures |

### 2.3 GLSL compilation behavior

- The `#version` directive is read by the legacy translator. The resulting version is at least `330`, at most `460`, and uses the `core` profile. A file without a version directive becomes `#version 330 core`; for example, `#version 150` becomes `#version 330 core`.
- `#extension` lines are preserved except for extensions treated as covered by the core profile.
- After `#version`, Moracolus injects device definitions, dimension definitions, option definitions, stage definitions, and then the shader body.
- Native-pack `#include` directives are resolved before compilation. Each file is included at most once per shader, with a maximum include depth of 32.
- If compilation fails, the driver's message is shown in the **Shader Info** tab and recorded in `logs/latest.log`.

## 3. Pack structure and `moracolus.json`

### 3.1 Pack discovery

A pack can be a directory or a `.zip` file inside `shaderpacks/`.

- `moracolus.json` at the root: native pack.
- `shaders/` at the root: OptiFine/Iris compatibility pack.
- Both: hybrid pack; Moracolus uses the native manifest.

Archives containing neither are ignored. The pack ID is based on the folder or filename, and saved options are stored under `config/moracolus/packs/`.

### 3.2 Recommended native pack layout

```text
MyShader/
├── moracolus.json            # Required
├── License.txt               # Recommended
├── lib/
│   └── common.glsl           # Optional reusable functions
├── post/
│   ├── bloom.fsh             # Post-processing passes
│   └── final.fsh
├── terrain/
│   ├── terrain.vsh           # Optional
│   └── terrain.fsh           # Optional
├── endportal/
│   └── portal.fsh            # Optional
├── textures/
│   └── lut.png               # Optional custom textures
└── lang/
    ├── en_us.lang            # Optional option labels
    └── pt_br.lang
```

Internal folder names are flexible: `moracolus.json` points to files using paths relative to the pack root. The manifest is required. The default language directory is `lang`, which can be changed through `import.lang`.

### 3.3 Manifest fields

The manifest's top-level fields include:

| Field | Purpose |
|---|---|
| `format` | Integer format version; currently `1`. Higher values are rejected. |
| `name`, `version`, `author`, `description`, `license` | Pack metadata shown in **Shader Info**. |
| `sunPathRotation` | Angle in degrees applied to the sun path. |
| `passes` | Post-processing pass list; required, and at least one pass must write to `screen`. |
| `targets` | Intermediate render textures. |
| `options` | User-configurable options. |
| `screens` | Option-menu organization. |
| `profiles` | Profiles/presets containing option values. |
| `presetLevels` | Optional quality level for each profile. |
| `terrain` | Terrain shader stage. |
| `endPortal` | End portal shader. |
| `textures` | Pack-provided textures. |
| `blocks` | Block IDs for the terrain stage. |
| `blockProperties` | Path to an OptiFine-format `block.properties` file. |
| `vanillaClouds` | Condition; when false, vanilla clouds are hidden. |
| `selectionOutline` | RGB selection-outline color. |
| `import` | Reuse options and language entries from an OptiFine-style pack. |

#### Post-processing pass example

```json
{
  "name": "bloom_blur",
  "vertex": "post/custom.vsh",
  "fragment": "post/blur.fsh",
  "output": "bloom_b",
  "when": "BLOOM && TIER >= 1",
  "optional": true,
  "interval": 2,
  "phase": 1
}
```

- `name`: pass name; defaults to a generated name such as `passN`.
- `vertex`: optional. If omitted, a full-screen vertex shader providing `mora_UV` is used.
- `fragment`: required fragment shader path.
- `output`: a target name, a list of targets, or `screen`. `outputs` is also accepted.
- `when`: optional condition; defaults to always enabled.
- `optional`: if compilation or execution fails, disable this pass rather than failing the entire pack.
- `interval`: run once every N frames, from 1 to 8.
- `phase`: frame phase within the interval.

Verified constraints include: `screen` cannot be combined with other outputs; all outputs from one pass must use the same scale; output count cannot exceed the GPU's color-attachment limit; and a pass cannot read and write the same target in one frame. Use `history: true` and read `tex_<name>_prev` for temporal feedback.

#### Render target example

```json
{
  "name": "bloom_a",
  "format": "rgba16f",
  "scale": 0.25,
  "filter": "linear",
  "mipmap": false,
  "history": false,
  "clear": false,
  "clearColor": [0, 0, 0, 0]
}
```

- Target names may contain letters, digits, and underscores. Reserved names include `scene`, `depth`, `noise`, and `screen`.
- `format` selects the target format; use a format supported by the device.
- `scale` is a resolution fraction or the ID of a numeric option, such as `BLOOM_SCALE`.
- `filter` is `linear` (default) or `nearest`.
- `mipmap` generates mipmaps after writing.
- `history` retains the previous frame and exposes it as `tex_<name>_prev`.
- `clear` clears each frame (ignored when history is enabled); `clearColor` sets the clear color.

Consult the actual pack schema and the installed build's diagnostics when using less common fields. Do not assume an undocumented field is supported.

### 3.4 Language files

Language files use `key=value` syntax, similar to OptiFine. Recognized keys include `option.<ID>`, `option.<ID>.comment`, `screen.<NAME>`, `screen.<NAME>.comment`, `value.<ID>.<VALUE>`, `profile`, `profile.comment`, and `profile.<NAME>`.

`en_us.lang` is loaded as a base; the active game language then overrides keys it contains.

### 3.5 Conditions

Conditions used by `when`, `vanillaClouds`, and `terrain.when` support `||`, `&&`, `!`, `==`, `!=`, `>=`, `<=`, `>`, `<`, and parentheses. Numeric expressions are accepted, for example `TIER >= 2`.

Available identifiers include `MOBILE`, `DESKTOP`, `TIER`, GPU vendor identifiers such as `ADRENO`, `MALI`, `POWERVR`, `XCLIPSE`, `NVIDIA`, `AMD`, `INTEL`, and `APPLE`; translation-layer identifiers such as `MOBILEGLUES`, `GL4ES`, `ANGLE`, and `ZINK`; dimensions such as `OVERWORLD`, `NETHER`, and `END`; `DIMENSION`, `true`, `false`, and pack option IDs. Where supported, `VULKAN` and `OPENGL` are also available. Boolean options evaluate to `1` or `0`, enums to their selected index, and numeric options to their numeric value. Unknown identifiers evaluate to `0`.

## 4. GLSL development reference

### 4.1 Version and profile

Write native shaders using GLSL core. Moracolus compiles native shaders as `#version 330 core` or later. Use `in`/`out` instead of `varying`/`attribute`, use `texture()` instead of `texture2D()`, and declare a fragment output, for example:

```glsl
layout(location = 0) out vec4 outColor;
```

Valid GLSL 1.50 syntax remains valid in GLSL 3.30, but the native format does not use a compatibility profile.

### 4.2 Full-screen pass inputs

With the default vertex shader, `in vec2 mora_UV` provides screen coordinates from 0 to 1. With a custom vertex shader, Moracolus defines `MORA_VERTEX_SHADER` and provides the `mora_Position` macro for a full-screen triangle. The triangle uses three vertices and no vertex attributes.

### 4.3 Native-pass samplers

| Sampler | Contents |
|---|---|
| `tex_scene` | Rendered world color |
| `tex_depth` | Scene depth, including translucent geometry |
| `tex_depth_solid` | Solid-block depth only |
| `tex_noise` | Repeating 256×256 noise texture |
| `tex_<target>` | Current contents of a render target |
| `tex_<target>_prev` | Previous frame for a target with `history: true` |
| `tex_<texture>` | Texture declared in `textures` |

Any other sampler name causes an `unknown sampler` error.

**Temporal target behavior:** targets using `history` swap buffers only on frames where a pass writes to them. This avoids stale/new-frame flickering when a pass has an `interval` greater than one or its `when` condition is false.

### 4.4 Common uniforms

Declare a uniform using its exact name, for example `uniform float mora_Rain;`. Undeclared uniforms have no cost.

| Category | Names and meaning |
|---|---|
| Screen/pass | `mora_Resolution`, `mora_TexelSize`, `mora_PassResolution`, `mora_AspectRatio`, `aspectRatio`, `viewWidth`, `viewHeight`, `mora_RenderScale` |
| Time | `mora_Time` / `frameTimeCounter`, `mora_FrameTime` / `frameTime`, `mora_Frame` / `frameCounter`, `mora_WorldTime` / `worldTime`, `worldDay`, `mora_TimeOfDay`, `mora_MoonPhase` / `moonPhase` |
| Camera/matrices | `mora_Projection` / `gbufferProjection`, `mora_ProjectionInv` / `gbufferProjectionInverse`, `mora_ModelView` / `gbufferModelView`, `mora_ModelViewInv` / `gbufferModelViewInverse`, previous projection/model-view matrices, `mora_CameraPos` / `cameraPosition`, `mora_PrevCameraPos` / `previousCameraPosition`, `mora_CameraDelta`, `mora_CameraPosWrapped`, `mora_EyePos` / `eyePosition`, `mora_RelativeEyePos`, `eyeAltitude`, `mora_Near` / `near`, `mora_Far` / `far` |
| Sun/sky/fog | `mora_SunDir`, `mora_SunDirView`, `mora_MoonDirView`, `mora_LightDirView`, `mora_UpDirView`, `sunPosition`, `moonPosition`, `upPosition`, `shadowLightPosition`, `mora_SunAngle` / `sunAngle`, `shadowAngle`, `mora_DayFactor`, `mora_SkyColor` / `skyColor`, `mora_FogColor` / `fogColor`, `mora_FogStart`, `mora_FogEnd`, `fogStart`, `fogEnd`, `fogMode` |
| Weather/player | `mora_Rain` / `rainStrength`, `mora_Thunder` / `thunderStrength`, `mora_Wetness` / `wetness`, `mora_EyeInWater` / `isEyeInWater`, `mora_Blindness` / `blindness`, `mora_NightVision` / `nightVision`, `mora_Darkness` / `darknessFactor`, `mora_Brightness` / `screenBrightness`, `hideGUI`, `mora_EyeBrightness`, `eyeBrightness` / `eyeBrightnessSmooth`, `mora_HeldLight`, `heldBlockLightValue`, `heldBlockLightValue2`, `mora_Dimension` |
| Device | `mora_Tier` (0–3) and `mora_Mobile` (1 on mobile) |
| Other | `atlasSize`, `noiseTextureResolution` (256), `alphaTestRef` (0.1) |
| Live options | `uniform <bool|int|float> <ID>;` is generated automatically for options marked `live`. |

On the Blaze3D path, these uniforms may be packed into the `std140` `MoraUniforms` block. Supported block types include `float`, `int`, `uint`, `bool`, vector types, `mat2`, `mat3`, `mat4`, and arrays of these types.

### 4.5 Injected defines

Device defines (all stages) include `MORACOLUS`, `MORA_FORMAT`, `MORA_PLATFORM_MOBILE` or `MORA_PLATFORM_DESKTOP`, `MORA_IS_MOBILE`, `MORA_TIER`, GPU identifiers such as `MORA_GPU_ADRENO` and `MORA_GPU_NVIDIA`, translation identifiers such as `MORA_TRANSLATOR_MOBILEGLUES`, and—where available—`MORA_BACKEND_OPENGL`, `MORA_BACKEND_VULKAN`, and `MORA_IS_VULKAN`. Mobile precision macros include `MORA_HIGHP`, `MORA_MEDIUMP`, and `MORA_LOWP`.

Dimension defines include `MORA_DIM_OVERWORLD`, `MORA_DIM_NETHER`, `MORA_DIM_END`, `MORA_DIM_OTHER`, and `MORA_DIMENSION`. Stage defines include `MORA_STAGE_POST`, `MORA_PASS_<PASS_NAME>`, `MORA_VERTEX_SHADER`, `MORA_FRAGMENT_SHADER`, and `MORA_STAGE_END_PORTAL`.

Native option defines:

- `bool`: `#define ID 1` or `0`.
- `int`: `#define ID 3` (example).
- `float`: `#define ID 0.45` (example).
- `enum`: `#define ID <index>` and `#define ID_<VALUE> <index>`.
- `live`: a uniform named `ID` plus `#define ID_LIVE`.

### 4.6 Common errors

| Message | Likely cause |
|---|---|
| `unknown sampler 'x'` | Unsupported sampler name; see the sampler table. |
| `reads and writes target` | A pass reads and writes the same target; use history and `tex_X_prev`. |
| `No pass writes to 'screen'` | The final screen pass is missing. |
| `has no fragment shader` | A pass has no `fragment` field. |
| `numeric option needs min/max or values` | An integer/float option is incomplete. |
| `enum option needs values` | An enum has no value list. |
| `File not found: x` | Incorrect path or `#include`. |

Driver errors such as `undeclared identifier` appear in **Shader Info** with the pass name and stage. A warning that a declared sampler is unused may occur when its usage is excluded by `#if`; it is not necessarily an error. To avoid it, declare the sampler inside the same conditional block in which it is used.

## 5. Creating a native shader pack

### 5.1 Prepare your environment

1. Install compatible versions of Fabric Loader, Sodium, Moracolus, and any additional dependencies required by your build.
2. Create a **directory** inside `shaderpacks/` rather than a ZIP while developing. You can edit files and press **R** to reload without repacking the archive.
3. Use an editor with GLSL and JSON syntax highlighting.
4. Keep `logs/latest.log` available; pack and compilation errors are recorded there.

### 5.2 First pack: saturation filter

Create `shaderpacks/MyFirst/moracolus.json`:

```json
{
  "format": 1,
  "name": "My First Shader",
  "version": "1.0.0",
  "author": "Your name",
  "passes": [
    { "name": "final", "fragment": "post/final.fsh", "output": "screen" }
  ],
  "options": [
    { "id": "SATURATION", "type": "float", "min": 0.0, "max": 2.0,
      "step": 0.05, "default": 1.2, "live": true }
  ]
}
```

Create `shaderpacks/MyFirst/post/final.fsh`:

```glsl
uniform sampler2D tex_scene;
layout(location = 0) out vec4 outColor;

void main() {
    vec3 c = texture(tex_scene, mora_UV).rgb;
    float l = dot(c, vec3(0.21, 0.72, 0.07));
    outColor = vec4(mix(vec3(l), c, SATURATION), 1.0);
}
```

Open the menu with **O**, select **My First Shader**, and apply it. Since `SATURATION` is marked `live`, changing it in the menu does not require recompilation.

### 5.3 Targets, bloom, and reduced-rate passes

A target can be rendered at a fraction of the screen resolution to reduce cost. A target with `history: true` can preserve previous-frame data for temporal effects. Passes can use `interval` and `phase` to run less frequently, but temporal effects may visibly stutter if updated too rarely. Validate the effect at the intended frame rate and on the target hardware.

Example manifest fragment:

```json
"targets": [
  { "name": "half", "format": "rgba16f", "scale": 0.5 },
  { "name": "accum", "format": "rgba16f", "scale": 0.5, "history": true }
],
"passes": [
  { "name": "downsample", "fragment": "post/downsample.fsh",
    "output": "half", "when": "GLOW" },
  { "name": "accumulate", "fragment": "post/accumulate.fsh",
    "output": "accum", "when": "GLOW", "interval": 1 },
  { "name": "final", "fragment": "post/final.fsh", "output": "screen" }
]
```

Ensure that a pass never reads and writes the same target in the same frame. Use a separate target or the previous-frame sampler as appropriate.

### 5.4 Options, screens, and profiles

Options can be `bool`, `int`, `float`, or `enum`. Define sensible defaults and valid ranges. Use screens to group related controls and profiles to expose coherent quality levels. Mark options as `live` only when they can safely be changed without recompiling the shader. For profile localization, provide `profile.<NAME>` entries in language files.

### 5.5 Terrain stage

A native pack can optionally provide terrain vertex and fragment shaders. Terrain shaders integrate with the Moracolus terrain stage and Sodium's rendering path, so they need more care than a full-screen post-processing pass. Validate solid, cutout, and translucent blocks; test water, glass, and portal surfaces; and ensure alpha values are correct. If terrain shader compilation fails, Moracolus may fall back to Sodium's normal terrain program.

### 5.6 Install, test, and distribute

Keep `moracolus.json` at the pack root. Test from an unpacked directory first, then create a ZIP with the manifest at the archive root. Include a license and clearly state the environments and backends you have actually tested.

## 6. Porting OptiFine/Iris shaders

### 6.1 Compatibility limitations

The compatibility path executes only a subset of the OptiFine/Iris pack format. Packs that rely on unsupported shadow pipelines, extra `gbuffers` stages, or Iris-specific features will usually require changes. Do not assume that a pack will behave identically just because it loads.

### 6.2 Legacy GLSL translation

The legacy translator can convert a number of older constructs, including `texture2D`/`texture3D`/`textureCube` to `texture`, some LOD/gradient texture calls, `varying`/`attribute` to `in`/`out`, and supported `gl_FragData` outputs. Some fixed-function constructs are replaced with full-screen equivalents. A missing vertex shader may be replaced with a default full-screen vertex shader that provides `texcoord`.

The compatibility path injects common OptiFine-style definitions such as `MC_VERSION`, `MC_GL_VERSION`, `MC_GLSL_VERSION`, quality values, platform/vendor identifiers, texture-format names, and Moracolus `MORA_*` definitions. Verify the actual values in the target environment rather than assuming every OptiFine feature exists.

### 6.3 Porting workflow

1. Put the original pack in `shaderpacks/`, select it, and inspect **Shader Info** for the detected format, limitations, and errors.
2. Identify the features the pack uses that Moracolus does not implement. Decide whether to remove them, approximate them, or replace them with native passes.
3. **Option A — keep the compatibility format:** modify the pack to avoid unsupported features. Use `#ifdef MORACOLUS` for Moracolus-specific sections when the same files must remain usable in other loaders.
4. **Option B — recommended hybrid pack:** add `moracolus.json` at the root and use `"import": { "options": "shaders" }` where supported to reuse options, screens, profiles, and language entries. Declare native passes for the effects you want to keep. Iris/OptiFine generally ignore the Moracolus manifest, but test this with your target loaders.
5. Replace unsupported shadow features with carefully tested approximations, such as sky-light-based lighting or screen-space occlusion, where appropriate.
6. Test the Overworld, Nether, and End, as well as underwater scenes, rain, daytime, and night.

### 6.4 Is the problem in the shader or in Moracolus?

Reduce the pack to the smallest reproducible case. Disable optional passes, remove unrelated effects, and inspect the compilation message and `latest.log`. If the same shader works on another supported backend but fails on one specific path, record the exact environment, driver/translation layer, and error. Avoid “fixes” that only hide the symptom or silently change unrelated rendering behavior.

## 7. Performance optimization

Mobile GPUs and translation layers can have stricter limits than desktop GPUs. Optimize based on measured cost rather than assumptions.

- Render expensive blur, bloom, and other full-screen effects at a reduced target scale where the visual result permits it.
- Combine small passes when doing so avoids extra render-target switches and memory traffic.
- Avoid unnecessary texture reads, branches, and expensive loops in fragment shaders.
- Prefer simple math and reuse intermediate results instead of recalculating them.
- Use `when` conditions and quality profiles to disable expensive effects on lower device tiers.
- Keep a genuinely playable **Low** or **Potato** profile, and ensure the **High** profile is reasonable for mid-range hardware.
- Adaptive resolution can reduce target scale when frame rate falls below the target, but it is not a substitute for an efficient pack.
- Validate performance on real target devices. A desktop result does not establish mobile performance, and results can vary between drivers and translation layers.

## 8. OpenGL, Vulkan, and translation layers

On the direct OpenGL path, the rendering environment requires the OpenGL capabilities expected by the game. On Android, OpenGL may be provided through a translation layer such as MobileGlues, GL4ES, ANGLE, or Zink. Moracolus detects supported translation-layer identifiers and exposes corresponding `MORA_TRANSLATOR_*` definitions.

On the Blaze3D path, Moracolus uses Minecraft's rendering abstraction and does not implement an independent Vulkan backend. When Minecraft is running with Vulkan, Moracolus prepares the shader and uniforms, then delegates compilation and execution to the game's rendering layer.

Practical guidance:

- Use `MORA_IS_VULKAN` or the `VULKAN` condition only when a real backend-specific difference requires it. Prefer shader code that works across both backends.
- On the Blaze3D path, non-sampler uniforms may be placed in a `std140` block. Do not use `layout(location = ...)` for uniforms.
- The Moracolus GPU benchmark uses OpenGL. When running through Vulkan, GPU benchmark data may be unavailable; classification can rely on CPU, memory, and GPU specifications instead.
- Translation layers may expose lower color-attachment limits. Keep pass output counts within the actual GPU limit.
- Extensions may be missing. Guard optional extension-dependent code with the appropriate preprocessor checks.
- Test with MobileGlues and, when possible, more than one mobile GPU family. Compare with the same pack on desktop to help distinguish shader bugs from driver or translation-layer issues.

## 9. Debugging

### 9.1 The pack does not appear

- Ensure `moracolus.json` or the `shaders/` directory is at the pack root, not nested inside an extra folder in the ZIP.
- Ensure the pack is a directory or a `.zip` file inside `shaderpacks/`.
- Press **R** to reload or use the menu's reload action.

### 9.2 The pack appears but fails

Open **Shader Info**. JSON errors identify the problematic field; compilation errors show the pack, pass, stage (`vertex` or `fragment`), and driver message. Check `logs/latest.log` for the full diagnostic.

### 9.3 A uniform has the wrong value or is always zero

- Verify the exact name in the uniform reference. Unknown names evaluate to zero.
- `live` options are supported for boolean, integer, and float values.
- On some backends, uniform arrays need a constant size.

### 9.4 A texture is missing or black

- An unsupported sampler name causes an error.
- A valid target that has never been written contains its clear color.
- Pack textures must be declared in `textures` and must be valid PNG files.

### 9.5 An effect flickers or appears delayed

- History targets swap only on frames in which they are written. If an effect must update every frame, avoid an `interval` greater than one.
- Check the `phase` value for interval-based passes.

### 9.6 Terrain disappears or uses Sodium's shader

Inspect the terrain status in **Shader Info**. If compilation fails on the current GPU, Moracolus may fall back to Sodium's normal terrain shader. Check the translucent pass and make sure its alpha output is not zero for water, glass, and other translucent blocks.

### 9.7 Useful logs

- Search `latest.log` for `(Moracolus)` to find device detection, preset selection, compilation failures, and disabled optional passes.
- The F3 debug screen can show the backend, pack, active passes, and render scale when **Show in F3** is enabled.

## 10. Testing and distribution

Before publishing, verify the following:

- [ ] `moracolus.json` is at the pack root and the pack opens without errors in **Shader Info**.
- [ ] All passes compile with important option combinations; test every profile.
- [ ] Test the Overworld, Nether, and End, including day/night, rain, and underwater scenes.
- [ ] Test every Moracolus build you intend to support.
- [ ] Where the environment offers both, test OpenGL and Vulkan separately.
- [ ] Test on at least one low-end device and one more capable device.
- [ ] `latest.log` has no recurring Moracolus errors during normal use.
- [ ] Compare FPS with and without the pack in the same scene and conditions.
- [ ] Declare profile levels if you want automatic preset selection to identify them reliably.
- [ ] Include the pack's license.
- [ ] If the pack also targets Iris/OptiFine, test it there independently.

If publishing through Modrinth, choose the appropriate project type and clearly list the Moracolus builds and rendering backends actually tested. Do not claim compatibility with an untested environment.

## 11. Automatic presets

### 11.1 Device tiers

The internal device tier is based on GPU family, platform, available memory, and benchmark data where valid. Software renderers such as llvmpipe or SwiftShader are treated as very low-tier devices. A valid benchmark result is combined with device specifications; thermal stability and memory can limit the resulting tier. GPU benchmark data may be unavailable on backends that do not use the OpenGL benchmark path.

Shaders see a reduced `MORA_TIER` value from 0 to 3. The internal device classification has more detailed levels, but packs should use the documented shader-facing tier rather than assuming a one-to-one mapping to a particular GPU model.

### 11.2 How profiles are recognized

Moracolus reads profiles from `profiles` in `moracolus.json` or supported `profile.*` entries in `shaders.properties`. Profile quality is determined in this order:

1. An explicit `presetLevels` entry in `moracolus.json` (most reliable).
2. Recognized profile names or localized `profile.<NAME>` labels. Common terms include:
   - **Potato:** `potato`, `batata`, `very low`, `ultra low`, `minimum`, `lowest`.
   - **Low:** `low`, `baixo`, `performance`, `lite`, `eco`, `battery`.
   - **Medium:** `medium`, `normal`, `balanced`, `equilibrado`.
   - **High:** `high`, `alto`, `fancy`, `quality`.
   - **Ultra/Extreme:** `ultra`, `very high`, `ultra+`, `extreme`, `maximum`, `cinematic`, `ultimate`.
3. If the name is unknown, the system estimates quality from the options changed by the profile. Enabling costly features or increasing quality-related values can rank a profile higher.

Name matching is heuristic. Use explicit `presetLevels` rather than relying on names when authoring a native pack.

### 11.3 Selection rules

Automatic selection targets an appropriate quality level for the device, but it is intentionally capped at **High**. Even on powerful hardware, the automatic system must not select Ultra, Ultra+, Extreme, or a higher tier. If High is unavailable, it selects the closest available profile below High. If a pack only has profiles above High, automatic selection leaves the pack unchanged and reports that no safe profile was selected. Packs without profiles are not modified.

### 11.4 Manual choices

- Automatic preset selection is applied only when the pack has no saved option choices that indicate prior user configuration.
- If the user changes a profile or option after automatic selection, the pack is treated as manually configured and is not automatically overwritten.
- If the device tier changes after a benchmark and the pack has not been manually changed, the automatic preset can be recalculated when the menu closes.
- Automatic preset state is stored in `config/moracolus/presets.properties`. Removing a pack's state entry and its saved options can return it to automatic selection.
- To disable automatic presets, set `"autoPreset": false` in `config/moracolus/moracolus.json` if that setting is available in the installed build.

### 11.5 Recommendations for shader authors

- Declare `presetLevels` in `moracolus.json` for every profile.
- Use clear profile names such as `POTATO`, `LOW`, `MEDIUM`, `HIGH`, and `ULTRA`, and localize their labels in `.lang` files.
- Make sure the High profile is usable on mid-range devices and provide a Low or Potato option.
- Do not assume that automatic selection can safely edit arbitrary third-party configuration formats. Test how your pack's profiles are represented and preserve user settings.

## 12. Quick reference

Minimal manifest:

```json
{ "format": 1, "passes": [ { "fragment": "f.fsh", "output": "screen" } ] }
```

| Purpose | Identifier |
|---|---|
| Fragment output | `layout(location = 0) out vec4 outColor;` |
| Screen coordinate | `mora_UV` |
| Scene color | `uniform sampler2D tex_scene;` |
| Scene depth | `uniform sampler2D tex_depth;` |
| Noise | `uniform sampler2D tex_noise;` |
| Target X | `uniform sampler2D tex_X;` (`tex_X_prev` with history) |
| Texel size | `uniform vec2 mora_TexelSize;` |
| Device tier | `MORA_TIER` define / `mora_Tier` uniform |
| Mobile condition | `#ifdef MORA_PLATFORM_MOBILE` |
| Reload shaders | Press **R** |
| Log file | `logs/latest.log`, search for `(Moracolus)` |

---

This guide describes documented Moracolus behavior and should be updated when the pack format or rendering interfaces change. Always validate shaders in the actual target environment before publishing compatibility claims.

## Appendix A — Complete manifest reference (expanded)

The following notes preserve the additional field-level details from the original tutorial. They supplement Section 3.3; they do not replace the examples above.

### Top-level fields

All top-level keys are optional except `passes`. `format` is an integer and must be `1`; larger values are rejected. `name`, `version`, `author`, `description`, and `license` are displayed in the Shader Info tab. `sunPathRotation` is an angle in degrees applied to the sun path. `passes` is required and at least one pass must write to `screen`. `targets` defines intermediate textures; `options` defines user options; `screens` organizes option menus; `profiles` defines profiles; `presetLevels` assigns quality levels to profiles; `terrain` configures the terrain stage; `endPortal` configures the End portal shader; `textures` declares pack textures; `blocks` maps block IDs; `blockProperties` points to an OptiFine-format `block.properties`; `vanillaClouds` is a condition that hides vanilla clouds when false; `selectionOutline` configures the RGB selection outline; and `import` reuses options and language data from an OptiFine pack.

### Pass fields and validation

A pass can include `name`, `vertex`, `fragment`, `output` (or `outputs`), `when`, `optional`, `interval`, and `phase`. The default name is generated, such as `passN`. `vertex` is optional; if omitted, a full-screen vertex shader supplying `mora_UV` is used. `fragment` is required. `output` may be a target, a list of targets, or `screen`. `when` defaults to enabled. An `optional` pass is disabled if compilation or execution fails instead of causing the whole pack to fail. `interval` runs the pass once every N frames (1–8), and `phase` chooses its frame within that interval.

Validation rules: `screen` cannot be combined with other outputs; all outputs from a pass must have the same scale; the output count cannot exceed the GPU's color-attachment limit; and a pass cannot read and write the same target during the same frame. For temporal feedback, set `history: true` on the target and read `tex_<target>_prev`.

### Render targets

A target has a `name` (letters, digits, and underscores; reserved names `scene`, `depth`, `noise`, and `screen` are not allowed), `format`, `scale`, `filter`, `mipmap`, `history`, `clear`, and `clearColor`. `scale` is a fraction of screen resolution or the ID of a numeric option, such as `BLOOM_SCALE`. `filter` is `linear` by default or `nearest`. `mipmap` requests mipmaps after writing. `history` preserves the previous frame. `clear` clears the target each frame, but is ignored when history is enabled. `clearColor` defaults to `[0, 0, 0, 0]`.

Supported formats, case-insensitive: `RGBA8` (default), `RGB10_A2`, `RGBA16F`, `R11F_G11F_B10F`, `R8`, `RG8`, `R16F`, `RG16F`, `RGBA32F`, and `R32F`. Equivalent OptiFine names are mapped to the closest supported format (for example, `RGB16` to `RGBA16F` and `RG32F` to `RG16F`). On phones, `RGBA32F` is automatically reduced to `RGBA16F`; on some paths, `R32F` may also be reduced to `R16F`.

### Options

An option ID uses letters, digits, and underscores, preferably `UPPER_SNAKE_CASE`. Types are `bool`, `int`, `float`, or `enum`. Numeric options can use `min`, `max`, and `step`; enums use `values`, and integer/float options may also use a fixed `values` list. `default` is the default value. `mobileDefault` optionally overrides the default on phones. `tierDefaults` optionally supplies defaults for `MORA_TIER` 0–3 and takes precedence over `mobileDefault`. `live: true` turns a supported option into a uniform that can change without recompilation; it does not apply to enums. Decimal values are saved with two decimal places, or more if `step` is smaller than `0.01`.

### Screens and profiles

Example:

```json
"screens": {
  "MAIN": ["<profile>", "<empty>", "[BLOOM]", "SATURATION"],
  "BLOOM": ["BLOOM_ENABLED", "BLOOM_STRENGTH"]
}
```

`MAIN` is the main screen. `[NAME]` opens a sub-screen, `<profile>` displays the profile selector, `<empty>` adds spacing, and `*` lists options not shown on any other screen.

A profile only changes the options it explicitly lists; other options remain unchanged. `presetLevels` is optional and tells Moracolus the quality tier of each profile: `0`/`potato`, `1`/`low`, `2`/`medium`, `3`/`high`, `4`/`ultra`, `5`/`extreme`.

### Terrain, End portal, textures, block IDs, outline, and imports

A terrain declaration can use `vertex`, `fragment`, and `when`; at least one of the two shader files must be supplied. The End portal fragment must define `vec3 moraEndPortal(vec3 rel)`, where `rel` is the vertex's relative position. Define `MORA_END_PORTAL_CUSTOM_FOG` to implement custom fog.

Textures can be declared as simple paths or objects, for example `"clouds": { "path": "textures/clouds.png", "blur": true, "clamp": false }`. PNG files are read as RGBA. In GLSL, use a sampler such as `uniform sampler2D tex_lut;` (or the texture's own name, such as `lut`).

Block IDs can be declared directly, for example:

```json
"blocks": {
  "1": ["minecraft:oak_leaves", "minecraft:birch_leaves"],
  "2": ["minecraft:water"]
}
```

Alternatively, set `blockProperties` to a path such as `shaders/block.properties`. IDs are exposed to terrain as `mora_BlockId` / `s.blockId` (or `mc_Entity.x` in legacy mode); blocks without an ID receive `-1`.

Only `selectionOutline` mode `rainbow` exists. Its `width` must be between 1 and 8. Example: `{ "mode": "rainbow", "when": "RGB_OUTLINE", "speed": 0.25, "scale": 0.20, "width": 2.0 }`.

`import` can reuse an OptiFine pack's options (`#define` and `const` with `[value list]` comments), screens, profiles, and text entries from within the same ZIP. `live` converts numeric options into uniforms; `disabled` hides options (boolean options become false). Includes beginning with `/` that do not exist at the pack root are also searched inside the folder named by `options`.

## Appendix B — Additional GLSL details

### Full-screen and native shader behavior

With the default vertex shader, `in vec2 mora_UV` is the screen coordinate in the 0–1 range and is declared by the mod. With a custom vertex shader, `MORA_VERTEX_SHADER` is defined and `mora_Position` maps to `vec2((gl_VertexID << 1) & 2, gl_VertexID & 2)`. Rendering uses a full-screen triangle with three vertices and no vertex attributes.

Native pass samplers are `tex_scene` (rendered world color), `tex_depth` (scene depth, including translucent geometry), `tex_depth_solid` (solid-block depth), `tex_noise` (256×256 repeating noise), `tex_<target>` (current target contents), `tex_<target>_prev` (previous frame for history targets), and `tex_<texture>` (declared pack textures). Any other name triggers `unknown sampler`.

In version 1.6.0, history targets swap buffers only on frames where a pass actually writes to them. Previously, a pass with an `interval` greater than 1—or a `when` condition that was false for that frame—could cause `tex_<target>` to alternate between a new and stale frame, producing flicker.

### Additional uniform reference

Declare uniforms using their exact names, for example `uniform float mora_Rain;`. Undeclared uniforms have no cost. Common aliases include:

- Resolution: `mora_Resolution`, `mora_TexelSize`, `mora_PassResolution`, `mora_AspectRatio` / `aspectRatio`, `viewWidth`, `viewHeight`, and `mora_RenderScale`.
- Time: `mora_Time` / `frameTimeCounter`, `mora_FrameTime` / `frameTime`, `mora_Frame` / `frameCounter`, `mora_WorldTime` / `worldTime`, `worldDay`, `mora_TimeOfDay`, `mora_MoonPhase` / `moonPhase`.
- Camera and matrices: `mora_Projection` / `gbufferProjection`, `mora_ProjectionInv` / `gbufferProjectionInverse`, `mora_ModelView` / `gbufferModelView`, `mora_ModelViewInv` / `gbufferModelViewInverse`, `mora_PrevProjection` / `gbufferPreviousProjection`, `mora_PrevModelView` / `gbufferPreviousModelView`, `mora_CameraPos` / `cameraPosition`, `mora_PrevCameraPos` / `previousCameraPosition`, `mora_CameraDelta`, `mora_CameraPosWrapped`, `mora_EyePos` / `eyePosition`, `mora_RelativeEyePos`, `eyeAltitude`, `mora_Near` / `near`, and `mora_Far` / `far`.
- Sun, moon, and sky: `mora_SunDir`, `mora_SunDirView`, `mora_MoonDirView`, `mora_LightDirView`, `mora_UpDirView`, `sunPosition`, `moonPosition`, `upPosition`, `shadowLightPosition`, `mora_SunAngle` / `sunAngle`, `shadowAngle`, `mora_DayFactor`, `mora_SkyColor` / `skyColor`, `mora_FogColor` / `fogColor`, `mora_FogStart`, `mora_FogEnd`, `fogStart`, `fogEnd`, and `fogMode`.
- Weather and player state: `mora_Rain` / `rainStrength`, `mora_Thunder` / `thunderStrength`, `mora_Wetness` / `wetness`, `mora_EyeInWater` / `isEyeInWater`, `mora_Blindness` / `blindness`, `mora_NightVision` / `nightVision`, `mora_Darkness` / `darknessFactor`, `mora_Brightness` / `screenBrightness`, `hideGUI`, `mora_EyeBrightness` (0–1), `eyeBrightness` / `eyeBrightnessSmooth` (0–240), `mora_HeldLight`, `heldBlockLightValue`, `heldBlockLightValue2`, and `mora_Dimension` (-1 Nether, 0 Overworld, 1 End, 2 other).
- Device: `mora_Tier` (0–3) and `mora_Mobile` (1 on mobile).
- Other: `atlasSize`, `noiseTextureResolution` (256), and `alphaTestRef` (0.1).
- Live options: a uniform of type `bool`, `int`, or `float` is generated automatically for each supported live option.

On the Blaze3D path, non-sampler uniforms are packed into the `std140` `MoraUniforms` block. Supported types are `float`, `int`, `uint`, `bool`, `vecN`, `ivecN`, `uvecN`, `bvecN`, `mat2`, `mat3`, `mat4`, and arrays of those types.

### Injected defines

Device defines (all stages): `MORACOLUS 100`, `MORA_FORMAT 1`, either `MORA_PLATFORM_MOBILE` or `MORA_PLATFORM_DESKTOP`, `MORA_IS_MOBILE` (0/1), `MORA_TIER` (0–3), `MORA_GPU_<ADRENO|MALI|POWERVR|XCLIPSE|NVIDIA|AMD|INTEL|APPLE|OTHER>`, and `MORA_TRANSLATOR_<MOBILEGLUES|GL4ES|ANGLE|ZINK|VIRGL|KRYPTON|LTW|NATIVE>`. When available, `MORA_BACKEND_<OPENGL|VULKAN|OTHER>` and `MORA_IS_VULKAN` (0/1) are defined. `MORA_HIGHP`, `MORA_MEDIUMP`, and `MORA_LOWP` expand to precision qualifiers on mobile and to nothing on PC.

Dimension defines: `MORA_DIM_OVERWORLD`, `MORA_DIM_NETHER`, `MORA_DIM_END`, `MORA_DIM_OTHER`, and `MORA_DIMENSION` (`0`, `-1`, `1`, or `2`). Stage defines include `MORA_STAGE_POST`, `MORA_PASS_<PASS_NAME>`, `MORA_VERTEX_SHADER` or `MORA_FRAGMENT_SHADER`, and `MORA_STAGE_END_PORTAL` for the portal. Native options are injected as `#define ID 1` or `0` for booleans, `#define ID 3` for integers, `#define ID 0.45` for floats, and `#define ID <index>` plus `#define ID_<VALUE> <index>` for enums. Live options get a `uniform ID;` and `#define ID_LIVE`.

### Frequent errors

- `unknown sampler 'x'`: unsupported sampler name; see the sampler list above.
- `reads and writes target`: the same target is read and written; use history and `tex_X_prev`.
- `No pass writes to 'screen'`: the final pass is missing.
- `has no fragment shader`: a pass has no `fragment` entry.
- `numeric option needs min/max or values`: an integer/float option is incomplete.
- `enum option needs values`: the enum has no list.
- `File not found: x`: incorrect file or `#include` path.

Driver errors such as `undeclared identifier` appear in Shader Info with the pass name and stage (`vertex`/`fragment`). Some backends may warn that a shader program does not use a sampler declared in the pipeline. This can happen when the code using the sampler was removed by a `#if`; it is not an error. To avoid the warning, declare the sampler inside the same `#if` block where it is used.

## Appendix C — Additional native-pack examples and terrain stage

### Targets, bloom, and reduced-rate passes

The development example used a half-resolution `half` target and a half-resolution `accum` target with `history: true`. A `downsample` pass writes to `half` when `GLOW` is enabled; an `accumulate` pass reads `tex_half` and `tex_accum_prev`, writes to `accum`, and runs every two frames; a final pass writes to `screen`. A reusable `lib/common.glsl` file can define a luminance function. Includes beginning with `/` are relative to the pack root; other includes are relative to the current file. A target must have been written before it is read in that frame or a previous frame; on first use, it contains its clear color. `interval: 2` runs every second frame; use `phase` to spread expensive passes across different frames.

### Options, screens, and profiles

Use `tierDefaults` to define defaults per device tier without requiring separate profiles. Use `mobileDefault` for a different default on phones. Declare `presetLevels` so automatic selection does not have to infer profile quality from names.

### Terrain stage interface

The terrain stage replaces Sodium's block shaders in the solid, cutout, and translucent passes. The author writes functions; Moracolus builds the full shader.

Fragment function:

```glsl
vec4 moraTerrainFragment(MoraSurface s)
```

`MoraSurface` contains: `vec4 albedo` (block texture color), `vec4 tint` (vertex color, biome tint, and Sodium AO), `vec3 lightColor` (vanilla lightmap color), `vec2 light` (x = block light, y = sky light, 0–1), `vec3 relPos` (camera-relative position), `float distance` (camera distance), `vec3 normal` (face normal), `float blockId` (ID from `blocks`/`blockProperties`, or -1), `vec2 uv` (atlas coordinates), and `vec2 midUV` (sprite center in the atlas).

The returned color receives Sodium fog automatically. Define `MORA_TERRAIN_CUSTOM_FOG` to provide custom fog; call `vec4 moraApplyFog(vec4 color, float distance)` if you want the original fog at a particular point. Alpha testing (`ALPHA_CUTOUT`) occurs before the function: 0.5 in the cutout pass and 0.01 in the translucent pass, matching Sodium. `MORA_TRANSLUCENT` is defined for the translucent pass (water, glass, Nether portal, and similar geometry).

Vertex function:

```glsl
void moraTerrainVertex(inout vec3 position, inout vec4 color,
                       inout vec2 lightCoord)
```

Available in the vertex stage: `mora_Normal`, `mora_Tangent`, `mora_BlockId`, `mora_TexCoord`, and `mora_MidTexCoord`.

Additional terrain samplers include `tex_<pack texture>`, `noisetex`/`tex_noise`, `lightmap`, `gtexture`/`tex`/`texture` (the atlas), `depthtex0/1/2` or `mora_SolidDepth` (solid depth, valid in the translucent pass), and `normals` (flat normal). Sodium `u_*` uniforms are also available, as are the `mora_*` uniforms listed in Section 4.4.

Example:

```glsl
vec4 moraTerrainFragment(MoraSurface s) {
    vec3 light = s.lightColor;
    float torch = s.light.x * s.light.x;
    light += vec3(1.00, 0.60, 0.30) * torch * 0.20;
    return s.albedo * s.tint * vec4(light, 1.00);
}
```

If terrain compilation fails on a particular GPU, Moracolus falls back to Sodium's shaders and records the failure in Shader Info (`PipelineManager.onTerrainFailure`).

## Appendix D — Complete legacy compatibility and mobile notes

The compatibility pipeline executes `shaders/deferred*.fsh`, `composite*.fsh` (indices 0–99), and `final.fsh` as post-processing, using `colortex0..15` and aliases such as `gcolor`, `gdepth`, `gnormal`, `composite`, and `gaux1..4`. It uses `gbuffers_terrain` (or, in order, `gbuffers_textured_lit`, `gbuffers_textured`, `gbuffers_basic`) for blocks and `gbuffers_water` for the translucent terrain pass through `TerrainTranslator`. It recognizes dimension folders (`world0`, `world-1`, `world1`) and `dimension.properties`; supported `shaders.properties` entries include `screen`, `sliders`, `profile.*`, `program.X.enabled`, `texture.noise`, `customTexture.*`, and `clouds`. It also reads `block.properties` (`mc_Entity`), options defined by `#define` or `const` with a `[value list]` comment, buffer formats (`colortexNFormat`), `Clear`, `ClearColor`, `MipmapEnabled`, `DRAWBUFFERS`, and `RENDERTARGETS`.

Not executed: shadow maps (`shadow*.fsh/vsh` do not run); `shadowtex*`, `shadow`, and `watershadow` samplers receive substitute depth, while `shadowcolor*` receives white; entity, hand, sky, cloud, particle, and block-entity gbuffers; geometry shaders; and compute shaders. Packs relying on shadows or gbuffers beyond terrain must be ported.

Legacy translation includes `texture2D`/`texture3D`/`textureCube` to `texture`, `texture2DLod` to `textureLod`, `texture2DGrad(ARB)` to `textureGrad`, `shadow2D`/`shadow2DLod` to `vec4(texture(...))`, `varying`/`attribute` to `in`/`out`, supported `gl_FragData[n]` outputs, and replacements for fixed-function full-screen constructs such as `ftransform()`, `gl_ModelViewProjectionMatrix`, `gl_MultiTexCoord0`, and `gl_Vertex`. `uniform sampler2D texture;` is renamed to `gtexture`. If the vertex shader is missing, a default full-screen vertex shader providing `texcoord` is used. Injected OptiFine-style defines include `MC_VERSION` (Minecraft-version-dependent), `MC_GL_VERSION 330`, `MC_GLSL_VERSION 330`, `MC_RENDER_QUALITY`, `MC_SHADOW_QUALITY`, `MC_HAND_DEPTH`, `MC_OS_*`, `MC_GL_VENDOR_*`, texture-format names such as `RGBA16F`, and the `MORA_*` definitions in Section 4.5.

### Android performance checklist

- Resolution: mobile GPUs are often limited by memory bandwidth. Use targets at `scale: 0.25` or `0.50` for bloom, blur, and low-frequency effects.
- Formats: `RGBA8` costs about half as much memory as `RGBA16F`; use `RGBA16F`/`R11F_G11F_B10F` only when HDR is needed. Avoid `RGBA32F` (it is already reduced to `RGBA16F` on mobile).
- Passes: each full-screen pass reads and writes the entire screen. Combine simple passes where possible.
- `interval` and `phase`: slow effects (clouds, diffuse reflections, accumulation) may run every 2–4 frames.
- Sampling: use separable blur (horizontal + vertical) instead of a 2D kernel. Linear filtering can sample two texels per fetch.
- Precision: use `MORA_MEDIUMP` for colors and `MORA_HIGHP` for depth and positions. These become precision qualifiers on mobile and disappear on PC.
- Branching: prefer option-based `#if` blocks (no runtime cost) over `if` statements using uniforms inside loops.
- Math: avoid long loops with `pow`, `exp`, or `sin`; lookup textures are often cheaper.
- Depth: `tex_depth_solid` and `tex_depth` require extra copies; declare them only in passes that need them.
- GPU differences: Adreno and Mali are tile-based. Reading a freshly written target in another pass may force the GPU to flush its tile; avoid many tiny chained passes.
- Use `tierDefaults` and conditions such as `TIER >= 2` to disable expensive effects on weaker devices, and declare presets using `presetLevels`.
- Moracolus adaptive resolution reduces target scale when FPS falls below the target, but does not replace an efficient shader pack.

### OpenGL, Vulkan, and translation layers

In the direct OpenGL path, Minecraft requires an OpenGL Core 3.3+ context. On Android, this can be provided through a translation layer such as MobileGlues, GL4ES, ANGLE, or Zink, detected by the mod and exposed as `MORA_TRANSLATOR_*`.

On the Blaze3D path, Moracolus uses Minecraft's rendering abstraction and does not have an independent Vulkan backend. When Minecraft runs on Vulkan, shaders are handed to the game's compiler after Moracolus resolves preprocessing and prepares `MoraUniforms`. Use `MORA_IS_VULKAN` or the `VULKAN` condition only for real behavioral differences; the default code should work on both backends. Non-sampler uniforms go into a `std140` block; do not use `layout(location=...)` on uniforms. The Moracolus GPU benchmark uses OpenGL; on Vulkan, that part is marked unavailable and classification uses CPU, memory, and GPU specifications.

Translation-layer cautions: drivers/translators may expose fewer color attachments, so Moracolus checks the limit and rejects passes with too many outputs; extensions may be missing, so guard them with `#ifdef GL_<extension>`; test MobileGlues and at least one Adreno and one Mali driver; compare with the same pack on PC to distinguish shader issues from driver/translator issues.

## Appendix E — Testing, troubleshooting, and distribution details

### Troubleshooting

**Pack missing from the list:** `moracolus.json` (or the `shaders/` directory) must be at the ZIP root; the file must end in `.zip` or be a directory; reload with **R** or the menu's Reload button.

**Pack listed but failing:** Shader Info reports the JSON key causing the problem, for example `moracolus.json: target name 'scene' is reserved`. Compilation messages identify `<pack>:<pass> [vertex|fragment]`; the driver message may be truncated to 1,200 characters in the UI, while the full log is in `latest.log`.

**Uniform always zero or incorrect:** verify the exact name in the uniform reference; unknown names are zero. `live` options only support bool/int/float. On some backends, uniform arrays require constant sizes.

**Texture missing or black:** unsupported sampler names produce an error; a valid sampler whose target has never been written contains its clear color. Pack textures must be declared under `textures` and be valid PNG files.

**Flickering or delayed effect:** in 1.6.0, history targets swap only when written. If the effect must update every frame, do not set an interval. Check `phase` on interval-based passes.

**Terrain missing or reverting to Sodium:** check the `Terrain:` line in Shader Info. If compilation fails on that GPU, Moracolus automatically falls back to Sodium. Translucent blocks (water, glass, Nether portal) use the translucent pass; check that the returned alpha is not zero.

**Useful logs:** search `latest.log` for `(Moracolus)` to find detected device information, presets, compile failures, and disabled optional passes. The F3 debug screen can show `[Moracolus]` with backend, pack, active passes, and scale when **Show in F3** is enabled.

### Pre-release checklist

- [ ] The ZIP has `moracolus.json` at its root and opens without Shader Info errors.
- [ ] All passes compile for important option combinations; test each profile.
- [ ] Test the Overworld, Nether, and End, day and night, in rain and underwater.
- [ ] Test every Moracolus build you intend to support.
- [ ] Where the game offers both, test OpenGL and Vulkan.
- [ ] Test at least one weak phone (`MORA_TIER` 0 or 1) and one powerful device.
- [ ] No `(Moracolus)` errors appear in `latest.log` during normal use.
- [ ] Measure FPS with and without the pack in the same scene (Hardware tab → Live → FPS).
- [ ] Declare profile `presetLevels` (Section 11).
- [ ] Include a license in the pack.
- [ ] If targeting Iris/OptiFine as well, test on those loaders too.

For Modrinth distribution, the mod has a Modrinth search tab that displays a “Moracolus” badge when a project is marked as compatible. Publish as a shader project and state exactly which Moracolus builds and rendering backends were tested.

## Appendix F — Full automatic preset rules

Device tier ranges from 0 to 5: Potato, Low, Medium, Strong, Ultra, and Ultimate. `device/TierLogic` calculates it using GPU model (Adreno, Mali, PowerVR, Xclipse, NVIDIA, AMD, Intel, Apple), platform (phone or PC), and total RAM. Software renderers such as llvmpipe and SwiftShader are always tier 0.

When a valid benchmark exists for the same GPU/backend, it is combined with the device specifications. The GPU score is a per-frame budget: how many full-screen passes (fill rate) and mathematical operations the GPU can perform per frame at 60 FPS and the current resolution. Thermal stability below 60% reduces the tier by one. RAM caps the maximum tier. Without a measured GPU result (for example, on Vulkan), the CPU result cannot raise the tier above specification + 1. The shader-visible `MORA_TIER` is mapped as follows: device tiers 0–1 → `MORA_TIER 0`; tier 2 → 1; tier 3 → 2; tiers 4–5 → 3.

Profile quality is inferred in this order:

1. `presetLevels` in `moracolus.json`, if declared (most reliable).
2. The profile name or its `profile.<NAME>` text in `.lang`. Recognized labels include:
   - **0 Potato:** `potato`, `batata`, `patata`, `toaster`, `very low`, `muito baixo`, `ultra low`, `minimum`, `lowest`.
   - **1 Low:** `low`, `baixo`, `bajo`, `bas`, `niedrig`, `laag`, `fast`, `performance`, `lite`, `eco`, `battery`.
   - **2 Medium:** `medium`, `medio`, `media`, `moyen`, `mittel`, `gemiddeld`, `normal`, `balanced`, `equilibrado`.
   - **3 High:** `high`, `alto`, `alta`, `haut`, `eleve`, `hoch`, `hoog`, `fancy`, `quality`.
   - **4 Ultra:** `ultra`, `very high`, `muito alto`, `highest`.
   - **5 Extreme:** `ultra+`, `ultra plus`, `extreme`, `extremo`, `max`, `maximum`, `insane`, `cinematic`, `epic`, `ultimate`.
3. If the name is not recognized, estimated profile cost is compared based on its options: enabled values rank above disabled values, higher values in lists/ranges rank above lower ones, and options containing terms such as `QUALITY`, `SHADOW`, `SAMPLES`, `STEPS`, `RES`, `DIST`, `AO`, `BLOOM`, `CLOUD`, or `REFLECT` carry more weight.

Selection target by device tier: tier 0 → Potato; 1 → Low; 2 → Medium; 3 → High (Medium on phones without a benchmark); tiers 4 and 5 → High. Automatic selection never goes above High, even on powerful devices. If Ultra/Ultra+/Extreme exists, High is selected; if High does not exist, the closest lower profile is used. If no profile exists at or below the device target, the lowest available profile up to High is used. If the pack only has profiles above High, nothing is changed and a chat message is shown. Packs without profiles are not changed; their defaults, including `tierDefaults` and `mobileDefault`, remain in effect.

Manual choices are preserved. Automatic preset selection only applies to packs with no saved options. Packs configured before 1.6.0 remain unchanged. If the user changes any option or profile after automatic selection, the pack becomes manual and is never automatically changed again. If device tier changes (for example, after running a benchmark) and the pack has not been manually changed, the preset is adjusted when the menu closes. `config/moracolus/presets.properties` stores each pack's state. Removing that pack's line (or the file) and its saved options under `config/moracolus/packs/` returns the pack to automatic mode. To disable automatic selection entirely, set `"autoPreset": false` in `config/moracolus/moracolus.json`.

Recommendations for authors: declare `presetLevels`; use clear profile names (`POTATO`, `LOW`, `MEDIUM`, `HIGH`, `ULTRA`) and `profile.<NAME>` text in `.lang`; ensure High is playable on mid-range devices and provide Low or Potato.
