> 翻译自 \_[README.md](README.md)  
> 原网站：[git branch -M master main](https://www.stellarium-labs.com/)  
> 原 github 仓库：[stellarium/stellarium-web-engine](https://github.com/stellarium/stellarium-web-engine)

# 星图网页工程

## 关于

Stellarium Web Engine 是一款基于 WebGL 技术开发的 JavaScript 天文馆渲染引擎，可嵌入网站中运行。

## 功能

- 大气模拟。
- Gaia 星数据库访问（超过 1 亿颗星）。
- HiPS surveys 渲染。
- 高分辨率行星纹理。
- 星座。
- 支持在天空视图中添加图层和形状。
- 地貌。

## 构建 JavaScript 版本

您需要确保已安装 emscripten 和 sconstruct。

    # 设置 emscripten 路径.
    source $PATH_TO_EMSDK/emsdk_env.sh

    # 构建 stellarium-web-engine.js 和 stellarium-web-engine.wasm
    # 这将同时复制文件到 html/static/js
    make js

    # 现在请参阅 apps/simple-html/ 来尝试在浏览器中运行该库。

## 贡献

为了使您的贡献能够被接受，您需要签署 [Stellarium Web 贡献者许可协议 (CLA)](doc/cla/sign-cla.md)。
