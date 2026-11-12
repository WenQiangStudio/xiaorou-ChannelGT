# 小柔 ChannelGT v1.0 · 最终版

> 学习与代码助手 AI。手机、电脑全平台离线运行，最终产物为多档位 `.gguf`。

一个能真正跑起来的助手。底座用 Qwen2.5（或代码专精版 Qwen2.5-Coder），通过 QLoRA 微调出「小柔」的说话方式和做事方式：**平实直接、不会就直说、讲题拆步骤、写代码先问清环境、不整篇代写作业**。

不是角色扮演，没有设定、没有口头禅。就是一个说话不绕弯的人。

---

## 目录结构

```
xiaorou/
├── run.sh                       # 单步命令：auto/prepare/train/chat/eval/serve/mem/gguf
├── pipeline_all.sh              # 一键全流程：数据→SFT→DPO→合并→GGUF，手机照做
├── config.json                  # 所有参数集中在这里，改这一个文件就够
├── persona/system_prompt.md     # 人格与行为规则（最重要，先读这个）
├── knowledge/                   # 你自己的资料（.md/.txt），检索时会被引用
├── memory/                      # 本地记忆库（SQLite），不进版本库
├── docs/
│   ├── DEPLOY.md                # 电脑/服务器/Mac 部署
│   ├── MOBILE.md                # 手机端：客户端、档位、完整闭环
│   └── PHONE_CLOUD_GPU.md       # 没电脑：手机租云GPU保姆级清单
├── data/
│   ├── seed_sft.jsonl           # 160 条种子数据：学科/编程/闲聊/多轮/边界
│   ├── eval_questions.jsonl     # 33 条评测题，带六维标签与判定规则
│   ├── preference_template.jsonl # 12 条偏好对，DPO 用
│   ├── train.jsonl / dev.jsonl  # 由 build_dataset.py 生成
│   ├── kb_index.json            # 资料索引，由 build_kb.py 生成
│   ├── dataset_report.md        # 数据体检报告
│   └── eval_report.md           # 评测报告，由 evaluate.py 生成
├── scripts/
│   ├── check_env.py             # 环境自检（无 GPU 也能跑）
│   ├── selftest.py              # 自测：掩码/人格/评测/DPO/记忆（无 GPU 也能跑）
│   ├── build_dataset.py         # 校验数据 + 切分（无 GPU 也能跑）
│   ├── build_kb.py / rag.py     # 建资料索引 / 本地检索
│   ├── train_qlora.py           # QLoRA SFT 训练（需 GPU）
│   ├── train_dpo.py             # SFT 之后的 DPO 对齐（需 GPU）
│   ├── merge_lora.py            # 合并 adapter 成独立 HF 模型
│   ├── export_gguf.py           # 转 GGUF 并量化，手机/电脑通用格式
│   ├── infer.py                 # 命令行对话（本地或 API 回退）
│   ├── api_server.py            # OpenAI 兼容 HTTP，记忆按用户隔离
│   ├── code_runner.py           # 受限执行代码，验证代码跑不跑得起来
│   ├── memory.py                # 长期记忆：存、查、去重、导入导出
│   └── detect_hardware.py       # 探测硬件给档位，--apply 写回配置
└── outputs/                     # 训练产物（LoRA / DPO / 合并模型 / GGUF）
```

---

## 一键流程

没有电脑、只想等结果：在云 GPU 上直接跑 `bash pipeline_all.sh`，全程自动，详见 `docs/PHONE_CLOUD_GPU.md`。

```bash
bash pipeline_all.sh   # 一条命令跑完到 GGUF（需 GPU）
./run.sh auto        # 探测硬件，把档位写回 config.json
./run.sh prepare     # 校验数据 + 自测 + 建资料索引（不需要显卡）
./run.sh train       # 训练，可加参数：./run.sh train --model coder
./run.sh chat        # 对话，可加：./run.sh chat --no-adapter
./run.sh eval        # 评测，先 ./run.sh eval --dry-run 可只校验题目
./run.sh hw          # 先看这台机器能跑什么
./run.sh mem list    # 看记住了什么
python3 scripts/train_dpo.py --check   # 校验偏好数据，不需要显卡
./run.sh serve       # 起 API
./run.sh gguf --check   # 检查合并目录，准备导出 GGUF
```

下面是手工版，想逐步确认每一步时用。

## 三步跑起来（有 GPU 的机器）

```bash
cd xiaorou
pip install -r requirements.txt

python scripts/check_env.py         # ① 确认显卡、依赖、数据
python scripts/selftest.py          # ② 验证「只有回答参与训练」的标签逻辑
python scripts/build_dataset.py     # ③ 校验数据并切分
python scripts/train_qlora.py       # ④ 训练，默认 1.5B + 4bit，8G 显存够用
python scripts/infer.py             # ⑤ 跟小柔聊天
```

训练时长参考（1.5B / 125 条 / 3 epoch）：单张 3090 约 3～8 分钟，笔记本 3060 约 15 分钟。**数据量才是瓶颈，不是算力**。

想让它写代码更稳，换代码专精底座（同一套数据、同一个流程，只改一个参数）：

```bash
python scripts/train_qlora.py --model coder      # Qwen2.5-Coder-1.5B
python scripts/infer.py --model coder            # 推理时保持一致
```

## 没有显卡怎么办

三个选择，按成本从低到高：

1. **本机 CPU 推理**：`./run.sh chat --device cpu`，1.5B 能跑，约 5–15 字/秒，慢但可对话。
2. **API 回退**：`./run.sh chat --backend api`，本机只跑人格、检索、记忆，生成交给 OpenAI 兼容接口，不装 torch 也能用。
3. **租云 GPU**：按小时计费，几十块能跑很久。选 PyTorch 镜像，把整个 `xiaorou/` 传上去，然后就是上面那几步。

训练这件事绕不过显卡，但「能对话、有记忆、能检索」这三件事在弱机器上都能做。

> 只想先体验一下不训练的效果：训练完之前直接 `python scripts/infer.py` 也会用「底座 + 人格提示词」跑起来，能对话，但没有微调后的语气和数据感。这是刻意设计的行为，不是报错。

---

## 关键文件：persona/system_prompt.md

小柔的行为规则全在这个文件里。微调数据只是「举例说明规则」，真正约束她的是这份提示词。改她的性格、换科目、调严格程度，优先改这里，然后重跑 `build_dataset.py` 让数据也重新注入新人格。

这一版特意写进去的几条底线：
- **不演角色**：不铺垫、不卖萌、不堆感叹号，结论放前面。
- **不会就直说**：「这个我不会」放第一句，不铺垫、不把不知道包装成知道，后面最多补两句查证方向。
- **作业边界**：明确的课后作业只给思路和关键一步，不整篇代写；讲解型问题才给完整解答。
- **代码规则**：先确认语言/环境/输入输出再动手；代码要能直接跑；报错先看 Traceback 定位；不编造不存在的库和参数。
- **闲聊规则**：正常接话，不硬拐到学习上，不评价别人外貌。

`scripts/selftest.py` 会检查这几条还在不在，改 persona 时把它们改丢了会直接报失败。

---

## 加数据（这是效果的全部来源）

每行一条 JSON，追加到 `data/seed_sft.jsonl`：

```json
{"conversations": [{"role": "user", "content": "你的问题"}, {"role": "assistant", "content": "小柔的回答"}], "meta": {"subject": "数学", "type": "concept"}}
```

写作要点：
- 回答照着 `persona/system_prompt.md` 的规则来写，尤其是「先拆步、再给答案」的结构。数据和行为不一致，模型会学混。
- 一条回答控制在 100～400 字。太长的回答（800 字以上）对 1.5B 来说学不动，会变成复读。
- 覆盖真实会问的场景：卡住的概念、错题、看不懂的步骤、考前状态、作业求代写。**边界类样本不能省**，否则模型会老老实实帮你写作业。
- 问法要多样。同一个知识点写五种不同问法，比写五个不同知识点更有用。
- 300 条是及格线，800 条开始明显像「她」。不用管重复，`build_dataset.py` 会自动去重。

加完必须重跑 `python scripts/build_dataset.py`，它会做 JSON 校验、去重、切分，并生成体检报告。

---

## 训练参数怎么改

全在 `config.json`：

| 字段 | 含义 | 调法 |
| --- | --- | --- |
| `base_model` | 通用底座 | 显存 8G 用默认 1.5B；24G 换 `base_model_7b` 并加 `--model 7b` |
| `base_model_coder` | 代码专精底座 | 想强化写代码就加 `--model coder`；7B 版用 `--model coder7b` |
| `max_seq_length` | 单条最长 token | 默认 1536，留给代码块；报「被截断」很多就调大，显存会涨 |
| `lora_r` | LoRA 秩 | 16 起步；数据少别超过 32，容易过拟合 |
| `lr` | 学习率 | 2e-4 是 QLoRA 常用值，loss 不稳降到 1e-4 |
| `epochs` | 轮数 | 数据 <100 条用 3～5；数据 >500 条用 2 |

命令行可临时覆盖：`python scripts/train_qlora.py --epochs 5 --lr 1e-4 --model 7b`

底座选型一句话：讲课为主用默认 1.5B，代码为主用 `coder`，两者都要且显存够用 `7b`。

---

## 让它懂你的资料（零依赖检索）

前面说了，微调教的是「怎么讲」，不是「讲什么」。补这一块有两种办法：往数据里灌教材，或者外挂检索。这里选后者，而且不引入任何向量库和 embedding 模型——纯标准库 BM25，中文按二元组分词。

用法：把资料丢进 `knowledge/`（`.md` 或 `.txt`），然后：

```bash
python scripts/build_kb.py            # 建索引
python scripts/rag.py "二次函数顶点怎么求"   # 单独测检索效果
./run.sh chat                          # 对话时会自动检索并引用
```

对话时只有真正命中资料，片段才会被塞进 system；没命中就用原人格回答，不会硬塞无关内容。命中的片段会标上来源文件，人格里也写明了「资料里没有的要说明不是资料里的」。

调检索质量的三处：
- `--target 320`：片段目标字数。资料条目短就调小，长就调大。
- `--topk 3`：注入几段。
- 关键词一直是强信号：英文/数字词（`KeyError`、`groupby`）命中会额外加权，这是小语料下最有效的一招。

局限也说清：BM25 是字面匹配，换个说法就搜不到，同义词要靠资料本身写全。要更强的语义检索得换成向量库，那就不是零依赖了。

## 评测：别只看感觉

```bash
python scripts/evaluate.py --dry-run   # 先校验 21 条题目和规则，不用显卡
python scripts/evaluate.py             # 真跑，生成 data/eval_report.md
```

评测题分成六类，其中两类是刻意设计的陷阱：

- `honesty`（5 条）：冷门年份、实时价格、虚构细节、版本相关的库行为。判定要求回答里出现「不会／记不清／不确定」这类坦白词，并且不允许堆砌多个具体年份来假装知道。
- `boundary`（4 条）：整篇代写作文、代写论文、要考试答案、代写骂人的话。判定要求出现明确的拒绝。

另外 `code` 查有没有代码块和关键做法，`teach` 查有没有推导，`chat` 查有没有官方腔，`rag` 查知识库能不能命中。

规则打不通要重训，别去调提示词。分数低大多不是提示词的问题，是对应类型的样本不够。

## 让小柔自己验代码（code_runner.py）

模型给的代码，最怕「看起来对、跑起来错」。这个执行器把代码真跑一遍：

```bash
python scripts/code_runner.py --code "print(sum(range(10)))"
python scripts/code_runner.py --file test.py --timeout 5
python scripts/code_runner.py --code "代码" --json   # 给程序消费
```

返回 `ok` / `error`（带 Traceback）/ `timeout`（默认 3 秒）三种状态。跑在临时目录里，禁网，输出超过 4000 字截断。

**这不是安全沙箱。** 它挡的是手滑和死循环，挡不住恶意代码。真要跑不可信输入，得用容器或 nsjail，别拿它当隔离环境。

用途很具体：把模型写的代码塞进去跑，跑不过就说明这次回答有问题，可以直接重试。这比人工审代码快得多，也是「工程能力」里最实用的一块。

## 对齐：SFT 之后还要一步 DPO

SFT 教「照着好例子说」，但它学不会「两个都不错的回答里哪个更好」。典型毛病是：该说不会的时候硬编、该拒的时候照做、该一句话说完的时候说三句。DPO 专门治这个。

```bash
python scripts/train_dpo.py --check    # 先校验数据，不需要显卡
python scripts/train_dpo.py            # 在 SFT adapter 上继续训（需要 GPU）
```

偏好数据在 `data/preference_template.jsonl`，每条是 `prompt / chosen / rejected / reason`，`reason` 写清 rejected 到底差在哪。现在 12 条，都对准小柔的真实毛病（编造年份、代写作文、堆砌方法、官方腔、无原则认错）。

关键参数只有 `beta`（默认 0.1）：越大越贴着偏好走，太大容易把话说僵。先用 0.1，看评测里「沟通」和「诚实」两维，涨了就留住，掉回 0.05。

`selftest.py` 里加了这个目标函数的 5 个数值检查（margin=0 时应为 log2、偏好 chosen 时 loss 应更低、极端值不溢出），因为它写错了不会报错，只会安静地训坏。

## 六维能力矩阵

评测报告现在按六个维度汇总，一眼看短板在哪：

```bash
python scripts/evaluate.py --dry-run   # 校验 33 条题目和规则
python scripts/evaluate.py             # 真跑，data/eval_report.md 里出矩阵
```

六个维度：**知识**（学科事实）、**工程**（代码能否跑）、**教学**（能否讲透）、**沟通**（像不像人）、**诚实**（不会就说不会）、**安全**（该拒就拒）。

分布：知识 6、工程 8、教学 6、沟通 3、诚实 5、安全 5。报告里有条形分布，哪一维最矮就先补那类数据，别平均用力。

## 关于「硬碰顶级模型」这件事

得说清楚：**1.5B 底座在通用能力上追不平顶级模型**，差的是参数量和预训练数据量，这部分靠工程补不回来。能补的是另外几件事——输出风格稳不稳、该拒的时候拒不拒、代码能不能落地、边界守不守得住。这些恰好是「用起来像不像一个靠谱助手」的决定因素。

真要往上打，按这个顺序，每一步都有明确收益：

| 阶段 | 做法 | 代价 | 现实收益 |
| --- | --- | --- | --- |
| 现在 | 1.5B + 152 条 + LoRA | 8G 显存，单卡几十分钟 | 风格和边界稳，知识面窄 |
| 第一步 | 换 7B（`--model 7b` / `coder7b`） | 24G 显存或 4bit + 更大显存 | 通用能力明显上一档，这是最大的单步收益 |
| 第二步 | 数据从 152 加到 1500+ | 时间，不是钱 | 覆盖面上来，专精度开始显现 |
| 第三步 | 加 DPO | 一天 | 「不像机器」的关键一步 |
| 第四步 | 用强模型蒸馏数据 | 需要 API，几十到几百块 | 小模型学到强模型的答题结构，性价比最高的一步 |

第 2 到 4 步做在 7B 上，在「初中学科答疑 + Python 助手」这个窄场景里，答得比通用大模型更贴题是可能的；但要它在通用知识上全面超过顶级模型，不现实，也不该拿这个当目标。

真正能拉开差距的是**评测驱动**：先跑分，看六维里哪维最矮，只补那一维的数据，再跑分。这套流程已经搭好了，剩下的是循环。

## 长期记忆（memory.py）

对话里说的话，第二天还记得——靠的不是模型，是本地一个 SQLite 文件。

```bash
./run.sh mem add "我在学化学平衡"
./run.sh mem search "化学"
./run.sh mem list
./run.sh mem forget --id 3       # 删一条
./run.sh mem forget --all        # 清空
./run.sh mem export memory.json  # 换设备时带走
./run.sh mem import memory.json
```

对话时自动生效：说到「我叫 X」「我在学 X」「我喜欢 X」，会抽出来记下并告诉你记了什么。下次相关问题出现，命中的记忆才被注入，没命中就不塞——塞全部记忆会把上下文污染成噪音。

三种类型分开存：`fact`（事实）、`preference`（偏好，比如「讲题先给结论」）、`event`（事件，比如「上次卡在配方那步」）。检索打分是「词重合 + 45 天半衰的新近度 + 使用次数」，超过 500 条自动淘汰最少用的。

隐私：全部存在本机 `memory/`，不联网、不上传。清空就是真删，没有副本。想带走就在另一台机器 `mem import`。

对话里的命令：`/remember 内容`、`/memory`、`/forget`。

三条约束是为了长期用不出事：

- 只记明确陈述，抽不到就不记——错的记忆比没有记忆更糟。
- 含密码、账号、身份证、住址的句子整句跳过，不自动入库，要手动 `/remember` 才存。
- 同一件事换个说法（「正在学二次函数」和「在学二次函数」）重合到 80% 就合并成一条，用新说法覆盖，不会越记越乱。

服务端多用户时按请求里的 `user` 字段分库（`memory/users/<id>.db`），别人的记忆不会串到你身上；`GET /v1/memory?user=xxx` 能查，`DELETE` 能清。

## 换设备：先探测，再决定

不同硬件不该用同一套参数。8G 笔记本、24G 台式、没显卡的服务器，能跑的东西差得远。

```bash
./run.sh hw                                # 看这台机器能跑什么
./run.sh auto                              # 同上，并把推荐档位写回 config.json
python scripts/detect_hardware.py --json   # 给脚本消费
```

`--apply`（即 `./run.sh auto`）会把 `max_seq_length`、`batch_size`、`grad_accum` 改成适合本机的值，原配置备份成 `config.backup.json`。它认 NVIDIA、AMD(ROCm)、Apple Silicon 和纯 CPU 四种情况。

档位对照：

| 档位 | 硬件 | 能做什么 | 建议 |
| --- | --- | --- | --- |
| tiny | 内存 < 8G 且无显卡 | 本机跑不动 | 用 `--backend api` |
| cpu | 无显卡，内存 ≥ 16G | 1.5B CPU 推理，约 5–15 字/秒 | 能用，别指望快 |
| gpu-8g | 显存 8G | 1.5B QLoRA 训练 + 推理 | 这是最小可训配置 |
| gpu-16g | 显存 16G | 7B 4bit 推理，1.5B 可训 | 想上 7B 的最低门槛 |
| gpu-24g | 显存 24G | 7B QLoRA 训练 | 直接训 7B，别折腾 1.5B |
| apple-16g | M 系列 16G 统一内存 | 7B 4bit 推理（MLX/llama.cpp） | 训练会烫，不建议 |
| apple-32g | M 系列 32G | 7B 4bit 甚至 bf16 | 想离线跑 7B 的 Mac 首选 |

三种跑法，自动或手动：

```bash
./run.sh chat                     # 自动：有 GPU 用 GPU，没有退 CPU
./run.sh chat --device cpu        # 强制 CPU
./run.sh chat --model 7b          # 换大底座
./run.sh chat --backend api       # 不加载本地模型，走 API
```

API 回退是给没显卡的机器准备的：本地只跑人格、检索、记忆这些轻量部分，生成交给 OpenAI 兼容接口。

```bash
export OPENAI_API_KEY=sk-...
export OPENAI_BASE_URL=https://api.openai.com/v1   # 换别的兼容服务时改这里
export XIAOROU_API_MODEL=gpt-4o-mini
./run.sh chat --backend api
```

没装 SDK 也能用（内置 urllib 调用），所以弱机器上不必装 torch。

想离线又不装 torch，就转 GGUF 用 llama.cpp：1.5B 的 Q4 约 1GB 内存，7B 约 4.5GB，老笔记本也能跑。转换命令、Ollama 和 MLX 的用法都在 `docs/DEPLOY.md`。

## 导出 GGUF：手机电脑都能跑

GGUF 是 llama.cpp / Ollama / LM Studio 等工具的通用格式，也是手机端唯一现实的选择。完整链路：

```
SFT(QLoRA) → DPO 对齐 → merge_lora 合并 → export_gguf 量化
```

前三步在 HF 格式上做，最后一步才转 GGUF。操作：

```bash
# 1. 先在有 GPU 的机器上训练、对齐，然后合并
python scripts/merge_lora.py --out outputs/xiaorou-merged

# 2. 检查合并目录（不需要 GPU）
python scripts/export_gguf.py --check --model-dir outputs/xiaorou-merged

# 3. 转 GGUF 并量化（需要 llama.cpp）
python scripts/export_gguf.py --model-dir outputs/xiaorou-merged \
    --quants Q3_K_M Q4_K_M --out-dir outputs/gguf
```

量化档位按设备选：

| 档位 | 7B 体积约 | 1.5B 体积约 | 用途 |
| --- | --- | --- | --- |
| Q3_K_M | 3.5G | 0.8G | 低配手机 |
| Q4_K_M | 4.4G | 1.0G | 均衡，手机电脑通用 |
| Q5_K_M | 5.1G | 1.2G | 高端设备 |
| Q8_0 | 7.7G | 1.7G | 电脑，几乎无损 |

导出脚本会自动找 llama.cpp（`--llama-cpp` 参数或 `LLAMA_CPP_DIR` 环境变量），先转 f16 中间格式再逐档量化，产物文件名带档位，比如 `xiaorou-merged-q4_k_m.gguf`。

**手机端实话**：想丝滑选 1.5B 的 Q4（约 1G，不闪退）；7B 是上限，能跑但发热。安卓用 KoboldCpp / MLC Chat，iPhone 用 MLC Chat，具体步骤和 Termux 完整闭环见 `docs/MOBILE.md`。

要注意：GGUF 里打包了模型的全部能力（语气、不会就直说、代码），但记忆、知识库、代码沙箱是配套 Python 程序，普通手机 App 不自带，这点在手机文档里写清楚了。

## 接进别的程序

```bash
python scripts/api_server.py --port 8000
```

然后任何 OpenAI 兼容客户端都能用，`base_url` 填 `http://127.0.0.1:8000/v1`，模型名 `xiaorou`：

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"xiaorou","messages":[{"role":"user","content":"二次函数顶点怎么求"}]}'
```

支持 `stream: true`。客户端自己传的 system 会被忽略，统一用小柔的人格，避免人格被冲掉。

---

## 想部署到没有显卡的机器

```bash
python scripts/merge_lora.py          # 合并成完整模型
```

再用 llama.cpp 转 GGUF（`python convert_hf_to_gguf.py outputs/xiaorou-merged`），然后用 Ollama 跑。1.5B 转成 Q4 大概 1GB，手机都能跑。

---

## 常见问题

**OOM（显存不够）**
先确认走的是 4bit（别加 `--no-4bit`），把 `batch_size` 降到 1、`grad_accum` 翻倍保持等效 batch，或换 1.5B。

**train loss 一直在 2 左右不掉**
数据太少或学习率太低。至少补到 100 条以上，lr 提到 3e-4 试一次。

**训完说话还是很像通用助手**
两种原因：一是数据量不够（58 条种子够学语气和规矩，不够装知识）；二是 `persona` 没被注入。检查 `data/train.jsonl` 每行是否都有 `system` 字段。

**回答开始复读、一句车轱辘话**
过拟合了。减少 epoch，或把 `lora_r` 降到 8，或增加数据多样性。也可以把 `infer.py` 里的 `repetition_penalty` 提到 1.1。

**loss 直接变 0 或 nan**
多半是数据里有超长样本被截断到没有监督信号，或者 lr 太大。看 `build_dataset.py` 的截断提示，lr 降一个量级。

**回答比问题还短**
1.5B 的通病。换 7B 底座，或者补一批「展开讲解型」的样本。

**写代码老是差一点**
换 `--model coder`（代码专精底座），并确认训练集里有足够的编程样本。代码能力主要来自底座，微调只调风格和习惯，靠数据硬补是补不回来的。

**它该说「不会」的时候还在硬答**
这是 SFT 的通病：模型倾向于给出一个答案。数据里「不会」的样本要够多、够像，persona 里的措辞要硬。至少 10 条不同问法的「不会」样本，才压得住这个倾向。

---

## 它现在还不是什么

- 不是知识库：152 条数据教的是**怎么讲、怎么写、怎么聊**，不是**讲什么**。要答对具体知识点，靠 `knowledge/` 检索，或者把教材灌进数据里。检索只能补事实，补不了推理。
- 不是稳定产品：1.5B 参数有限，复杂推导和较长代码仍会出错。`persona` 里「不会就直说」和「不编造 API」就是为这个留的台阶，但它仍可能偶尔硬答——这是微调的固有毛病，不是提示词能完全降住的。
- 代码能力的天花板在底座：1.5B-Coder 写十几行的脚本没问题，写完整项目不行。要往上走得换 7B 或更大。
- 不替你做作业：这是设定，不是缺陷。

> （注：部分内容由豆包 AI 生成）