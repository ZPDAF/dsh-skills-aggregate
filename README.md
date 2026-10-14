# DSH 技能聚合源

给 **设置 → 技能**（`@michengai/dsh-skills-manager`）用的自建聚合技能源。
把多个公开仓库里的 `SKILL.md` 精简镜像进**一个**仓库，面板里添加**一次**就能浏览并一键安装全部技能。

## 为什么需要它（实测依据）

面板的仓库来源只接受 `owner/repo` 形式的 GitHub 仓库，并会**下载整库 zip**，硬上限 **32 MiB**，
随后扫描全库每一个 `SKILL.md`（每库上限 **500 个**）。实测上游体量：

| 上游 | SKILL.md | 仓库大小 | 能否直接给面板用 |
|---|---|---|---|
| `anthropics/skills` | 20 | 11.13 MB | ✅ 可以 |
| `LeastBit/Anthropics_skills_zh-CN` | 17 | 8.07 MB | ✅ 可以 |
| `lubiny0601-spec/Skill-hub` | 70 | 33.96 MB | ❌ 超 32 MiB |
| `sickn33/agentic-awesome-skills` | 8218 | 311.89 MB | ❌ 超 9.7 倍，且超 500 技能上限 |
| `VoltAgent/awesome-agent-skills` | 0 | 0.22 MB | ❌ 纯清单，无技能正文 |
| `lza6/Agent-Skills-Hub` | 0 | 16.86 MB | ❌ 无技能正文 |

所以正确形态不是「指向上游」，而是**筛选式精简镜像**：只挑技能目录、剔大资产、硬性裁到预算内。

## 目录结构

```
aggregator/
├── manifest.json                   # 上游清单 + 优先级 + 覆盖规则（唯一需要人工维护的文件）
├── scripts/
│   ├── build.mjs                   # CI 构建：走 api.github.com（需 GITHUB_TOKEN）
│   ├── build-local.mjs             # 本机构建：走 codeload 单次 zip，不吃 API 配额
│   ├── verify.mjs                  # 约束校验：复刻面板全部硬限制
│   └── panel-truth-check.mjs       # 排他性验证：直接调用面板自己的解码器
├── .github/workflows/build.yml     # 定时/手动重建并推回仓库
└── package.json
```

## 部署步骤

### 1. 建仓库并推上去

本仓库按这套流程实际部署的参数：

| 项 | 值 |
|---|---|
| 本地目录 | `C:\Skills\Repo` |
| 远程仓库 | `ZPDAF/dsh-skills-aggregate` |
| 分支 | `main` |

```powershell
cd C:\Skills\Repo
git init -b main
git add -A
git commit -m "init: aggregate skill source"
git remote add origin https://github.com/ZPDAF/dsh-skills-aggregate.git
git push -u origin main
```

### 2. 本地先跑一次确认能出东西

```powershell
cd C:\Skills\Repo
node scripts/build-local.mjs          # 走 codeload，不吃 api.github.com 配额
node scripts/verify.mjs
node scripts/panel-truth-check.mjs    # 需在装了面板的机器上跑
```

`verify.mjs` 与 `panel-truth-check.mjs` 都必须 **EXIT=0** 才算通过。

> `build-local.mjs` 默认把产物写进**仓库根目录**（`skills/` + `index.json` + `SKILLS.md`），
> 因为面板下载的是整库 zip、会扫描里面每一个 `SKILL.md`。
> 若只想试跑，用 `$env:AGG_OUT` 指到临时目录，再 `$env:AGG_ROOT` 指到同一目录去验证。
> `panel-truth-check.mjs` 会从 `DSH_PROFILE_DIR` 自动定位面板安装位置，可用 `PANEL_MODULES` 覆盖。

### 3. 让 CI 生成正式产物

推到 GitHub 后，`.github/workflows/build.yml` 会在 **每周一 03:17 UTC** 自动重建，
也可在 Actions 页面手动 `Run workflow`。CI 会用 `GITHUB_TOKEN`（5000 次/小时）
拉取上游，把生成的 `skills/ index.json SKILLS.md` **提交回本仓库**。

> 为什么要 CI：`api.github.com` 匿名只有 60 次/小时，一次完整重建要读上千个 blob，
> 实测本机匿名配额已耗尽（`core.remaining = 0`），必须用 `GITHUB_TOKEN`。
> 本机构建则改走 codeload 单次整包 zip（`build-local.mjs`），不消耗 API 配额。

### 4. 在 DSH 里添加

```
设置 → 技能 → 技能仓库 → 添加仓库
  填入： ZPDAF/dsh-skills-aggregate
  扫描后即可逐项安装
```

面板安装只写入全局 DSH 目录 `~/.dsh/skills`，且**同名技能已存在时会拒绝安装**，不会覆盖本地文件。

## 维护方式：只改 manifest.json

```jsonc
{
  "owner": "某作者",
  "name": "某仓库",
  "ref": "main",              // 分支/标签；无法确定时填 main
  "category": "community-cn", // 决定产物落在 skills/<category>/ 下
  "license": "见上游",
  "priority": 5,              // 数字大 = 优先；用于人工阅读，覆盖由 overrides 决定
  "include": ["skills"],      // 只镜像这些前缀目录
  "exclude": ["template"],    // 排除这些前缀
  "overrides": ["docx"]       // 允许替换前面来源里同名的技能
}
```

**优先级规则**：面板**拒绝同名技能**。默认先到先得，后面的来源遇到同名直接跳过；
只有当它在 `overrides` 里显式列出该名字时，才会替换掉前面的版本。

`manifest.json` 现有顺序即为优先级（官方 → 中文覆盖官方 → 社区兜底）：

| 顺序 | 来源 | 类别 | 作用 |
|---|---|---|---|
| 1 | `anthropics/skills` | `official` | 官方基础技能 |
| 2 | `LeastBit/Anthropics_skills_zh-CN` | `official-zh` | 官方技能中文化，`overrides` 16 个同名项 |
| 3 | `lubiny0601-spec/Skill-hub` | `community-cn` | 社区中文技能，兜底不覆盖 |

## 构建期自动执行的裁剪

| 规则 | 阈值 | 原因 |
|---|---|---|
| 整库体积 | ≤ 26 MiB（`AGG_BUDGET_BYTES`） | 留余量给面板 32 MiB 硬上限 |
| 技能数 | ≤ 500 | 面板每库扫描上限 |
| 单个 `SKILL.md` | ≤ 256 KiB 且 frontmatter 合法 | 面板判定为 invalid 会被跳过 |
| 单文件资产 | ≤ 512 KiB | 大资产挤占预算且对技能无用 |
| 二进制扩展名 | 直接丢弃 | zip/视频/模型权重等 |
| 测试脚手架路径 | 任意深度的 `evals/tests/fixtures/examples` 段直接剔除 | 其内嵌 `SKILL.md` 会被面板扫到并造成重名冲突 |
| 重名技能 | 未声明的重复直接跳过 | 面板拒绝同名技能 |

## 当前产出的验证结果

在 `C:\Skills\Repo` 本机构建（`build-local.mjs`，codeload 路径）实测：

```
payload      : 17.83 MiB / 32 MiB cap
skills       : 72 / 500
SKILL.md sum : 0.82 MiB / 4 MiB
files        : 809
verify.mjs            -> EXIT=0
panel-truth-check.mjs -> 面板自身解码器解码 822 文件，判定 72/72 技能合法，zip 9.25 MiB
```

## 版权

产物中的技能文件**版权归各自上游作者**，本仓库只做筛选与镜像。
分发或商用前请逐个核对上游许可证；`index.json` 为每个技能记录了 `source` 与 `sourceUrl` 便于溯源。
