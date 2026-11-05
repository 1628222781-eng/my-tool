# my-tool · 鸿蒙设备协同微专业案例库

本仓库收录「鸿蒙端边云设备告警与巡检协同」微专业训练过程中的全套报告与任务记录，
按 FDE（发现 → 定义 → 设计 → 构建 → 测试 → 试点 → 上线 → 运营）流程归档。

> 所有案例基于合成场景访谈，需求为待验证假设；不代表真实客户授权、生产上线或实际收益。

## 目录结构

```
my-tool/
├── README.md                 # 本文件
├── LICENSE                   # 开源许可证
├── CHANGELOG.md              # 版本记录
├── CONTRIBUTING.md           # 贡献指南
├── requirements.txt          # Python 依赖
├── src/my_tool/              # 源代码（预留）
├── docs/
│   ├── design.md             # 设计说明
│   ├── api.md                # 接口说明
│   ├── FDE全流程训练记录.md  # 全流程串联
│   └── reports/              # 分阶段报告
│       ├── 01-discovery/     # 业务挖掘
│       ├── 02-research/      # 现场调研 / 角色访谈
│       ├── 03-requirements/  # 需求定义
│       ├── 04-design/        # 方案设计
│       ├── 05-build/         # 构建集成
│       ├── 06-test/          # 测试评估
│       ├── 07-pilot/         # 客户 / 试点
│       ├── 08-release/       # 上线交接
│       ├── 09-operations/     # 运营迭代
│       └── 99-misc/          # 其他任务粘贴稿
├── tests/                    # 测试（预留）
├── examples/                 # 示例（预留）
├── scripts/                  # 构建 / 部署脚本
├── config/                   # 默认配置
├── data/                     # 样例数据
└── .github/                  # Issue / PR 模板与 CI
```

## 快速阅读顺序

1. [`docs/FDE全流程训练记录.md`](docs/FDE全流程训练记录.md) — 全流程串联
2. [`docs/reports/01-discovery/`](docs/reports/01-discovery/) — 为什么立项
3. 按 `02 → 09` 顺序读各阶段报告
4. `*-粘贴版.txt` 是任务执行时的工作稿，可对照正式报告看思路演进

## 安装与运行

本仓库当前以文档为主，`src/` 下预留了 Python 包骨架，暂无独立可运行程序。

```bash
# 克隆
git clone <your-repo-url>
cd my-tool

# （可选）建虚拟环境
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

## 许可证

见 [LICENSE](./LICENSE)。
