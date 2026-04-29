## 包的加载方式

shader pack可以是：

- 一个目录
- 一个`.zip`

无论哪种形式，包根目录都必须直接包含：

```text
configs.json
```

推荐目录结构：

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

`configs.json`里的所有文件路径都必须是相对于包根目录的相对路径，绝对路径会直接报错。如果是zip，loader会先解压，再把解压目录本身当作包根目录；实际临时目录位于`radiance目录/temp/shaders/world/ray_tracing/<zip主名>_<路径哈希>`，也就是渲染器工作目录下的`temp/shaders/world/ray_tracing/`子目录。zip包不能再多套一层目录，`configs.json`必须位于压缩包最外层。shader编译阶段的`#include`会先相对当前shader文件所在目录解析，再相对pack根目录解析。

如果用户指定的pack加载失败，系统会回退到built-in pack。如果pack写了`"requires_emission": true`，但当前运行环境没有启用chunk emission采集，系统也会回退到built-in pack。

## 运行时资源和绑定

### 自定义runtime resource

`sampled_binding`、`storage_binding`和buffer的`binding`只决定自定义runtime resource在“自定义资源descriptor set”中的binding编号。

当前分配规则如下：`ray_tracing` stage中的自定义runtime resource位于`set 5`，`post_render` stage中的自定义runtime resource位于`set 4`。如果pack完全没有声明任何`textures`或`buffers`，这一整组自定义资源set不会创建。

对应关系：例如`sampled_binding: 3`表示该texture出现在对应stage的自定义资源set中的`binding 3`，`storage_binding: 7`表示该texture出现在对应stage的自定义资源set中的`binding 7`，`binding: 24`表示该buffer出现在对应stage的自定义资源set中的`binding 24`。

示例：

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

在`ray_tracing` stage中，这个示例里的`history`会占用自定义资源set中的`binding 3`和`binding 7`，`reservoirs`会占用`binding 24`。对应的shader声明如下：

```glsl
layout(set = 5, binding = 3) uniform sampler2D historySampler;
layout(set = 5, binding = 7, rgba16f) uniform image2D historyImage;
layout(set = 5, binding = 24) buffer Reservoirs {
  vec4 data[];
} reservoirs;
```

如果同一组资源在`post_render` stage中使用，binding编号不变，只需将set编号改为`4`。

### `ray_tracing` module

`ray_tracing` module的内置descriptor set与binding如下：

这些set按资源类别划分。`set 0`存放游戏基础贴图，`set 1`存放加速结构和场景几何数据，`set 2`存放world / sky uniform buffer，`set 3`存放radiance维护的主要输出图像，`set 4`存放SHARC资源，`set 5`存放shader pack自定义runtime resource；如果存在execution变量且pack声明了自定义runtime resource，execution变量会位于后续的`set 6`。

- `set 0`
  - `binding 0`：radiance维护的`combined image sampler`数组，是游戏的基础贴图存放的地方，容量`4096`
- `set 1`
  - `binding 0`：TLAS
  - `binding 1`：`blasOffsetsBuffer`
  - `binding 2`：`indexBufferAddr`
  - `binding 3`：`lastIndexBufferAddr`
  - `binding 4`：`positionBufferAddr`
  - `binding 5`：`materialBufferAddr`
  - `binding 6`：`lastPositionBufferAddr`
  - `binding 7`：`textureMappingBuffer`
  - `binding 8`：`lastObjToWorldMat`
  - `binding 9`：`chunkPackedData`。只在启用chunk emission采集且数据存在时绑定
- `set 2`
  - `binding 0`：`worldUniformBuffer`
  - `binding 1`：`lastWorldUniformBuffer`
  - `binding 2`：`skyUniformBuffer`
- `set 3`
  - `binding 0`：`out:radiance`
  - `binding 1`：`out:diffuse_albedo_metallic`
  - `binding 2`：`out:specular_albedo`
  - `binding 3`：`out:normal_roughness`
  - `binding 4`：`out:motion_vector`
  - `binding 5`：`out:linear_depth`
  - `binding 6`：`out:specular_hit_depth`
  - `binding 7`：`out:first_hit_depth`
  - `binding 8`：`out:first_hit_diffuse_direct_light`
  - `binding 9`：`out:first_hit_diffuse_indirect_light`
  - `binding 10`：`out:first_hit_specular`
  - `binding 11`：`out:first_hit_clear`
  - `binding 12`：`out:first_hit_base_emission`
  - `binding 13`：`out:fog_image`
  - `binding 14`：`out:first_hit_refraction`
- `set 4`
  - `binding 0`：`SharcConfigBuffer`
  - `binding 1`：`SharcHashEntriesBuffer`
  - `binding 2`：`SharcAccumulationBuffer`
  - `binding 3`：`SharcResolvedBuffer`
  - `binding 4`：`SharcLockBuffer`
- `set 5`
  - shader pack自定义runtime texture / buffer
  - 具体binding编号直接来自`sampled_binding`、`storage_binding`和buffer的`binding`

说明：`set 4`是SHARC专用资源组，不在`configs.json`中声明。自定义`rgen` / `comp`如果要直接接入SHARC，应按`set 4`的固定binding声明。

### `post_render` module

`post_render` module的内置descriptor set与binding如下：

`post_render` module中的set划分与`ray_tracing`不同。`set 0`存放游戏基础贴图以及`out:first_hit_depth`的storage image，`set 1`存放world / sky uniform buffer，`set 2`存放`textureMappingBuffer`，`set 3`存放post阶段常用输出图像，`set 4`存放shader pack自定义runtime resource；如果存在execution变量且pack声明了自定义runtime resource，execution变量会位于后续的`set 5`。

- `set 0`
  - `binding 0`：radiance维护的`combined image sampler`数组，是游戏的基础贴图存放的地方，容量`4096`
  - `binding 2`：`out:first_hit_depth`的storage image
- `set 1`
  - `binding 0`：`worldUniformBuffer`
  - `binding 1`：`skyUniformBuffer`
- `set 2`
  - `binding 0`：`textureMappingBuffer`
- `set 3`
  - `binding 0`：`out:ldr`
  - `binding 1`：`out:first_hit_depth`
  - `binding 2`：`out:hdr`
  - `binding 3`：`out:motion_vector`
  - `binding 4`：`out:normal_roughness`
- `set 4`
  - shader pack自定义runtime texture / buffer
  - 具体binding编号直接来自`sampled_binding`、`storage_binding`和buffer的`binding`

### execution变量

execution变量位于单独的storage buffer set中，固定使用`binding 0`。

当前分配规则如下：`ray_tracing` stage中，如果pack声明了任意`textures`或`buffers`，execution变量位于`set 6, binding 0`；如果pack完全没有声明runtime texture / buffer，execution变量位于`set 5, binding 0`。`post_render` stage中，如果pack声明了任意`textures`或`buffers`，execution变量位于`set 5, binding 0`；如果pack完全没有声明runtime texture / buffer，execution变量位于`set 4, binding 0`。

示例：如果pack声明了上面的`history`和`reservoirs`，那么`ray_tracing` stage中的execution变量buffer应按`set 6, binding 0`声明，`post_render` stage中的execution变量buffer应按`set 5, binding 0`声明。如果pack没有任何自定义runtime texture / buffer，这两个stage中的set编号都会各减`1`。因此，是否声明自定义runtime resource会直接影响execution变量在shader中的set编号。

## 根配置结构

最常见的根结构如下：

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

字段说明：

- `radiance`：shader pack的manifest，用于识别这个目录或zip是否应被当成shader pack，以及在列表里显示什么名字；不参与后文这些运行时资源字段的解析
- `requires_emission`：表示这个pack是否要求运行时提供chunk emission数据，默认`false`
- `sharc`：可选。声明整个shader pack共享的一次SHARC update / resolve配置，只作用于`ray_tracing` stage
- `attributes`：声明用户可调的shader参数
- `textures`：声明运行时texture资源
- `buffers`：声明运行时storage buffer资源
- `execution`：描述`ray_tracing` stage中pass的执行流程
- `execution_post`：描述`post_render` stage中pass的执行流程
- `passes`：声明这个pack里有哪些pass。要求它是非空数组

loader只解析自己认识的字段。`radiance`这类manifest元数据不参与后文的运行时资源与执行流程配置。

## `manifest`

当前`configs.json`里通常应当带一个`radiance` manifest。对需要被识别成shader pack的目录或zip来说，它是必需字段。例如：

```json
{
  "radiance": {
    "shader_pack": true,
    "display_name": "§8§kXX §r§b§lAdvanced§r §8§kXX§r"
  }
}
```

它用于标记shader pack并提供显示名。`radiance`是manifest根对象，也可以直接写成布尔值；`radiance.shader_pack`控制这个包是否被识别成shader pack，`radiance.display_name`控制shader pack列表中的显示名。

如果`radiance`是布尔值，它只表示是否识别为shader pack，不能提供显示名。如果`radiance`是object但没写`shader_pack`，会按`true`处理。如果根对象里完全没有`radiance`，这个目录或zip不会被识别成shader pack条目。如果没有可用显示名，列表最终会回退到去掉扩展名后的zip或目录名。`radiance`用于shader pack识别，必须提供；`display_name`用于控制列表显示名；pass、texture、execution以及其他运行时行为由后文的运行时字段配置。

## `sharc`

`sharc`用于声明整个shader pack共享的一次[SHARC](https://github.com/NVIDIA-RTX/SHARC) update / resolve配置，只作用于`ray_tracing` stage。例如：

```json
{
  "sharc": {
    "update_pass": "final_compose",
    "resolve_comp": "sharc_resolve.comp"
  }
}
```

字段：

- `update_pass`：一个已有`ray_tracing` pass的名字。runtime会使用这个pass的`rgen`、`hit_groups`、`miss`、`inputs`、`outputs`和dispatch尺寸表达式来执行全局SHARC update
- `resolve_comp`：SHARC resolve使用的compute shader路径。它必须是相对于pack根目录的相对路径

如果根对象存在`sharc`且当前运行时的SHARC开关开启，runtime会在整个`ray_tracing` stage开始时，先执行一次全局SHARC update，再执行一次resolve。后续真正执行`execution`里的各个pass时，只有写了`query_sharc: true`的`ray_tracing` pass能使用SHARC query路径。

`update_pass`对应的`rgen`会额外编译一份SHARC update路径，并注入`USE_SHARC=1`和`SHARC_UPDATE=1`。带有`query_sharc: true`的`rgen`会注入`USE_SHARC=1`和`SHARC_QUERY=1`。`resolve_comp`会在update之后执行，用于整理SHARC缓存条目。

全局SHARC update的ray tracing dispatch数量本身就是下采样后的数量，不是全分辨率dispatch。假设`update_pass`计算出的原始追踪尺寸为`width`和`height`，运行时实际发起的update dispatch尺寸是：

```text
update_width  = ceil(width  / sharc_update_downsample_factor)
update_height = ceil(height / sharc_update_downsample_factor)
update_depth  = 1
```

shader中的`gl_LaunchIDEXT`对应的是下采样后的block坐标，再由shader在block内选取一个全分辨率像素，因此它不是“全分辨率dispatch，只是在shader里少做一些事”。

## `attributes`

`attributes`用于声明可调参数，以及这些参数如何映射为shader宏定义。

每个attribute都是一个对象：

```json
{
  "name": "translation.key.or.id",
  "type": "float_range:0.0-1.0",
  "default_value": "0.5",
  "define": "RT_EXAMPLE"
}
```

字段：

- `name`：属性名。通常写翻译key
- `type`：属性类型
- `default_value`：默认值。这里必须是字符串
- `define`：可选。描述如何把当前attribute值转成shader宏

### 支持的attribute类型

`attributes`支持以下类型：

- `bool`
- `int`
- `float`
- `string`
- `vec3`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

`default_value`在`attributes`中始终必须是字符串。`vec3`约定用`x,y,z`的字符串格式。`enum:`后面的每个选项都是固定字符串，运行时按字符串精确匹配。

### `define`的真实语法

`define`支持三种结构。下文按loader实际解析的JSON形状分类。

#### 写法1：字符串

```json
{
  "name": "my_attr",
  "type": "float",
  "default_value": "1.0",
  "define": "RT_MY_ATTR"
}
```

当前attribute值会直接映射到一个宏名，等价的GLSL形式如下：

```glsl
#define RT_MY_ATTR 1.0
```

#### 写法2：对象，且value全部是字符串

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

`define`是一个“宏名 -> 表达式字符串”的表，右侧字符串是loader执行`X`替换时使用的表达式模板。`X`会被当前attribute值替换。

#### 写法3：对象，且value全部是对象

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

外层key是当前attribute的原始字符串值，内层是“宏名 -> 表达式字符串”的表；当前值命中哪个外层key，就应用哪一组定义。

内层value也可写表达式。例如：

```json
{
  "define": {
    "mode.a": {
      "RT_MODE_SCALE": "((X) + 1)"
    }
  }
}
```

该写法合法。loader对内层value的处理方式与写法2相同，仍执行`X`替换。

需区分两层语义：外层key的匹配使用的是attribute的原始字符串值，内层表达式里的`X`使用的是转换后的值（`bool`会被转换成`1`或`0`，`enum`会被转换成枚举下标`0/1/2/...`）。

但是下面这种混合形式不合法：

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

原因是loader会把`define` object解析成以下两类之一：“value全是字符串”和“value全是对象”。两类不能混用。

### attribute值如何传给`define`

在做`define`替换前，当前attribute值会先转换成loader使用的中间表示：`bool`会被转换成`1`或`0`，`enum`会被转换成枚举下标`0/1/2/...`，其他类型直接使用原字符串。

`bool`常见真值通常写成`render_pipeline.true`、`true`或`1`。`enum`的当前值如果和`enum:`里列出的字符串对不上，就不会得到预期的枚举下标。

## `textures`

`textures`用来声明shader pack运行时会用到的图像资源。

texture分类如下：

- `file`：从包内文件导入贴图数据
- `intermediate`：由runtime在显存中创建图像资源，供pass读写

一个texture对象的基本样子如下：

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

`sampled_binding`、`storage_binding`和buffer的`binding`都用于描述runtime资源绑定到descriptor set中的`binding`槽位，也就是shader在GPU侧通过哪个binding号访问这份资源。它们不描述pass的执行顺序、barrier安排或哪个pass负责读写；这些由`execution` / `execution_post`、`inputs`和`outputs`负责。

绑定对象：`sampled_binding`表示texture以sampled image形式暴露给shader，常见声明对象为`sampler2D` / `samplerCube` / `sampler2DArray` / `sampler3D`；`storage_binding`表示texture以storage image形式暴露给shader，常见声明对象为`imageLoad` / `imageStore`。

绑定形式：仅声明`sampled_binding`时，该texture以sampled image形式绑定；仅声明`storage_binding`时，该texture以storage image形式绑定；同时声明两者时，该texture同时提供sampled image和storage image两种绑定形式。

约束：`configs.json`中不写descriptor set编号，只写binding编号。所有runtime texture / buffer的binding共用同一套编号空间，不能重复。

### 通用字段

- `name`：资源名
- `type`：`file`或`intermediate`
- `dimension`：`2d`、`2d_array`、`3d`、`cube`，默认`2d`
- `sampled_binding`：可选
- `storage_binding`：可选
- `filter`：`linear`或`nearest`，默认`linear`
- `address_mode`：`repeat`、`clamp_to_edge`、`mirrored_repeat`，默认`repeat`
- `shared`：是否跨所有frame共享一份资源，默认`false`

### `file`贴图

示例：

```json
{
  "name": "noise",
  "type": "file",
  "path": "textures/noise.png",
  "sampled_binding": 3
}
```

`path`必填且必须是相对路径，`sampled_binding`必填，`format`可选且默认`R8G8B8A8_UNORM`。当前支持的导入维度是`2d`、`2d_array`和`3d`，`cube`不能作为imported texture。

运行时行为：`2d` imported texture通过图片加载器读取，当前要求`format`必须是`R8G8B8A8_UNORM`。`2d_array`和`3d` imported texture会直接把文件当原始二进制blob读入。

常用文件形式：

- `2d`：常用文件形式为`.png`等普通图片
- `2d_array` / `3d`：常用文件形式为`.raw`等原始二进制数据

对于imported `2d_array` / `3d`，还必须额外提供：`width`、`height`和`depth`。

并且文件字节数必须严格等于：

```text
width * height * depth * 每像素字节数
```

### `intermediate`贴图

示例：

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

`format`必填，`width` / `height` / `depth`可以写数字，也可以写字符串表达式，`scale`只能是数字。

尺寸写法：`2d` / `cube`二选一使用`width` + `height`或`scale`；`2d_array` / `3d`必须显式写`width`、`height`和`depth`。

### `scale`

`scale`表示：这张`intermediate` texture的宽高按当前参考渲染尺寸进行缩放。

计算方式如下：

```text
width  = max(1, int(referenceWidth  * scale))
height = max(1, int(referenceHeight * scale))
```

其中`referenceWidth` / `referenceHeight`对应当前渲染阶段使用的参考输出尺寸，与`RENDER_WIDTH` / `RENDER_HEIGHT`同级。

示例：`scale: 1`表示与参考尺寸同宽高，`scale: 0.5`表示半分辨率，`scale: 2`表示双倍分辨率。

`scale`只影响宽高，不影响layer数或depth。

### 支持的format

当前loader支持这些格式字符串：

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

`buffers`用来声明runtime storage buffer资源。

示例：

```json
{
  "name": "reservoir",
  "type": "intermediate",
  "size": "RENDER_WIDTH * RENDER_HEIGHT * 16",
  "binding": 24
}
```

字段：

- `name`：资源名
- `type`：当前只能是`intermediate`
- `size`：必填。可以是数字，也可以是字符串表达式
- `binding`：必填。表示这个buffer在shader runtime resource descriptor set中使用的binding槽位
- `shared`：可选，默认`false`

说明：`binding`与texture的`sampled_binding` / `storage_binding`共用一个全局编号空间。`size`的表达式也是在runtime中求值的。

## `passes`

`passes`用于声明shader pack中的各个GPU执行步骤，例如：

- 一次full-screen draw
- 一次ray tracing dispatch
- 一次compute dispatch
- 一次post-render draw

`passes`只负责声明“有哪些步骤”和“每一步长什么样”。

执行顺序与执行条件由后面的`execution` / `execution_post`定义。

### 资源列表

当前`inputs` / `outputs`的语法如下，它们是对象：

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

`inputs` / `outputs`都是可选的，`images` / `buffers`也都是可选的；省略时表示空列表，对象中只允许`images`和`buffers`。

`inputs` / `outputs`的意义是：它们用于告诉runtime这个pass会读哪些资源、会写哪些资源，runtime会据此生成资源barrier和布局转换。

它与binding的区别是：binding解决shader通过哪个binding号访问资源，`inputs` / `outputs`解决pass声称自己读写了哪些资源。

### 通用规则

每个pass都必须是一个object，并且必须带有`type`。

通用字段：

- `type`：pass类型
- `name`：pass名
- `stage`：可选。支持`ray_tracing`和`post_render`，默认`ray_tracing`
- `inputs`：可选。声明本pass会读取哪些image / buffer
- `outputs`：可选。声明本pass会写入哪些image / buffer
- `define`：可选。声明这个pass额外注入的宏定义

`ray_tracing` pass只能放在`ray_tracing` stage，`render` pass目前只能放在`post_render` stage（渲染天气、粒子等。后续会将区块数据也加进来，这样就可以光栅渲染了），`define`中的value都必须是字符串。

### `full_screen` pass

示例：

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

字段：

- `name`：pass名
- `fragment`：fragment shader路径
- `target`：full-screen draw的输出目标image名
- `inputs`：可选。声明读取的资源
- `outputs`：可选。声明写入的资源
- `face`：可选，整数0到5，用于cube target的某一面
- `layer`：可选，无符号整数，用于array target的某一层

如果`outputs.images`里没有`target`对应的贴图，loader会自动补进去；如果`outputs`整体省略，loader也会自动把`target`视为唯一image output。`face`和`layer`不能同时写，`face`只能用于cube target，`layer`只能用于array target。

`ray_tracing` stage存在以下特殊行为：如果`target`是cube，且没有写`face`也没有写`layer`，runtime会将该pass隐式执行6次，并分别向shader注入：

```glsl
FACE=0
FACE=1
FACE=2
FACE=3
FACE=4
FACE=5
```

`post_render` stage下更严格：full-screen pass只能写自己的`target`；如果target是数组图像，必须显式写`layer`。

### `ray_tracing` pass

示例：

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

字段：

- `name`：pass名
- `inputs`：可选。声明读取的资源
- `outputs`：可选。声明写入的资源
- `rgen`：ray generation shader路径
- `default_hit_group`：默认hit group的名字
- `hit_groups`：各几何组名对应的hit shader组配置
- `miss`：miss shader列表
- `width`：可选，默认`RENDER_WIDTH`
- `height`：可选，默认`RENDER_HEIGHT`
- `depth`：可选，默认`1`
- `query_sharc`：可选，默认`false`

#### `query_sharc`

`query_sharc`用于声明这个`ray_tracing` pass是否查询SHARC缓存。

它只控制这个pass自己的query路径，不决定update和resolve使用哪个shader；update和resolve由根级`sharc`字段统一配置。

如果根对象存在`sharc`且当前运行时的SHARC开关开启，runtime会在整个`ray_tracing` stage开始时先执行一次全局SHARC update和一次resolve。之后，每个`query_sharc: true`的`ray_tracing` pass在自己的query dispatch前会插入SHARC相关barrier，并以`USE_SHARC=1`和`SHARC_QUERY=1`编译其`rgen`。

`query_sharc: false`的pass按普通`ray_tracing` pass执行，不注入SHARC query宏，也不会为SHARC插入额外barrier。

如果某个pass同时被`sharc.update_pass`引用，并且自身又写了`query_sharc: true`，它会同时参与两件事：第一，这个pass的配置会被用来执行全局SHARC update；第二，当`execution`真正执行到这个pass时，它仍会按自己的`width` / `height` / `depth`执行一次普通query dispatch。

全局SHARC update使用下采样后的dispatch尺寸，query dispatch使用各pass自己的完整追踪尺寸。update dispatch的宽高计算方式如下：

```text
update_width  = ceil(width  / sharc_update_downsample_factor)
update_height = ceil(height / sharc_update_downsample_factor)
```

其中`width`和`height`是`update_pass`自身计算出来的追踪尺寸，`sharc_update_downsample_factor`由运行时的SHARC下采样系数决定。query dispatch仍然使用各pass自己配置出来的完整`width` / `height` / `depth`。

### `hit_groups`

`hit_groups`解决的问题是：当TLAS/BLAS中某段几何被射线命中时，应该使用哪一组hit shader。

这里的group name对应运行时的几何组名。当前实现中，对应关系如下：Java侧传给native的每段几何的`geometryGroupName`，这个字符串当前直接来自`RenderLayer.name`。

对chunk / entity几何：一段几何在Java侧属于哪个`RenderLayer`，native侧就会拿到哪个`RenderLayer.name`，然后用这个字符串去`hit_groups`里查同名group，匹配是纯字符串精确匹配。

另外runtime还会额外注入一个保留组名：`shadow`。

它是TLAS构建阶段额外插入的合成group，用于shadow相关hit group选择。

如果pack没有显式提供名为`shadow`的hit group，当前runtime会把shadow几何回退到`default_hit_group`。

### `hit_groups`的解析规则

解析流程如下：

1. `hit_groups`必须是非空object
2. `default_hit_group`指向的名字必须真的存在于`hit_groups`中
3. 每个group的`type`只能是`triangle`或`aabb`
4. `shaders`中可显式写`rchit`、`rahit`、`rint`
5. 如果某个非默认group最终仍然既没有`rchit`，也没有`rint`，它会回退到默认group的`rchit`
6. 如果某个group最终拥有`rint`，runtime会把它当成procedural hit group
7. 如果你把`type`写成`aabb`，但最终没有`rint`，会直接报错
8. 每个group最终至少必须有：`rchit`或`rint`

其他细节：默认group会被排在第一个，其他group会按名字排序。`miss`必须是非空数组。如果runtime找不到某个几何组名对应的hit group，会回退到`default_hit_group`。

### `compute` pass

示例：

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

字段：

- `name`：pass名
- `compute`：compute shader路径
- `inputs`：可选。声明读取的资源
- `outputs`：可选。声明写入的资源
- `gx`：dispatch的x方向工作组数量
- `gy`：dispatch的y方向工作组数量
- `gz`：dispatch的z方向工作组数量

作用：用一个compute shader做一次dispatch。

`compute` shader路径必填，`gx` / `gy` / `gz`可选且默认都是`1`。当前runtime只在`ray_tracing` stage支持compute pass；如果把compute pass放到`post_render` stage，后续初始化阶段会报错。

### `render` pass

`render` pass只用于`post_render` stage。

它的作用是：从post-render队列中取出某类几何，并使用vertex/fragment shader进行图形管线绘制。

它有两种shader配置方式。

#### 写法1：单一shader对

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

#### 写法2：多variant shader

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

字段：

- `name`：pass名
- `content`：这个pass要消费哪一类post-render几何
- `vertex` / `fragment`：单一shader对写法；与`shader`二选一
- `shader`：多variant写法；与`vertex` / `fragment`二选一
- `inputs`：可选。声明读取的资源
- `outputs`：可选。声明写入的资源
- `depth_test`：可选。是否开启深度测试
- `depth_write`：可选。是否写深度
- `depth_compare`：可选。深度比较函数
- `color_blend`：可选。颜色混合配置

深度与混合相关字段的取值规则如下：

- `depth_test`：布尔值，支持`true`或`false`，默认`true`
- `depth_write`：布尔值，支持`true`或`false`。默认值按`content`决定，`text`、`name_tag`、`star`的默认值为`true`，其他`content`的默认值为`false`
- `depth_compare`：字符串，支持`never`、`less`、`equal`、`less_or_equal`、`greater`、`not_equal`、`greater_or_equal`、`always`。默认值通常为`less`，`name_tag`的默认值为`always`
- `color_blend`：布尔值，支持`true`或`false`，默认`true`。`true`表示启用标准alpha混合，颜色混合参数为`src_alpha`和`one_minus_src_alpha`；`false`表示关闭颜色混合，颜色输出直接覆盖目标

`content`的作用是：选择这个pass要消费哪一类post-render几何队列。

当前支持：

- `weather`
- `particle`
- `text`
- `name_tag`
- `star`

### `render` pass的variant匹配规则

`shader` object里的key对应：post-render队列里每段几何携带的`contentName`。

runtime选择variant的规则是：

1. 先拿到当前几何的`contentName`
2. 在`shader` map中查同名key
3. 如果找不到，再回退到`default`
4. 如果连`default`也没有，这段几何就不会命中任何variant

variant匹配对象为`contentName`。`content`用于选择post-render几何队列，`RenderLayer.name`不参与这一步variant匹配。

`contentName`来源如下：

- `weather`
  - `/weather/rain`
  - `/weather/snow`
  - 默认配置：`/weather/default`
- `particle`
  - `contentName`由粒子类型的`Identifier`生成
  - 粒子类型的`namespace`为`minecraft`时：`/particle/<path>`
  - 粒子类型的`namespace`不是`minecraft`时：`/particle/<namespace>/<path>`
  - 默认配置：`/particle/default`
- `text`
  - `/text/default`
- `name_tag`
  - `/name_tag/default`
- `star`
  - 使用pass的默认pipeline，不参与这套variant选择逻辑

适用范围：`weather`很适合用多variant shader。`particle`也可以用多variant shader，但前提是明确知道要匹配哪些粒子`contentName`。`text`和`name_tag`当前通常只会命中各自的`default`，`star`实际上只会使用默认pipeline。

### `post_render`运行时额外限制

`post_render` runtime限制如下：`render` pass必须输出`out:ldr`，并且额外只允许再输出一个`out:first_hit_depth`。

### 常用内置图像名

#### `ray_tracing` stage常用目标

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

#### `post_render` stage常见内置图像名

- `out:ldr`
- `out:first_hit_depth`
- `out:hdr`
- `out:motion_vector`
- `out:normal_roughness`

## execution与execution_post

`execution`和`execution_post`用于描述stage执行流程，属于配置部分。`execution`控制`ray_tracing` stage，`execution_post`控制`post_render` stage。它们决定某个stage中pass的执行顺序，并允许用简单变量和条件控制哪些pass会执行。

基本结构：

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

### 变量作用域与生命周期

执行过程如下：每次进入某个stage执行前，runtime都会先从`global_variables`复制出一份新的变量表，然后按`commands`执行。`assign`改动的变量只对当前这次stage执行有效，当前帧结束后，这些运行中改动不会自动写回配置。

结论：`global_variables`表示这个stage的初始变量表，不表示跨帧持久化变量。

用户长期保存的设置应写在`attributes`中。execution变量只用于当前stage执行。

### `global_variables`

作用：它为某个stage提供一组初始execution变量，这些变量在这个stage的所有command和pass中都可见。

数组元素必须为单键object，每个object声明一个变量。

简写：

```json
{
  "RT_READY": false
}
```

完整写法：

```json
{
  "RT_READY": {
    "type": "bool",
    "default_value": false
  }
}
```

#### 支持的类型

这里要分“自动推断”与“显式声明”两件事。

##### 自动推断支持

如果不写`type`，loader只会从`default_value`自动推断这几类：

- `bool`
- `int`
- `float`
- `string`

##### 显式`type`时，当前runtime有明确处理的类型

- `bool`
- `int`
- `float`
- `string`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

说明：

- `bool`：可参与条件判断，shader中会映射成布尔宏
- `int` / `int_range`：在shader中会映射成`int(...)`
- `float` / `float_range`：在表达式和shader中按浮点处理
- `enum:...`：底层仍以数值下标工作
- `string`：语法上允许，但不能直接参与数值表达式；上传到shader时也不适合作为普通execution数值使用

不支持：

- `vec3`
- 任意object / array类型

#### 名字规则

execution变量名必须满足：第一个字符必须是英文字母，后续字符只能是字母、数字、下划线。

例如：`RT_READY`合法，`_READY`不合法。

保留名字：`if`、`true`、`false`。

### 没有`local_variables`列表

作用：当前实现没有单独的`local_variables`声明区，局部变量是通过`assign`在command树中动态引入的。

loader会递归扫描整个command tree，把这些变量也收集到execution变量表中。

#### 局部变量支持哪些类型

这里同样分“自动推断”与“显式声明”。

##### `assign`自动推断

未声明`type`时：布尔字面量会推断成`bool`，整数字面量会推断成`int`，浮点字面量会推断成`float`。如果右侧是字符串表达式，loader在静态收集阶段通常只能把它先当成一个占位`float`。

##### `assign`显式`type`

显式`type`时，当前runtime有明确处理的类型与`global_variables`一致：

- `bool`
- `int`
- `float`
- `string`
- `int_range:min-max`
- `float_range:min-max`
- `enum:value0-value1-value2`

不支持：

- `vec3`
- object / array

### `commands`

作用：它组成某个stage的执行程序，决定哪些pass会执行、执行顺序如何、是否循环、是否分支。

每个command条目必须为单键object，每个object声明一个命令。

当前支持四种命令：`assign`、`pass`、`if_else`、`while`。

注意：命令键名为`pass`。

### `assign`

作用：它用于给execution变量赋值，也可以创建尚未在`global_variables`中声明的局部变量。

简写：

```json
{
  "assign": {
    "RT_X": 1,
    "RT_Y": "RT_X + 1"
  }
}
```

完整写法：

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

简写支持布尔、数字、字符串表达式，完整写法只允许`type`和`value`。如果显式`type`与已知变量类型冲突，会报错。

### `pass`

作用：它用于调用一个已经在`passes`中声明过的pass。

示例：

```json
{
  "pass": "world"
}
```

示例中的`world`对应同一个stage内某个pass的`name`。

### `if_else`

作用：它用于做条件分支。

示例：

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

`condition`必填，`then`必填，`else`可省略。

### `while`

作用：它用于在条件成立时重复执行一组命令。

示例：

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

说明：runtime内部有循环上限，用来避免死循环。

### `commands`为空时的默认行为

如果某个stage：完全没写execution block，或者写了execution block但`commands`为空，

loader会按`passes`数组中的原始顺序自动生成默认命令：

```json
[
  { "pass": "first_pass" },
  { "pass": "second_pass" }
]
```

并且按stage过滤：`execution`只会拿到`stage == ray_tracing`的pass，`execution_post`只会拿到`stage == post_render`的pass。

### 表达式

作用：它可用于条件判断、赋值、dispatch size以及runtime texture / buffer尺寸表达式。

表达式由当前内置的`tinyexpr`扩展解析，适合写数值和布尔控制流。

常见写法：`RT_X + 1`、`RT_I < 4`、`!RT_READY`、`RT_A && RT_B`、`if(RT_FLAG, 1, 0)`。

表达式上下文中的常用名字包括：当前stage的execution变量、可数值化的attribute define、`RENDER_WIDTH`、`RENDER_HEIGHT`。

`ray_tracing` stage还提供：`CHUNK_SIZE_X`、`CHUNK_SIZE_Y`、`CHUNK_SIZE_Z`、`CHUNK_BOTTOM_SECTION_COORD`、`CHUNK_NUM`。

### execution变量如何进入shader

作用：它用于让shader在GLSL里直接读到execution变量。

对descriptor set来说，execution变量位于单独的storage buffer set中，位置紧跟runtime texture / buffer之后：`ray_tracing` module中，如果pack声明了任意`textures`或`buffers`，execution变量位于`set 6`；如果pack完全没有runtime texture / buffer，execution变量位于`set 5`。`post_render` module中，如果pack声明了任意`textures`或`buffers`，execution变量位于`set 5`；如果pack完全没有runtime texture / buffer，execution变量位于`set 4`。

runtime会自动注入对应的GLSL声明。需要手写set/binding的对象主要是前述runtime texture / buffer资源。

runtime会自动生成一段GLSL并注入shader。

生成结果形式如下：

```glsl
layout(set = EXECUTION_SET, binding = 0, std430) readonly buffer RtExecutionVariables {
    float RT_X_;
    float RT_READY_;
} rtExecutionVariables;

#define RT_X int(rtExecutionVariables.RT_X_)
#define RT_READY (rtExecutionVariables.RT_READY_ != 0.0)
```

因此：所有execution变量底层都以`float`上传；`bool`会被包装成`!= 0.0`；`int` / `int_range` / `enum`会被包装成`int(...)`；`float` / `float_range`会直接作为浮点使用。

## lang文件

语言文件通常放在：

```text
lang/en_us.lang
lang/zh_cn.lang
...
```

格式：

```text
# comment
render_pipeline.example.name=Example
render_pipeline.example.mode.a=Mode A
render_pipeline.example.mode.b=Mode B
```

一行一个`key=value`，以`#`开头的行会被忽略。loader会先加载`en_us`，再加载当前语言。语言代码会被标准化，例如`zh-CN.lang`、`zh_cn.lang`、`ZH CN.lang`最终都能映射到类似`zh_cn`。

建议至少给这些内容提供翻译：attribute的`name`和enum的每个选项。

## 示例

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

这个示例声明了一个attribute、一个intermediate texture，以及两个pass。`pack.example.exposure`会导出为shader宏`RT_EXPOSURE`，供shader读取用户设置的曝光值。`lut`是一个`128 x 32`的shared intermediate texture，`build_lut`会把它作为渲染目标写入，`world`会把它作为输入图像读取；`sampled_binding: 0`和`storage_binding: 1`表示shader可以分别通过sampled image和storage image两种形式访问它。

执行流程如下：`execution.global_variables`中的`RT_LUT_READY`初始为`false`。第一次执行时，`if_else`条件成立，runtime先执行`build_lut`，用`atmosphere/build_lut.frag`生成`lut`，然后将`RT_LUT_READY`设为`true`。后面的`pass: "world"`每次都会执行，它会读取`lut`并输出到`out:radiance`。从第二次执行开始，`RT_LUT_READY`已经为`true`，因此`build_lut`会被跳过，只保留`world`这一遍ray tracing dispatch。`execution_post`为空，表示这个示例没有额外的`post_render`阶段流程。

## 运行时设置文件

shader attribute的用户设置不会写回`configs.json`。

运行时会在shader pack zip的旁边生成一个文本文件：

```text
internal.zip.txt
```

这个文件是运行时状态，不需要随shader pack一起分发。
