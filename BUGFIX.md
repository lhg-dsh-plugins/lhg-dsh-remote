# BUGFIX — v0.8.51

## Bug 标题

盘符根工作区（`D:\`）下 `relPathUnder()` 对所有子路径恒返回 `null`：镜像同步的同步状态键退化为纯文件名互相碰撞、路径型忽略规则在第一层以下永不命中

## 证据链（修复前）

`lib/paths.js` 的 `relPathUnder(root, p)` 是 `lib/sync.js`（syncTree/pushTree）与 `lib/search.js` 计算工作区相对路径的唯一入口，调用方一律以 `relPathUnder(root, rp) || name` 兜底——返回 `null` 时相对路径静默退化为**纯文件名**。

盘符根本是合法工作区（上游 issue #60 已明确把 POSIX `/` 与 Windows `D:\` 都定为合法远端工作区，`isFilesystemRoot()` 亦收 `D:\`/`D:`），但 `relPathUnder` 对它失效。离线复现（内存版 MemFs SFTP，与上游自有测试同手法，经 `toSftpPath` 适配层模拟生产 `lib/sftp-surface.js` 的路径转换；未做任何真实网络连接）：

- `relPathUnder('D:\\', 'D:\\a\\same.txt')` → `null`（应为 `'a/same.txt'`）
- `relPathUnder('D:', 'D:\\b\\only-b.txt')` → `null`（应为 `'b/only-b.txt'`）
- 以 `D:\` 为工作区跑 `syncTree`：`nextState` 的键为 `["only-b.txt", "same.txt"]`——`a/same.txt` 与 `b/same.txt` 共用一个 `same.txt` 状态条目，三方比对（R/L/S）会把两个不同文件当作同一个：一个文件的已同步状态可让另一个文件被误判 unchanged 而永不镜像（或误报/漏报冲突）。
- 忽略规则 `b/**` 下 `b/same.txt` 仍被镜像下载——相对路径退化为 `same.txt` 后，任何带路径形状的 ignore 模式在第一层以下都匹配不到（search 的 ignore 过滤共用同一入口，同受影响）。

复现脚本在基线（`git stash` 回退修复）下 exit 1、4 项断言失败；修复后 exit 0、全过。

## 根因

`relPathUnder` 内部的 `posix()` 规范化把盘符根写成 `'/d:/'`（尾随斜杠是 `'X:/'` 拼接的一部分），而子路径规范化为 `'/d:/a/…'`（无尾随斜杠）；随后代码用 `prefix = r + '/'` 拼前缀，得到 `'/d://'`，没有任何子路径以它开头，于是恒返回 `null`。POSIX 根 `'/'` 因规范化结果是空串而偶然可用，未被既有测试发现；`test/root-workspace.test.js` 的根工作区同步用例只覆盖了 POSIX `/`。

## 改动文件

- `lib/paths.js`（唯一逻辑改动）：拼前缀前对根与目标的规范化结果统一剥除尾随斜杠（空串归一为 `'/'`），根自身相等判断（返回 `''`）与 POSIX/UNC 行为保持不变。
- `test/paths.test.js`：`relPathUnder` 用例补盘符根/根自身断言（上游既有用例未改）。
- `test/root-workspace.test.js`：新增盘符根工作区 `syncTree` 回归用例（状态键为完整相对路径 + `b/**` 忽略生效）。两处新测试在未修复的 `lib/paths.js` 上恰 2 红、修复后全绿。
- `package.json` / `package-lock.json`：version 0.8.50 → 0.8.51；`package.json` 的 `repository.url` 改指本 fork 落点仓。包内无版本戳常量（`lib/update.js` 的 `LOADED_VERSION` 运行时读 package.json）。
- `README.md`：顶部追加中文 lhg fork 头，上游原文一字未删；`LICENSE` 原样保留；新增 `UPSTREAM.md` / 本文件。

## 复验结果

- 复现脚本：修复前 exit 1（4 失败）→ 修复后 exit 0（5 项断言全过，含对照组 `a/same.txt` 正常镜像）。
- 全量 `npm test`：修复后 661 个测试 660 过，与修复前基线逐项一致、零回归；`node check.mjs` 通过。
- 新增回归测试定向复跑：`test/paths.test.js` + `test/root-workspace.test.js` 共 21 个全过。

## 验证边界

- 唯一失败与基线相同：`test/docs-images.test.js` 的截图边缘裁剪检查需要 `python` 可执行文件，本沙箱缺失（`spawnSync python ENOENT`），属环境伪影、与本改动无关（基线同样失败，已以 `git stash` 对照确认）。
- 验证全程离线：未连真实 SSH/SFTP 服务，SFTP 行为由上游自带 MemFs 测试替身模拟（与上游 `test/root-workspace.test.js` 既有根工作区用例同一手法）；真实 Windows 远程主机上的端到端同步未做。
- open issue #63（本地 session 工具分层暴露 + `localTools` 配置项）是功能改造而非本缺陷，未在本版处理。
