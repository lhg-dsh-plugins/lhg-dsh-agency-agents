# 本版修复（BUGFIX）

## 标题

代码生成脚本 `scripts/generate-roster.mjs` 无法运行：`walk()` 给**文件**条目也拼尾随 `/` 构造 URL，`readFile('…/xxx.md/')` 以 `ENOTDIR` 崩溃，`src/client/roster.ts` 无法重新生成。

版本：上游 1.2.10 → 本版 1.2.11（patch +1）。

## 证据链

1. **修复前（上游固定 SHA `18144c7…` 原样运行）**：

   ```
   $ node scripts/generate-roster.mjs
   Error: ENOTDIR: not a directory, open '/…/assets/agency-agents/academic/academic-anthropologist.md/'
       at async readFile (node:internal/fs/promises:2003:14)
       at async file:///…/scripts/generate-roster.mjs:42:17
   退出码 1
   ```

   在遍历到的第一个 `.md` 文件上即崩溃，脚本头注释声明的用法（`node scripts/generate-roster.mjs` 重新生成 roster）完全不可用。

2. **最小复现**：单独按现行写法构造 URL 并读取——

   ```js
   const fileUrl = new URL('academic-anthropologist.md' + '/', dir) // 现行写法
   await readFile(fileUrl, 'utf8') // → ENOTDIR: …/academic-anthropologist.md/
   const okUrl = new URL('academic-anthropologist.md', dir)          // 修正写法
   await readFile(okUrl, 'utf8')   // → 正常读出 7878 字符
   ```

3. **基线对照**：`git stash` 回退本版改动后重跑生成器，同样以 ENOTDIR 退出码 1 失败，证明缺陷在上游原代码中、非本版引入。

4. **基线其余项**：上游 `pnpm test` 243/243、`pnpm typecheck`、`pnpm build`、`pnpm verify` 全部通过；open issue 仅 2 条功能请求（#27、#16），无 bug 报告——本缺陷为静态审查 + 复现坐实。

## 根因

`walk()` 对目录与文件条目统一执行 `new URL(e.name + '/', dir)`：

- 对**目录**，尾随 `/` 是必需的——URL 相对解析以路径末段为界，没有 `/` 时子条目会解析到父级目录；
- 对**文件**，尾随 `/` 使 pathname 变成「文件路径/」，POSIX 上 `open()` 返回 `ENOTDIR`，`readFile` 必然失败。

即同一行代码对目录正确、对文件必错，而该脚本的唯一用途就是读取文件生成花名册。

## 改动文件

仅 1 个文件，`scripts/generate-roster.mjs` 的 `walk()`：按条目类型构造 URL——目录条目拼 `e.name + '/'` 递归，文件条目用 `new URL(e.name, dir)` 不拼斜杠。生成逻辑、排序、输出格式均未动。

（另按 fork 集成口径：`package.json` 版本 1.2.10 → 1.2.11、`repository.url` 改为本 fork 仓库；README 顶部追加 fork 说明；新增本文件与 `UPSTREAM.md`。lock 文件不含根包版本字段，无版本戳常量需同步。）

## 复验结果

- 修复后：`node scripts/generate-roster.mjs` 正常输出「已生成 331 条 roster -> src/client/roster.ts」，且 `git diff src/client/roster.ts` **为空**——重新生成结果与上游已提交的花名册逐字节一致，修复只恢复生成路径、零产出漂移。
- 回归：`pnpm test` 243/243 通过；`pnpm typecheck` 退出码 0；`pnpm verify` 全部发布验证通过。

## 验证边界

- 本缺陷位于维护者侧的代码生成脚本，不影响已发布插件的运行时行为（`roster.ts` 已随包提交且内容正确）；影响面是「新增/修改专家后按文档重新生成花名册」这一维护流程。
- ENOTDIR 为 POSIX 文件系统语义，本版在 Linux 上实测；Windows 下打开带尾随斜杠的文件路径同样非法，但未在 Windows 实机复验，如实标注。
