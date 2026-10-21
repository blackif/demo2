# 自动检查文件并创建 Issue 的方案说明

## 1. 目的

使用 GitHub Actions 定期检查仓库中指定文件是否存在。如果文件缺失，则自动创建 GitHub Issue，提醒维护者补齐文件。

## 2. 检查目标

- 仓库：`blackif/demo2`
- 目标文件：`sap/document/template/issues.md`
- 工作流文件建议路径：`.github/workflows/check-required-file.yml`
- 检查频率：每 5 分钟一次，并支持通过 `workflow_dispatch` 手动触发。

> GitHub Actions 的定时任务可能延迟执行，并不保证每 5 分钟准时启动。

## 3. Issue 标识规则

采用「固定标题 + 专用 Label」识别自动创建的 Issue。

- 固定标题：`[Auto Check] Missing file: sap/document/template/issues.md`
- 专用 Label：`auto-file-check`

工作流应使用目标文件路径生成固定标题，并同时检查标题和专用 Label，以识别该文件对应的未关闭 Issue。不要只依赖可被人工修改的标题，也不要把不同目标文件的 Issue 混在一起。

首次启用前，应确保仓库中存在 `auto-file-check` Label；也可以在工作流中加入自动创建 Label 的步骤。

## 4. 检查与处理逻辑

1. 检出仓库默认分支的代码。
2. 检查目标文件是否存在。
3. 如果文件存在：不创建新 Issue；如果存在对应的未关闭自动 Issue，可选择自动关闭它。
4. 如果文件不存在：查询是否已有标题和 Label 均匹配的未关闭 Issue。
5. 如果已有匹配的未关闭 Issue：不重复创建。
6. 如果没有匹配的未关闭 Issue：创建新 Issue，并添加 `auto-file-check` Label。

## 5. 重复 Issue 的行为

| 文件状态 | 对应的未关闭 Issue | 处理 |
|---|---|---|
| 不存在 | 不存在 | 创建 Issue |
| 不存在 | 已存在 | 不创建重复 Issue |
| 存在 | 已存在 | 可自动关闭该 Issue |
| 不存在 | 之前的 Issue 已关闭 | 按当前规则重新创建 Issue |

因此，如果文件一直没有补齐，只要对应 Issue 保持 Open 状态，后续检查就不会重复创建 Issue。若人工关闭 Issue 但文件仍然缺失，下一次检查会重新创建 Issue；如不希望这样，需要额外设计关闭后抑制重建的规则。

## 6. 权限要求

工作流至少需要：

```yaml
permissions:
  contents: read
  issues: write
```

创建 Issue 可使用 GitHub Actions 提供的 `GITHUB_TOKEN`。同时应确认仓库的 Actions 设置允许工作流使用所需权限。

## 7. 建议的工作流触发器

```yaml
on:
  schedule:
    - cron: '*/5 * * * *'
  workflow_dispatch:
```

## 8. 维护注意事项

- 定时工作流文件必须提交到仓库默认分支，定时触发才会按 GitHub 的规则运行。
- 文件路径、固定标题和 Label 名称应保持一致；修改时要同步更新检查和去重逻辑。
- 检查的是工作流检出的分支；如需检查其他分支，应显式指定分支。
- 本文档描述的是方案与预期行为，不代表工作流已经创建或启用。
