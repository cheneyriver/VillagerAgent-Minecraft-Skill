# VillagerAgent / VillagerBench 论文复现指南（中文）

这份文档面向**在本仓库代码基础上**复现论文实验（或至少跑通 benchmark）的用户，目标是把“从零到跑出 `result/` 结果”说明清楚。

本仓库里常用的 benchmark（环境）主要是 4 类：
- **Meta（原子能力/评测）**：`env_type.meta`
- **Construction（建造对照蓝图）**：`env_type.construction`
- **Farming（农场做菜/合成）**：`env_type.farming`
- **Puzzle / Escape Room（合作逃脱房间）**：`env_type.puzzle`

其中你提到的“完整跑一遍 benchmark 对应的三个任务”，一般指 **Construction + Farming + Puzzle**（Meta 常用作冒烟和补充评测）。

---

## 0. 你需要准备什么

### 系统依赖
- **Python**：3.8+（建议 3.10）
- **Java**：用于 Minecraft Server（建议 `openjdk-17`）
- **Node.js + npm**：用于 Mineflayer 相关依赖

### 代码依赖（仓库内）
在仓库根目录执行：

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

npm install
python js_setup.py
```

说明：
- `npm install` 会根据 `package.json` 安装 `mineflayer` / `minecraft-data` / `mineflayer-pathfinder` 等。
- `python js_setup.py` 会 `require(...)` 这些包，用来快速确认 JS 依赖是否正确安装。

---

## 1. 配置 API_KEY_LIST（非常关键）

批量跑实验时（`config.py` / `start_with_config.py`）默认会读根目录的 `API_KEY_LIST`，其中最关键的是 `AGENT_KEY`：

```json
{
  "AGENT_KEY": ["YOUR_KEY_1", "YOUR_KEY_2"],
  "OPENAI": ["OPTIONAL_OPENAI_KEY"],
  "Qwen": ["OPTIONAL_QWEN_KEY"]
}
```

注意：
- **不要把真实 key 提交到 git**。如果你要长期使用，建议在本地自行忽略该文件（例如加入 `.gitignore`），或用环境变量/私密文件管理。
- 代码里不同位置可能用到 `OPENAI` 或 `AGENT_KEY`，但批量脚本里最常见的是 `AGENT_KEY`。

---

## 2. 启动 Minecraft 1.19.2 服务器

本仓库在 `MineCraft/MineCraft/MineCraft/` 目录下提供了启动/停止脚本（推荐优先用脚本）。

---

## 2-快速流程（脚本整合版，推荐直接照抄）

下面把“启动服务器 → 赋权 → meta 冒烟 → 跑三大 benchmark → 停止服务器”整理成尽量少跳转的脚本清单。

> 说明：`/op` 这一步必须在 Minecraft 控制台里做（或你用管理员账号进入游戏里做）。其余都在命令行做。

### 终端 A：启动 Minecraft（新世界）

```bash
cd MineCraft/MineCraft/MineCraft
chmod +x start_server_fresh.sh stop_server.sh
./start_server_fresh.sh
```

### Minecraft 服务器控制台：一次性 /op（建议）

等以下账号加入后（第一次跑任务时会自动加入），在控制台执行：

```text
/op build_judge
/op farm_judge
/op escape_judge
/op meta_judger
/op gen_judger

/op Alice
/op Bob
/op Cindy
/op David
```

> 你实际用到几个 agent，就 /op 到几个（`start_with_config.py` 默认从 `Alice` 开始取名字）。

### 终端 B：先跑 meta （推荐 2 条）

```bash
cd /path/to/VillagerAgent-Minecraft-Skill
source venv/bin/activate

python config.py --task meta --meta_task_num 2 --api_model qwen-max --host 127.0.0.1 --port 25565 --agent_num 1
python start_with_config.py --config qwen_max_launch_config_meta.json
```

### 终端 B：再跑三大 benchmark（construction / farming / puzzle）

```bash
cd /path/to/VillagerAgent-Minecraft-Skill
source venv/bin/activate

# Construction（注意：config.py 默认只生成 1 个 idx=5，需要你按论文设置改 range 才能“完整跑”）
python config.py --task construction --api_model qwen-max --host 127.0.0.1 --port 25565 --agent_num 2
python start_with_config.py --config qwen_max_launch_config_construction.json

# Farming（注意：config.py 默认只生成 1 个 idx=10，需要你按论文设置改 range 才能“完整跑”）
python config.py --task farming --api_model qwen-max --host 127.0.0.1 --port 25565 --agent_num 2
python start_with_config.py --config qwen_max_launch_config_farming.json

# Puzzle（config.py 会生成一组 max_task_num × seed 的组合任务）
python config.py --task puzzle --api_model qwen-max --host 127.0.0.1 --port 25565 --agent_num 2
python start_with_config.py --config qwen_max_launch_config_puzzle.json
```

### 终端 C：停止 Minecraft

```bash
cd MineCraft/MineCraft/MineCraft
./stop_server.sh
```

> 结果都在 `result/<task_name>/score.json`（每条任务一个目录）。

---

## 2.1 启动服务器 jar（不用脚本时）

在你的 Minecraft Server 目录下运行（示例）：

```bash
java -Xmx4G -Xms4G -jar minecraft_server.1.19.2.jar nogui
```

首次运行后请在 `eula.txt` 里把 `eula=false` 改为 `eula=true`。

---

## 3. 其他说明（模型/结果/排查）

### 3.1 结果在哪里？
- 每条任务完成后会生成目录：`result/<task_name>/`
- 关键文件：`result/<task_name>/score.json`

### 3.2 常见问题
- **bot 加不进服务器**：检查 `server.properties` 是否 `online-mode=false`
- **没权限执行命令**：按上面控制台 `/op`（judge + agent）

