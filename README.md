# org-backup

把 GitHub 组织 [`yuxinrun`](https://github.com/yuxinrun) 下的**全部公开仓库**，以裸镜像（bare mirror）方式单向备份到个人账号 [`yuxink`](https://github.com/yuxink)，用于防止组织仓库误删后无法恢复。

- 触发：每天 UTC 02:00（北京时间 10:00）+ 手动 `workflow_dispatch`
- 方式：GitHub Actions + `gh` CLI + `git clone --mirror` / `git push --mirror`
- 备份内容：分支、tag、完整提交历史
- 不删除任何个人侧仓库：源仓库被删除后，镜像保留为最后一版存档

## 工作原理

1. 用 `ORG_READ_PAT` 调 GitHub API，分页拉取 `yuxinrun` 的所有公开仓库。
2. 对每个仓库，用 `PERSONAL_PAT` 检查 `yuxink/<repo>` 是否存在：
   - 不存在：自动创建空仓库（默认 `private`），并关闭该镜像仓库的 Actions，避免同步推送触发镜像里的工作流、消耗额度；
   - 已存在：直接覆盖同步。
3. `git clone --mirror` 拉取源仓库到 Runner 临时目录，剔除 `refs/pull/*`（GitHub 不允许向目标仓库写入这类引用）。
4. `git push --mirror` 强制推送全部引用到个人侧镜像，并把镜像的默认分支对齐到源仓库。
5. 单个仓库失败不会中断其他仓库；结束时汇总结果到 Actions Summary，有失败则任务标记失败。

仓库文件只有两个：

```
.github/workflows/backup-all.yml
README.md
```

## 部署步骤

### 1. 准备两个 PAT

在 GitHub 右上角头像 → **Settings** → **Developer settings** → **Personal access tokens** 生成。

#### ORG_READ_PAT（读取组织公开仓库）

- 推荐类型：**Classic**
- 勾选：`public_repo`（最小权限，够用于公开仓库）
- 如果以后要备份私有仓库，改为勾选 `repo`
- 如果 `yuxinrun` 组织启用了 SAML SSO，生成后需要在 token 页面点 **Configure SSO** → **Authorize** 授权该组织
- 若想用 **Fine-grained**：Resource owner 选 `yuxinrun`，Repository access 选 **Public repositories (read-only)**，权限 `Contents: Read-only` + `Metadata: Read-only`。组织若限制 fine-grained token，需要组织管理员批准

#### PERSONAL_PAT（创建并推送镜像仓库）

- 推荐类型：**Classic**（fine-grained 建仓能力不如 classic 明确，用 classic 最稳妥）
- 必须勾选：
  - `repo`（私有镜像仓库的读写；镜像默认 private）
  - `workflow`（镜像仓库里含 `.github/workflows/*.yml` 时，推送这些文件必须要有该 scope，否则会被拒绝）
- Fine-grained 备选（需自行验证建仓是否可用）：Resource owner 选 `yuxink`，Repository access 选 **All repositories**，权限：
  - `Administration: Read and write`（创建仓库）
  - `Contents: Read and write`（推送代码）
  - `Workflows: Read and write`（推送 workflow 文件）
  - `Metadata: Read-only`（必选）

> 建议给两个 PAT 都设置有效期（如 90 天），到期前轮换并更新 Secret；过期后备份任务会直接失败。

### 2. 配置 Secrets

进入 `yuxinrun/org-backup` → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**，添加：

| Secret 名称 | 值 |
| --- | --- |
| `ORG_READ_PAT` | 第 1 步生成的只读 PAT |
| `PERSONAL_PAT` | `yuxink` 账号的读写 PAT |

### 3. 确认仓库设置

- 仓库可见性：**Public**
- 默认分支：**main**
- **Settings** → **Actions** → **General** → Workflow permissions 保持默认即可（本工作流不使用 `GITHUB_TOKEN` 写权限）

### 4. 首次手动运行

**Actions** → **backup-all** → **Run workflow** → 分支选 `main` → 运行。

运行结束后查看：
- 运行日志中的 `Preflight` 步骤应输出 `PERSONAL_PAT 所属账号：yuxink`
- 本次运行的 **Summary** 页会有一张表，列出每个仓库的成功/失败状态

## PAT 权限勾选清单（速查）

### Classic PAT

| Secret | 必须勾选 | 说明 |
| --- | --- | --- |
| `ORG_READ_PAT` | `public_repo` | 读取公开仓库；若备份私有仓库改为 `repo` |
| `PERSONAL_PAT` | `repo`、`workflow` | `workflow` 用于推送镜像中的 Actions 工作流文件 |

补充：

- 组织开启 SSO 时，`ORG_READ_PAT` 还需在 token 页面对 `yuxinrun` 执行 **Authorize**。
- 只读场景可选 `read:org`，但拉取公开仓库列表并不需要。

### Fine-grained PAT

| Secret | Resource owner | Repository access | 权限 |
| --- | --- | --- | --- |
| `ORG_READ_PAT` | `yuxinrun` | Public repositories (read-only) | `Contents: Read-only`、`Metadata: Read-only` |
| `PERSONAL_PAT` | `yuxink` | All repositories | `Administration: R/W`、`Contents: R/W`、`Workflows: R/W`、`Metadata: Read-only` |

## 部署后校验

以下命令建议先 `gh auth login`，镜像仓库默认是 private，直接 `git ls-remote` 需要已登录的凭据。

1. **仓库数量核对**：源仓库列表减去个人侧同名仓库，应为空。

   ```bash
   gh api --paginate "orgs/yuxinrun/repos?type=public&per_page=100" \
     --jq '.[] | select(.private == false) | .name' | sort > /tmp/src.txt

   gh repo list yuxink --limit 1000 --json name --jq '.[].name' | sort > /tmp/dst.txt

   comm -23 /tmp/src.txt /tmp/dst.txt   # 输出为空 = 每个公开仓库都有对应镜像
   ```

2. **引用（分支/tag/提交）一致性抽查**：对任意几个仓库执行，输出为空表示 refs 完全一致。

   ```bash
   repo=some-repo
   diff \
     <(git ls-remote --refs "https://github.com/yuxinrun/$repo.git" | grep -v refs/pull | sort) \
     <(git ls-remote --refs "https://github.com/yuxink/$repo.git" | sort)
   ```

   > `--refs` 会排除 `HEAD`；`refs/pull/*` 因 GitHub 限制不做同步，比较时一并排除。

3. **可恢复性验证**：克隆镜像仓库并确认历史完整。

   ```bash
   git clone --mirror "https://github.com/yuxink/$repo.git" /tmp/check.git
   git -C /tmp/check.git log --oneline -3
   git -C /tmp/check.git tag | head
   ```

4. **删除语义演练（可选，建议用测试仓库）**：
   - 在源仓库删除一个分支 → 手动触发 workflow → 镜像侧同名分支应消失；
   - 在源仓库改名或更换默认分支 → 手动触发 → 镜像默认分支应跟随；
   - 删除整个源测试仓库 → 再次触发 → 个人侧镜像**仍存在**且不再更新。

5. **定时触发确认**：次日检查 Actions 是否有 `schedule` 触发的运行记录。GitHub 定时任务在高峰期可能延迟数分钟到数十分钟。

## 重要注意事项

1. **单向备份**：`yuxink` 下的镜像仓库只做存档，不要在镜像仓库里开发或提交。下次同步会用 `--mirror` 强制覆盖，个人侧的改动会被丢弃。
2. **`--mirror` 的删除语义**：源仓库删除分支或 tag，同步时镜像侧对应引用也会被删除；这是“完全一致”的镜像行为，不是增量保留。
3. **源仓库整体删除**：脚本只遍历当前仍存在的源仓库，因此已删除的源仓库不会再被同步，但个人侧镜像**保留不删除**，作为最后一版存档。脚本从不删除个人账号下的任何仓库。
4. **不是 fork**：镜像不是 GitHub Fork，没有父子关系，也不会共享网络/贡献图；源仓库删除不影响个人备份。
5. **镜像可见性**：默认新建为 `private`，避免把历史中已删除的敏感内容暴露公开。需要公开时把 workflow 中 `MIRROR_VISIBILITY` 改为 `public`——只影响之后新建的镜像，已有镜像需在仓库设置中手动调整。
6. **镜像 Actions 被关闭**：脚本在新建镜像时会关闭其 Actions，防止镜像中的 `on: push` 工作流被同步推送触发。手动创建的旧镜像如需关闭，请在仓库 Settings → Actions 中处理。
7. **重命名**：源仓库改名后，新名字会被视为新仓库并新建镜像；旧镜像保留为旧名字的存档。
8. **备份范围有限**：只备份 git 对象（分支、tag、提交历史）。不包含 Issues、PR、Wiki、Releases 附件、Actions Secrets；**Git LFS 对象不会随 `git mirror` 复制**（若仓库使用 LFS，需额外 `git lfs fetch --all` 方案）；submodule 只保留引用，不展开其内容。
9. **定时任务可能被停用**：公共仓库若连续 60 天没有任何提交活动，GitHub 会自动停用定时 workflow。建议定期手动触发一次；若长期无人维护，可在其他仓库加 keep-alive 提交，或改为每月手动跑一次。
10. **PAT 轮换**：PAT 过期会导致任务失败（Preflight 会给出明确报错）。更换账号或 token 权限后，记得同步更新 Secrets。
11. **运行时长**：仓库多或仓库大时，单次运行可能较久；workflow 已设置 180 分钟超时。若超时，可拆分多个 workflow 分批处理。
12. **私有源仓库不受支持**：当前只拉取 `yuxinrun` 的公开仓库（`private == false`）。如需备份私有仓库，需要给 `ORG_READ_PAT` 增加 `repo` 权限并调整过滤逻辑。

## 故障排查

| 现象 | 常见原因 |
| --- | --- |
| Preflight 报 `PERSONAL_PAT 必须属于 yuxink` | Secret 填错，或用了其他账号的 token |
| `gh api orgs/yuxinrun` 返回 401/403 | `ORG_READ_PAT` 无效、过期，或组织 SSO 未授权 |
| 建仓失败 403 | `PERSONAL_PAT` 缺少 `repo` 权限，或 fine-grained token 不支持建仓 |
| 推送报 `refusing to allow a Personal Access Token to create or update workflow` | `PERSONAL_PAT` 缺少 `workflow` scope |
| `git push --mirror` 报默认分支无法删除 | 源仓库改过默认分支；workflow 会自动对齐后重试，仍失败时手动检查镜像默认分支 |
| 仓库列表为空 | 组织名写错，或 org 下没有公开仓库 |
