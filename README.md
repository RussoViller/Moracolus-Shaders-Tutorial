# Moracolus — Shader Development & Porting Guide

Official community documentation for creating native shaders for Moracolus and adapting existing shader packs to its rendering environment.

Moracolus is a client-side Minecraft Java Edition mod with its own shader-pack system, designed with portability across desktop computers and Android devices in mind. This guide covers the native pack format, OptiFine/Iris compatibility, GLSL, rendering backends, optimization, debugging, and automatic quality presets.

- **Documentation Mod version:** 1.6.5
- **Mod author:** RussoViller
- **Mod license:** [RVCL-1.0](https://github.com/RussoViller/russostyle-saves/blob/main/RVCL-1.0.md)

> This guide is updated for Moracolus 1.6.5 and the builds for Minecraft 1.21.1, 1.21.11, 26.2, and 26.3. The pack format is shared by all four builds, but Minecraft's rendering internals are not: 1.21.1 and 1.21.11 only have OpenGL, while 26.2 and 26.3 can run on OpenGL or Vulkan and compile shaders through Minecraft's own rendering layer. Section 8 explains exactly what changes for shader authors. Treat backend-specific behavior as build-dependent and test the exact target build; dependency version ranges alone do not guarantee support.

### What is new in 1.6.5

- **Lighting library** (`#include "moracolus:lighting.glsl"`): soft sun/moon shading, screen-space cast shadows (up to 48 blocks long), sun/moon light rays, sky reflections and screen-space reflections, all controlled by three global settings in the **Hardware** tab and adapted automatically for phones. See [Section 13](#13-lighting-library-shading-light-rays-and-reflections).
- **New defines and conditions** for those settings (`MORA_SHADOW_QUALITY`, `MORA_LIGHT_RAYS_QUALITY`, `MORA_REFLECTION_QUALITY`, `MORA_SHADOWS`, `MORA_LIGHT_RAYS`, `MORA_REFLECTIONS` and related step counts).
- **Output guard**: NaN values written by post-processing passes are replaced with 0 and infinities are clamped, so one invalid pixel can no longer spread through history targets and turn the screen black (Section 4.7).
- **1.21.11 fix**: depth (`tex_depth`, `depthtex0`) was read as 0 on OpenGL ES/MobileGlues, which removed clouds and broke depth-based lighting (Section 8.4).
- **26.3 fixes for OpenGL on desktop GPUs**: black-output recovery that clears render targets before disabling post-processing, automatic `location` for unmatched fragment inputs, and a startup log line describing the graphics backend (Section 8.5).
- **Moracolus Lite 1.1.0**: the built-in pack gained a **Lighting** screen that uses the lighting library.
- **Quick shader list (L)**: press **L** while playing to open a small panel with only the shader packs, on top of the game, without pausing it. Tap a pack to switch to it or tap the first entry to turn shaders off.

### Supported build matrix

| Minecraft build | Rendering API | How Moracolus builds and runs shaders | Depth seen by Minecraft | Depth seen by shader packs |
|---|---|---|---|---|
| **1.21.1** | OpenGL only | Moracolus compiles and links its own GLSL programs directly with the OpenGL driver and binds textures to texture units itself. | Standard (near = 0, far = 1) | Standard |
| **1.21.11** | OpenGL only | Same direct OpenGL path as 1.21.1, but Minecraft now uses sampler objects and its GPU-device abstraction; Moracolus sets its own sampling state (Section 8.4). | Standard | Standard |
| **26.2** | OpenGL or Vulkan | Shaders are handed to Minecraft's rendering layer (render pipelines). Non-sampler uniforms are packed into a `std140` uniform block. | Reversed (near = 1, far = 0) | Standard: Moracolus converts depth and matrices |
| **26.3** | OpenGL or Vulkan | Minecraft compiles every shader to SPIR-V first. On OpenGL the SPIR-V is translated back to GLSL 330 and compiled by the driver; on Vulkan the SPIR-V is used directly. | Reversed | Standard: Moracolus converts depth and matrices |

The table describes the code paths implemented in the 1.6.5 builds. It is not a claim that every shader pack or every GPU/driver combination has been tested. The guide's examples describe the pack interface, not a promise that every legacy shader feature works on every build.

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
13. [Lighting library: shading, light rays and reflections](#13-lighting-library-shading-light-rays-and-reflections)

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
- **L:** open the quick shader list while playing (only inside a world). The game is not paused; tap a pack to switch, tap outside the panel or press **L**/**Esc** to close.

Useful paths:

- Shader packs: `<game directory>/shaderpacks`
- Mod configuration: `config/moracolus/moracolus.json`
- Saved pack options: `config/moracolus/packs/<pack-id>.properties`
- Automatic preset state: `config/moracolus/presets.properties`
- Benchmark results: `config/moracolus/benchmark.properties`
- Lighting settings: `shadowQuality` (0–5), `lightRaysQuality` (0–3) and `reflectionQuality` (0–4) in `config/moracolus/moracolus.json`, also editable in **Hardware → Lighting, shadows and reflections**

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
| `config/MoraLighting` | Lighting settings, per-platform step counts and lighting defines |
| `glsl/OutputGuard` | NaN/Inf guard added to post-processing outputs |
| `assets/moracolus/glsl/lighting.glsl` | Built-in lighting library (`moracolus:lighting.glsl`) |

### 2.3 GLSL compilation behavior

- The `#version` directive is read by the legacy translator. The resulting version is at least `330`, at most `460`, and uses the `core` profile. A file without a version directive becomes `#version 330 core`; for example, `#version 150` becomes `#version 330 core`.
- `#extension` lines are preserved except for extensions treated as covered by the core profile.
- After `#version`, Moracolus injects device definitions, dimension definitions, option definitions, stage definitions, and then the shader body.
- Native-pack `#include` directives are resolved before compilation. Each file is included at most once per shader, with a maximum include depth of 32.
- `#include "moracolus:<file>"` loads a file shipped inside the mod instead of the pack. In 1.6.5 the available file is `moracolus:lighting.glsl` (Section 13). Only use it in files that Iris/OptiFine never load (for example the files of a native or hybrid pack), because those loaders cannot resolve the `moracolus:` prefix.
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

Available identifiers include `MOBILE`, `DESKTOP`, `TIER`, GPU vendor identifiers such as `ADRENO`, `MALI`, `POWERVR`, `XCLIPSE`, `NVIDIA`, `AMD`, `INTEL`, and `APPLE`; translation-layer identifiers such as `MOBILEGLUES`, `GL4ES`, `ANGLE`, and `ZINK`; dimensions such as `OVERWORLD`, `NETHER`, and `END`; `DIMENSION`, `true`, `false`, and pack option IDs. Where supported, `VULKAN` and `OPENGL` are also available. Since 1.6.5, `MORA_SHADOWS` (0–5), `MORA_LIGHT_RAYS` (0–3) and `MORA_REFLECTIONS` (0–4) return the levels chosen in the Hardware tab, for example `"when": "MORA_LIGHT_RAYS >= 1 && OVERWORLD"`. These three names take precedence over pack options with the same name. Boolean options evaluate to `1` or `0`, enums to their selected index, and numeric options to their numeric value. Unknown identifiers evaluate to `0`.

## 4. GLSL development reference

### 4.1 Version and profile

Write native shaders using GLSL core. Moracolus compiles native shaders as `#version 330 core` or later. Use `in`/`out` instead of `varying`/`attribute`, use `texture()` instead of `texture2D()`, and declare a fragment output, for example:

```glsl
layout(location = 0) out vec4 outColor;
```

Valid GLSL 1.50 syntax remains valid in GLSL 3.30, but the native format does not use a compatibility profile.

### 4.2 Full-screen pass inputs

With the default vertex shader, `in vec2 mora_UV` provides screen coordinates from 0 to 1. With a custom vertex shader, Moracolus defines `MORA_VERTEX_SHADER` and provides `mora_Position`. On 26.2/26.3 it is a macro for a full-screen triangle (three vertices, no vertex attributes, values 0–2); on 1.21.1/1.21.11 it is a vertex attribute of a full-screen quad (values 0–1). `gl_Position = vec4(mora_Position * 2.0 - 1.0, 0.0, 1.0); mora_UV = mora_Position;` is correct on all builds.

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

Lighting defines (all stages, since 1.6.5): `MORA_LIGHTING`, `MORA_SHADOW_QUALITY`, `MORA_SHADOW_STEPS`, `MORA_SHADOW_SOFT`, `MORA_SHADOW_DISTANCE`, `MORA_SHADOW_RANGE`, `MORA_LIGHT_RAYS_QUALITY`, `MORA_LIGHT_RAYS_SAMPLES`, `MORA_REFLECTION_QUALITY`, `MORA_REFLECTION_STEPS`, and `MORA_REFLECTION_REFINE`. Their values are listed in Section 13.3.

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

### 4.7 Output guard (NaN and infinity)

Since 1.6.5, Moracolus wraps the `main()` function of every post-processing fragment shader (native passes and Iris/OptiFine `deferred`/`composite`/`final` programs). After your code runs, each output declared at global scope outside `#if` blocks (`out vec4`, `vec3`, `vec2`, `float`, and fixed-size arrays such as `mora_FragData[3]`) goes through:

```glsl
c = clamp(mix(c, vec4(0.0), isnan(c)), vec4(-3.0e38), vec4(3.0e38));
```

- NaN becomes 0 and ±infinity becomes a large finite value. Normal values, including large HDR values in `RGBA16F`/`RGBA32F` targets, are unchanged.
- Why it matters: desktop GPUs and mobile GPUs with reduced precision produce NaN for operations such as `sqrt` of a negative number, `pow` with a negative base, `normalize(vec3(0.0))`, or `0.0 / 0.0`. Mesa's software renderer often produces different results for the same code, so a pack can look correct in one environment and black in another. A single NaN pixel in a target with `history: true` spreads through blur and temporal passes until the whole screen is black.
- Outputs declared inside `#if`/`#ifdef` blocks are not guarded (the wrapper cannot know whether they exist). Declare outputs unconditionally when possible.
- The guard costs a few ALU instructions per pixel. To disable it in a specific shader, add `#define MORA_NO_OUTPUT_GUARD` anywhere in that shader file.
- The guard hides the symptom; it does not fix the math. If a pass needs it, look for the operation that produces NaN.

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

### 8.1 Which API each build uses

- **1.21.1 and 1.21.11 only have OpenGL.** Minecraft runs on an OpenGL core-profile context. On Android that context usually comes from a translation layer (MobileGlues, GL4ES, ANGLE, Zink, and others) that turns desktop OpenGL into OpenGL ES or Vulkan underneath. Moracolus talks to that OpenGL context directly.
- **26.2 and 26.3 can use OpenGL or Vulkan.** Minecraft chooses the backend, and Moracolus never creates its own Vulkan renderer: it gives its shaders to Minecraft's rendering layer, which compiles and executes them on the active backend. `MORA_BACKEND_OPENGL`, `MORA_BACKEND_VULKAN`, `MORA_IS_VULKAN`, and the `VULKAN`/`OPENGL` conditions tell a pack which one is active.
- On Android with 26.2/26.3, Vulkan runs on the phone's native Vulkan driver (Adreno, Mali, Xclipse, PowerVR), while OpenGL still goes through a translation layer. Both must be tested; they are different drivers.

### 8.2 How a shader reaches the GPU on each build

| Step | 1.21.1 / 1.21.11 (OpenGL) | 26.2 (OpenGL or Vulkan) | 26.3 (OpenGL or Vulkan) |
|---|---|---|---|
| Preprocessing and `#include` | Moracolus | Moracolus | Moracolus |
| Uniforms | Individual `glUniform*` calls per program, looked up by name | Packed into the `std140` block `MoraUniforms` | Packed into the `std140` block `MoraUniforms` |
| Compilation | OpenGL driver compiles the GLSL text | Minecraft's rendering layer compiles the pipeline for the active backend | Minecraft compiles GLSL to SPIR-V for both backends; on OpenGL the SPIR-V is converted back to GLSL 330 and compiled by the driver |
| Varyings (`in`/`out`) | Matched by name | Matched by Minecraft's pipeline | Must have `layout(location = N)`; Moracolus adds matching locations automatically |
| `gl_VertexID` | Native | Native | Rewritten to `gl_VertexIndex` by Moracolus |
| Full-screen geometry | 4-vertex quad; `mora_Position` is a vertex attribute in 0–1 | Triangle; `mora_Position` is a macro over `gl_VertexID` | Same as 26.2 |
| Depth buffer convention | Standard (far = 1) | Reversed (near = 1); converted by Moracolus | Reversed; converted by Moracolus |
| Texture sampling state | Set per texture by Moracolus (1.21.11: also overrides Minecraft's sampler objects) | Minecraft sampler objects chosen by Moracolus | Minecraft sampler objects chosen by Moracolus |
| GPU benchmark | OpenGL | OpenGL only; unavailable on Vulkan | OpenGL only; unavailable on Vulkan |

### 8.3 Shader compatibility differences between 1.21.x and 26.2/26.3+

Code that follows these rules runs unchanged on all four builds:

1. **Write ordinary uniform declarations.** On 26.2/26.3 Moracolus moves every non-sampler uniform into the `MoraUniforms` block for you. Do not put `layout(location = ...)` or `layout(binding = ...)` on uniforms, and do not declare your own uniform blocks with the same names. Numeric initializers such as `uniform float strength = 0.5;` are used as fallback values when Moracolus has no value for that name.
2. **Do not rely on the vertex index pattern.** In a custom full-screen vertex shader, use `gl_Position = vec4(mora_Position * 2.0 - 1.0, 0.0, 1.0); mora_UV = mora_Position;`. This works with the 1.21.x quad and the 26.x triangle.
3. **Match every fragment `in` with a vertex `out` of the same name and type.** 1.21.x links by name; 26.3 assigns locations by name. A fragment input with no vertex output gets a free location on 26.3 since 1.6.5, but its value is undefined everywhere.
4. **Use the depth values Moracolus gives you as standard depth.** `tex_depth`, `tex_depth_solid`, `depthtex0–2`, and `mora_SolidDepth` are always 0 near and 1 far, with sky at exactly 1.0, on every build. `gbufferProjection`/`mora_Projection` and their inverses match that convention. Do not try to detect or undo reversed-Z yourself.
5. **Expect depth precision to depend on the device on 26.x.** The converted depth is written to `R32F`; if the device cannot render to it, Moracolus falls back to `R16F` and then `RGBA16F`/`RGBA8`. Terrain depth is clamped below the sky value (0.99989, 0.9990234 or 0.9960784 depending on the format) so that `depth >= 0.9999` stays a reliable sky test, but distant terrain loses precision on the fallback formats. For distances, prefer `moraViewDistance()`/`moraViewPos()` from the lighting library, which use the real projection matrix, instead of formulas based on the `near`/`far` uniforms (`far` is the render distance, not the projection's far plane).
6. **Keep GLSL strict.** 26.3 compiles through glslang before any driver sees the code, so code that only a lenient driver accepts can fail there. Typical problems are functions that do not return a value on every path, array sizes that are not compile-time constants, and features that need an extension the shader does not enable. Shader Info shows the compiler message.
7. **Declare samplers next to their use.** Some backends warn about samplers that are declared but removed by `#if`. Declare them inside the same `#if` block.
8. **Avoid backend `#ifdef`s unless a reproducible difference requires them.** When one is needed, use `MORA_IS_VULKAN`/`MORA_BACKEND_*` or the `VULKAN`/`OPENGL` conditions rather than the Minecraft version.

### 8.4 OpenGL-only builds (1.21.1 and 1.21.11)

- 1.21.1 uses Minecraft's classic OpenGL path. Textures created by Minecraft carry their own filtering state, so sampling the main depth and color textures behaves as on any OpenGL driver.
- 1.21.11 moved Minecraft to a GPU-device abstraction with OpenGL sampler objects. Moracolus binds its own textures without Minecraft's samplers, so the texture's own state is used. Minecraft no longer sets a filter on the main depth texture, which leaves OpenGL's default (linear, with mipmaps). Desktop drivers accept that, but OpenGL ES (used underneath by MobileGlues and other Android layers) treats a depth texture with linear filtering as incomplete and returns 0 for every read. Before 1.6.5 this made every pixel look like terrain touching the camera on Android: no sky, no clouds over the sky, and wrong depth-based lighting, fog and occlusion. Since 1.6.5 Moracolus sets the depth texture to nearest filtering, clamp-to-edge and no comparison before every post-processing frame.
- On both builds the GPU benchmark, device detection and translation-layer defines (`MORA_TRANSLATOR_*`) use the same OpenGL context the game uses.

### 8.5 Builds with OpenGL and Vulkan (26.2 and 26.3)

- **Uniforms**: packed into a `std140` block. Supported block types are `float`, `int`, `uint`, `bool`, vectors, `mat2`, `mat3`, `mat4`, and arrays of these. Uniform arrays need constant sizes.
- **Depth and matrices**: Minecraft renders with reversed depth. Moracolus converts the depth texture into a standard-depth copy (with a hardware sky mask when available) and gives packs a standard projection matrix. View bobbing is moved into the model-view matrix so that `gbufferProjection` stays a clean perspective projection.
- **26.2**: shaders are compiled for the active backend by Minecraft's pipeline compiler. OpenGL and Vulkan can still differ in driver behavior, precision and supported formats.
- **26.3**: every shader is compiled to SPIR-V. On Vulkan, the driver receives SPIR-V. On OpenGL, Minecraft converts that SPIR-V back to GLSL 330 and the desktop or mobile OpenGL driver compiles the result. Consequences for authors:
  - Errors can come from glslang (before any driver) or from the final OpenGL compile; Shader Info shows both. Names in driver messages may be renamed (for example `_uniform_00_01`).
  - Moracolus adds `layout(location = N)` to varyings, adds `GL_ARB_separate_shader_objects` for GLSL versions below 410, and rewrites `gl_VertexID`/`gl_InstanceID`.
  - `gl_FragCoord.z` in terrain shaders follows the standard depth convention.
- **Desktop GPUs on OpenGL (26.3)**: the main differences from Mesa's software renderer are NaN/precision behavior and stricter drivers. 1.6.5 adds the output guard (Section 4.7), a recovery step when the post-processing output turns completely black (all render targets are cleared once before post-processing is disabled), and a `Graphics: ...` line in `latest.log` with backend, vendor, GPU, driver and depth convention. Include that line in bug reports.
- **GPU benchmark**: uses OpenGL. On Vulkan it is marked unavailable and classification uses CPU, memory and GPU specifications.

### 8.6 Translation layers and mobile drivers

- Translation layers may expose fewer color attachments; Moracolus checks the limit and rejects passes with too many outputs.
- Extensions may be missing; guard optional code with `#ifdef GL_<extension>`.
- Float render formats depend on the device. `RGBA32F` is reduced to `RGBA16F` on phones; on 26.x `R32F` may fall back to `R16F`.
- Test MobileGlues and at least one Adreno and one Mali device, and compare with the same pack on a PC to separate shader bugs from driver or translation-layer issues.

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
| Lighting library | `#include "moracolus:lighting.glsl"` |
| Mod lighting levels | `MORA_SHADOW_QUALITY`, `MORA_LIGHT_RAYS_QUALITY`, `MORA_REFLECTION_QUALITY` |
| Shading factor | `moraShadow(tex_depth, mora_UV, depth)` |
| Exact distance | `moraViewDistance(mora_UV, depth)` |
| Disable output guard | `#define MORA_NO_OUTPUT_GUARD` |

## 13. Lighting library: shading, light rays and reflections

### 13.1 What it is

Moracolus 1.6.5 ships a GLSL library, `moracolus:lighting.glsl`, and three global settings in **Hardware → Lighting, shadows and reflections**. A pack includes the library and calls its functions; the player chooses the quality once, in the mod, for every compatible pack.

| Setting | Levels | Default | Config key |
|---|---|---|---|
| Shading | Off, Super simple, Super light, Light, Medium, Heavy, Ultra realistic | Light (Super light on low-tier phones at first run) | `shadowQuality` 0–6 |
| Light rays | Off, Light, Medium, High | Light | `lightRaysQuality` 0–3 |
| Reflections | Off, Simple, Medium, Advanced, Extreme | Simple | `reflectionQuality` 0–4 |

Changing a setting marks the menu as modified; press **Apply** to recompile the pack with the new defines.

All effects are screen-space: they use the depth buffer of the current frame. Moracolus does not render shadow maps, so anything outside the screen cannot cast a shadow or appear in a reflection. From **Light** upwards, the shading traces rays towards the sun or moon through the depth buffer, so walls, trees, roofs and floating structures that are on screen cast soft shadows; higher levels trace longer rays with more steps. This is what keeps the library cheap enough for phones.

Configs saved by the first 1.6.5 builds (six shading levels) are converted automatically to the new numbering.

### 13.2 Including it

```glsl
#include "moracolus:lighting.glsl"
```

- Include it only in fragment shaders: native post-processing passes and the terrain fragment shader. It uses `gl_FragCoord` and derivatives, which do not exist in vertex shaders. It works on all four builds and both backends.
- The library declares its own uniforms (`moraL_Proj`, `moraL_ProjInv`, `moraL_LightDir`, `moraL_SunDir`, `moraL_UpDir`, `moraL_Day`, `moraL_Rain`, `moraL_Sky`, `moraL_Fog`, `moraL_EyeLight`). Do not declare these names yourself. It does not declare samplers: you pass your own `tex_depth`/`tex_scene` to its functions.
- If the file is compiled without Moracolus's defines, every effect compiles to a no-op.
- Iris and OptiFine cannot resolve `moracolus:` includes. In a hybrid pack, only include the library from the Moracolus files.

### 13.3 Quality levels and the defines they produce

Values in each cell are **PC / phone**. On phones with `MORA_TIER 0` and on Mali or PowerVR GPUs, step and sample counts are reduced by another 25% (minimum 2).

| Shading level | `MORA_SHADOW_QUALITY` | `MORA_SHADOW_DIRECTIONAL` | `MORA_SHADOW_STEPS` | `MORA_SHADOW_SOFT` (rays) | `MORA_SHADOW_DISTANCE` (blocks) | `MORA_SHADOW_RANGE` (blocks) |
|---|---|---|---|---|---|---|
| Off | 0 | 0 | 0 | 0 | 0.00 | 0.00 |
| Super simple (surfaces facing down are darker) | 1 | 0 | 0 | 0 | 0.00 | 0.00 |
| Super light (sun/moon direction) | 2 | 1 | 0 | 0 | 0.00 | 0.00 |
| Light | 3 | 1 | 8 / 6 | 1 / 1 | 6.00 | 32 / 24 |
| Medium | 4 | 1 | 14 / 10 | 1 / 1 | 16.00 | 48 / 32 |
| Heavy | 5 | 1 | 24 / 14 | 2 / 1 | 32.00 | 80 / 48 |
| Ultra realistic | 6 | 1 | 40 / 20 | 3 / 2 | 48.00 | 128 / 64 |

`MORA_SHADOW_DISTANCE` is the length of the shadow ray. Steps are spaced quadratically (dense near the surface, sparse far away), so short contact shadows stay sharp and long cast shadows stay soft.

| Light rays level | `MORA_LIGHT_RAYS_QUALITY` | `MORA_LIGHT_RAYS_SAMPLES` |
|---|---|---|
| Off | 0 | 0 |
| Light | 1 | 8 / 6 |
| Medium | 2 | 16 / 10 |
| High | 3 | 32 / 16 |

| Reflections level | `MORA_REFLECTION_QUALITY` | `MORA_REFLECTION_STEPS` | `MORA_REFLECTION_REFINE` |
|---|---|---|---|
| Off | 0 | 0 | 0 |
| Simple (sky only) | 1 | 0 | 0 |
| Medium | 2 | 12 / 8 | 0 |
| Advanced | 3 | 24 / 12 | 4 / 3 |
| Extreme | 4 | 48 / 20 | 6 / 4 |

`MORA_LIGHTING` is defined as `1` whenever the library is available. All loop limits are compile-time constants, so drivers can unroll them and there is no cost for disabled features.

### 13.4 Functions

| Function | Returns | Notes |
|---|---|---|
| `vec3 moraViewPos(vec2 uv, float depth)` | View-space position | Uses the real projection matrix. |
| `float moraViewDistance(vec2 uv, float depth)` | Distance from the camera in blocks | Exact, unlike `linearizeDepth(depth) * far`. |
| `float moraLinearZ(float depth)` | View-space Z (negative in front of the camera) | Cheap; used inside loops. |
| `vec3 moraViewToScreen(vec3 viewPos)` | Screen UV (xy) and depth (z) | |
| `bool moraIsSky(float depth)` | `depth >= 0.9999` | |
| `vec3 moraViewNormal(sampler2D depthTex, vec2 uv, vec3 viewPos)` | View-space normal reconstructed from depth | Uses derivatives at low quality and four depth reads at Medium shading or Advanced reflections and above. Call it outside non-uniform branches. |
| `float moraSunShading(vec3 normal, vec3 lightDir, float skyLight)` | Soft directional shading factor (1 = lit) | Wrapped lighting: faces turned away from the sun are at most about 45% darker, so both sides of a hill stay readable. Normal and light direction in the same space. `skyLight` 0–1 (for example `s.light.y` in the terrain stage). |
| `float moraHemisphereShading(vec3 normal)` | 0.78–1.00 | Surfaces facing down are darker. Used by Super simple. |
| `float moraContactShadow(sampler2D depthTex, vec2 uv, vec3 viewPos, vec3 normal, float dither)` | Cast-shadow factor (1 = lit) | Screen-space ray towards the sun/moon, `MORA_SHADOW_DISTANCE` blocks long. 1.0 when `MORA_SHADOW_STEPS` is 0. Pass a dither close to 0.5 (for example `0.5 + (moraDither(gl_FragCoord.xy) - 0.5) * 0.35`) for clean shadows without temporal filtering. |
| `float moraShadow(sampler2D depthTex, vec2 uv, float depth)` | Combined shading factor (1 = lit) | Hemisphere shading (Super simple) or directional shading (Super light and up) from reconstructed normals, plus cast shadows from Light upwards, faded underground using the player's sky light. |
| `vec3 moraLightRays(sampler2D depthTex, vec2 uv)` | Additive light-ray color | Radial sky-mask blur towards the sun or moon. Designed for a reduced-resolution pass. |
| `vec3 moraSkyReflection(vec3 dirView)` | Sky color in a view direction | Uses sky color, fog color and a sun/moon highlight. |
| `vec4 moraReflection(sampler2D sceneTex, sampler2D depthTex, vec2 uv, float depth, vec3 normal)` | rgb = reflection, a = confidence of the screen-space hit | Sky only at Simple; screen-space reflections from Medium. |
| `float moraFresnel(float cosTheta, float f0)` | Schlick Fresnel term | |
| `float moraDither(vec2 fragCoord)` | Interleaved gradient noise 0–1 | Static (no frame flicker). |
| `vec3 moraLightColor()` | Current sun/moon color | Warmer at sunrise/sunset, dim blue at night. |

### 13.5 Tutorial: add the three effects to a native pack

**1. Light rays in their own half-resolution pass.** In `moracolus.json`:

```json
"targets": [
  { "name": "rays", "format": "rgba8", "scale": 0.50, "filter": "linear", "clear": true }
],
"passes": [
  { "name": "rays", "fragment": "post/rays.fsh", "output": "rays", "optional": true,
    "when": "MORA_LIGHT_RAYS >= 1 && OVERWORLD" },
  { "name": "final", "fragment": "post/final.fsh", "output": "screen" }
]
```

`post/rays.fsh`:

```glsl
#include "moracolus:lighting.glsl"

uniform sampler2D tex_depth;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(moraLightRays(tex_depth, mora_UV), 1.00);
}
```

**2. Shading and rays in the final pass.** `post/final.fsh`:

```glsl
#include "moracolus:lighting.glsl"

uniform sampler2D tex_scene;
uniform sampler2D tex_depth;
#if MORA_LIGHT_RAYS_QUALITY > 0 && defined(MORA_DIM_OVERWORLD)
uniform sampler2D tex_rays;
#endif

layout(location = 0) out vec4 outColor;

void main() {
    float depth = texture(tex_depth, mora_UV).r;
    vec3 color = texture(tex_scene, mora_UV).rgb;
#if MORA_SHADOW_QUALITY > 0 && defined(MORA_DIM_OVERWORLD)
    color *= mix(1.00, moraShadow(tex_depth, mora_UV, depth), SHADING_STRENGTH);
#endif
#if MORA_LIGHT_RAYS_QUALITY > 0 && defined(MORA_DIM_OVERWORLD)
    color += texture(tex_rays, mora_UV).rgb * RAYS_STRENGTH;
#endif
    outColor = vec4(color, 1.00);
}
```

`SHADING_STRENGTH` and `RAYS_STRENGTH` are ordinary pack options (for example `"live": true` floats), so the player controls the look in the pack and the cost in the mod.

**3. Reflections on surfaces you choose.** The library does not know which pixels are water or metal; your pack does. A common native-pack approach is to treat pixels where translucent depth is in front of solid depth as water or glass:

```glsl
uniform sampler2D tex_depth_solid;

float solid = texture(tex_depth_solid, mora_UV).r;
if (depth < solid - 0.00002 && depth < 0.9999) {
    vec3 p = moraViewPos(mora_UV, depth);
    vec3 n = moraViewNormal(tex_depth, mora_UV, p);
    vec4 r = moraReflection(tex_scene, tex_depth, mora_UV, depth, n);
    float f = moraFresnel(clamp(dot(-normalize(p), n), 0.00, 1.00), 0.02);
    color = mix(color, r.rgb, f);
}
```

**4. Terrain-stage soft shading (cheapest).** In a terrain fragment shader the face normal and sky light are known, so shading costs a few instructions per pixel and no extra texture reads:

```glsl
#include "moracolus:lighting.glsl"

uniform vec3 mora_SunDir;

vec4 moraTerrainFragment(MoraSurface s) {
    vec4 c = s.albedo * s.tint * vec4(s.lightColor, 1.00);
#if MORA_SHADOW_QUALITY > 0 && defined(MORA_DIM_OVERWORLD)
    vec3 light = mora_SunDir.y >= 0.00 ? mora_SunDir : -mora_SunDir;
    c.rgb *= mix(1.00, moraSunShading(s.normal, light, s.light.y), SHADING_STRENGTH);
#endif
    return c;
}
```

### 13.6 Cost and recommendations

- **Super simple** and **Super light** add no texture reads in the terrain-stage approach and only the normal reconstruction in post-processing. They are the safe choice for weak phones.
- **Light** and **Medium** are designed for mid-range phones: 6–10 depth reads per pixel on phones.
- **Heavy** and **Ultra realistic** are meant for PCs and high-end phones. Ultra realistic on a PC traces 3 rays of 40 steps (120 depth reads per pixel); run it at full resolution only on strong GPUs.
- Light rays cost `MORA_LIGHT_RAYS_SAMPLES` depth reads per pixel of the pass that calls them; render them in a target with `scale` 0.25–0.50.
- Reflections cost up to `MORA_REFLECTION_STEPS + MORA_REFLECTION_REFINE` depth reads, but only for the pixels where you call `moraReflection`.
- Adreno and Mali are tile-based GPUs: keep rays in one small pass and do shading inside the final pass instead of adding more full-screen passes.
- Combine the mod level with a pack option for intensity. The mod level decides the cost; the pack option decides how strong the effect looks.

### 13.7 Iris/OptiFine packs

Packs loaded through the Iris/OptiFine compatibility path also receive all `MORA_*` defines, including the lighting defines, and can include `moracolus:lighting.glsl` inside `#ifdef MORACOLUS` blocks. Shadow passes (`shadow.vsh/fsh`) are still not executed; `shadowtex*` samplers keep receiving substitute textures. Use the library's contact shadows as a replacement where a shadow map is not essential.

---

This guide describes documented Moracolus behavior for the 1.6.5 release family and should be updated when the pack format or rendering interfaces change. Always validate shaders in the actual target environment before publishing compatibility claims.

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

With the default vertex shader, `in vec2 mora_UV` is the screen coordinate in the 0–1 range and is declared by the mod. With a custom vertex shader, `MORA_VERTEX_SHADER` is defined. On 26.2/26.3 `mora_Position` maps to `vec2((gl_VertexID << 1) & 2, gl_VertexID & 2)` and rendering uses a full-screen triangle with three vertices and no vertex attributes. On 1.21.1/1.21.11 `mora_Position` is an `in vec2` attribute of a four-vertex full-screen quad.

Native pass samplers are `tex_scene` (rendered world color), `tex_depth` (scene depth, including translucent geometry), `tex_depth_solid` (solid-block depth), `tex_noise` (256×256 repeating noise), `tex_<target>` (current target contents), `tex_<target>_prev` (previous frame for history targets), and `tex_<texture>` (declared pack textures). Any other name triggers `unknown sampler`.

In the 1.6.0+ implementation, history targets swap buffers only on frames where a pass actually writes to them. Previously, a pass with an `interval` greater than 1—or a `when` condition that was false for that frame—could cause `tex_<target>` to alternate between a new and stale frame, producing flicker.

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

**Flickering or delayed effect:** in the 1.6.0+ implementation, history targets swap only when written. If the effect must update every frame, do not set an interval. Check `phase` on interval-based passes.

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


## Appendix G — Version-specific porting notes for 1.21.1, 1.21.11, 26.2, and 26.3

This appendix supplements the shared shader-pack guide. It is especially relevant when a shader such as VanillaEX works on one Moracolus build but fails, renders incorrectly, or exposes broken interface behavior on another.

### G.1 Keep the shader format shared; isolate compatibility differences

- Keep `moracolus.json`, shader passes, options, profiles, and shared GLSL as common as possible.
- Only add backend- or version-specific code when a reproducible difference requires it.
- Use `MORA_IS_VULKAN`, `MORA_BACKEND_VULKAN`, `MORA_BACKEND_OPENGL`, or the supported `VULKAN`/`OPENGL` conditions for backend-specific shader behavior. Confirm that the relevant define exists in the exact build before relying on it.
- Do not use Minecraft version numbers as a substitute for checking actual rendering capabilities.
- When a shader fails on one build, reduce the issue to a minimal pass or option before changing the entire pack.

### G.2 Minecraft 1.21.1 and 1.21.11 — OpenGL builds

The 1.6.5 JARs for Minecraft 1.21.1 and 1.21.11 use the OpenGL rendering path. On 1.21.11, versions before 1.6.5 returned 0 for every depth read on OpenGL ES/MobileGlues (Section 8.4); if a pack showed no clouds and strange depth-based lighting only on 1.21.11 Android, update Moracolus before changing the pack. Their integration code is version-specific, even though they share the same broad Moracolus shader-pack format.

- Test shader compilation, post-processing, terrain rendering, translucent blocks, clouds, and the End portal separately.
- Test the UI independently from the rendering pipeline. A menu issue does not automatically mean the shader is broken.
- Do not copy mixin assumptions or Minecraft class references from 1.21.1 directly into 1.21.11. The supplied builds use different Minecraft/Fabric/Sodium compatibility metadata and different mixin sets.
- On Android, include the actual launcher, GPU, and OpenGL translation layer in bug reports. “Android OpenGL” is not one uniform driver environment.
- If VanillaEX works on one OpenGL device but not another, compare the GLSL compiler log, driver/translation-layer information, target formats, and pass output limits before assuming the shader source is the cause.

### G.3 Minecraft 26.2 — OpenGL and Vulkan

The 26.2 build supports OpenGL and Vulkan through Minecraft's rendering abstraction. See Sections 8.2–8.5 for the uniform-block, depth and compilation differences.

- Keep shader code backend-neutral by default.
- Verify uniform handling on the actual rendering path. On the Blaze3D path, non-sampler uniforms may be packed into the `std140` `MoraUniforms` block.
- Avoid `layout(location = ...)` on uniforms when using that uniform-block path.
- Do not assume a successful OpenGL compile guarantees Vulkan compilation, or vice versa.
- If a pass uses multiple render targets, test the actual color-attachment limit and target formats exposed by the device.
- Benchmark data may differ by backend. The Moracolus GPU benchmark uses OpenGL; Vulkan runs may not provide an equivalent GPU benchmark result.

### G.4 Minecraft 26.3 — rendering and GUI integration

Minecraft 26.3 must be treated as a distinct compatibility target rather than as a drop-in replacement for 26.2. The supplied 26.3 build has a separate mixin set and additional integration points for the game's rendering and GUI APIs. Its UI and rendering paths must therefore be validated independently.

When investigating a 26.3 issue, separate it into one of these categories:

1. **UI/layout issue:** a screen does not open, controls are misplaced, text or icons are missing, scrolling is incorrect, or an interaction does not respond.
2. **Shader compilation issue:** a pass fails to compile, with an error naming the shader, pass, stage, or backend.
3. **Rendering-state issue:** the pack loads but output is black, stale, incorrectly blended, or rendered at the wrong stage.
4. **Terrain integration issue:** blocks, translucent surfaces, or chunk rendering fail or fall back to Sodium's normal shader.
5. **Backend-specific issue:** the same minimal shader behaves differently between OpenGL and Vulkan.

Do not treat these categories as interchangeable. A GUI regression should first be reproduced with shaders disabled or with a minimal pack. A shader regression should be reproduced without unrelated UI customization. If a bug occurs only with a particular combination, record that combination precisely.

**26.3 UI regression checklist**

- Open the main Moracolus menu before and after enabling a shader.
- Visit every tab used in the report, including shader selection, Shader Info, settings, benchmarking, and Modrinth when available.
- Test mouse input and touch input where supported.
- Test scrolling, sliders, toggles, dropdowns, confirmation dialogs, and returning to the previous screen.
- Resize the game window or change GUI scale where those controls are available.
- Test with the default theme and then with custom theme color, opacity, animations, and other UI effects.
- Confirm that UI failures are not caused by an exception in shader loading or a rendering backend.
- Capture `logs/latest.log` immediately after reproducing the problem. Include the first relevant exception and the surrounding stack trace, not only the final error line.

**26.3 shader/backend checklist**

- Start with a minimal full-screen pass sampling only `tex_scene`.
- Test OpenGL and Vulkan separately using the same pack, scene, options, resolution, and device whenever possible.
- Add VanillaEX features back incrementally, beginning with simple post-processing and then more complex passes or terrain behavior.
- Check uniform names and types, sampler names, render-target formats, output count, GLSL version/profile, and backend-specific defines.
- Test the Overworld, Nether, End, water, glass, day/night transitions, weather, and camera movement.
- If the failure is Vulkan-only or OpenGL-only, preserve both logs and report the backend explicitly.
- Do not claim that a shader is fully compatible with 26.3 merely because its menu entry appears or the pack loads. Confirm that the relevant effects render correctly.

### G.5 VanillaEX and other complex ports

For a large existing pack such as VanillaEX, use a staged port rather than making many unrelated edits at once:

1. Keep an untouched copy of the original shader pack.
2. Record the exact Moracolus JAR, Minecraft version, Sodium version, backend, device/GPU, driver or translation layer, and shader options.
3. Confirm that the pack is discovered and identify whether Moracolus treats it as a native, legacy, or hybrid pack.
4. Fix compile errors before investigating visual differences.
5. Validate a simple final pass, then add effects in small groups.
6. Validate terrain and translucent rendering separately from post-processing.
7. Compare OpenGL and Vulkan results only after the same scene and options work on at least one path.
8. Keep each fix tied to a specific reproducible bug. Avoid changing shared shader code to compensate for a suspected mod-side issue until the evidence points to the shader.
9. Retest the fixed shader on the other supported builds, because a fix for one Minecraft rendering path can introduce a regression in another.
10. Document known limitations and the exact environments tested when publishing the port.

### G.6 What to include in a useful bug report

Include:

- Moracolus version and exact Minecraft build (`1.21.1`, `1.21.11`, `26.2`, or `26.3`).
- Sodium and Fabric Loader versions.
- OpenGL or Vulkan, plus the launcher and translation layer on Android if applicable.
- GPU model and device model where relevant.
- Shader-pack name/version and whether the pack is native, legacy, or hybrid.
- Steps to reproduce, expected behavior, and actual behavior.
- Whether the issue persists with shaders disabled or with a minimal shader.
- The relevant portion of `logs/latest.log` and any Shader Info compilation message.
- For UI bugs, the screen/tab, input method, GUI scale, and UI customization settings.
- For backend bugs, logs from each backend tested, when available.

A useful report distinguishes confirmed observations from guesses. “VanillaEX fails on 26.3 OpenGL with this compile error” is more actionable than “26.3 is broken.”

### G.7 Compatibility statement for pack authors

The shared Moracolus pack format is intended to make shader creation and porting easier across supported builds. Minecraft-version compatibility, shader-format compatibility, and backend compatibility are separate things: a pack may be valid in the native format but still depend on a rendering feature unavailable in a particular build or driver.

Always list the exact Minecraft builds and backends you have tested. Treat 1.21.1 and 1.21.11 as OpenGL targets in this release family, and test OpenGL and Vulkan independently on 26.2 and 26.3.
