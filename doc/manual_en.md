## Package Loading

A shader pack can be:

- a directory
- a `.zip`

In either form, the pack root must directly contain:

```text
configs.json
```

Recommended directory layout:

```text
my_pack/
  configs.json
  lang/
    en_us.lang
    zh_cn.lang
  textures/
    ...
  common/
    ...
  ray_tracing/
    ...
  post/
    ...
```

All file paths in `configs.json` must be relative to the pack root. Absolute paths are rejected immediately. If the input is a zip, the loader extracts it first and uses the extracted directory itself as the pack root. The temporary directory is located at `radiance path/temp/shaders/world/ray_tracing/<zip_stem>_<path_hash>`, under the renderer working directory at `temp/shaders/world/ray_tracing/`. A zip pack must not contain an extra wrapper directory. `configs.json` must be at the top level of the archive. During shader compilation, `#include` is resolved relative to the current shader file first, then relative to the pack root.

If the user-specified pack fails to load, the system falls back to the built-in pack. If the pack sets `"requires_emission": true` but the current runtime does not have chunk-emission collection enabled, the system also falls back to the built-in pack.

## Runtime Resources and Bindings

### Custom Runtime Resources

`sampled_binding`, `storage_binding`, and buffer `binding` only determine the binding number of a custom runtime resource inside the custom resource descriptor set.

The current allocation rules are as follows. In the `ray_tracing` stage, custom runtime resources are placed in `set 5`. In the `post_render` stage, they are placed in `set 4`. If the pack declares no `textures` or `buffers` at all, this entire custom-resource set is not created.

Examples: `sampled_binding: 3` means that the texture appears at `binding 3` in the custom-resource set for that stage. `storage_binding: 7` means the texture appears at `binding 7`. `binding: 24` means the buffer appears at `binding 24`.

Example:

```json
{
  "textures": [
    {
      "name": "history",
      "type": "intermediate",
      "format": "rgba16f",
      "width": "RENDER_WIDTH",
      "height": "RENDER_HEIGHT",
      "sampled_binding": 3,
      "storage_binding": 7
    }
  ],
  "buffers": [
    {
      "name": "reservoirs",
      "type": "intermediate",
      "size": "RENDER_WIDTH * RENDER_HEIGHT * 16",
      "binding": 24
    }
  ]
}
```

In the `ray_tracing` stage, `history` in this example occupies `binding 3` and `binding 7` in the custom resource set, and `reservoirs` occupies `binding 24`. The corresponding shader declarations are:

```glsl
layout(set = 5, binding = 3) uniform sampler2D historySampler;
layout(set = 5, binding = 7, rgba16f) uniform image2D historyImage;
layout(set = 5, binding = 24) buffer Reservoirs {
  vec4 data[];
} reservoirs;
```

If the same resources are used in the `post_render` stage, the binding numbers stay the same and only the set number changes to `4`.

### `ray_tracing` Module

The built-in descriptor sets and bindings of the `ray_tracing` module are:

These sets are grouped by resource category. `set 0` stores the game's base textures, `set 1` stores acceleration structures and scene-geometry data, `set 2` stores the world / sky uniform buffers, `set 3` stores the main output images maintained by Radiance, `set 4` stores SHARC resources, and `set 5` stores custom runtime resources declared by the shader pack. If execution variables exist and the pack declares custom runtime resources, they are placed in the following `set 6`.

- `set 0`
  - `binding 0`: the `combined image sampler` array maintained by Radiance. This is where the game's base textures are stored. Capacity: `4096`
- `set 1`
  - `binding 0`: TLAS
  - `binding 1`: `blasOffsetsBuffer`
  - `binding 2`: `indexBufferAddr`
  - `binding 3`: `lastIndexBufferAddr`
  - `binding 4`: `positionBufferAddr`
  - `binding 5`: `materialBufferAddr`
  - `binding 6`: `lastPositionBufferAddr`
  - `binding 7`: `textureMappingBuffer`
  - `binding 8`: `lastObjToWorldMat`
  - `binding 9`: `chunkPackedData`. This is bound only when chunk-emission collection is enabled and the data exists.
- `set 2`
  - `binding 0`: `worldUniformBuffer`
  - `binding 1`: `lastWorldUniformBuffer`
  - `binding 2`: `skyUniformBuffer`
- `set 3`
  - `binding 0`: `out:radiance`
  - `binding 1`: `out:diffuse_albedo_metallic`
  - `binding 2`: `out:specular_albedo`
  - `binding 3`: `out:normal_roughness`
  - `binding 4`: `out:motion_vector`
  - `binding 5`: `out:linear_depth`
  - `binding 6`: `out:specular_hit_depth`
  - `binding 7`: `out:first_hit_depth`
  - `binding 8`: `out:first_hit_diffuse_direct_light`
  - `binding 9`: `out:first_hit_diffuse_indirect_light`
  - `binding 10`: `out:first_hit_specular`
  - `binding 11`: `out:first_hit_clear`
  - `binding 12`: `out:first_hit_base_emission`
  - `binding 13`: `out:fog_image`
  - `binding 14`: `out:first_hit_refraction`
- `set 4`
  - `binding 0`: `SharcConfigBuffer`
  - `binding 1`: `SharcHashEntriesBuffer`
  - `binding 2`: `SharcAccumulationBuffer`
  - `binding 3`: `SharcResolvedBuffer`
  - `binding 4`: `SharcLockBuffer`
- `set 5`
  - custom runtime textures / buffers declared by the shader pack
  - the binding numbers come directly from `sampled_binding`, `storage_binding`, and buffer `binding`

`set 4` is the dedicated SHARC resource set and is not declared in `configs.json`. If a custom `rgen` or `comp` shader needs direct SHARC access, it should declare the fixed bindings in `set 4`.

### `post_render` Module

The built-in descriptor sets and bindings of the `post_render` module are:

The set layout in `post_render` differs from `ray_tracing`. `set 0` stores the game's base textures and the storage image of `out:first_hit_depth`, `set 1` stores the world / sky uniform buffers, `set 2` stores `textureMappingBuffer`, `set 3` stores the output images commonly used in the post stage, and `set 4` stores custom runtime resources declared by the shader pack. If execution variables exist and the pack declares custom runtime resources, they are placed in the following `set 5`.

- `set 0`
  - `binding 0`: the `combined image sampler` array maintained by Radiance. This is where the game's base textures are stored. Capacity: `4096`
  - `binding 2`: the storage image of `out:first_hit_depth`
- `set 1`
  - `binding 0`: `worldUniformBuffer`
  - `binding 1`: `skyUniformBuffer`
- `set 2`
  - `binding 0`: `textureMappingBuffer`
- `set 3`
  - `binding 0`: `out:ldr`
  - `binding 1`: `out:first_hit_depth`
  - `binding 2`: `out:hdr`
  - `binding 3`: `out:motion_vector`
  - `binding 4`: `out:normal_roughness`
- `set 4`
  - custom runtime textures / buffers declared by the shader pack
  - the binding numbers come directly from `sampled_binding`, `storage_binding`, and buffer `binding`

### execution Variables

Execution variables live in a dedicated storage-buffer set and always use `binding 0`.

The current allocation rules are as follows. In the `ray_tracing` stage, if the pack declares any `textures` or `buffers`, execution variables are located at `set 6, binding 0`; if the pack declares no runtime textures / buffers, they are located at `set 5, binding 0`. In the `post_render` stage, if the pack declares any `textures` or `buffers`, execution variables are located at `set 5, binding 0`; if the pack declares no runtime textures / buffers, they are located at `set 4, binding 0`.

Example: if the pack declares `history` and `reservoirs` as shown above, the execution-variable buffer should be declared at `set 6, binding 0` in the `ray_tracing` stage and at `set 5, binding 0` in the `post_render` stage. If the pack declares no custom runtime textures or buffers, both set numbers decrease by `1`. Therefore, the presence of custom runtime resources directly affects the set number used by execution variables in shaders.

## Root Configuration Structure

The most common root structure is:

```json
{
  "radiance": {
    "shader_pack": true,
    "display_name": "Example Shader Pack"
  },
  "requires_emission": false,
  "sharc": {
    "update_pass": "world",
    "resolve_comp": "sharc_resolve.comp"
  },
  "attributes": [],
  "textures": [],
  "buffers": [],
  "execution": {
    "global_variables": [],
    "commands": []
  },
  "execution_post": {
    "global_variables": [],
    "commands": []
  },
  "passes": []
}
```

Fields:

- `radiance`: the shader pack manifest. It determines whether this directory or zip should be treated as a shader pack, and what name should be shown in the list. It does not participate in the runtime resource parsing described later.
- `requires_emission`: whether this pack requires runtime chunk-emission data. Default: `false`
- `sharc`: optional. Declares one shared SHARC update / resolve configuration for the entire shader pack. It only applies to the `ray_tracing` stage.
- `attributes`: declares user-adjustable shader parameters
- `textures`: declares runtime texture resources
- `buffers`: declares runtime storage buffer resources
- `execution`: describes pass execution for the `ray_tracing` stage
- `execution_post`: describes pass execution for the `post_render` stage
- `passes`: declares the passes in this pack. It must be a non-empty array.

The loader only parses fields it recognizes. Manifest metadata such as `radiance` does not participate in the runtime resource and execution-flow configuration described later.

## `manifest`

`configs.json` should normally include a `radiance` manifest. For a directory or zip that should be recognized as a shader pack, this field is required. Example:

```json
{
  "radiance": {
    "shader_pack": true,
    "display_name": "§8§kXX §r§b§lAdvanced§r §8§kXX§r"
  }
}
```

It marks the shader pack and provides the display name. `radiance` is the manifest root object, and it may also be written as a boolean directly. `radiance.shader_pack` controls whether the pack is recognized as a shader pack. `radiance.display_name` controls the display name shown in the shader pack list.

If `radiance` is a boolean, it only controls shader-pack recognition and cannot provide a display name. If `radiance` is an object but does not contain `shader_pack`, it is treated as `true`. If the root object has no `radiance` at all, the directory or zip is not recognized as a shader pack entry. If no display name is available, the list falls back to the zip or directory name without the extension. `radiance` is required for shader-pack recognition. `display_name` controls the list label. Passes, textures, execution, and all other runtime behavior are configured by the runtime fields described later.

## `sharc`

`sharc` declares one shared [SHARC](https://github.com/NVIDIA-RTX/SHARC) update / resolve configuration for the entire shader pack. It only applies to the `ray_tracing` stage. Example:

```json
{
  "sharc": {
    "update_pass": "final_compose",
    "resolve_comp": "sharc_resolve.comp"
  }
}
```

Fields:

- `update_pass`: the name of an existing `ray_tracing` pass. The runtime uses that pass's `rgen`, `hit_groups`, `miss`, `inputs`, `outputs`, and dispatch-size expressions to execute the global SHARC update.
- `resolve_comp`: the compute shader path used for SHARC resolve. It must be a path relative to the pack root.

If the root object contains `sharc` and the runtime SHARC switch is enabled, the runtime executes one global SHARC update and one resolve at the start of the entire `ray_tracing` stage. When the commands in `execution` are later executed, only `ray_tracing` passes with `query_sharc: true` can use the SHARC query path.

The `rgen` referenced by `update_pass` is compiled once more as a SHARC update path with `USE_SHARC=1` and `SHARC_UPDATE=1`. An `rgen` with `query_sharc: true` is compiled with `USE_SHARC=1` and `SHARC_QUERY=1`. `resolve_comp` runs after the update and is responsible for resolving SHARC cache entries.

The ray tracing dispatch count for the global SHARC update is already downsampled. It is not a full-resolution dispatch. If the original trace size computed by `update_pass` is `width` and `height`, the actual update dispatch size issued by the runtime is:

```text
update_width  = ceil(width  / sharc_update_downsample_factor)
update_height = ceil(height / sharc_update_downsample_factor)
update_depth  = 1
```

In the shader, `gl_LaunchIDEXT` corresponds to a downsampled block coordinate. The shader then selects one full-resolution pixel inside that block. This is not a full-resolution dispatch with reduced work performed inside the shader.

## `attributes`

`attributes` declares adjustable parameters and how those parameters are mapped to shader macros.

Each attribute is an object:	

```json
{
  "name": "translation.key.or.id",
  "type": "float_range:0.0-1.0",
  "default_value": "0.5",
  "define": "RT_EXAMPLE"
}
```

Fields:

- `name`: attribute name. In most packs this is written as a translation key.
- `type`: attribute type
- `default_value`: default value. It must be a string here.
- `define`: optional. Describes how the current attribute value is converted into shader macros.

### Supported Attribute Types

`attributes` supports the following types:

- `bool`
- `int`
- `float`
- `string`
- `vec3`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

In `attributes`, `default_value` must always be a string. `vec3` is written as a string in `x,y,z` form. Each option after `enum:` is a fixed string, and runtime matching is exact string matching.

### The Actual `define` Syntax

`define` supports three structures. The classification below follows the actual JSON shapes accepted by the loader.

#### Form 1: String

```json
{
  "name": "my_attr",
  "type": "float",
  "default_value": "1.0",
  "define": "RT_MY_ATTR"
}
```

The current attribute value is mapped directly to one macro name. The equivalent GLSL form is:

```glsl
#define RT_MY_ATTR 1.0
```

#### Form 2: Object With String Values

```json
{
  "name": "my_attr",
  "type": "float",
  "default_value": "2.0",
  "define": {
    "RT_MY_ATTR": "((X) * 3.0 + 1.0)",
    "RT_MY_ATTR_INT": "int(X)"
  }
}
```

`define` is a map from macro name to expression string. The string on the right-hand side is an expression template used by the loader when replacing `X`. `X` is replaced with the current attribute value.

#### Form 3: Object With Object Values

```json
{
  "name": "my_mode",
  "type": "enum:mode.a-mode.b",
  "default_value": "mode.a",
  "define": {
    "mode.a": {
      "RT_MODE": "0",
      "RT_MODE_IS_A": "1"
    },
    "mode.b": {
      "RT_MODE": "1",
      "RT_MODE_IS_B": "1"
    }
  }
}
```

The outer keys are the original string values of the attribute. The inner objects are maps from macro name to expression string. When the current value matches an outer key, that group of definitions is applied.

Inner values may also be expressions. Example:

```json
{
  "define": {
    "mode.a": {
      "RT_MODE_SCALE": "((X) + 1)"
    }
  }
}
```

This form is valid. The loader handles the inner values in the same way as form 2 and still performs `X` replacement.

Two layers of meaning must be distinguished. Matching of the outer keys uses the attribute's original string value. `X` inside the inner expressions uses the converted value (`bool` becomes `1` or `0`; `enum` becomes the enum index `0/1/2/...`).

However, the following mixed form is invalid:

```json
{
  "define": {
    "RT_A": "1",
    "mode.a": {
      "RT_B": "2"
    }
  }
}
```

The reason is that the loader parses a `define` object as exactly one of these two categories: "all values are strings" or "all values are objects". The two categories cannot be mixed.

### How Attribute Values Are Passed To `define`

Before `define` replacement, the current attribute value is converted into the intermediate representation used by the loader. `bool` becomes `1` or `0`. `enum` becomes the enum index `0/1/2/...`. Other types use the original string directly.

Common true values for `bool` are typically written as `render_pipeline.true`, `true`, or `1`. If the current value of an `enum` does not match the strings listed after `enum:`, the expected enum index is not produced.

## `textures`

`textures` declares image resources used by the shader pack at runtime.

The categories are:

- `file`: imports texture data from a file inside the pack
- `intermediate`: creates an image resource in GPU memory at runtime for pass read/write access

A texture object typically looks like this:

```json
{
  "name": "trans_lut",
  "type": "intermediate",
  "format": "R16G16B16A16_SFLOAT",
  "width": 256,
  "height": 64,
  "sampled_binding": 0,
  "storage_binding": 1
}
```

### binding

`sampled_binding`, `storage_binding`, and buffer `binding` all describe which `binding` slot a runtime resource occupies in a descriptor set, that is, which binding number the shader uses on the GPU side to access that resource. They do not describe pass order, barrier placement, or which pass is responsible for reading or writing the resource. Those are described by `execution` / `execution_post`, `inputs`, and `outputs`.

Binding object: `sampled_binding` exposes the texture to shaders as a sampled image, typically accessed through declarations such as `sampler2D`, `samplerCube`, `sampler2DArray`, or `sampler3D`. `storage_binding` exposes the texture as a storage image, typically accessed with `imageLoad` and `imageStore`.

Binding forms: if only `sampled_binding` is declared, the texture is bound only as a sampled image. If only `storage_binding` is declared, it is bound only as a storage image. If both are declared, the texture is exposed in both forms.

Constraint: `configs.json` only provides binding numbers, not descriptor set numbers. All runtime textures and buffers share one global binding-number space and cannot reuse the same binding numbers.

### Common Fields

- `name`: resource name
- `type`: `file` or `intermediate`
- `dimension`: `2d`, `2d_array`, `3d`, or `cube`. Default: `2d`
- `sampled_binding`: optional
- `storage_binding`: optional
- `filter`: `linear` or `nearest`. Default: `linear`
- `address_mode`: `repeat`, `clamp_to_edge`, or `mirrored_repeat`. Default: `repeat`
- `shared`: whether one resource is shared across all frames. Default: `false`

### `file` Textures

Example:

```json
{
  "name": "noise",
  "type": "file",
  "path": "textures/noise.png",
  "sampled_binding": 3
}
```

`path` is required and must be relative. `sampled_binding` is required. `format` is optional and defaults to `R8G8B8A8_UNORM`. Currently supported imported dimensions are `2d`, `2d_array`, and `3d`. `cube` cannot be used as an imported texture.

Runtime behavior: imported `2d` textures are loaded through the image loader, and currently require `format` to be `R8G8B8A8_UNORM`. Imported `2d_array` and `3d` textures are read directly as raw binary blobs.

Common file forms:

- `2d`: typically `.png` or other ordinary image files
- `2d_array` / `3d`: typically `.raw` or other raw binary formats

For imported `2d_array` / `3d`, you must also provide `width`, `height`, and `depth`.

The file size must be exactly:

```text
width * height * depth * bytes_per_pixel
```

### `intermediate` Textures

Example:

```json
{
  "name": "sky_cube",
  "type": "intermediate",
  "dimension": "cube",
  "format": "R16G16B16A16_SFLOAT",
  "width": 128,
  "height": 128,
  "sampled_binding": 2
}
```

`format` is required. `width` / `height` / `depth` may be numbers or string expressions. `scale` can only be a number.

Size forms: for `2d` / `cube`, use either `width` + `height` or `scale`. For `2d_array` / `3d`, `width`, `height`, and `depth` must all be provided explicitly.

### `scale`

`scale` means that the width and height of the `intermediate` texture are scaled from the current reference render size.

The calculation is:

```text
width  = max(1, int(referenceWidth  * scale))
height = max(1, int(referenceHeight * scale))
```

Here `referenceWidth` / `referenceHeight` are the reference output dimensions used by the current render stage, at the same level as `RENDER_WIDTH` / `RENDER_HEIGHT`.

Example: `scale: 1` means the same width and height as the reference size, `scale: 0.5` means half resolution, and `scale: 2` means double resolution.

`scale` only affects width and height. It does not affect the layer count or depth.

### Supported `format` Strings

The loader currently supports these format strings:

- `R8_UNORM`
- `R8G8_UNORM`
- `R8G8B8A8_UNORM`
- `R16_SFLOAT`
- `R16G16_SFLOAT`
- `R16G16B16A16_SFLOAT`
- `R32_SFLOAT`
- `R32G32_SFLOAT`
- `R32G32_UINT`
- `R32G32B32A32_SFLOAT`
- `R32G32B32A32_UINT`

## `buffers`

`buffers` declares runtime storage buffer resources.

Example:

```json
{
  "name": "reservoir",
  "type": "intermediate",
  "size": "RENDER_WIDTH * RENDER_HEIGHT * 16",
  "binding": 24
}
```

Fields:

- `name`: resource name
- `type`: currently only `intermediate`
- `size`: required. It may be a number or a string expression.
- `binding`: required. The binding slot used by this buffer in the shader runtime-resource descriptor set.
- `shared`: optional. Default: `false`

`binding` shares one global number space with texture `sampled_binding` / `storage_binding`. The `size` expression is also evaluated at runtime.

## `passes`

`passes` declares the GPU execution steps in the shader pack, for example:

- one full-screen draw
- one ray tracing dispatch
- one compute dispatch
- one post-render draw

`passes` only declares which steps exist and what each step looks like.

Execution order and execution conditions are defined later by `execution` / `execution_post`.

### Resource Lists

The current syntax of `inputs` / `outputs` is object-based:

```json
{
  "inputs": {
    "images": ["trans_lut", "sky_cube"],
    "buffers": ["reservoir"]
  },
  "outputs": {
    "images": ["out:radiance"],
    "buffers": []
  }
}
```

`inputs` / `outputs` are optional, and `images` / `buffers` are also optional. If omitted, they mean empty lists. Only `images` and `buffers` are allowed inside these objects.

The meaning of `inputs` / `outputs` is that they tell the runtime which resources the pass reads and writes. The runtime uses this to generate resource barriers and layout transitions.

Their difference from binding is: binding determines which binding number the shader uses to access a resource, while `inputs` / `outputs` determine which resources the pass declares as read or written.

### Common Rules

Each pass must be an object and must contain `type`.

Common fields:

- `type`: pass type
- `name`: pass name
- `stage`: optional. Supports `ray_tracing` and `post_render`. Default: `ray_tracing`
- `inputs`: optional. Declares which images / buffers the pass reads.
- `outputs`: optional. Declares which images / buffers the pass writes.
- `define`: optional. Declares extra macros injected into this pass.

`ray_tracing` passes may only appear in the `ray_tracing` stage. `render` passes may currently only appear in the `post_render` stage, for weather, particles, and similar content. Chunk data is planned to be added later, at which point rasterized world rendering will also be possible. All values in `define` must be strings.

### `full_screen` Pass

Example:

```json
{
  "type": "full_screen",
  "name": "sky_cube",
  "fragment": "atmosphere/sky_cube.frag",
  "inputs": {
    "images": ["trans_lut"]
  },
  "target": "sky_cube"
}
```

Fields:

- `name`: pass name
- `fragment`: fragment shader path
- `target`: output image name of the full-screen draw
- `inputs`: optional. Declares read resources.
- `outputs`: optional. Declares written resources.
- `face`: optional. Integer from 0 to 5, used for a specific face of a cube target.
- `layer`: optional. Unsigned integer, used for a specific layer of an array target.

If `outputs.images` does not contain the texture referenced by `target`, the loader adds it automatically. If `outputs` is omitted entirely, the loader also treats `target` as the only image output. `face` and `layer` cannot be used together. `face` only applies to cube targets. `layer` only applies to array targets.

In the `ray_tracing` stage, there is a special behavior: if `target` is a cube and neither `face` nor `layer` is provided, the runtime executes the pass implicitly six times and injects the following into the shader:

```glsl
FACE=0
FACE=1
FACE=2
FACE=3
FACE=4
FACE=5
```

The `post_render` stage is stricter: a full-screen pass may only write to its own `target`, and if the target is an array image, `layer` must be provided explicitly.

### `ray_tracing` Pass

Example:

```json
{
  "type": "ray_tracing",
  "name": "world",
  "inputs": {
    "images": ["trans_lut", "sky_cube"]
  },
  "outputs": {
    "images": ["out:radiance"]
  },
  "rgen": "ray_tracing/world.rgen",
  "default_hit_group": "default",
  "hit_groups": {
    "default": {
      "type": "triangle",
      "shaders": {
        "rchit": "ray_tracing/default.rchit",
        "rahit": "ray_tracing/default.rahit"
      }
    }
  },
  "miss": [
    {
      "name": "world",
      "shader": "ray_tracing/world.rmiss"
    }
  ]
}
```

Fields:

- `name`: pass name
- `inputs`: optional. Declares read resources.
- `outputs`: optional. Declares written resources.
- `rgen`: ray generation shader path
- `default_hit_group`: the name of the default hit group
- `hit_groups`: hit-shader group configuration for each geometry group name
- `miss`: miss shader list
- `width`: optional. Default: `RENDER_WIDTH`
- `height`: optional. Default: `RENDER_HEIGHT`
- `depth`: optional. Default: `1`
- `query_sharc`: optional. Default: `false`

#### `query_sharc`

`query_sharc` declares whether this `ray_tracing` pass queries the SHARC cache.

It only controls this pass's own query path. It does not determine which shader is used for update or resolve. Update and resolve are configured centrally by the root-level `sharc` field.

If the root object contains `sharc` and the runtime SHARC switch is enabled, the runtime performs one global SHARC update and one resolve at the start of the `ray_tracing` stage. After that, each `ray_tracing` pass with `query_sharc: true` inserts a SHARC-related barrier before its own query dispatch and compiles its `rgen` with `USE_SHARC=1` and `SHARC_QUERY=1`.

`query_sharc: false` passes execute as ordinary `ray_tracing` passes. They do not receive SHARC query macros and do not insert extra SHARC barriers.

If a pass is referenced by `sharc.update_pass` and also sets `query_sharc: true`, it participates in two roles. First, its configuration is used for the global SHARC update. Second, when `execution` actually executes that pass, it still performs a normal query dispatch using its own `width` / `height` / `depth`.

The global SHARC update uses downsampled dispatch dimensions. Query dispatch uses the full trace dimensions of each pass. The update dispatch width and height are computed as:

```text
update_width  = ceil(width  / sharc_update_downsample_factor)
update_height = ceil(height / sharc_update_downsample_factor)
```

Here `width` and `height` are the trace dimensions computed by `update_pass` itself, and `sharc_update_downsample_factor` is the runtime SHARC downsample factor. Query dispatch continues to use each pass's own full `width` / `height` / `depth`.

### `hit_groups`

`hit_groups` solves this problem: when a ray hits a piece of geometry in the TLAS/BLAS, which hit-shader group should be used.

The group name here corresponds to the runtime geometry-group name. In the current implementation, the relationship is as follows: the `geometryGroupName` passed from Java to native for each piece of geometry currently comes directly from `RenderLayer.name`.

For chunk / entity geometry: if a piece of geometry belongs to some `RenderLayer` on the Java side, native receives that `RenderLayer.name`, then looks up the same string in `hit_groups`. The match is exact string matching.

The runtime also injects one reserved group name: `shadow`.

This is an extra synthetic group inserted during TLAS construction for shadow-related hit-group selection.

If the pack does not explicitly provide a hit group named `shadow`, the runtime currently falls back to `default_hit_group` for shadow geometry.

### Parsing Rules of `hit_groups`

The parsing flow is:

1. `hit_groups` must be a non-empty object.
2. The name referenced by `default_hit_group` must actually exist in `hit_groups`.
3. Each group's `type` must be either `triangle` or `aabb`.
4. `shaders` may explicitly contain `rchit`, `rahit`, and `rint`.
5. If a non-default group still has neither `rchit` nor `rint` after parsing, it falls back to the default group's `rchit`.
6. If a group ends up with `rint`, the runtime treats it as a procedural hit group.
7. If `type` is `aabb` but the group does not end up with `rint`, this is a hard error.
8. Each group must end up with at least `rchit` or `rint`.

Other details: the default group is placed first, and the remaining groups are sorted by name. `miss` must be a non-empty array. If the runtime cannot find a hit group for a geometry group name, it falls back to `default_hit_group`.

### `compute` Pass

Example:

```json
{
  "type": "compute",
  "name": "blur",
  "inputs": {
    "images": ["ping"]
  },
  "outputs": {
    "images": ["pong"]
  },
  "compute": "post/blur.comp",
  "gx": "(RENDER_WIDTH + 7) / 8",
  "gy": "(RENDER_HEIGHT + 7) / 8",
  "gz": 1
}
```

Fields:

- `name`: pass name
- `compute`: compute shader path
- `inputs`: optional. Declares read resources.
- `outputs`: optional. Declares written resources.
- `gx`: workgroup count in the x direction
- `gy`: workgroup count in the y direction
- `gz`: workgroup count in the z direction

This pass performs one dispatch with one compute shader.

The `compute` shader path is required. `gx` / `gy` / `gz` are optional and all default to `1`. The current runtime only supports compute passes in the `ray_tracing` stage. If a compute pass is placed in the `post_render` stage, initialization later fails.

### `render` Pass

`render` passes are only used in the `post_render` stage.

Their purpose is to take one category of geometry from the post-render queue and draw it through a graphics pipeline with vertex / fragment shaders.

There are two shader-configuration forms.

#### Form 1: One Shader Pair

```json
{
  "type": "render",
  "stage": "post_render",
  "name": "post_render_particle",
  "content": "particle",
  "vertex": "post/particle.vert",
  "fragment": "post/particle.frag"
}
```

#### Form 2: Multi-Variant Shader

```json
{
  "type": "render",
  "stage": "post_render",
  "name": "post_render_weather",
  "content": "weather",
  "shader": {
    "default": {
      "vertex": "post/weather.vert",
      "fragment": "post/weather_default.frag"
    },
    "/weather/rain": {
      "vertex": "post/weather.vert",
      "fragment": "post/weather_rain.frag"
    },
    "/weather/snow": {
      "vertex": "post/weather.vert",
      "fragment": "post/weather_snow.frag"
    }
  }
}
```

Fields:

- `name`: pass name
- `content`: which category of post-render geometry this pass consumes
- `vertex` / `fragment`: the single-shader-pair form. Mutually exclusive with `shader`.
- `shader`: the multi-variant form. Mutually exclusive with `vertex` / `fragment`.
- `inputs`: optional. Declares read resources.
- `outputs`: optional. Declares written resources.
- `depth_test`: optional. Whether depth testing is enabled.
- `depth_write`: optional. Whether depth is written.
- `depth_compare`: optional. Depth comparison function.
- `color_blend`: optional. Color blend configuration.

Rules for depth and blend related fields:

- `depth_test`: boolean, `true` or `false`. Default: `true`
- `depth_write`: boolean, `true` or `false`. The default depends on `content`. The default is `true` for `text`, `name_tag`, and `star`, and `false` for other `content`.
- `depth_compare`: string. Supported values are `never`, `less`, `equal`, `less_or_equal`, `greater`, `not_equal`, `greater_or_equal`, and `always`. The default is usually `less`. For `name_tag`, the default is `always`.
- `color_blend`: boolean, `true` or `false`. Default: `true`. `true` enables standard alpha blending with `src_alpha` and `one_minus_src_alpha`. `false` disables color blending and makes the color output overwrite the target directly.

The role of `content` is to select which category of post-render geometry queue this pass consumes.

Currently supported:

- `weather`
- `particle`
- `text`
- `name_tag`
- `star`

### Variant Matching Rules For `render` Pass

The keys in the `shader` object correspond to the `contentName` carried by each piece of geometry in the post-render queue.

The runtime chooses a variant as follows:

1. It obtains the `contentName` of the current geometry.
2. It looks for the same key in the `shader` map.
3. If no exact match is found, it falls back to `default`.
4. If `default` is also missing, that piece of geometry matches no variant.

The matching target is `contentName`. `content` selects the post-render geometry queue. `RenderLayer.name` does not participate in this variant selection.

`contentName` is produced as follows:

- `weather`
  - `/weather/rain`
  - `/weather/snow`
  - default form: `/weather/default`
- `particle`
  - `contentName` is generated from the particle type's `Identifier`
  - if the particle type's `namespace` is `minecraft`: `/particle/<path>`
  - if the particle type's `namespace` is not `minecraft`: `/particle/<namespace>/<path>`
  - default form: `/particle/default`
- `text`
  - `/text/default`
- `name_tag`
  - `/name_tag/default`
- `star`
  - uses the pass's default pipeline and does not participate in this variant-selection path

Applicability: `weather` is well-suited for multi-variant shaders. `particle` can also use multi-variant shaders, but only if the required particle `contentName` values are known in advance. `text` and `name_tag` currently usually only hit their respective `default` variants. `star` effectively only uses the default pipeline.

### Additional Runtime Restrictions In `post_render`

The `post_render` runtime currently requires that a `render` pass must output `out:ldr`, and may additionally output at most one `out:first_hit_depth`.

### Common Built-In Image Names

#### Common Targets In The `ray_tracing` Stage

- `out:radiance`
- `out:diffuse_albedo_metallic`
- `out:specular_albedo`
- `out:normal_roughness`
- `out:motion_vector`
- `out:linear_depth`
- `out:specular_hit_depth`
- `out:first_hit_depth`
- `out:first_hit_diffuse_direct_light`
- `out:first_hit_diffuse_indirect_light`
- `out:first_hit_specular`
- `out:first_hit_clear`
- `out:first_hit_base_emission`
- `out:fog_image`
- `out:first_hit_refraction`

#### Common Built-In Image Names In The `post_render` Stage

- `out:ldr`
- `out:first_hit_depth`
- `out:hdr`
- `out:motion_vector`
- `out:normal_roughness`

## execution and execution_post

`execution` and `execution_post` describe stage execution flow and belong to configuration. `execution` controls the `ray_tracing` stage. `execution_post` controls the `post_render` stage. They determine the pass execution order inside a stage and allow simple variables and conditions to control which passes execute.

Basic structure:

```json
{
  "execution": {
    "global_variables": [],
    "commands": []
  },
  "execution_post": {
    "global_variables": [],
    "commands": []
  }
}
```

### Variable Scope and Lifetime

The execution process is as follows. Each time a stage begins execution, the runtime first copies a fresh variable table from `global_variables`, then executes `commands`. Variables modified by `assign` only affect the current stage execution. Those runtime changes are not automatically written back into configuration after the current frame ends.

Conclusion: `global_variables` is the initial variable table of this stage. It is not a cross-frame persistent variable store.

Long-lived user settings should be placed in `attributes`. Execution variables are only used during the current stage execution.

### `global_variables`

Purpose: provides a set of initial execution variables for one stage. These variables are visible to all commands and passes in that stage.

Each array element must be a single-key object, and each object declares one variable.

Short form:

```json
{
  "RT_READY": false
}
```

Full form:

```json
{
  "RT_READY": {
    "type": "bool",
    "default_value": false
  }
}
```

#### Supported Types

Two separate questions must be distinguished here: automatic inference and explicit declaration.

##### Types Supported By Automatic Inference

If `type` is omitted, the loader only infers these types from `default_value` automatically:

- `bool`
- `int`
- `float`
- `string`

##### Types Explicitly Handled By The Current Runtime When `type` Is Provided

- `bool`
- `int`
- `float`
- `string`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

Notes:

- `bool`: can participate in condition evaluation and is mapped to a boolean-style macro in shaders
- `int` / `int_range`: mapped to `int(...)` in shaders
- `float` / `float_range`: handled as floating-point values in both expressions and shaders
- `enum:...`: still works as a numeric index internally
- `string`: syntactically allowed, but cannot directly participate in numeric expressions; it is also not suitable as a normal numeric execution value uploaded to shaders

Not supported:

- `vec3`
- arbitrary object / array types

#### Naming Rules

Execution variable names must satisfy the following rule: the first character must be an English letter, and the remaining characters may only be letters, digits, or underscores.

Example: `RT_READY` is valid. `_READY` is invalid.

Reserved names: `if`, `true`, `false`.

### No `local_variables` List

Current behavior: there is no dedicated `local_variables` declaration block. Local variables are introduced dynamically by `assign` inside the command tree.

The loader recursively scans the entire command tree and collects those variables into the execution-variable table as well.

#### Types Supported By Local Variables

The same distinction between automatic inference and explicit declaration applies here.

##### Automatic Inference In `assign`

If `type` is omitted: boolean literals are inferred as `bool`, integer literals as `int`, and floating-point literals as `float`. If the right-hand side is a string expression, the static collection stage usually can only treat it as a placeholder `float`.

##### Explicit `type` In `assign`

When `type` is explicitly provided, the types explicitly handled by the current runtime are the same as for `global_variables`:

- `bool`
- `int`
- `float`
- `string`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

Not supported:

- `vec3`
- object / array

### `commands`

Purpose: forms the execution program for one stage. It determines which passes execute, in what order, whether loops occur, and whether branching occurs.

Each command entry must be a single-key object, and each object declares one command.

The currently supported commands are `assign`, `pass`, `if_else`, and `while`.

Note: the command key name is `pass`.

### `assign`

Purpose: assigns values to execution variables, and may also create local variables that were not declared earlier in `global_variables`.

Short form:

```json
{
  "assign": {
    "RT_X": 1,
    "RT_Y": "RT_X + 1"
  }
}
```

Full form:

```json
{
  "assign": {
    "RT_X": {
      "type": "int",
      "value": "RT_X + 1"
    }
  }
}
```

The short form supports booleans, numbers, and string expressions. The full form only allows `type` and `value`. If an explicit `type` conflicts with the known type of an existing variable, the loader reports an error.

### `pass`

Purpose: invokes a pass that has already been declared in `passes`.

Example:

```json
{
  "pass": "world"
}
```

Here `world` is the `name` of some pass in the same stage.

### `if_else`

Purpose: performs conditional branching.

Example:

```json
{
  "if_else": {
    "condition": "!RT_READY",
    "then": [
      { "pass": "trans_lut" }
    ],
    "else": []
  }
}
```

`condition` is required. `then` is required. `else` may be omitted.

### `while`

Purpose: repeatedly executes a group of commands while the condition remains true.

Example:

```json
{
  "while": {
    "condition": "RT_I < 4",
    "then": [
      { "pass": "step" },
      { "assign": { "RT_I": "RT_I + 1" } }
    ]
  }
}
```

The runtime contains an internal loop limit to avoid infinite loops.

### Default Behavior When `commands` Is Empty

If a stage either has no execution block at all, or has an execution block with an empty `commands` array,

the loader automatically generates default commands in the original order of the `passes` array:

```json
[
  { "pass": "first_pass" },
  { "pass": "second_pass" }
]
```

Stage filtering is also applied: `execution` only receives passes whose `stage == ray_tracing`, and `execution_post` only receives passes whose `stage == post_render`.

### Expressions

Purpose: used in conditions, assignments, dispatch sizes, and runtime texture / buffer size expressions.

Expressions are parsed by the built-in extended `tinyexpr` parser and are suitable for numeric and boolean control flow.

Common forms: `RT_X + 1`, `RT_I < 4`, `!RT_READY`, `RT_A && RT_B`, `if(RT_FLAG, 1, 0)`.

Common names available in the expression context include the current stage's execution variables, numerically usable attribute definitions, `RENDER_WIDTH`, and `RENDER_HEIGHT`.

The `ray_tracing` stage also provides `CHUNK_SIZE_X`, `CHUNK_SIZE_Y`, `CHUNK_SIZE_Z`, `CHUNK_BOTTOM_SECTION_COORD`, and `CHUNK_NUM`.

### How Execution Variables Enter Shaders

Purpose: allows shaders to read execution variables directly in GLSL.

At the descriptor-set level, execution variables are placed in a dedicated storage-buffer set located immediately after runtime textures / buffers. In the `ray_tracing` module, if the pack declares any `textures` or `buffers`, execution variables are in `set 6`; if the pack declares no runtime textures / buffers, they are in `set 5`. In the `post_render` module, if the pack declares any `textures` or `buffers`, execution variables are in `set 5`; if the pack declares no runtime textures / buffers, they are in `set 4`.

The runtime automatically injects the corresponding GLSL declaration. The objects for which you usually need to write explicit set/binding declarations by hand are the runtime textures / buffers described earlier.

The runtime automatically generates one GLSL snippet and injects it into the shader.

The generated form is:

```glsl
layout(set = EXECUTION_SET, binding = 0, std430) readonly buffer RtExecutionVariables {
    float RT_X_;
    float RT_READY_;
} rtExecutionVariables;

#define RT_X int(rtExecutionVariables.RT_X_)
#define RT_READY (rtExecutionVariables.RT_READY_ != 0.0)
```

Therefore, all execution variables are uploaded as `float` at the storage level. `bool` is wrapped as `!= 0.0`. `int` / `int_range` / `enum` are wrapped as `int(...)`. `float` / `float_range` are used directly as floating-point values.

## lang Files

Language files are typically placed at:

```text
lang/en_us.lang
lang/zh_cn.lang
...
```

Format:

```text
# comment
render_pipeline.example.name=Example
render_pipeline.example.mode.a=Mode A
render_pipeline.example.mode.b=Mode B
```

Each line is one `key=value` pair. Lines beginning with `#` are ignored. The loader first loads `en_us`, then loads the current language. Language codes are normalized. For example, `zh-CN.lang`, `zh_cn.lang`, and `ZH CN.lang` are all normalized to a form like `zh_cn`.

It is recommended to provide translations at least for these items: the `name` of each attribute and every option of each enum.

## Example

```json
{
  "attributes": [
    {
      "name": "pack.example.exposure",
      "type": "float_range:0.0-8.0",
      "default_value": "1.0",
      "define": "RT_EXPOSURE"
    }
  ],
  "textures": [
    {
      "name": "lut",
      "type": "intermediate",
      "shared": true,
      "format": "R16G16B16A16_SFLOAT",
      "width": 128,
      "height": 32,
      "sampled_binding": 0,
      "storage_binding": 1
    }
  ],
  "execution": {
    "global_variables": [
      {
        "RT_LUT_READY": false
      }
    ],
    "commands": [
      {
        "if_else": {
          "condition": "!RT_LUT_READY",
          "then": [
            { "pass": "build_lut" },
            { "assign": { "RT_LUT_READY": true } }
          ]
        }
      },
      {
        "pass": "world"
      }
    ]
  },
  "execution_post": {
    "global_variables": [],
    "commands": []
  },
  "passes": [
    {
      "type": "full_screen",
      "name": "build_lut",
      "fragment": "atmosphere/build_lut.frag",
      "target": "lut"
    },
    {
      "type": "ray_tracing",
      "name": "world",
      "inputs": {
        "images": ["lut"]
      },
      "outputs": {
        "images": ["out:radiance"]
      },
      "rgen": "ray_tracing/world.rgen",
      "default_hit_group": "default",
      "hit_groups": {
        "default": {
          "type": "triangle",
          "shaders": {
            "rchit": "ray_tracing/default.rchit"
          }
        }
      },
      "miss": [
        {
          "name": "world",
          "shader": "ray_tracing/world.rmiss"
        }
      ]
    }
  ]
}
```

This example declares one attribute, one intermediate texture, and two passes. `pack.example.exposure` is exported as the shader macro `RT_EXPOSURE`, which lets shaders read the user-selected exposure value. `lut` is a shared `128 x 32` intermediate texture. `build_lut` writes to it as a render target, and `world` reads it as an input image. `sampled_binding: 0` and `storage_binding: 1` mean that the shader can access it both as a sampled image and as a storage image.

The execution flow is as follows. `RT_LUT_READY` in `execution.global_variables` starts as `false`. On the first execution, the `if_else` condition is true, so the runtime runs `build_lut` first, uses `atmosphere/build_lut.frag` to generate `lut`, and then assigns `RT_LUT_READY = true`. The following `pass: "world"` runs every time, reads `lut`, and writes to `out:radiance`. Starting from the second execution, `RT_LUT_READY` is already `true`, so `build_lut` is skipped and only the `world` ray tracing dispatch remains. `execution_post` is empty, which means this example has no additional `post_render` stage flow.

## Runtime Settings File

User-selected shader attribute settings are not written back to `configs.json`.

At runtime, a side-by text file is generated next to the shader pack zip:

```text
internal.zip.txt
```

This file is runtime state and does not need to be distributed together with the shader pack.
