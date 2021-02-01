---
title: Zink Failcase 分析
date: 2021-01-27 15:53:33
categories:
- something
tags: 
- OpenGL
---

## ARB_vertex_type_2_10_10_10_rev

### **一、Overview**

该特性提供两种新的顶点属性数据格式：
- 有符号2.10.10.10顶点数据格式
- 无符号2.10.10.10顶点数据格式

每个格式描述了一个4分量流，法线、切线、Binormal和其他顶点属性通常可以在不引入明显瑕疵的情况下以较低的精度指定，从而减少它们消耗的内存量和内存带宽。

### **二、Piglit 测试结果**

1. ```./bin/arb_vertex_type_2_10_10_10_rev-array_types -auto -fbo```

    测试结果：

    ```
    testing: RGBA SINT 
    testing: RGBA SNORM 
    testing: RGBA UINT 
    testing: RGBA UNORM 
    testing: BGRA SNORM 
    Probe color at (85,5)   
    Expected: 0.000000 0.250000 0.500000 1.000000   
    Observed: 0.501961 0.250980 0.000000 1.000000 
    testing: BGRA UNORM
    ``` 
<!--more-->
2. ```./bin/draw-vertices-2101010 -auto```

    测试结果：
    ```
    Int vertices - 2/10/10/10
    Unsigned Int vertices - 2/10/10/10
    Int Color - 2/10/10/10
    Probe color at (45,5)
    Expected: 1.000000 0.000000 0.000000 0.333000
    Observed: 1.000000 0.000000 0.000000 0.000000
    warning: driver uses GL 4.2+ rules for signed normalized to float conversions
    Unsigned Int Color - 2/10/10/10
    Int BGRA Color - 2/10/10/10
    Probe color at (85,5)
    Expected: 0.000000 0.000000 1.000000 0.333000
    Observed: 1.000000 0.000000 0.000000 0.000000
    Probe color at (85,5)
    Expected: 0.000000 0.000000 1.000000 0.000000
    Observed: 1.000000 0.000000 0.000000 0.000000
    Unsigned Int BGRA Color - 2/10/10/10
    Int 2/10/10/10 - test ABI
    Unsigned 2/10/10/10 - test ABI
    ```


### **三、原因分析**

注意到测试结果中B和R位置发生了交换, 而在Zink 中，该特性与Vulkan映射如下：

    [PIPE_FORMAT_R10G10B10A2_UNORM] = VK_FORMAT_A2B10G10R10_UNORM_PACK32,
    [PIPE_FORMAT_R10G10B10A2_SNORM] = VK_FORMAT_A2B10G10R10_SNORM_PACK32,
    [PIPE_FORMAT_B10G10R10A2_UNORM] = VK_FORMAT_A2R10G10B10_UNORM_PACK32,
    [PIPE_FORMAT_B10G10R10A2_SNORM] = VK_FORMAT_A2B10G10R10_SNORM_PACK32,
    [PIPE_FORMAT_R10G10B10A2_USCALED] = VK_FORMAT_A2B10G10R10_USCALED_PACK32,
    [PIPE_FORMAT_R10G10B10A2_SSCALED] = VK_FORMAT_A2B10G10R10_SSCALED_PACK32,
    [PIPE_FORMAT_B10G10R10A2_USCALED] = VK_FORMAT_A2R10G10B10_USCALED_PACK32,
    [PIPE_FORMAT_B10G10R10A2_SSCALED] = VK_FORMAT_A2B10G10R10_SSCALED_PACK32,
    [PIPE_FORMAT_R10G10B10A2_UINT] = VK_FORMAT_A2B10G10R10_UINT_PACK32,
    [PIPE_FORMAT_B10G10R10A2_UINT] = VK_FORMAT_A2R10G10B10_UINT_PACK32,


通过对比分析，上述代码中的如下两个对应关系存在问题：

    [PIPE_FORMAT_B10G10R10A2_SNORM] = VK_FORMAT_A2B10G10R10_SNORM_PACK32,
    [PIPE_FORMAT_B10G10R10A2_SSCALED] = VK_FORMAT_A2B10G10R10_SSCALED_PACK32,

应修改为：

    [PIPE_FORMAT_B10G10R10A2_SNORM] = VK_FORMAT_A2R10G10B10_SNORM_PACK32,
    [PIPE_FORMAT_B10G10R10A2_SSCALED] = VK_FORMAT_A2R10G10B10_SSCALED_PACK32,

修改后，Piglit相关测试通过：

1. ```./bin/arb_vertex_type_2_10_10_10_rev-array_types -auto -fbo```

    测试结果：

    ```
    testing: RGBA SINT
    testing: RGBA SNORM
    testing: RGBA UINT
    testing: RGBA UNORM
    testing: BGRA SNORM
    testing: BGRA UNORM
    PIGLIT: {"result": "pass" }

2. ```./bin/draw-vertices-2101010 -auto```
    
    测试结果：

    ```
    testing: RGBA SINT
    testing: RGBA SNORM
    testing: RGBA UINT
    testing: RGBA UNORM
    testing: BGRA SNORM
    testing: BGRA UNORM
    PIGLIT: {"result": "pass" }


## Border Color

### **一、Overview** 

目前大多数bordercolor相关测试均无法通过，但相关测试在去除bordercolor参数后即可通过。包括：

``arb_texture_float`` 相关:

1. ``./bin/texwrap GL_ARB_texture_float bordercolor swizzled -auto -fbo``
2. ``./bin/texwrap GL_ARB_texture_float bordercolor -auto -fbo``

``arb_texture_rectangle`` 相关:

1. ``./bin/texwrap RECT GL_RGBA8 bordercolor -auto -fbo``
2. ``./bin/texwrap RECT GL_RGBA8 proj bordercolor -auto -fbo``

``arb_texture_rg`` 相关:

1. ``./bin/texwrap GL_ARB_texture_rg bordercolor swizzled -auto -fbo``
2. ``./bin/texwrap GL_ARB_texture_rg bordercolor -auto -fbo``
3. ``./bin/texwrap GL_ARB_texture_rg-float bordercolor swizzled -auto -fbo``
4. ``./bin/texwrap GL_ARB_texture_rg-float bordercolor -auto -fbo``

``ext_packed_float`` 相关:

1. ``./bin/texwrap GL_EXT_packed_float bordercolor swizzled -auto -fbo``
2. ``./bin/texwrap GL_EXT_packed_float bordercolor -auto -fbo``

测试结果样例：

    Testing GL_ARB_texture_float.
    Testing the border color only.
    Testing GL_ALPHA16F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (128,125) @ 11,0
    Expected: 0 0 0 229
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_ALPHA16F_ARB, border color only" : "fail"}}
    Testing GL_LUMINANCE16F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 255
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_LUMINANCE16F_ARB, border color only" : "fail"}}
    Testing GL_LUMINANCE_ALPHA16F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 216
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA16F_ARB, border color only" : "fail"}}
    Testing GL_INTENSITY16F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 19
    Observed: 0 0 0 0
    PIGLIT: {"subtest": {"GL_INTENSITY16F_ARB, border color only" : "fail"}}
    Testing GL_RGB16F, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 172 95 255
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_RGB16F, border color only" : "fail"}}
    Testing GL_RGBA16F, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 172 95 216
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_RGBA16F, border color only" : "fail"}}
    Testing GL_ALPHA32F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 0 0 0 216
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_ALPHA32F_ARB, border color only" : "fail"}}
    Testing GL_LUMINANCE32F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 255
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_LUMINANCE32F_ARB, border color only" : "fail"}}
    Testing GL_LUMINANCE_ALPHA32F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 216
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA32F_ARB, border color only" : "fail"}}
    Testing GL_INTENSITY32F_ARB, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 19 19 19
    Observed: 0 0 0 0
    PIGLIT: {"subtest": {"GL_INTENSITY32F_ARB, border color only" : "fail"}}
    Testing GL_RGB32F, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 172 95 255
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_RGB32F, border color only" : "fail"}}
    Testing GL_RGBA32F, border color only
    Fail with LINEAR and GL_CLAMP at (95,125) @ 0,0
    Expected: 19 172 95 216
    Observed: 0 0 0 255
    PIGLIT: {"subtest": {"GL_RGBA32F, border color only" : "fail"}}
    PIGLIT: {"result": "fail" }


### **二、原因分析**

1. 上述测试去除bordercolor参数后均能通过，初步分析相关特性已实现，问题出在bordercolor上

1. 失败样例均为LINEAR + GL_CLAMP

1. Zink到Vulkan 纹理相关转换如下

    ```
    static VkSamplerAddressMode
    sampler_address_mode(enum pipe_tex_wrap filter)
    {
        switch (filter) {
        case PIPE_TEX_WRAP_REPEAT: return VK_SAMPLER_ADDRESS_MODE_REPEAT;
        case PIPE_TEX_WRAP_CLAMP: return VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE; /* not technically correct, but kinda works */
        case PIPE_TEX_WRAP_CLAMP_TO_EDGE: return VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
        case PIPE_TEX_WRAP_CLAMP_TO_BORDER: return VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER;
        case PIPE_TEX_WRAP_MIRROR_REPEAT: return VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT;
        case PIPE_TEX_WRAP_MIRROR_CLAMP: return VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE; /* not technically correct, but kinda works */
        case PIPE_TEX_WRAP_MIRROR_CLAMP_TO_EDGE: return VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE;
        case PIPE_TEX_WRAP_MIRROR_CLAMP_TO_BORDER: return VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE; /* not technically correct, but kinda works */
        }
        unreachable("unexpected wrap");
    }
    ```

    注意到zink中的PIPE_TEX_WRAP_CLAMP映射到VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE，该处的注释也标识了这样技术上会存在一些问题，初步怀疑是由此处导致

    注：Vulkan 纹理支持：

    ```
    typedef enum VkSamplerAddressMode {
        VK_SAMPLER_ADDRESS_MODE_REPEAT = 0,
        VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT = 1,
        VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE = 2,
        VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER = 3,
        // Provided by VK_VERSION_1_2, VK_KHR_sampler_mirror_clamp_to_edge
        VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE = 4,
        VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE_KHR = VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE,
    } VkSamplerAddressMode;
    ```
    缺少```PIPE_TEX_WRAP_CLAMP```和```PIPE_TEX_WRAP_MIRROR_CLAMP```以及```PIPE_TEX_WRAP_MIRROR_CLAMP_TO_BORDER```的直接对应

1. CLAMP, CLAMP_TO_EDGE区别
    - GL_CLAMP ：在边界处双线性处理，
    - GL_CLAMP_TO_EDGE：边界处采用纹理边缘自己的的颜色


    WRT the CLAMP_TO_EDGE and CLAMP distinction, we have a little test app which illustrates this to some degree. Note that many implementations don’t get GL_CLAMP right, but this matters little in games since the GL_CLAMP_TO_EDGE behavior is what the overwhelming majority of developers expect when using GL_CLAMP. TRIBES, for example, shipped with this wrong and had to issue a patch using the GL_CLAMP_TO_EDGE extension to eliminate the black grid lines that showed up on their terrain on implementations which did GL_CLAMP right.

    请注意，许多实现都无法正确使用GL_CLAMP，但这在游戏中意义不大，因为GL_CLAMP_TO_EDGE行为是绝大多数开发人员在使用GL_CLAMP时所期望的行为。 例如，TRIBES附带此错误，必须使用GL_CLAMP_TO_EDGE扩展程序发布修补程序，以消除在正确执行GL_CLAMP的实现中出现在其地形上的黑色网格线。

1. Zink针对性处理
    ```
    static inline bool
    wrap_needs_border_color(unsigned wrap)
    {
        return wrap == PIPE_TEX_WRAP_CLAMP || wrap == PIPE_TEX_WRAP_CLAMP_TO_BORDER ||
                wrap == PIPE_TEX_WRAP_MIRROR_CLAMP || wrap == PIPE_TEX_WRAP_MIRROR_CLAMP_TO_BORDER;
    }

    need_custom |= wrap_needs_border_color(state->wrap_s);
    need_custom |= wrap_needs_border_color(state->wrap_t);
    need_custom |= wrap_needs_border_color(state->wrap_r);

    if (screen->info.have_EXT_custom_border_color &&
        screen->info.border_color_feats.customBorderColorWithoutFormat && need_custom) {
        cbci.sType = VK_STRUCTURE_TYPE_SAMPLER_CUSTOM_BORDER_COLOR_CREATE_INFO_EXT;
        cbci.format = VK_FORMAT_UNDEFINED;
        /* these are identical unions */
        memcpy(&cbci.customBorderColor, &state->border_color, sizeof(union pipe_color_union));
        sci.pNext = &cbci;
        sci.borderColor = VK_BORDER_COLOR_INT_CUSTOM_EXT;
        UNUSED uint32_t check = p_atomic_inc_return(&screen->cur_custom_border_color_samplers);
        assert(check <= screen->info.border_color_props.maxCustomBorderColorSamplers);
        *custom_border_color = true;
   }
    ```

    通过代码调试，发现测试中customBorderColor存储的是float，参考panfrost对应代码：

    ```
    cfg.border_color_r = cso->border_color.f[0];
    cfg.border_color_g = cso->border_color.f[1];
    cfg.border_color_b = cso->border_color.f[2];
    cfg.border_color_a = cso->border_color.f[3];
    ```
    怀疑zink中使用的类型错误，
    ```sci.borderColor = VK_BORDER_COLOR_INT_CUSTOM_EXT;```应该为```sci.borderColor = VK_BORDER_COLOR_FLOAT_CUSTOM_EXT;```

## ARB_seamless_cube_map
### 一、Overview

立方体贴图是一种纹理类型，类似于二维纹理有两个维度。但是，每个mipmap级别有6个面，每个面与其他面具有相同的大小。立方体贴图的宽度和高度必须相同（即：立方体贴图是正方形）。

seamless特性影响的是采样的范围。在cubemap中，面之间不起作用，这将在立方体贴图的面上形成一个接缝。这在过去是一个硬件限制，但现代硬件能够跨立方体面边界进行插值。

### 二、Piglit测试

初始化纹理：

    glBindTexture(GL_TEXTURE_CUBE_MAP_ARB, 1);
    glTexParameteri(GL_TEXTURE_CUBE_MAP_ARB, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP_ARB, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP_ARB, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP_ARB, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP_ARB, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    for (i = 0; i < 6; i++) {
        glTexImage2D(targets[i], 0, GL_RGBA8, 1, 1, 0, GL_RGB, GL_FLOAT, colors[i]);
    }

纹理是6张大小为1*1的图，即一个像素点，点的颜色为：

    static const float colors[6][3] = {
        {1, 0, 0},
        {0, 1, 1},
        {0, 1, 0},
        {1, 0, 1},
        {0, 0, 1},
        {1, 1, 0}
    };

测试一共分为4组：

    glClear(GL_COLOR_BUFFER_BIT);

    draw_quad(10, 10, 0.99, 0, 1);
    draw_quad(40, 10, 1, 0, 0.99);

    glEnable(GL_TEXTURE_CUBE_MAP_SEAMLESS);
    draw_quad(70, 10, 0.99, 0, 1);
    draw_quad(100, 10, 1, 0, 0.99);
    glDisable(GL_TEXTURE_CUBE_MAP_SEAMLESS);

    pass = piglit_probe_pixel_rgb(20, 20, colors[4]) && pass;
    pass = piglit_probe_pixel_rgb(50, 20, colors[0]) && pass;
    pass = piglit_probe_pixel_rgb(80, 20, violet) && pass;
    pass = piglit_probe_pixel_rgb(110, 20, violet) && pass;
    

测试结果中，反而是是开启了GL_TEXTURE_CUBE_MAP_SEAMLESS的两组测试通过，没有开启GL_TEXTURE_CUBE_MAP_SEAMLESS的测试失败：

    
    Mesa: User error: GL_INVALID_OPERATION in glBindTexture(target mismatch)
    Probe color at (20,20)
    Expected: 0.000000 0.000000 1.000000
    Observed: 0.498039 0.000000 0.501961
    Probe color at (50,20)
    Expected: 1.000000 0.000000 0.000000
    Observed: 0.501961 0.000000 0.498039
    PIGLIT: {"result": "fail" }
    
### 三、原因分析

失败样例中，测试分别以（10，10）和（40，10）为顶点，20为边长绘制矩形，采样坐标位于矩形正中间，zink输出的结果为期望结果与相邻纹理图片的线性采样。

在OpenGL中，创建纹理需要同时创建图像与制定采样参数；在Vulkan中，图像与采样器是分开创建/设计的。目前Vulkan的采样器包含如下参数

- sType - Type of the structure. It should be equal to a VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO value.
- spNext - Pointer reserved for extensions.
- flags - Must be set to zero. This parameter is reserved for future use.
- magFilter - Type of filtering (nearest or linear) used for magnification.
- minFilter - Type of filtering (nearest or linear) used for minification.
- mipmapMode - Type of filtering (nearest or linear) used for mipmap lookup.
- addressModeU - Addressing mode for U coordinates that are outside of a <0.0; 1.0> range.
- addressModeV - Addressing mode for V coordinates that are outside of a <0.0; 1.0> range.
- addressModeW - Addressing mode for W coordinates that are outside of a <0.0; 1.0> range.
- mipLodBias - Value of bias added to mipmap’s level of detail calculations. If we want to offset fetching data from a specific mipmap, we can provide a value other than 0.0.
- anisotropyEnable - Parameter defining whether anisotropic filtering should be used.
- maxAnisotropy - Maximal allowed value used for anisotropic filtering (clamping value).
- compareEnable - Enables comparison against a reference value during texture lookups.
- compareOp–Type of comparison performed during lookups if the compareEnable parameter is set to true.
- minLod - Minimal allowed level of detail used during data fetching. If calculated level of detail (mipmap level) is lower than this value, it will be clamped.
- maxLod - Maximal allowed level of detail used during data fetching. If the calculated level of detail (mipmap level) is greater than this value, it will be clamped.
- borderColor - Specifies predefined color of border pixels. Border color is used when address mode includes clamping to border colors.
- unnormalizedCoordinates - Usually (when this parameter is set to false) we provide texture coordinates using a normalized <0.0; 1.0> range. When set to true, this parameter allows us to specify that we want to use unnormalized coordinates and address texture using texels (in a <0; texture dimension> range, similar to OpenGL’s rectangle textures).

换言之，Vulkan目前缺乏立方体问题采样相关参数设置，默认使用的是跨立方体面边界采样插值。
这样做的好处是能够使立方体贴图的边缘更加平滑。


## Provoking Vertex
### 一、Overview
[Provoking Vertex](https://www.khronos.org/opengl/wiki/Primitive#Provoking_vertex) 是输出图元中的一个顶点，该顶点对图元有特殊的意义。例如，当开启平面着色时，只有 provoking vertex 的颜色会被使用，该图元生成的每个片段的输入也都是 provoking vertex 的输出。

| Primitive type | `GL_FIRST_VERTEX_CONVENTION` | `GL_LAST_VERTEX_CONVENTION` |
| ----------- | :-----------: | :-----------: |
| `GL_POINTS`      | `i` | `i` |
| `GL_LINES`      | `2i - 1` | `2i` |
| `GL_LINE_LOOP`      | `i` | `i​ + 1, if i​ < the number of vertices. 1 if i​ is equal to the number of vertices.` |
| `GL_LINE_STRIP`      | `i` | `i + 1` |
| `GL_TRIANGLES`      | `3i - 2` | `3i` |
| `GL_TRIANGLE_STRIP`      | `i` | `i + 2` |
| `GL_TRIANGLE_FAN`      | `i + 1` | `i + 2` |
| `GL_LINES_ADJACENCY`      | `4i - 2` | `4i - 1` |
| `GL_LINE_STRIP_ADJACENCY`      | `i + 1` | `i + 2` |
| `GL_TRIANGLES_ADJACENCY`      | `6i - 5` | `6i - 1` |
| `GL_TRIANGLE_STRIP_ADJACENCY`      | `2i - 1` | `2i + 3` |

> [ARB_provoking_vertex Overview](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_provoking_vertex.txt)  
This extension provides an alternative provoking vertex convention
    for rendering lines, triangles, and (optionally depending on the
    implementation) quads.

### 二、Piglit 测试结果

`./bin/arb-provoking-vertex-clipped-geometry-flatshading -auto`

```
Probe color at (158,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
PIGLIT: {"result": "fail" }
```

`./bin/arb-provoking-vertex-render -auto`

```
GL_QUADS_FOLLOW_PROVOKING_VERTEX_CONVENTION = 0
Probe color at (40,80)
  Expected: 0.000000 1.000000 0.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,80)
  Expected: 1.000000 1.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
Failure for GL_LINES, GL_LAST_VERTEX_CONVENTION
Probe color at (40,80)
  Expected: 0.000000 1.000000 0.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,80)
  Expected: 0.000000 0.000000 1.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_LINE_STRIP, GL_LAST_VERTEX_CONVENTION
Probe color at (40,80)
  Expected: 0.000000 1.000000 0.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,80)
  Expected: 0.000000 0.000000 1.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_LINE_LOOP, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 0.000000 0.000000 1.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_TRIANGLES, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 0.000000 0.000000 1.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_TRIANGLE_STRIP, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 0.000000 0.000000 1.000000
  Observed: 0.000000 1.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
Failure for GL_TRIANGLE_FAN, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_QUADS, GL_FIRST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_QUADS, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 1.000000 0.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
Failure for GL_QUAD_STRIP, GL_FIRST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 1.000000 1.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Failure for GL_QUAD_STRIP, GL_LAST_VERTEX_CONVENTION
Probe color at (40,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 1.000000 0.000000
Probe color at (120,79)
  Expected: 1.000000 0.000000 0.000000
  Observed: 0.000000 0.000000 1.000000
Failure for GL_POLYGON, GL_LAST_VERTEX_CONVENTION
PIGLIT: {"result": "fail" }
```

`./bin/provoking-vertex -auto`

```
Probe color at (225,90)
  Expected: 0.000000 0.000000 0.900000
  Observed: 0.898039 0.000000 0.000000
PIGLIT: {"result": "fail" }
```

### 三、原因分析

OpenGL 的 provoking vertex 支持 `GL_FIRST_VERTEX_CONVENTION` 和 `GL_LAST_VERTEX_CONVENTION` 两种模式，为了排查问题，对这两种分别进行了测试。

`./bin/arb-provoking-vertex-clipped-geometry-flatshading -auto` 的关键绘图指令：

```
glProvokingVertexEXT(GL_LAST_VERTEX_CONVENTION_EXT);

const int y1 = piglit_height / 3;

glBegin(GL_TRIANGLE_STRIP);
    glColor3fv(cyan);
    glVertex3i(piglit_width + 1, y1, 0);
    glColor3fv(yellow);
    glVertex3i(piglit_width + 2, y1, 0);
    glColor3fv(blue);
    glVertex3i(piglit_width + 3, y1, 0);
    glColor3fv(green);
    glVertex3i(piglit_width / 2, y1 * 2, 0);
    glColor3fv(red);
    glVertex3i(piglit_width - 1, y1 * 2, 0);
glEnd();
```

使用的是 `GL_LAST_VERTEX_CONVENTION`，修改成 `GL_FIRST_VERTEX_CONVENTION` 的等价形式：

```
glProvokingVertexEXT(GL_FIRST_VERTEX_CONVENTION_EXT);

const int y1 = piglit_height / 3;

glBegin(GL_TRIANGLE_STRIP);
    glColor3fv(blue);
    glVertex3i(piglit_width + 1, y1, 0);
    glColor3fv(green);
    glVertex3i(piglit_width + 2, y1, 0);
    glColor3fv(red);
    glVertex3i(piglit_width + 3, y1, 0);
    glColor3fv(green);
    glVertex3i(piglit_width / 2, y1 * 2, 0);
    glColor3fv(red);
    glVertex3i(piglit_width - 1, y1 * 2, 0);
glEnd();
```

测试通过：

```
PIGLIT: {"result": "pass" }
```

等价的绘图指令，`GL_FIRST_VERTEX_CONVENTION` 设置能通过，而 `GL_LAST_VERTEX_CONVENTION` 设置无法通过，初步怀疑是 zink 对 `GL_LAST_VERTEX_CONVENTION` 的处理有问题。

搜索 zink 和 vulkan 与 provoking vertex 相关的[资料](https://github.com/KhronosGroup/Vulkan-Docs/issues/1173)发现：

>For API comparison purposes, in D3D, Metal, and Vulkan the first vertex is always the provoking vertex while in OpenGL both the first vertex or the last vertex can be the provoking vertex.

说的是 vulkan 只支持以第一个顶点作为 provoking vertex。

所以最终问题定位到了 vulkan，vulkan 不支持与 `GL_LAST_VERTEX_CONVENTION` 相对应的绘图设置。

### 四、进一步验证

`./bin/arb-provoking-vertex-render -auto` 测试中 quads 和 `GL_LAST_VERTEX_CONVENTION` 相关的注释掉：

```cpp
static const GLenum modes[] = {
    GL_LINES,
    GL_LINE_STRIP,
    GL_LINE_LOOP,
    GL_TRIANGLES,
    GL_TRIANGLE_STRIP,
    GL_TRIANGLE_FAN,
    // GL_QUADS,
    // GL_QUAD_STRIP,
    GL_POLYGON

};
int i;
bool pass = true;

glViewport(0, 0, piglit_width, piglit_height);
glShadeModel(GL_FLAT);

glGetBooleanv(GL_QUADS_FOLLOW_PROVOKING_VERTEX_CONVENTION, &quads_pv);
printf("GL_QUADS_FOLLOW_PROVOKING_VERTEX_CONVENTION = %u\n", quads_pv);

for (i = 0; i < ARRAY_SIZE(modes); i++) {
    pass = test_mode(modes[i], GL_FIRST_VERTEX_CONVENTION) && pass;
    // pass = test_mode(modes[i], GL_LAST_VERTEX_CONVENTION) && pass;
}

pass = piglit_check_gl_error(GL_NO_ERROR) && pass;

piglit_report_result(pass ? PIGLIT_PASS : PIGLIT_FAIL);
```

测试结果：

```
GL_QUADS_FOLLOW_PROVOKING_VERTEX_CONVENTION = 0
PIGLIT: {"result": "pass" }
```

`./bin/provoking-vertex -auto` 测试中注释掉 `GL_LAST_VERTEX_CONVENTION` 相关的：

```cpp
...
pass = pass && piglit_probe_pixel_rgb(150, 90, red);
pass = pass && piglit_probe_pixel_rgb(150, 170, red);

// pass = pass && piglit_probe_pixel_rgb(225, 90, blue);
// pass = pass && piglit_probe_pixel_rgb(225, 170, blue);

glFinish();
piglit_present_results();
```

测试结果：

```
PIGLIT: {"result": "pass" }
```

## arb_texture_float

此扩展添加了带有 16 位和 32 位浮点数的纹理内部格式。32 位浮点数采用标准 IEEE 浮点格式。16 位浮点数具有 1 个符号位，5 个指数位和 10 个尾数位。

### 一、FBO Blending Formats

将源纹理 ($s$) 和目标纹理 ($d$) 通过指定的混合方式进行混合（先绘制目标纹理，再绘制源纹理）。

| | |
| --- | --- |
| `CLEAR` | $0$ |
| `NOR` | $\neg (s \lor d)$ |
| `AND_INVERTED` | $\neg s \land d$ |
| `COPY_INVERTED` | $\neg s$ |
| `AND_REVERSE` | $s \land \neg d$ |
| `INVERT` | $\neg d$ |
| `XOR` | $s \oplus d$ |
| `NAND` | $\neg (s \land d)$ |
| `AND` | $s \land d$ |
| `EQUIV` | $\neg (s \oplus d)$ |
| `NOOP` | $d$ |
| `OR_INVERTED` | $\neg s \lor d$ |
| `COPY` | $s$ |
| `OR_REVERSE` | $s \lor \neg d$ |
| `OR` | $s \lor d$ |
| `SET` | $1$ |

```
Accepted by the <internalFormat> parameter of TexImage1D,
TexImage2D, and TexImage3D:

    RGBA32F_ARB                      0x8814
    RGB32F_ARB                       0x8815
    ALPHA32F_ARB                     0x8816
    INTENSITY32F_ARB                 0x8817
    LUMINANCE32F_ARB                 0x8818
    LUMINANCE_ALPHA32F_ARB           0x8819
    RGBA16F_ARB                      0x881A
    RGB16F_ARB                       0x881B
    ALPHA16F_ARB                     0x881C
    INTENSITY16F_ARB                 0x881D
    LUMINANCE16F_ARB                 0x881E
    LUMINANCE_ALPHA16F_ARB           0x881F
```

### 二、Piglit 测试结果

`./bin/fbo-blending-formats GL_ARB_texture_float -auto -fbo`

```
Using test set: GL_ARB_texture_float
Testing GL_RGB16F
Probe color at (192,0)
  Expected: 0.800000 0.300000 0.500000 1.000000
  Observed: 0.849609 0.349854 0.599609 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_RGB16F" : "fail"}}
Testing GL_RGBA16F
PIGLIT: {"subtest": {"GL_RGBA16F" : "pass"}}
Testing GL_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_ALPHA16F_ARB" : "pass"}}
Testing GL_LUMINANCE16F_ARB
Probe color at (192,0)
  Expected: 0.800000 0.000000 0.000000 1.000000
  Observed: 0.849609 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_LUMINANCE16F_ARB" : "fail"}}
Testing GL_LUMINANCE_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA16F_ARB" : "pass"}}
Testing GL_INTENSITY16F_ARB
Probe color at (192,0)
  Expected: 0.810000 0.000000 0.000000 1.000000
  Observed: 0.849609 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_INTENSITY16F_ARB" : "fail"}}
Testing GL_RGB32F
Probe color at (192,0)
  Expected: 0.800000 0.300000 0.500000 1.000000
  Observed: 0.850000 0.350000 0.600000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_RGB32F" : "fail"}}
Testing GL_RGBA32F
PIGLIT: {"subtest": {"GL_RGBA32F" : "pass"}}
Testing GL_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_ALPHA32F_ARB" : "pass"}}
Testing GL_LUMINANCE32F_ARB
Probe color at (192,0)
  Expected: 0.800000 0.000000 0.000000 1.000000
  Observed: 0.850000 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_LUMINANCE32F_ARB" : "fail"}}
Testing GL_LUMINANCE_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA32F_ARB" : "pass"}}
Testing GL_INTENSITY32F_ARB
Probe color at (192,0)
  Expected: 0.810000 0.000000 0.000000 1.000000
  Observed: 0.850000 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_INTENSITY32F_ARB" : "fail"}}
PIGLIT: {"result": "fail" }
```

### 三、原因分析

| Pass 的格式 | Fail 的格式 |
| --- | --- |
| `RGBA16F` | `RGB16F` |
| `ALPHA16F` |  |
| `LUMINANCE_ALPHA16F` | `LUMINANCE16F` |
|  | `INTENSITY16F` |
| `RGBA32F` | `RGB32F` |
| `ALPHA32F` |  |
| `LUMINANCE_ALPHA32F` | `LUMINANCE32F` |
|  | `INTENSITY32F` |

混合函数 (`glBlendFunc(blendsrc, blenddst)`)：

| `blendsrc` | `blenddst` | 结果（仅考虑 Fail 的格式） |
| --- | --- | --- |
| `GL_CONSTANT_COLOR` | `GL_ONE_MINUS_CONSTANT_COLOR` | Pass |
| `GL_DST_COLOR` | `GL_ONE_MINUS_DST_COLOR` | Pass |
| `GL_SRC_COLOR` | `GL_ONE_MINUS_SRC_COLOR` | Pass |
| `GL_DST_ALPHA` | `GL_ONE_MINUS_DST_ALPHA` | Fail |
| `GL_SRC_ALPHA` | `GL_ONE_MINUS_SRC_ALPHA` | Pass |

也就是说，失败的 case 都是格式中没有 alpha 通道且混合函数为 `glBlendFunc(GL_DST_ALPHA, GL_ONE_MINUS_DST_ALPHA)`。

该混合函数下没有 alpha 通道的混合规则：

> When there are no bits for the alpha channel, we always expect to read an alpha value of 1.0.

> DST_ALPHA/ONE_MINUS_DST_ALPHA with an implicit destination alpha value of 1.0 means that the result color should be identical to the source color, (if there are any bits to store that color that is).

为了发现问题，选择了 `RGB16F` 和 `RGBA16F` 两种格式使用 GDB 一步步调试与纹理格式相关的函数调用，发现在函数 `teximage (src/mesa/main/teximage.c)` 中

```
texFormat = _mesa_choose_texture_format(ctx, texObj, target, level,
                                        internalFormat, format, type);
```

返回了相同的 PIPE 格式 `PIPE_FORMAT_R16G16B16A16_FLOAT`。

进一步追踪调用栈：

```
#0  0x00007ffff6b85abe in zink_is_format_supported (pscreen=0x7ffff6f8f7c0 <format_info>, format=6408,
    target=4294957840, sample_count=0, storage_sample_count=4294958032, bind=0)
    at ../src/gallium/drivers/zink/zink_screen.c:713
#1  0x00007ffff619ca3e in find_supported_format (screen=0x5555555d3f10, formats=0x7ffff6c79ae8 <format_map+6344>,
    target=PIPE_TEXTURE_2D, sample_count=0, storage_sample_count=0, bindings=10, allow_dxt=1 '\001')
    at ../src/mesa/state_tracker/st_format.c:1066
#2  0x00007ffff619cc58 in st_choose_format (st=0x555555d94ff0, internalFormat=34843, format=6408, type=5126,
    target=PIPE_TEXTURE_2D, sample_count=0, storage_sample_count=0, bindings=10, swap_bytes=false, allow_dxt=true)
    at ../src/mesa/state_tracker/st_format.c:1162
#3  0x00007ffff619d05c in st_ChooseTextureFormat (ctx=0x555555d32af0, target=3553, internalFormat=34843, format=6408,
    type=5126) at ../src/mesa/state_tracker/st_format.c:1327
#4  0x00007ffff648bdef in _mesa_choose_texture_format (ctx=0x555555d32af0, texObj=0x555555dbba90, target=3553, level=0,
    internalFormat=34843, format=6408, type=5126) at ../src/mesa/main/teximage.c:2834
#5  0x00007ffff648c88a in teximage (no_error=false, pixels=0x0, imageSize=0, type=5126, format=6408, border=0, depth=1,
    height=256, width=256, internalFormat=34843, level=0, target=3553, texObj=0x555555dbba90, dims=2,
    compressed=0 '\000', ctx=0x555555d32af0) at ../src/mesa/main/teximage.c:3065
#6  teximage_err (ctx=0x555555d32af0, compressed=0 '\000', dims=2, target=3553, level=0, internalFormat=34843,
    width=256, height=256, depth=1, border=0, format=6408, type=5126, imageSize=0, pixels=0x0)
    at ../src/mesa/main/teximage.c:3195
#7  0x00007ffff648f284 in _mesa_TexImage2D (target=3553, level=0, internalFormat=34843, width=256, height=256, border=0,
    format=6408, type=5126, pixels=0x0) at ../src/mesa/main/teximage.c:3266
#8  0x0000555555558b6e in test_format (format=0x55555555a780 <arb_texture_float>)
    at /home/zink/piglit/tests/fbo/fbo-blending-formats.c:154
#9  0x0000555555557bbb in fbo_formats_display (test_format=0x555555557f29 <test_format>)
    at /home/zink/piglit/tests/fbo/fbo-formats.h:712
#10 0x0000555555559a4d in piglit_display () at /home/zink/piglit/tests/fbo/fbo-blending-formats.c:422
#11 0x00007ffff7f0591e in run_test (gl_fw=0x55555556feb0, argc=2, argv=0x7fffffffe2b8)
    at /home/zink/piglit/tests/util/piglit-framework-gl/piglit_fbo_framework.c:52
#12 0x00007ffff7ef5e26 in piglit_gl_test_run (argc=2, argv=0x7fffffffe2b8, config=0x7fffffffe160)
    at /home/zink/piglit/tests/util/piglit-framework-gl.c:229
#13 0x0000555555557d25 in main (argc=2, argv=0x7fffffffe2b8) at /home/zink/piglit/tests/fbo/fbo-blending-formats.c:46
```

定位到了 zink 层面的函数 `zink_is_format_supported`，里面有部分代码：

```cpp
if ((bind & PIPE_BIND_SAMPLER_VIEW) || (bind & PIPE_BIND_RENDER_TARGET)) {
    /* if this is a 3-component texture, force gallium to give us 4 components by rejecting this one */
    unsigned bpb = util_format_get_blocksizebits(format);
    if (bpb == 24 || bpb == 48 || bpb == 96)
      return false;
}
```

按照代码的逻辑，不支持 3 个通道的纹理格式，会强制转换为 4 通道。为了测试是否是该问题，将里面的判断注释了：

```cpp
if ((bind & PIPE_BIND_SAMPLER_VIEW) || (bind & PIPE_BIND_RENDER_TARGET)) {
    /* if this is a 3-component texture, force gallium to give us 4 components by rejecting this one */
    // unsigned bpb = util_format_get_blocksizebits(format);
    // if (bpb == 24 || bpb == 48 || bpb == 96)
    //   return false;
}
```

重新编译 mesa，运行 piglit 测试：

```
Using test set: GL_ARB_texture_float
Testing GL_RGB16F
PIGLIT: {"subtest": {"GL_RGB16F" : "pass"}}
Testing GL_RGBA16F
PIGLIT: {"subtest": {"GL_RGBA16F" : "pass"}}
Testing GL_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_ALPHA16F_ARB" : "pass"}}
Testing GL_LUMINANCE16F_ARB
Probe color at (192,0)
  Expected: 0.800000 0.000000 0.000000 1.000000
  Observed: 0.849609 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_LUMINANCE16F_ARB" : "fail"}}
Testing GL_LUMINANCE_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA16F_ARB" : "pass"}}
Testing GL_INTENSITY16F_ARB
Probe color at (192,0)
  Expected: 0.810000 0.000000 0.000000 1.000000
  Observed: 0.849609 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_INTENSITY16F_ARB" : "fail"}}
Testing GL_RGB32F
Probe color at (192,0)
  Expected: 0.800000 0.300000 0.500000 1.000000
  Observed: 0.850000 0.350000 0.600000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_RGB32F" : "fail"}}
Testing GL_RGBA32F
PIGLIT: {"subtest": {"GL_RGBA32F" : "pass"}}
Testing GL_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_ALPHA32F_ARB" : "pass"}}
Testing GL_LUMINANCE32F_ARB
Probe color at (192,0)
  Expected: 0.800000 0.000000 0.000000 1.000000
  Observed: 0.850000 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_LUMINANCE32F_ARB" : "fail"}}
Testing GL_LUMINANCE_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA32F_ARB" : "pass"}}
Testing GL_INTENSITY32F_ARB
Probe color at (192,0)
  Expected: 0.810000 0.000000 0.000000 1.000000
  Observed: 0.850000 0.000000 0.000000 1.000000
  when testing FBO result, blending with DST_ALPHA.
PIGLIT: {"subtest": {"GL_INTENSITY32F_ARB" : "fail"}}
PIGLIT: {"result": "fail" }
```

发现格式 `RGB16F` 的测试确实通过了。

为了验证其他格式，在一番探索之后（主要是 vulkan 还不支持许多 GL 的格式），在 mesa 上层加入了一些代码：

```cpp
texFormat = _mesa_choose_texture_format(ctx, texObj, target, level,
                                        internalFormat, format, type);
switch (internalFormat) {
case GL_RGB16F: texFormat = PIPE_FORMAT_R16G16B16_FLOAT; break;
case GL_RGBA16F: texFormat = PIPE_FORMAT_R16G16B16A16_FLOAT; break;
case GL_ALPHA16F_ARB: texFormat = PIPE_FORMAT_R16G16B16A16_FLOAT; break;
case GL_LUMINANCE16F_ARB: texFormat = PIPE_FORMAT_R16G16B16_FLOAT; break;
case GL_LUMINANCE_ALPHA16F_ARB: texFormat = PIPE_FORMAT_R16G16B16A16_FLOAT; break;
case GL_INTENSITY16F_ARB: texFormat = PIPE_FORMAT_R16G16B16_FLOAT; break;
case GL_RGB32F: texFormat = PIPE_FORMAT_R32G32B32_FLOAT; break;
case GL_RGBA32F: texFormat = PIPE_FORMAT_R32G32B32A32_FLOAT; break;
case GL_ALPHA32F_ARB: texFormat = PIPE_FORMAT_R32G32B32A32_FLOAT; break;
case GL_LUMINANCE32F_ARB: texFormat = PIPE_FORMAT_R32G32B32_FLOAT; break;
case GL_LUMINANCE_ALPHA32F_ARB: texFormat = PIPE_FORMAT_R32G32B32A32_FLOAT; break;
case GL_INTENSITY32F_ARB: texFormat = PIPE_FORMAT_R32G32B32_FLOAT; break;
default: break;
}
```

进行强制的格式转换。

编译后测试：

```
Using test set: GL_ARB_texture_float
Testing GL_RGB16F
PIGLIT: {"subtest": {"GL_RGB16F" : "pass"}}
Testing GL_RGBA16F
PIGLIT: {"subtest": {"GL_RGBA16F" : "pass"}}
Testing GL_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_ALPHA16F_ARB" : "pass"}}
Testing GL_LUMINANCE16F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE16F_ARB" : "pass"}}
Testing GL_LUMINANCE_ALPHA16F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA16F_ARB" : "pass"}}
Testing GL_INTENSITY16F_ARB
PIGLIT: {"subtest": {"GL_INTENSITY16F_ARB" : "pass"}}
Testing GL_RGB32F - fbo incomplete (status = GL_FRAMEBUFFER_UNSUPPORTED)
PIGLIT: {"subtest": {"GL_RGB32F" : "skip"}}
Testing GL_RGBA32F
PIGLIT: {"subtest": {"GL_RGBA32F" : "pass"}}
Testing GL_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_ALPHA32F_ARB" : "pass"}}
Testing GL_LUMINANCE32F_ARB - fbo incomplete (status = GL_FRAMEBUFFER_UNSUPPORTED)
PIGLIT: {"subtest": {"GL_LUMINANCE32F_ARB" : "skip"}}
Testing GL_LUMINANCE_ALPHA32F_ARB
PIGLIT: {"subtest": {"GL_LUMINANCE_ALPHA32F_ARB" : "pass"}}
Testing GL_INTENSITY32F_ARB - fbo incomplete (status = GL_FRAMEBUFFER_UNSUPPORTED)
PIGLIT: {"subtest": {"GL_INTENSITY32F_ARB" : "skip"}}
PIGLIT: {"result": "pass" }
```

测试通过。
