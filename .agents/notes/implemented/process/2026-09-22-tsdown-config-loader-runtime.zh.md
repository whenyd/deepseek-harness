# Agent Note: 无原生 TypeScript 支持时的 tsdown 配置加载

Status: implemented

[English](2026-09-22-tsdown-config-loader-runtime.md) | 中文

## 问题

在 Node 运行时报告 `process.features.typescript` 为 false 的全新克隆上，`pnpm run build` 会失败；部分打包发布的 Node 22 构建正是如此，而声明的引擎范围 `^22.19.0 || >=24.0.0` 允许这些运行时。既没有 Bun 也没有原生 TypeScript 支持时，tsdown 的 `--config-loader auto` 会选定 `unrun` 加载器，而 tsdown 只把 `unrun` 声明为可选 peerDependency，pnpm 因此从不安装它：构建停在 `Failed to import module "unrun"`。

补上这个缺失的对等依赖后还会暴露第二个失败。unrun 加载器执行的是每个 tsdown 配置的编译副本，它替换出的 `import.meta.url` 使 [`tsdown.client.ts`](../../../../packages/client/tsdown.client.ts) 中的 `new URL('../..', import.meta.url)` 解析到 `packages/` 而不是仓库根目录。预设的 `workspaceManifest` 于是按 `packages/packages/*/*/package.json` 匹配，找不到任何文件，构建以 `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-terminal-controller` 失败。自动加载器选择原生路径的运行时从不导入 unrun，也就不会遇到这两个失败。

## 决策

根 [`package.json`](../../../../package.json) 在 `devDependencies` 中与 tsdown 的其他可选对等依赖 `tsx`、`publint` 和 `typescript` 并列声明 `unrun`（`^0.3.1`），使自动加载器在引擎范围内的每个运行时都能解析到加载器。它只用于开发工具链：任何发布产物都不包含它，[`THIRD_PARTY_NOTICES.md`](../../../../THIRD_PARTY_NOTICES.md) 将其列为 MIT 开发工具。

[`tsdown.client.ts`](../../../../packages/client/tsdown.client.ts) 改为从 `process.cwd()` 向上查找到最近的 `pnpm-workspace.yaml` 来定位仓库根目录。workspace 构建以仓库根目录为工作目录求值各包配置，`pnpm --filter <pkg> bundle` 以包目录运行；两种形式都能到达仓库根目录，不依赖配置加载器的 `import.meta.url`。

`pnpm-lock.yaml` 新增 `unrun@0.3.1` 与 tsdown 的对等依赖后缀。

## 考虑过的替代方案

**要求使用具备原生 TypeScript 支持的 Node 运行时。** 引擎下限是一个版本范围，而同一版本的 `process.features.typescript` 因构建而异，版本检查无法选出能加载这些配置的运行时。

**在每个 tsdown 调用点传入 `--config-loader tsx`。** tsdown 只通过命令行或编程 API 接受该值，配置文件无法设置；根脚本与每个包级 `bundle`/`watch` 脚本都需要这个标志，而新增调用点可能悄悄退回有问题的默认值。

**保留基于 `import.meta.url` 的仓库根目录，改用能保留它的加载器。** URL 相对上溯在原生与 tsx 加载器下正确，在 unrun 下错误；工作目录上溯在三种加载器下给出同一个根目录。

## 后果

全新安装后在引擎范围内的任何运行时都能加载 tsdown 配置，加载器选择不再依赖编入 Node 二进制的特性。代价是一个仅用于开发的依赖、重新生成的第三方声明文件与 lockfile 条目，以及一个假定进程在仓库内启动的预设：在仓库外启动的构建会抛出 `tsdown: no pnpm-workspace.yaml above the working directory marks the repository root`，而不是匹配到错误的目录。

验证：在 `process.features.typescript` 为 false、加载器为 unrun 的 Node v22.22.1 上，`pnpm run build` 完整通过；回滚工作目录上溯会复现 `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-terminal-controller`；`pnpm run verify-third-party-notices` 与冻结 lockfile 安装均通过。

## 相关

- [TSC-first 构建与单一编译器归属](2026-06-17-ts-build-config.zh.md) 负责构建阶段与 tsdown 的打包职责；本记录只补充 tsdown 如何加载配置。
- [在工作区 tsdown 之后打包 Desktop 主进程](2026-09-22-desktop-main-bundle-after-workspace-tsdown.zh.md) 负责根工作区构建列表与 Desktop 打包顺序。
