> 翻译自 _[internals.md](internals.md)_.

# 星图网页引擎内部结构

# 文件结构

    ext_src/            所有依赖项的源代码
    src/algos/          通用天文算法代码，仅依赖增强型天体位置计算库（ERFA）和C语言标准库
    src/modules/        所有功能模块代码（恒星、银河等）
    src/projections/    所有投影算法代码
    src/utils/          工具类代码

    tools/              各类Python脚本
    html/               用于测试JS版本的简易网页
    data/               数据文件目录（需通过运行tools/makedata.py脚本生成填充数据）

# 代码结构

尽管代码基于 C 语言编写，但采用了轻量级的面向对象架构，所有类均继承自根类 `obj_t`，形成单一继承体系。

    +-------+
    | obj_t | 根类
    +-------+
        ^
        |
        +----------------+---------------+--------------+-------------
        |                |               |              |
    +---------+   +------------+   +---------+   +------------+   +---
    | core_t  |   | observer_t |   | stars_t |   | milkyway_t |   | ... 子类
    +---------+   +------------+   +---------+   +------------+   +---

继承机制的实现方式为：将 `obj_t` 类型作为结构体的首个成员，示例如下：

    struct my_obj_t {
        obj_t obj;
        // 其他成员变量
    };

方法和类属性均在 `obj_klass_t` 结构体实例中定义（定义在 obj.h 中）。

例如要定义一个可渲染对象的类，我们需要声明一个全局的 `obj_klass_t` 实例，并将渲染方法指针设置为具有正确签名的函数：

    static int my_render(const obj_t *obj, const painter_t *painter);

    static obj_klass_t my_obj_klass = {
        .render = my_render,
    };

为了创建给定类的新对象实例，我们必须将 obj 属性的 klass 指针分配给相应的 klass 实例。我们还需要指定对象的大小、ID 和父对象。这些都可以通过 `obj_create` 函数完成：

    obj_t *obj;
    obj = obj_create(&my_obj_klass, (int)sizeof(my_obj_t), "MyID", parent)

为简化调用，也可使用 OBJ_CREATE 宏：

    obj_t *obj;
    obj = OBJ_CREATE(my_obj, "MyID", parent);

当前支持定义的类方法包括：

    create      创建类的新实例
    init        初始化对象，在 create 方法调用后调用（如果指定）
    del         当对象引用计数降为零时调用
    update      每个程序迭代调用
    render      每个渲染调用
    get         用于根据查询字符串查找子对象
    gui         用于 GUI 交互
    list        列出模块中定义的天空对象

有两种特殊情况的对象：天空对象和模块。

天空对象代表天文源。它们必须实现 update 方法，该方法应设置 obj.pos 结构体的值：

    // equ, J2000.0, AU geocentric pos and speed.历元、地心坐标系下的位置与速度，单位 AU
    double pvg[2][3];
    // Can be set to INFINITY for stars that are too far to have a对于距离过远、无法用 AU 表示位置的恒星，可设为 INFINITY
    // position un AU.
    double unit;
    double g_ra, g_dec;
    double ra, dec;
    double az, alt;

模块对象在启动时自动创建，至少必须定义 create 方法。为了将类注册为模块，我们必须在模块源文件中放入 MODULE_REGISTER 宏。（内部使用 gcc 构造函数属性）。例如，`src/modules/cardinal.c` 是一个简单的模块代码。

# 属性

除对象方法与结构体成员外，类还可定义动态属性。动态属性具备以下优势：

- 可从 JavaScript 访问
- 可为属性赋值回调函数，当属性值改变时调用
- 可为属性赋值元数据，如文档、可能值列表等，简化 GUI 生成
- 属性访问 API 提供自动类型转换，例如： 可直接传入天区对象为位置属性赋值。

为对象添加动态属性时，需在`obj_klass_t` 结构体实例中定义 `attributes` 成员。以观测者类（observer_t）的经度属性为例：

    ATTR("longitude", .offset = offsetof(observer_t, elong), .base = "f"),

该示例定义了名为 longitude 的属性，类型为双精度浮点数（base = "f"），本质是对象结构体成员 `elong` 的别名。

我们可以像这样读取属性值：

    double elong;
    obj_call(obj, "longitude", ">f", &elong);

这样设置属性值：

    obj_call(obj, "longitude", "f", 1.4);

`obj_call` 语法定义如下:

    int obj_call(obj_t *obj, const char *func, const char *sig, ...);

# 栈

Internally an attribute is actually a function operating on a stack. In
the case of a simple attribute, the function first set the value if the
passed stack is not empty, and then push the value in the stack.

`obj_call` works by first creating the stack, then calling the function,
and finally getting the output values from the stack.

The signature argument has the form:

    INPUTS ['>' OUTPUTS]

Where inputs and outputs are a list of variables declaration:

    TYPE[size]

Types are:

    'i': int
    'f': double
    'b': boolean
    's': string
    'p': pointer

Instead of using `obj_call`, we could also manually manipulate the stack:

    const attribute_t *attr = obj_get_attr(obj, "longitude");
    stack_t *s;

    // Set the longitude:
    s = stack_create();
    stack_push_f(s, 1.4);
    attr->fn(obj, attr, s);
    stack_delete(s);

    // Get the longitude:
    s = stack_create();
    attr->fn(obj, attr, s);
    elong = stack_get_f(s, -1); // Last value on the stack.
    stack_delete(s);

The `stack_push` and `stack_get` are convenience functions that allow to
push or get several values from a stack in a single call using a signature
string.

In addition to the base type (int, double, string, boolean), we can also
assign an type name to a stack value to give it some semantic meaning.

All the types are defined in src/types:

    - `d_angle`
    - `h_angle`
    - `mjd`
    - `color`
    - `obj`
    - `v3`
    - `v3radec`
    - `v3altaz`
    - `dist`
    - `mag`

This is mostly used so that we can convert those types to string values.

# Assets manager

All the assets are compiled into the binary. The tool makedata.py generates
the file src/assets.inl, that is included by assets.c. The function
`asset_get_data` is used to retrieve any asset by url:

    asset_get_data("asset://some_asset", &size, &code)

As a convenience, if we use a normal url, the asset manager will try to load
the data using an http request or direct file access.

# Ephemeris algorithm

I try to use the erfa (open source version of the sofa library) as much as
possible. All the code is in `ext_src/erfa/erfa.c`.

Erfa is used to compute:

    - Earth rotation and position.
    - Star positions.
    - Major planets positions.
    - Referential conversion matrices.
    - Time conversion

Some additional code not covered by erfa is in src/algos. The rule is that
all the code in this dir should only depend on erfa and nothing else.

    - deltat computation (very basic for the moment).
    - moon position.
    - saturn ring.
    - some more time manipulation functions.
    - some convenience string format functions.

# Projections

A projection represents any function that goes from the 3D space to the
2D and/or reverse. Some projections also support projecting to OpenGL clipping
space.

All the projections are represented by the type `projection_t`.
Currently we support the following projections:

    - Perspective
    - Stereographic
    - Mercator
    - Toast     (only backward)
    - Healpix   (only backward)

The signature of the projection function is:

    // Projection flags.
    enum {
        PROJ_NO_CLIP            = 1 << 0,
        PROJ_BACKWARD           = 1 << 1,
        PROJ_TO_NDC_SPACE       = 1 << 2,
        PROJ_ALREADY_NORMALIZED = 1 << 3,
    };

    bool project(const projection_t *proj, int flags,
                 int out_dim, const double *v, double *out);

Example of usages:

    // Project a 3d pos to clipping space:
    double pos[3] = {1, 0, 0};
    double out[4];
    project(proj, 0, 4, pos, out);

    // Optimization if we know the input is normalized:
    double pos[3] = {1, 0, 0};
    double out[4];
    project(proj, PROJ_ALREADY_NORMALIZED, 4, pos, out);

    // Project a 3d pos to NDC space (xyz divided by w):
    double pos[3] = {1, 0, 0};
    double out[3];
    project(proj, PROJ_TO_NDC_SPACE, 3, pos, out);

    // Project from a 2d pos back to the sphere.
    double pos[2] = {0.5, 0.5};
    double out[3];
    project(proj, PROJ_BACKWARD, 3, pos, out);

In addition we can check if a projecting a line or quad would intersects a
discontinuity:

    int projection_intersect_discontinuity(const projection_t *proj,
                                           double (*p)[3], int n);

In case of discontinuity some projections can implement a split method,
that returns two new projections that are similar to the original one but
with a different discontinuity position.

    P                         P1                     P2
    +---------------+         +---------------+      +---------------+
    |               |         |               |      |               |
    |--A         B--|      B--|--A            |      |            B--|--A
    |               |         |               |      |               |
    |               |         |               |      |               |
    +---------------+         +---------------+      +---------------+

    For example here we try to project a segment using a cylindrical
    projection P.  The projection returns the points A and B, but also tells
    us that we are facing an discontinuty.  We split the projection into
    P1 and P2, each of them projecting the segment A-B properly without
    discontinuty.

# Quad rendering

The function used to render a quad mapped into the sky sphere is `paint_quad`:

    int paint_quad(const painter_t *painter,
                   texture_t *tex,
                   double uv[4][2],
                   const projection_t *proj,
                   int grid_size);

We don't pass the actual 3d coordinates of the quad, but instead the quad
projected into the UV space and the projection function used to go from
the 3d space to the UV space. That way there is no ambiguity as to the
exact position of each pixel in the texture.

The chain of coordinates transformations is as follow:

    Texture UV coordinates (2d, 0 to 1)
            |
            | Inverse projection `*proj` (for example healpix)
            |
            v
    3d model sphere coordinates (3d vector)
            |
            | Multiplication by 3x3 mat `painter->rm2h`
            | (default to identity)
            |
            v
    Horizontal cartesian coordinates (without refraction)
            |
            | Optional refraction computation.
            |
            v
    Observed cartesian coordinates
            |
            | Multiplication by 3x3 mat `painter->ro2v`.
            | default to the rotation for observer altitude and azimuth
            | plus Z up to Y up switch.
            |
            v
    OpenGL view coordinates
            |
            | Render projection `painter->proj`.
            | Usually set by the core depending on user preferences.
            | If there is any effects like refraction it is also applied.
            |
            v
    OpenGL clipping coordinates

Note that there are two projections used: the first one defines how the texture
itself projects into the sphere (argument `proj`), it depends on the texture
used. The second one defines how to project the sphere into the screen.

Internally the `paint_quad`function will try to subdivide the original UV
surface into smaller parts so that we don't have projection errors, and
it also take care of discontinuities. We can specify the number of
subdivisions we want with the `grid_size` argument.

# `traverse_surface` function

This high level function can be used to split an UV coordinate quad into
smaller tiles. It is useful for hierarchical data structure used for example
in the stars module or healpix surveys.

        level 0          level 1          level 2
       +-----------+    +-----+-----+    +--+--+--+--+
       |           |    |     |     |    |  |  |  |  |
       |           |    |     |     |    +--+--+--+--+
       |           |    |     |     |    |  |  |  |  |
       |           |    +-----+-----+    +--+--+--+--+
       |           |    |     |     |    |  |  |  |  |
       |           |    |     |     |    +--+--+--+--+
       |           |    |     |     |    |  |  |  |  |
       +-----------+    +-----+-----+    +--+--+--+--+

The function takes a callback, and call it for each tile at every level,
starting from 0. We can control how deep we go, if we filter non visible
tiles, and if we handle discontinuities split using the return value of the
callback.

The way it works is that for each tile the function will first check if
it is visible, then check if its projection intersects a discontinuity,
in which case it attempts to split the projection.

The callback get called at every stage of the algorithm, and can decide to:

- stop the tests of this tile and not go to deeper tiles (return 0).
- stop the tests, but go to deeper tiles (return 1).
- keep the normal test process, and go to deeper tiles (return 2).
- totally abort the traversal of all tiles (return 3).

Example of usages:

To visit all the tiles up to a certain level, no matter if they are visible
or not:

    if (node->level > max_level) return 0;
    ...
    return 1;

To visit all the visible tiles, up to a certain level, but ignore the
discontinuities (for example used to render the stars tiles).

    if (step == 0) return 2; // Check for visibiliy and call again if so.
    if (node->level > max_level) return 0; // Stop here.
    ...
    return 1;

To visit all the visible tiles, and also take care of the discontinuities
(used in the hips module):

    if (step < 2) return 2; // Check both visible and discontinuities.
    if (node->level > max_level) return 0; // Stop here.
    ...
    return 1;
