# 贡献指南

感谢你愿意为 `my-tool` 出一份力。

## 提交前
1. Fork 本仓库并克隆到本地。
2. 新建分支：`git checkout -b feat/your-topic` 或 `docs/your-topic`。
3. 保持提交信息简洁：`docs: 补充 06-test 评估口径`。

## 目录约定
- 新报告请放入 `docs/reports/` 下对应的阶段目录，命名 `<阶段>-<主题>.md`。
- 通用工具代码放入 `src/my_tool/utils/`，核心业务放入 `src/my_tool/core/`。
- 配置样例放 `config/default.yaml`，真实密钥不要提交。

## 提交 PR
- PR 标题写清楚改了什么。
- 涉及报告内容时，在 PR 描述里说明对应阶段、证据来源。
- 等待 CI 通过后再请求 review。
