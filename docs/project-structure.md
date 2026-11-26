# 项目结构与模块边界

## 总体设计

采用一个 Python 项目、一套模型主线、多种运行入口。按职责组织代码，学习顺序由路线文档表达，不建立 `lesson01`、`lesson02` 等完整代码副本。

下面是目标结构，不代表现在已经创建全部文件。只在当前阶段需要时创建对应模块；文件名也可以随实现细化。

```text
nano-ai/
├── README.md
├── AGENTS.md
├── .agents/skills/learning-notes/SKILL.md  # 项目内学习笔记技能
├── pyproject.toml              # Python 版本、依赖和开发工具
├── uv.lock                     # 确定工具链后生成并提交
├── .gitignore
├── configs/                    # 可提交的实验输入配置
│   ├── pretrain/               # 本地小实验、GPU 正式训练
│   ├── finetune/               # 后续 SFT / LoRA 配置
│   └── eval/                   # 固定评估的配置
├── src/nano_ai/
│   ├── data/                  # 语料读取、切分、编码、序列构建
│   ├── tokenization/          # tokenizer 训练、加载和版本信息
│   ├── models/                # 模型配置、attention、Transformer
│   ├── training/              # 训练循环、优化器、checkpoint
│   ├── evaluation/            # 验证 loss、生成质量、任务指标
│   ├── inference/             # 自回归生成、采样和后续服务逻辑
│   └── cli/                   # 准备数据、训练、评估、生成的薄入口
├── scripts/                   # 租卡准备、环境检查等运维脚本
├── tests/                     # 小而关键的正确性测试
├── notebooks/                 # 可选的探索与图示，不承载正式训练逻辑
├── knowledge/                 # 按主题组织的知识笔记，README.md 为入口
├── docs/
│   ├── project-structure.md
│   ├── roadmap.md
│   ├── decisions/             # 重要选择、备选方案与理由
│   └── experiments/           # 可提交的实验报告
│       └── template.md
├── data/                      # 本地数据与缓存，不提交
│   ├── raw/
│   └── processed/
└── runs/                      # 每次运行的产物，不提交
    └── <run-id>/
        ├── config.yaml        # 本次实际生效的完整配置
        ├── metadata.json      # 代码、数据、tokenizer、环境与种子
        ├── metrics.jsonl
        ├── checkpoints/
        └── samples/
```

## 关键模块的职责

| 模块 | 负责什么 | 边界 |
| --- | --- | --- |
| `data` | 将语料转换成训练和评估批次 | 不负责模型前向或优化器更新 |
| `tokenization` | 文本与 token IDs 的转换及词表工件管理 | tokenizer 必须与模型、数据版本对应 |
| `models` | 根据输入计算 logits，表达模型结构 | 不读取训练目录，不管理实验 |
| `training` | 计算训练目标、反向传播、更新参数、恢复状态 | 调用模型与数据模块，避免把所有逻辑塞进一个脚本 |
| `evaluation` | 定义与执行评估，汇总结果 | 划分验证与最终测试用途，避免评估污染 |
| `inference` | 生成 token、采样、停止条件 | 训练和独立推理共用生成逻辑 |
| `cli` | 解析参数、加载配置、串联模块 | 保持轻薄，不成为核心实现所在地 |

第一版允许把相关逻辑保留在少数几个文件中。模块边界用于帮助理解，不要求预先创建一整套类层次。

## 主数据流

```text
原始文本
  → 数据划分与清理
  → tokenizer 编码
  → token 序列与批次
  → 模型 logits
  → next-token loss
  → 梯度与参数更新
  → checkpoint
  → 独立评估与生成
```

使用自训练 tokenizer 时，只用训练划分拟合词表。数据集需要在切块之前按合适的文档或来源单位划分，并检查跨划分重复。

## 初始技术选择

- **Python + PyTorch**：直接观察张量、自动微分和训练循环，不先引入高层训练框架。
- **优先使用 uv 管理环境**：在建立工程时确认兼容性；本地与 GPU 环境记录各自实际依赖和硬件信息。
- **简单配置**：先使用显式参数与单个配置文件；配置增多后采用 YAML，不引入复杂继承系统。
- **本地正确性与远程性能分工**：CPU 能运行小检查；本地是否启用其他加速后端，在实际环境确认后决定。正式 GPU 实验以 CUDA 为主。
- **先保存文件记录实验**：配置、JSONL 指标与 Markdown 报告足够起步；需要跨实验比较时再考虑实验平台。
- **后续使用成熟工具**：微调阶段再引入 Transformers / PEFT 等；服务阶段根据目标选择推理引擎，不把已有模型强行适配为教学模型。

暂不加入分布式训练、Web 管理台、向量数据库、通用 Agent 框架和复杂部署平台。它们在出现具体问题时再引入。

## 代码、实验报告与大文件

问答和讨论中形成的可复习知识统一放在 `knowledge/`，使用稳定文件名、按主题分目录，由 `knowledge/README.md` 串联；版本由 Git 保存。组织与维护规则见项目内技能 `.agents/skills/learning-notes/SKILL.md`。项目规划、决策和实验留在 `docs/`，通过链接关联知识页，避免重复维护。

Git 保存代码、配置、小型合成测试样本、报告和必要图表。`data/` 与 `runs/` 只保存可重新获取或单独备份的本地产物。

实验报告记录结果摘要，并指向完整工件的保存位置。忽略文件不是备份方案；在销毁租卡实例之前，需要转存 checkpoint、日志和配置。

依赖锁文件不能保证 GPU 环境完全一致，还需记录 PyTorch、CUDA、驱动与 GPU 型号。复现也不意味着跨硬件逐位一致。
