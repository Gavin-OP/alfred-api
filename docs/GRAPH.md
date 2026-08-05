# GRAPH — 引擎规范：Node / Graph / Action

> 配套 [GLOSSARY.md](./GLOSSARY.md) 与 [ADR-0018](./adr/0018-engine-terminology-node-graph-action.md)、[ADR-0019](./adr/0019-graph-as-python.md)。
> 本文件是**引擎本体的权威规范**：三层抽象如何落地、Node 怎么写、Graph 怎么画、Action 怎么落库。

---

## 1. 三层抽象一览

```
Action  ──（多个组成）──▶  Node  ──（经 Edge 串联/分支/并行/循环）──▶  Graph
 (原子写)                  (LLM抽取+out)                        (LangGraph StateGraph, Python)
```

- **Action**：引擎最底层原子写原语。1:1 映射 `services` 函数，由 `ActionExecutor` 单一落库。无 LLM、无控制流。
- **Node**：处理单元 = LLM 抽取（`prompt` + `output_schema`）→ 产出结构化 `out` → 推一组 Action 进共享 state 的 `pending_actions`。**不含控制流**。
- **Graph**：带共享 state 的 LangGraph `StateGraph`。Node 经 Edge 连成流；支持 DAG 与环（循环）。**用 Python 声明，不是 YAML**。

---

## 2. Node（配置 = `node.yaml`）

Node 是**声明式配置**，承载身份、路由、分块、prompt、output_schema。**控制流不在这里**——Node 之间怎么连由 Graph 的 Python 代码决定。

```yaml
# alfred/nodes/extract_job/node.yaml
name: extract_job
domain: job
version: 1
description: 从招聘链接/文本抽取岗位结构化数据
match:                      # 声明式预筛（L1）用
  keywords: [职位, 招聘, jd, hiring, 校招, 社招]
  url_patterns: ["*/jobs/*", "*/careers/*", "*/recruit/*"]
  modalities: [text, link]
  priority: 90
chunking:
  enabled: true
  strategy: by_section
prompt:
  system: |
    你是 Alfred 的求职抽取节点。从用户输入抽取岗位结构化数据，
    严格按 output_schema 输出，不要编造字段。
  user: "{{ input.text }}"   # Jinja2，输入来自 AlfredState.input
output_schema:               # JSON Schema，LLM 输出会被校验
  type: object
  properties:
    company: { type: string }
    title: { type: string }
    location: { type: string }
    requirements:
      type: array
      items:
        type: object
        properties:
          kind: { enum: [skill, qualification, experience] }
          text: { type: string }
          importance: { enum: [must, nice] }
  required: [company, title]
actions:                     # 抽取出的 out 经 Jinja2 映射到一组 Action（推入 pending_actions）
  - kind: upsert_company
    args: { name: "{{ out.company }}" }
  - kind: upsert_job_posting
    args: { title: "{{ out.title }}", location: "{{ out.location }}", company_name: "{{ out.company }}" }
  - for_each: "{{ out.requirements }}"
    as: req
    kind: link_requirement
    args: { kind: "{{ req.kind }}", text: "{{ req.text }}", importance: "{{ req.importance }}" }
```

### Node 处理入口（Python，薄壳）
每个 Node 一个 Python 函数，由 Graph 注册。`_run_node` 负责调 LLM、校验、把 action 推入 state：

```python
def extract_job_node(state: AlfredState) -> AlfredState:
    manifest = load_manifest("extract_job")
    out = call_llm(prompt=manifest.prompt, schema=manifest.output_schema, input=state["input"])
    pending = render_actions(manifest.actions, out=out, state=state)  # 展开 for_each/as
    state["extracted"]["extract_job"] = out
    state["pending_actions"].extend(pending)
    return state
```
> 静态流程（这个 Node 之后去哪）**不写在 Node 里**，由 Graph 的 `add_edge` 决定；只有需要**数据驱动动态路由**时才在 Node 内 `return Command(goto="...")`。

---

## 3. Graph（Python 声明，非 YAML）

Graph = LangGraph `StateGraph`。**作者写 Python**，YAML 只做 Node 配置。

```python
# alfred/graphs/job.py
from langgraph.graph import StateGraph, START, END
from alfred.engine.state import AlfredState
from alfred.nodes.extract_job import extract_job_node
from alfred.nodes.capture_experience import capture_experience_node
from alfred.nodes.record_application import record_application_node
from alfred.nodes.generate_resume import generate_resume_node

def build_job_graph() -> CompiledGraph:
    b = StateGraph(AlfredState)
    b.add_node("extract_job", extract_job_node)
    b.add_node("capture_experience", capture_experience_node)
    b.add_node("record_application", record_application_node)
    b.add_node("generate_resume", generate_resume_node)
    # 静态 Edge（控制流画在这里）
    b.add_edge(START, "extract_job")
    b.add_edge("extract_job", "capture_experience")
    # 条件路由：抽完岗位后，若用户意图是投递则走投递，否则结束
    b.add_conditional_edges("extract_job", route_after_extract,
                            {"apply": "record_application", "done": END})
    b.add_edge("record_application", "generate_resume")
    b.add_edge("generate_resume", END)
    return b.compile()
```

### 动态路由（仅数据驱动时）
```python
def route_after_extract(state: AlfredState) -> str:
    intent = state["extracted"]["extract_job"].get("user_intent")
    return "apply" if intent == "apply" else "done"
```
> 静态分支/并行/循环一律在 Graph 里声明；Node 内部返回 `Command(goto=...)` 仅用于**运行时才知道目标**的路由（如循环重试、按 LLM 输出跳不同 Node）。

---

## 4. AlfredState（共享状态）

```python
class AlfredState(TypedDict, total=False):
    input: SkillInput                 # L0 归一化后的输入（text/link/image/audio）
    chat_message_id: str | None
    thread_id: str | None
    memory_context: list[Memory]      # 召回的长期记忆（混合加权 + 时间衰减）
    extracted: dict[str, Any]         # 各 Node 的 out，按 node 名索引
    pending_actions: list[Action]     # dry_run 预览（HITL 前不落库）
    ui_intent: UiIntent | None        # 客户端下一帧该渲染什么（无头引擎产出）
    confirmations: dict[str, bool]    # 用户对各 pending action 的批准
    errors: list[str]
```

---

## 5. HITL：dry_run → commit 迁移为 interrupt / resume

沿用"脏数据不入库"原则：

1. Graph 跑完捕获类 Node，`pending_actions` 仅停留在 state（不写 DB）。
2. Graph 在提交点 `interrupt()`，把 `ui_intent`（确认预览）交回客户端。
3. 用户确认/修改后，客户端 `resume()` 带 `confirmations` 回 Graph。
4. `ActionExecutor` 仅执行被批准的 Action，真正落库；丢弃的 Action 在 `chat_message` 标注"已丢弃，原始内容保留"。

> 与 [ADR-0008](./adr/0008-services-as-single-write-path.md) 一致：**Action 是唯一写入口**，Graph 不直接写 DB。

---

## 6. 跨域 Subgraph（常驻）

`memory` / `capture` / `chat` 是**跨域 subgraph**，被外层 router 或任意 Domain Graph `add_subgraph` 复用，不归某个领域独有。例如 `capture_memory` 节点在所有领域输入后都可选触发，把自由内容写入 `memory`。

---

## 7. 约定（写代码前必读）

- Graph **只声明控制流**，Node **只描述自身处理 + Action 列表**，Action **只落库**。三层职责不越界。
- Node 配置用 `node.yaml`；Graph 用 Python；**禁止用 YAML 画控制流**。
- 新增领域 = 新建 `alfred/graphs/<domain>.py` + 若干 `alfred/nodes/*/node.yaml` + 私有扩展表（带 `user_id`）。
- 凡新增 Action，必须在 `ACTION_KINDS` 注册并在 `executor.py` 实现 handler，且支持 `dry_run`。
