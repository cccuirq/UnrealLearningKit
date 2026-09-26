# Unreal Learning Kit

Unreal Learning Kit is an Unreal Engine project for teaching game design, robotics, and block-based programming. This repository contains course example levels, Blueprints, assets, and project configuration.

## Requirements

- Unreal Engine 5.4
- A Windows or macOS development environment that supports Unreal Engine

Open [`UnrealLearningKit_54.uproject`](UnrealLearningKit_54.uproject) with Unreal Engine 5.4 to load the project.

## Project Contents

- `Content/BlockGame`: block-based game examples
- `Content/Hour_of_Code`: Hour of Code levels and Blueprints
- `Content/LearningKit_Games`: Learning Kit game examples and MP2 Blueprint assets
- `Config`: project configuration

## Git LFS

Unreal `.uasset` and `.umap` files are managed with Git LFS. Install and enable Git LFS, then fetch the LFS objects after cloning:

```sh
git lfs install
git lfs pull
```

Unreal Engine may generate local `Intermediate`, `Saved`, and `DerivedDataCache` directories when the project is opened. These generated files are not committed to the repository.
