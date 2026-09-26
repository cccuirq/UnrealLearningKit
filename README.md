# Unreal Learning Kit

Unreal Learning Kit 是一个面向游戏设计、机器人和积木编程教学的 Unreal Engine 项目。本仓库包含课程示例关卡、蓝图、素材和相关配置。

## 环境要求

- Unreal Engine 5.4
- 支持 Unreal Engine 的 Windows 或 macOS 开发环境

项目文件：[`UnrealLearningKit_54.uproject`](UnrealLearningKit_54.uproject)。在 Unreal Engine 5.4 中打开该文件即可载入项目。

## 内容

- `Content/BlockGame`：积木游戏示例
- `Content/Hour_of_Code`：Hour of Code 教学关卡与蓝图
- `Content/LearningKit_Games`：Learning Kit 游戏示例及 MP2 蓝图资源
- `Config`：项目配置

## 版本控制说明

Unreal 的 `.uasset` 和 `.umap` 资源使用 Git LFS 管理。克隆仓库后请安装并启用 Git LFS，再获取 LFS 文件：

```sh
git lfs install
git lfs pull
```

打开项目后，Unreal Engine 可能会在本机生成 `Intermediate`、`Saved` 和 `DerivedDataCache` 等目录；这些生成文件不会提交到仓库。
