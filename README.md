# Elysia CoTrain

**An open intelligence, built together.**

[English](README.en.md) · [Architecture](docs/architecture.md) · [Roadmap](docs/roadmap.md) · [Contribute](CONTRIBUTING.md)

Elysia 是一个社区协作训练研究原型：共同研究模型架构、贡献计算、复现实验，逐步建设可独立运行和分叉的开放模型。

## 我们的愿景

我们希望 AI 的未来可以由更多人共同建设。无论你贡献的是算力、模型设计、代码、数据、验证还是文档，你都应该能够理解项目如何运作，并在公开的证据上参与改进。

Elysia 的长期目标是一个开放的协作网络：模型的研发过程可复现，贡献可追溯，成果可获得，运行与维护可以由不同的人接续。任何单一维护者离开，都不应让整个项目失去继续发展的可能。

我们承诺清楚区分已实现的功能和未来愿景，公开失败与限制，并保留独立运行和分叉的路径。当前版本仍依赖单个协调器；去中心化是需要逐步验证的工程目标。我们不宣称已经实现完全去中心化，也不以发行代币作为参与前提。

**欢迎共同建设：**运行一次演示、复现一个实验、报告一个缺陷，或者提出一个更好的设计。你不需要拥有昂贵显卡才能帮助项目。先看 [贡献指南](CONTRIBUTING.md) 与 [可接手的任务](docs/community-tasks.md)。

完整中英文愿景见 [VISION.md](VISION.md)。

名字取意于希腊典故 Elysium。**本项目与 Weaviate 的 [Elysia](https://github.com/weaviate/elysia)、ElysiaJS 等同名项目无关联。**建议 GitHub 仓库名称为 `elysia-cotrain`，Python 内部包名暂为 `opencotrain`。

## 现在能做什么

- 从零训练一个小型字节级 causal Transformer；宽度、层数和注意力头可配置。
- 多个独立客户端通过 HTTP 领取不同数据分片，执行真实 PyTorch 训练。
- SQLite 持久化任务租约、检查点、验收记录；超时重分配，重复提交不重复计入。
- 对版本、参数形状、有限数值和更新幅度进行服务端检查。
- 每轮等工作量参数平均，运行独立验证集；候选损失超过上一版的 1.02 倍时停止该实验并保留上一版。
- 导出模型，在本地生成文本；导出实验报告并比较多个架构实验。
- 两客户端一键演示、协议故障测试、CI、社区提案模板。

**这是受信任参与者的研究原型，不是可用于聊天的成品模型。**演示数据很小，损失下降不代表语言能力达标。v0.1 使用一个协调器，没有实现 P2P 发现、分片推理、匿名节点防投毒、密码学计算证明或链上治理。

初始本地验证：12 项测试通过；两个 CPU worker 进程完成 3 轮训练。30,432 参数的演示模型验证损失由 5.753 降至 3.316，但生成文本尚无实用性。完整条件与限制见 [验证报告](docs/validation.md)。

## 五分钟起步

需要 Python 3.11+。首版已在 Windows / Python 3.13 / CPU 实测；其他环境以实际 CI 与设备测试为准。首次下载 PyTorch 的时间取决于网络。

```powershell
git clone https://github.com/dbruce090-a11y/Elysia-cotrain.git
cd Elysia-cotrain
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
python -m pip install -e .
python -m opencotrain demo
```

Linux/macOS 激活环境使用 `source .venv/bin/activate`，后续命令相同。GPU 用户需自行安装匹配设备的 PyTorch；`--device cuda` / `mps` 代码路径尚未在本项目实测。

演示自动启动一个本地协调器和两个独立 worker 进程，训练 3 轮，然后关闭全部演示服务。输出位置为 `runs/demo/<timestamp-id>/`：

```text
report.json                 配置、数据/代码哈希、任务、每轮验证结果
elysia-checkpoint.json       最新通过质量门槛的模型
demo-worker-a.log            客户端 A 日志
demo-worker-b.log            客户端 B 日志
coordinator.sqlite          持久化实验状态
```

运行模型：把下面路径替换成演示实际输出路径。

```powershell
python -m opencotrain generate --checkpoint runs/demo/<run>/elysia-checkpoint.json --prompt "People " --tokens 80
python -m unittest discover -s tests -v
```

生成结果通常不连贯；这是训练链路验证模型。模型文件只使用 JSON 数值张量，加载时不执行文件内代码。JSON 传输体积大，仅适合初版小模型。

## 手动运行多个客户端

协调器终端：

```powershell
$env:ELYSIA_TOKEN = python -c "import secrets; print(secrets.token_urlsafe(32))"
python -m opencotrain create --name "Elysia tiny v1" --rounds 3 --shards 2
python -m opencotrain serve
```

记下 `create` 返回的 experiment ID。通过私密渠道让两个 worker 获得相同 token；不要把 token 写入公开 Issue 或 Git。两个 worker 必须安装相同代码快照。

每个 worker 分别打开终端，设置 token 后运行，使用不同名字：

```powershell
$env:ELYSIA_TOKEN = "<the-secret-from-coordinator>"
python -m opencotrain worker --experiment <id> --name worker-a
```

第二个客户端使用 `--name worker-b`。同名客户端会领到同一份未完成租约，名字不是可抵抗伪造的身份。

查看结果与导出：

```powershell
python -m opencotrain status --experiment <id> --output runs/tiny-report.json
python -m opencotrain export --experiment <id> --output runs/elysia.json
```

默认仅监听 `127.0.0.1`。跨电脑测试可在受信任的私有网络绑定指定地址，并让 worker 使用 `--url https://<your-endpoint>`（TLS 由反向代理提供）或受保护的 VPN 地址。不要把这个带共享令牌的 HTTP 原型直接暴露给匿名公网用户。

`Ctrl+C` 停止 worker。任务在租约到期后可重新领取。默认租约 300 秒，长任务需在创建实验时提高 `--lease-seconds`；当前没有训练中的租约续期。重新启动同一 `--db` 的协调器可继续未完成实验。尚未提交的本地梯度进度不会恢复。

## 如何共创模型

```powershell
python -m opencotrain create --name "wider experiment" --config configs/wider.json
```

复制配置，修改模型宽度、层数等，然后创建独立实验。修改模型代码时，所有参与该实验的 worker 必须使用相同代码；不要在运行中的实验上热替换代码。不同结构的权重不能直接平均。

```powershell
python -m opencotrain compare runs/tiny-report.json runs/wider-report.json
```

相同 token 数不等于相同计算预算。公平比较需同时记录墙钟时间、设备、参数量和资源成本。实验示例见 [experiments.md](docs/experiments.md)。

## 设计来源与边界

| 参考项目 | 借鉴的设计方向 | v0.1 实际状态 |
|---|---|---|
| Hivemind | 多节点训练、聚合与容错 | 自实现集中协调参考协议，未集成其 P2P 网络 |
| Petals | 模型分片推理 | 当前仅本地完整模型推理；分片服务在路线图 |
| OpenDiLoCo / PRIME | 降低同步频率、弹性训练 | 使用简单本地 AdamW + 参数平均，不是 DiLoCo |
| BOINC | 任务租约、重试、验证与贡献记录 | 已实现租约和协议验证；未实现重复计算验真 |
| Gensyn / Bittensor | 执行、验证、评价分工 | 暂由协调器评测，无证明系统、代币或链上评分 |
| OLMo | 代码、数据、配置和检查点可复现 | 原创小语料、数据哈希、源码哈希、JSON 检查点 |

没有复制这些项目的源码，也不暗示其认可或合作。详细链接与许可边界见 [references.md](docs/references.md)。

## 共同建设

最需要贡献的方向：真实多机基准、GPU 兼容性、低通信聚合、独立验证者、模型分片服务、持久化镜像与设备资源限制。每项都有范围与验收标准，见 [社区任务清单](docs/community-tasks.md)。

代码与随仓库原创玩具语料采用 [MIT](LICENSE)。训练外部数据或引入外部模型时必须单独记录来源与许可证。本项目不宣称拥有所有未来贡献者的版权，也不宣称免受任何法律约束。“共同建设”通过开放获取、独立复现、可迁移与可分叉落实，见 [GOVERNANCE.md](GOVERNANCE.md)。
