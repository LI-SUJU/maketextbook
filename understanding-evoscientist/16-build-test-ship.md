# Chapter 16 — Build, Test, and Ship

> **This chapter answers:**
> - How do you build, test, and contribute to EvoScientist as a developer?
> - How do you test an AI agent *without* ever calling a real (nondeterministic, paid) LLM?
> - Why are certain dependency versions pinned with comments that read like postmortems?
> - What does the CI pipeline actually gate, and why is it split into four separate workflows?

Every chapter until now has zoomed into a region of the master diagram that a *user's request* flows through: a surface hands a prompt to the gateway, the gateway drives the orchestrator's model⇄tools loop, middleware wraps each call, memory distills the turn. This chapter zooms into the one region drawn deliberately *to the side* of that runtime map — the region a request never touches, but every line of code passes through before it ships: **build, test, and CI**. This is the developer's map, not the agent's. It answers a different question than "how does a request run?" It answers "how does a contributor take an idea, turn it into code that provably works, and get it into a released version — all without an OpenAI bill and without a flaky test?"

That last clause is the whole point of this chapter, and it hides a genuinely hard problem. EvoScientist *is* an LLM agent. Its central behavior — the ReAct loop (the reason→act→observe cycle, Ch 3), the HITL pause before a dangerous tool (human-in-the-loop, Ch 7), the delegation to sub-agents (Ch 6) — is orchestrated by a model whose outputs are nondeterministic and whose calls cost money and take seconds. A test suite that called a real model would be slow, expensive, flaky, and would need API keys in CI. So how does a project whose *entire reason for existing* is "put a model in a loop" test that loop hundreds of times in fifteen seconds, offline, for free? The answer — a scripted fake model driving the *real* graph — is the crown jewel of this chapter, and we build up to it deliberately. But first, the ordinary machinery: how a contributor works day to day.

---

## 1. The contributor lifecycle, end to end

Before any deep mechanism, hold the whole loop in your head. A contributor's life on EvoScientist is a short, repeatable cycle:

```
clone → uv sync --dev → edit code → uv run pytest → uv run ruff → git push / PR
                                          ↑                              │
                                          └──────── fix ─────────────────┘
                                                                         ↓
                                              4 CI workflows must pass → merge
                                                                         ↓
                                              version bump + v* tag → PyPI + ghcr release
```

`CONTRIBUTING.md:35` spells out the human-facing version of this: "Run the test suite (no API keys needed): `uv run pytest`". The parenthetical is not a throwaway — it is a promise the test architecture works hard to keep, and §3 is the story of how. But notice the tool at the front of nearly every step: **uv**. Before we can read a single test, we need to know what `uv` is, because it is how the project is installed, how tests are run, and how the wheel is built. It is the ground the rest stands on.

---

## 2. uv: one tool for dependencies, environments, builds, and running

### Intuition

If you come from Python, you have met the traditional split of jobs: `pip` installs packages, `venv` creates isolated environments, `pip-tools` or `poetry` lock exact versions for reproducibility, and `python -m build` produces a distributable artifact. Four concerns, historically four tools, and a lot of "works on my machine" pain gluing them together.

**uv** — glossed in the ledger as *Astral's Rust-based Python package/project manager* — collapses all four into one binary. Astral is the company behind `ruff` (the linter you'll meet in §5); `uv` is their package-and-project manager written in Rust, which is why it is fast enough that people notice. For this chapter you can think of `uv` as "poetry, but faster, and it also creates the virtualenv and runs your commands inside it for you." If you have never used poetry either, the plain mental model is: **one command that reads a spec file, computes an exact set of package versions, installs them into a private environment, and then runs your code against exactly those versions.**

### Mechanism

Two files drive `uv`, and they play distinct roles:

- `pyproject.toml` — the *human-authored* spec. It lists dependencies with loose ranges ("I need `langchain>=1.3`"), the metadata that will go into the built package, and tool configuration. This is what a developer edits.
- `uv.lock` — the *machine-generated* resolution. It records the single exact version of *every* package (direct and transitive) that satisfies the spec, plus content hashes. In this repo it is a real, checked-in, ~1 MB file. This is what makes an install *reproducible*: two developers who run against the same lock get byte-identical dependency trees.

The commands you saw in the lifecycle map onto these files:

| Command | What it does |
|---|---|
| `uv sync --dev` | Read `pyproject.toml` + `uv.lock`, create `.venv/`, install everything including the dev group |
| `uv run pytest` | Run `pytest` *inside* that environment (no manual `activate` needed) |
| `uv run ruff check .` | Same, for the linter |
| `uv build` | Produce the distributable sdist + wheel |
| `uv sync --frozen` | Sync but **refuse to update the lock** — fail if `pyproject.toml` and `uv.lock` disagree |

The `--frozen` flag is worth pausing on, because it is exactly the guarantee CI and Docker need. A plain `uv sync` will, if it notices `pyproject.toml` changed, silently re-resolve and rewrite `uv.lock`. That is convenient at your desk and *dangerous* in an automated build — you want the build to use the exact versions someone reviewed and committed, not whatever the resolver picks today. `--frozen` turns "helpfully update the lock" into "error out if the lock is stale," converting a silent drift into a loud failure. We will see it in the Dockerfile in §7.

### Detail: dependency groups vs. optional-dependencies

Open `pyproject.toml` and you find something that looks redundant: the dev tooling is declared *twice*.

```toml
[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=1.0",
    ...
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    ...
]
telegram = ["python-telegram-bot>=21.0"]
```
`pyproject.toml:52-72`

These two blocks look like a copy-paste mistake, but they answer two different questions, and the distinction is worth teaching because it trips up nearly everyone.

`[project.optional-dependencies]` (a standard from PEP 621) declares *extras* — named bundles a **consumer** of your package can opt into when they install it. Because the package publishes them, an end user who runs `pip install "EvoScientist[telegram]"` gets the Telegram bot library pulled in. Each channel EvoScientist supports (Telegram, Discord, Slack, WeChat, Feishu, QQ) is one extra, plus `stt`, `oauth`, and an `all-channels` superset — this is how the wheel stays lean by default while letting users add exactly the integrations they want (`pyproject.toml:73-97`).

`[dependency-groups]` (the newer PEP 735 standard) declares *development-only* groups that are **not** published with the package — a consumer installing EvoScientist from PyPI can never accidentally pull in `pytest`. They exist purely for people working *on* the repo, via `uv sync --dev`.

So why is `dev` in *both* blocks? Inference from the shape of the file: the `optional-dependencies` `dev` is a compatibility shim so that a plain `pip install ".[dev]"` — for a contributor who has not adopted `uv` — still works, while the `dependency-groups` `dev` is the `uv`-native, unpublished path. The project is hedging across two eras of Python packaging so that neither a `uv` user nor a `pip` user is left out. The lesson generalizes: **extras are for your users; groups are for your maintainers**, and if you need to serve both installers you may declare the maintainer set in both places.

One more line rewards a look — the build backend:

```toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"
```
`pyproject.toml:110-112`

`uv` manages *and runs* the build, but it does not *replace* the thing that actually assembles the wheel. That job is delegated to **setuptools**, the oldest and most compatible build backend in the ecosystem. This is a conservative, deliberate choice: `uv` has its own faster backend, but setuptools is what every pip in the world already understands, and it lets the project do one thing `uv`'s backend makes fussier — ship data files. The `[tool.setuptools.package-data]` block (`:117-122`) declares that `subagents/*.yaml`, `langgraph_dev/*.json`, and the *entire* `skills/**/*` tree are packaged into the wheel as data. That is how a `pip install` of EvoScientist arrives with its sub-agent definitions (Ch 6) and its built-in skills (Ch 12) already on disk — they are not code, they are baggage the build carries along.

---

## 3. The crown jewel: testing an agent without an LLM

Now the hard problem. We have an agent whose behavior is produced by a model. We want hundreds of fast, deterministic, offline tests. The naïve approaches all disappoint, and seeing *why* they disappoint is the fastest route to appreciating what EvoScientist actually does.

### Intuition: what doesn't work, and the move that does

You could **mock the HTTP layer** — intercept the outgoing request to the model provider and hand back a canned JSON response. This works, but it is brittle in a specific way: you are now testing against your *belief* about what the provider's wire protocol looks like. Change providers, or let the provider tweak its streaming format, and your mock is a lie your tests believe. Worse, mocking HTTP tests the *plumbing to the model* while leaving the part you actually care about — the LangGraph state machine, the middleware onion, the tool-calling loop, the interrupt logic — running against a fake shape rather than the real graph.

You could **mock the whole agent** — replace `create_deep_agent` with a stub that returns scripted events. Now you are not testing the graph at all; you are testing your stub. Any bug in how the *real* graph wires the model node to the tool node, or how an interrupt propagates through super-steps, sails straight past.

EvoScientist makes a different, more surgical move, and it is the single most important idea in this chapter:

> **Substitute a fake *chat model* — a scripted, deterministic object that satisfies LangChain's chat-model interface — into the *real* `create_deep_agent()`. Everything above the model is genuine: the actual LangGraph StateGraph, the real ToolNode, the real `interrupt_on` HITL wiring, the real `InMemorySaver` checkpointer. Only the one nondeterministic, paid, slow component — the model itself — is replaced with a list of pre-written answers.**

Recall from Ch 3 that a chat model is just "the LLM wrapped behind a uniform interface that takes messages and returns a message." Because that interface is narrow and well-defined, you can implement it with a class that ignores its input and returns the *next* answer off a scripted list. The agent framework cannot tell the difference — it calls the same method, gets back an `AIMessage` (possibly with tool calls), and drives its loop exactly as it would with a real model. The test gets to assert on the *behavior of the real machine* while feeding it a deterministic script. Hermetic (no network), fast (no inference), and free.

The linchpin is a class LangChain ships for exactly this purpose: **`FakeMessagesListChatModel`** — a chat model whose "intelligence" is a Python list of messages it returns in order. EvoScientist subclasses it with a two-line tweak and feeds it into the genuine graph.

### Mechanism: the fake model, in three lines

Here is the entire subclass, at the top of the deepest test file in the suite:

```python
class _ToolCallingFakeModel(FakeMessagesListChatModel):
    def bind_tools(self, tools, *, tool_choice=None, **kwargs):
        return self
```
`tests/test_stream_events.py:41-43`

That is it. `FakeMessagesListChatModel` already does the important part — you construct it with `responses=[...]`, a list of `AIMessage` objects, and each time the graph invokes it, it hands back the next one. The only thing the base class lacks for our purposes is `bind_tools`. Recall from Ch 3 that before an agent runs, LangChain calls `model.bind_tools(tools)` so the model knows the schemas of the tools it may call; the returned object is what actually gets invoked in the loop. A real model's `bind_tools` returns a configured copy that will emit tool calls. The fake model does not need to *decide* to call tools — the test author already scripted the tool calls into the `responses` list — so `bind_tools` just returns `self` unchanged, a no-op. Three lines convert LangChain's generic fake into a fake that survives being wired into a real tool-calling agent.

### Detail: following one request end to end — the HITL test

Now we walk the single best "follow a request through the whole system" test in the codebase: `test_live_deepagents_v3_hitl_emits_tool_call_and_single_interrupt` (`tests/test_stream_events.py:689-741`). It exercises, in one hermetic function, the ReAct loop, tool calling, the HITL interrupt (Ch 7), the checkpointer (Ch 13), and the v3 streaming event vocabulary (Ch 15) — every layer this book has taught, driven by a scripted model. Read it in four beats.

**Beat one — a real tool.** The test defines an ordinary tool with the `@tool` decorator (Ch 3):

```python
@tool
def echo_tool(value: str) -> str:
    """Echo a deterministic value."""
    return f"echo:{value}"
```
`tests/test_stream_events.py:692-695`

This is not a fake. It is a genuine LangChain tool that the real `ToolNode` would execute. Its output is deterministic (`echo:ok`) so the test can assert on it, but structurally it is exactly what a production tool looks like.

**Beat two — a scripted model.** The fake model is constructed with a single response: an `AIMessage` carrying one tool call.

```python
model = _ToolCallingFakeModel(
    responses=[
        AIMessage(
            content="",
            tool_calls=[
                {
                    "name": "echo_tool",
                    "args": {"value": "ok"},
                    "id": "call_echo_1",
                }
            ],
        )
    ]
)
```
`tests/test_stream_events.py:697-710`

This is the script. When the graph's `"model"` node runs, this exact `AIMessage` comes back — an empty text body and one tool call requesting `echo_tool(value="ok")`. Note there is *no second response* in the list. That is deliberate: the test expects the graph to pause at the interrupt *before* it ever asks the model a second time, so a second scripted answer would never be consumed. The shape of the `responses` list encodes the expected control flow.

**Beat three — the real agent, with real HITL.** Here is the move that makes this test valuable rather than a toy:

```python
agent = create_deep_agent(
    model=model,
    tools=[echo_tool],
    system_prompt="Use tools when requested.",
    interrupt_on={"echo_tool": True},
    checkpointer=InMemorySaver(),
)
```
`tests/test_stream_events.py:711-717`

Everything here is production machinery. `create_deep_agent` (Ch 4) is the *actual* deepagents factory that compiles the *actual* LangGraph StateGraph. `interrupt_on={"echo_tool": True}` is the real HITL configuration that tells the graph to pause and surface an approval request before executing `echo_tool` — the same `interrupt()` primitive from Ch 3 that Ch 7 built the human-in-the-loop stance on. `InMemorySaver` is LangGraph's real in-memory checkpointer (the ledger-defined checkpointer, Ch 3/13 — here the throwaway in-memory variant rather than the `sessions.db`-backed `PruningCheckpointer`, because the test needs no durability, only the ability to pause and resume within one run). The only fake object in the whole construction is `model`. The test is asserting against the behavior of the genuine graph.

> A subtlety worth flagging: Ch 7 taught you that EvoScientist's *production* code deliberately does **not** use the `interrupt_on=` kwarg — PR #202 showed it cascades to every sub-agent and breaks parallel `task` calls, so production attaches HITL as a main-agent-only middleware instead. This test uses the raw `interrupt_on=` kwarg anyway, and that is fine: the test's job here is to verify that the *deepagents/LangGraph HITL mechanism itself* emits one tool call and exactly one interrupt in the right order. It is testing the substrate the production middleware is built on, at its simplest entry point.

**Beat four — drive the real streaming pipeline and assert.** The test does not call the agent directly. It runs it through `stream_agent_events` — the very same v3 event translator from Ch 15 that every surface (CLI, channels, headless JSON) consumes:

```python
events = [
    event
    async for event in stream_agent_events(agent, "run echo", "live-deepagents-hitl")
]

tool_calls = [e for e in events if e.get("type") == "tool_call"]
interrupts = [e for e in events if e.get("type") == "interrupt"]
assert tool_calls == [{
    "type": "tool_call", "name": "echo_tool",
    "args": {"value": "ok"}, "id": "call_echo_1",
}]
assert len(interrupts) == 1
assert events.index(tool_calls[0]) < events.index(interrupts[0])
assert interrupts[0]["action_requests"][0]["name"] == "echo_tool"
```
`tests/test_stream_events.py:719-741`

Read what these assertions actually prove. The scripted model emitted a tool call; the *real* graph turned that into exactly one `tool_call` streaming event (the Ch 15 vocabulary), then paused; the *real* HITL wiring produced exactly *one* `interrupt` event (not zero, not two — the "single" in the test name is guarding against a real bug class where an interrupt double-fires); and the ordering assertion (`index(tool_call) < index(interrupt)`) confirms the user sees "the agent wants to run echo_tool" *before* "approve?", which is the whole UX point of HITL. Finally, the interrupt carries an `action_requests` payload naming the tool and its arguments — the data a surface needs to render the approval prompt.

Sit with what this one function covers: a request enters (`"run echo"`), flows through the real StateGraph's model node, produces a real tool call, hits the real interrupt, and comes out the far end as the exact normalized events three different renderers rely on — and it runs in milliseconds with no network and no key. This is the payoff of the "substitute the model, keep the machine" design. It is the reason the book's Ch 3–15 mechanisms are *testable at all*, and it is why the dossier nominates this exact test as the canonical "follow one request end-to-end" artifact.

### The other two fakes: faking around the graph, not inside it

The fake *model* handles the agent's core loop. Two more fake modules handle the two *boundaries* the agent talks across, so that surfaces and streaming can be tested without a running graph either.

`tests/fakes.py` (~650 lines) fakes the **outer** boundary — the world the gateway talks to. It provides a full in-memory stand-in for the LangGraph SDK client (`FakeLangGraphClient` and friends, built on plain dicts), plus `FakeCommandUI`, real `Channel` subclasses (`StubChannel`, `QueueFakeChannel`), a `FakeThreadStore`, and a `FakeGraphGateway`. Recall from Ch 15 that the `GraphGateway` Protocol is the seam giving every surface one thread/stream API over *either* local or remote graph execution. Because it is a Protocol (structural typing), a test can hand any surface a `FakeGraphGateway` and drive it without ever standing up a `langgraph dev` subprocess (Ch 14). The fakes import `LangGraphClient` from the real SDK (`tests/fakes.py:11`) precisely so they stay type-compatible with the code under test.

`tests/stream_v3_fakes.py` fakes the **inner** boundary in the other direction — the raw event protocol the graph emits. Where the HITL test drove a real graph and let it produce real v3 events, these builders let a test *hand-write* raw DeepAgents v3 protocol events and feed them straight into `stream_agent_events` (`tests/stream_v3_fakes.py:11`, `:42-77`). Functions like `protocol_event`, `message_delta`, and `message_finish` construct the exact wire-shaped dicts the v3 streaming protocol emits (`{"type": "event", "method": "messages", "params": {...}}`), and `FakeV3Agent` / `HangingV3Agent` / `ErroringV3Agent` play them back — the last two on purpose modeling a stream that hangs or errors mid-flight, so the Ch 15 event processor's timeout and error handling can be tested against adversarial inputs a real model would rarely produce on demand. This ties directly back to Ch 15: it lets the *translator* be tested in isolation from the *graph*, complementing the HITL test's approach of testing them together.

Three fakes, three altitudes: `_ToolCallingFakeModel` replaces the model *inside* the real graph; `stream_v3_fakes` replaces the graph's raw output *beneath* the translator; `fakes.py` replaces the SDK/gateway world *around* the whole thing. Between them, every layer this book taught can be exercised offline.

### The one place a real model is still (partly) touched

For completeness: not every test drives a graph. The provider-registry tests (Ch 8) work at a lower level — they patch `init_chat_model` itself and assert on the *arguments* the registry passes it, never constructing a model at all:

```python
@patch("EvoScientist.llm.models.init_chat_model")
def test_uses_default_model_when_none(self, mock_init):
    get_chat_model()
    call_kwargs = mock_init.call_args[1]
    expected_model_id, expected_provider = MODELS[DEFAULT_MODEL]
    assert call_kwargs["model"] == expected_model_id
    assert call_kwargs["model_provider"] == expected_provider
```
`tests/test_llm.py:163-175`

This is a different, simpler archetype — a *contract test* on the provider-routing table (Ch 8), verifying that "the default model resolves to this model_id and this provider" without any graph or fake model. The two techniques coexist: contract tests for the registry's *inputs*, the fake-model integration tests for the graph's *behavior*.

---

## 4. Test isolation: two war stories

A hermetic test suite has a second enemy besides the model: **the developer's own machine**. A test that passes because of a stray file in your home directory is a test that will fail on someone else's, or in CI, or when pytest runs the files in a different order. Two `autouse` fixtures — fixtures that run automatically around *every* test without being requested — encode hard-won lessons about this, and both read like small incident reports.

> ### 事故档案 / Origin story: the leaking `.env` (issue #322)
>
> **背景 / Background.** EvoScientist reads configuration through `get_effective_config`, which — to honor the config precedence ladder from Ch 10 — calls `load_dotenv(find_dotenv(usecwd=True), override=True)`. That is: "find the nearest `.env` file walking up from the working directory, and load its keys into `os.environ`, overriding what's there." Perfectly reasonable at runtime.
>
> **经过 / What happened.** In a test run, the working directory *is the repo*, and the repo has a real `.env` (a developer's, gitignored, holding their keys and base URLs). So the first test that touched config loaded that developer's `.env` into `os.environ` — and `os.environ` persists for the *entire pytest process*. Worse than leaking a secret: a line like `MINIMAX_BASE_URL=` (present but *empty*) made `os.environ.get(key, default)` return `""` instead of the intended default, silently breaking *unrelated* tests that ran later. A test's pass/fail now depended on whether some earlier test happened to load config, and on the exact contents of one developer's machine.
>
> **代价 / Cost.** Order-dependent, machine-dependent failures — the worst kind, because they vanish when you try to reproduce them and reappear in CI. Issue #322.
>
> **机制化 / Mechanized.** The `_isolate_dotenv` autouse fixture (`tests/conftest.py:153-171`) monkeypatches `find_dotenv` to return a fixed path that *does not exist*, so `load_dotenv` becomes a harmless no-op for every test. The docstring on the fixture is itself the postmortem — it names the empty-value trap and cites #322. And note the small craft in the fix: rather than pointing at a temp directory (which would allocate a fresh temp dir per test), it points at one fixed nonexistent path (`tests/conftest.py:7`), making the fixture cheap enough to run on all ~hundreds of tests without a thought.

The second story is subtler and teaches a real `pytest` gotcha. EvoScientist's dangerous-mode flag (Ch 9) is applied via a *direct* `os.environ` assignment (`apply_config_to_env`), not through pytest's `monkeypatch`. That matters because of how `monkeypatch` undoes things:

```python
@pytest.fixture(autouse=True)
def _restore_dangerous_env():
    _sentinel = object()
    _prev = os.environ.get("EVOSCIENTIST_DANGEROUS_MODE", _sentinel)
    yield
    if _prev is _sentinel:
        os.environ.pop("EVOSCIENTIST_DANGEROUS_MODE", None)
    else:
        os.environ["EVOSCIENTIST_DANGEROUS_MODE"] = _prev
```
`tests/test_config.py:33-48`

The docstring names the trap exactly: "monkeypatch's `delenv` of an originally-absent key records no undo." That is, `pytest`'s `monkeypatch` fixture knows how to restore a variable it *changed*, but if you ask it to `delenv` a variable that was never set, there is nothing for it to restore, so it records no undo action — and meanwhile the code under test set the variable via raw `os.environ`, which `monkeypatch` never saw. The result: a test that flips dangerous mode on leaks `EVOSCIENTIST_DANGEROUS_MODE=1` into every later test, an order-dependent landmine (the docstring calls out `pytest-randomly`, which shuffles test order precisely to surface such bugs). The fixture defends manually with the classic snapshot/restore idiom, using a unique `_sentinel` object to distinguish "was absent" from "was set to empty string" — because those two states must be restored differently, and `None` can't tell them apart.

There is a third, quieter isolation mechanism worth naming because it protects the fake-model story itself. Recall from Ch 6 that EvoScientist monkeypatches deepagents' *private* async-subagent internals (we return to this in §6). If a test triggers that patch as a side effect, later tests would run against patched internals. So `conftest.py:107-120` captures the *original* deepagents functions at conftest load time — before any test can import EvoScientist and trigger the patch — giving the `restore_model_passthrough_patch` fixture a stable "truly unpatched" baseline to reset to (`tests/conftest.py:123-150`). The lesson across all three: **an autouse fixture is where a project writes down what it learned the hard way about its own global state.**

---

## 5. ruff and pre-commit: style as a machine-checkable gate

With correctness handled by pytest, *consistency* is handled by **ruff** — Astral's Rust-based linter and formatter (the same company as `uv`). "Lint" means static analysis that flags likely bugs and style violations without running the code; "format" means mechanically reshaping code to a canonical layout (indentation, quote style, line breaks). Historically these were two tools (`flake8` + `black`); ruff does both, fast.

The configuration lives in `pyproject.toml` under `[tool.ruff]` and `[tool.ruff.lint]`. The selected rule groups (`:168-179`) read as a curated set: `E`/`W` (pycodestyle), `F` (pyflakes — undefined names, unused imports), `I` (isort — import ordering), `B` (bugbear — likely-bug patterns), `C4` (comprehension performance), `UP` (pyupgrade — modernize syntax), `PT` (pytest style), plus pylint errors, ruff-specific rules, and refurb. One deliberate exclusion stands out: `E501` (line too long) is ignored (`:180-182`) with the comment "Formatter takes care of that" — the formatter reflows lines to the 88-character limit, so flagging length as a *lint error* would be redundant noise. This is a small instance of a good principle: don't have two tools police the same thing and disagree.

`pre-commit` is a git hook framework that runs checks automatically when you `git commit`, catching problems before they ever reach a push. The config (`.pre-commit-config.yaml`) pins ruff's pre-commit hook to a specific version (`v0.15.17`) and runs `ruff-check --fix` then `ruff-format`. Note what it does *not* include: a pytest hook. Tests can be slow; blocking every commit on the full suite would be hostile, so pre-commit stays a fast style gate and leaves behavior to CI. There is an honest wrinkle here worth flagging in the book's spirit of "the repo is the law": pre-commit runs ruff *through the hook framework*, while CI runs `uv run ruff check` *directly* (§6), and the two pin ruff via different mechanisms — the hook pins `v0.15.17`, CI takes whatever `uv sync --dev` resolves under `ruff>=0.5`. These two paths *can* drift, and nothing forces them to agree. The project accepts that small risk in exchange for pre-commit being a lightweight local convenience rather than an exact mirror of CI.

---

## 6. Pinning as an incident log; monkeypatching a private API

Open `pyproject.toml`'s dependency list again and read the *comments*. They do not describe what a package is; they describe how it *hurt someone*:

```toml
    # 0.11.0+ SSE regressions: teardown noise + mid-turn ReadTimeout
    "openrouter>=0.10.8,<0.11.0",
    ...
    # <8.2.7: 8.2.7 Kitty "report-all-keys" breaks CJK input on iTerm2
    "textual>=8.0,<8.2.7",
```
`pyproject.toml:28-29`, `:48-49`

You met the first of these as a full origin story in Ch 8 — the `openrouter<0.11` cap (commit e9857dd: the maintainers first tried patching the SDK's SSE stream leak, then abandoned the patch and pinned the version instead — the "patch vs. pin" lesson). The second is `textual<8.2.7`, and its comment tells the whole story in one line: a terminal-library update whose new Kitty "report-all-keys" key-reporting mode broke CJK (Chinese/Japanese/Korean) input on the iTerm2 terminal, so the TUI framework is capped just below it. We do not dwell on either; the point for *this* chapter is structural. **A pin with a comment is a permanent, code-adjacent postmortem.** A future contributor who sees `openrouter<0.11` and thinks "let me bump this, it's out of date" reads the comment and learns, in one line, that someone already tried and it went badly. The dependency file has become an incident log that travels with the code and is impossible to lose.

That is the *loose* end of the dependency-safety spectrum: cap a version and write down why. The *tight* end is a fresh example this chapter owns — what to do when you depend not just on a package's version but on its *private internals*.

### Living dangerously with someone else's private API

Recall from Ch 6 that async sub-agents run as remote graphs in the `langgraph dev` subprocess, and that EvoScientist needs to inject the CLI's chosen model into deepagents' async-launch machinery. The trouble is that the functions doing that launch — `_build_start_tool` and `_build_update_tool` — are *private* to deepagents (the leading underscore is the universal Python convention for "not part of the public API; may change without notice"). EvoScientist monkeypatches them anyway. This is genuinely risky: a deepagents upgrade could rename or remove either function, and a naïve patch would crash the CLI at startup with an `AttributeError`.

The pin `deepagents[quickjs]~=0.6.12` (`pyproject.toml:19`) is the first line of defense. The `~=` operator is the **compatible-release** specifier: `~=0.6.12` means ">= 0.6.12 but < 0.7" — accept patch bumps within the `0.6.x` line, refuse the `0.7` line that might reshape internals. But a version cap is a blunt instrument; it can't protect against a private function vanishing in a patch release. So the *code* is written to degrade gracefully:

```python
def _patch_deepagents_model_passthrough() -> None:
    global _model_passthrough_patched
    if _model_passthrough_patched:
        return

    try:
        from deepagents.middleware import async_subagents as ds_mod
    except ImportError:
        return

    orig_build_start = getattr(ds_mod, "_build_start_tool", None)
    orig_build_update = getattr(ds_mod, "_build_update_tool", None)
    if orig_build_start is None or orig_build_update is None:
        return
    ...
    ds_mod._build_start_tool = _patched_build_start
    ds_mod._build_update_tool = _patched_build_update
    _model_passthrough_patched = True
```
`EvoScientist/llm/patches.py:1201-1237`

Three defenses stack here, and each earns its place. First, the **idempotent flag** (`_model_passthrough_patched`, checked at `:1210`): the patch is safe to call on every CLI startup and from `_maybe_swap_async_subagents`, because a second call is a no-op — you never double-wrap. Second, the **import guard** (`try/except ImportError`, `:1213-1216`): if the module isn't even there, quietly do nothing. Third — the load-bearing one — the **`getattr(..., None)` fallback** (`:1222-1225`): rather than accessing `ds_mod._build_start_tool` directly (which raises if it's gone), the code *asks* for it with a default of `None`, and if either private function has been renamed or removed, `_patch_deepagents_model_passthrough` simply returns without patching. The comment at `:1218-1221` notes this pattern is used consistently throughout the file. The effect: a deepagents update that reshapes these internals costs EvoScientist a *degraded feature* (async model passthrough silently off) instead of a *crash at startup*. This is the same defense-in-depth stance you saw in Ch 9's sandbox — assume the thing you don't control will change, and fail soft. **Version pin plus getattr-guarded monkeypatch is how you responsibly depend on an API you were never promised.**

### Dependabot: automate the boring pins, hand-curate the dangerous ones

This same philosophy explains an otherwise-odd configuration choice. **Dependabot** is GitHub's bot that watches your dependencies and opens PRs to bump them. EvoScientist scopes it to exactly one ecosystem:

```yaml
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    ...
    groups:
      base-images:
        patterns: ["*"]
```
`.github/dependabot.yml`

Docker base images *only*. Python dependencies are deliberately *not* automated. Why automate one and not the other? Because the two carry different risk profiles. A Docker base-image bump — a new digest for the same `python3.11-trixie-slim` — is almost always a security patch to the OS layer, mechanical and safe, and the config even groups all image bumps into a *single* weekly PR (the comment: "keeps reviewer load low and lets us validate trixie/uv/node bumps as a coherent set"). But a Python dependency bump is exactly the class of change the incident-log comments warn about: `openrouter 0.11` and `textual 8.2.7` were *newer versions* that broke things in subtle, hard-to-catch ways an automated bot would happily propose. So those pins are **hand-curated** — a human reads the changelog, tests, and decides. The rule the config embodies: **automate the changes that are mechanical and low-risk; keep a human in the loop for the changes that have already bitten you.** It is the same human-on-the-loop instinct that governs the product (Ch 1), applied to the supply chain.

---

## 7. The Docker image: five concepts in one file

The Dockerfile packages EvoScientist into a container — a self-contained, runnable image. It is short enough to read as a walkthrough, and it demonstrates a handful of production-Docker techniques worth naming. Read it top to bottom as three stages that pass artifacts down a line.

**The header and the pinned bases.** The file opens with `# syntax=docker/dockerfile:1.7`, which opts into BuildKit's modern Dockerfile features (we need one of them below). Then the base images:

```dockerfile
ARG BASE_IMAGE=ghcr.io/astral-sh/uv:python3.11-trixie-slim@sha256:7936cc66...
ARG NODE_IMAGE=node:24-trixie-slim@sha256:735dd688...
```
`Dockerfile:3-4`

Both bases are **digest-pinned** — the `@sha256:...` suffix nails the image to an exact content hash, not a moving tag. `python3.11-trixie-slim` could be repointed at new content tomorrow; the digest cannot. This is the container equivalent of `uv.lock`: reproducibility by pinning to content, not names. And these two `ARG`s are exactly what Dependabot bumps (§6) — the comment in `dependabot.yml` even says it reads "ARG-bound base refs."

**Stage one — nodejs, copy-only.** `FROM ${NODE_IMAGE} AS nodejs` (`:6`) pulls in a full Node.js image, but *only so later stages can copy the binaries out of it*. Node is needed at *runtime* (for MCP servers, Ch 10, and the QuickJS code interpreter, Ch 9) but not for *building* the Python package. Rather than install Node into the final image with a package manager, the build lifts just the `node`, `npm`, and `npx` binaries from an official image (`:39-42`). This is a clean way to vendor a foreign-ecosystem dependency without its whole toolchain.

**Stage two — the uv builder, split for cache.** `FROM ${BASE_IMAGE} AS builder` (`:9`) is where dependencies get installed, and it is written for maximum cache reuse across builds:

```dockerfile
COPY pyproject.toml uv.lock README.md ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev --extra all-channels

COPY EvoScientist ./EvoScientist
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-editable --extra all-channels
```
`Dockerfile:18-26`

Two techniques here. First, the **layer split**: the build copies *only* `pyproject.toml` and `uv.lock` first and installs dependencies, *then* copies the application source and installs the project itself. Docker caches each layer, and a layer is only rebuilt if its inputs changed. Because dependencies change far less often than source code, this split means editing an EvoScientist `.py` file re-runs only the fast second `uv sync`, not the slow dependency download — the expensive layer stays cached. Second, `--mount=type=cache,target=/root/.cache/uv` is a **BuildKit cache mount** (the feature the `# syntax` header unlocked): it persists `uv`'s download cache *across builds* without baking it into the image, so even a cache-busting change reuses already-downloaded packages. And note `uv sync --frozen` (§2) — the build refuses to silently re-resolve; if `uv.lock` is stale, the build *fails*, which is exactly what you want in an automated pipeline. The `UV_*` environment variables at `:11-14` tune this: `UV_COMPILE_BYTECODE=1` precompiles `.pyc` for faster startup, `UV_PYTHON_DOWNLOADS=never` forbids `uv` from fetching its own Python (the base image already has one), and `UV_PROJECT_ENVIRONMENT=/opt/venv` puts the venv at a fixed path the runtime stage will copy.

**Stage three — a clean runtime, non-root, with an init.** `FROM ${BASE_IMAGE} AS runtime` (`:29`) starts from a *fresh* base rather than continuing from the builder — so all the build-time scratch (caches, dev headers) is left behind and only the finished venv is copied in (`COPY --from=builder /opt/venv /opt/venv`, `:49`). Two production-hardening touches close the file. It creates and switches to a **non-root user** (`evosci`, uid/gid 1000, `:44-47`, `:66`): if the container is ever compromised, the attacker is not root. And the entrypoint is wrapped in **tini**:

```dockerfile
ENTRYPOINT ["tini", "--", "evosci"]
```
`Dockerfile:75`

`tini` is a tiny init program. The problem it solves is that when a process runs as PID 1 inside a container (as the entrypoint does), the OS gives it special responsibilities — reaping zombie child processes and forwarding signals — that ordinary programs don't implement. A Python CLI made PID 1 might not forward `SIGTERM` to its children, so `docker stop` would hang until it timed out and `SIGKILL`'d everything. `tini` becomes PID 1 instead, does the init job correctly, and forwards signals to `evosci` — so the container shuts down cleanly. This matters especially for EvoScientist, which spawns child processes (the `langgraph dev` subprocess of Ch 14, background OS processes, ccproxy). One small binary; correct process lifecycle.

The `.dockerignore` is the file's silent partner: it excludes VCS metadata, CI, docs, tests, *and* all local runtime state (`.env`, `runs/`, `workspace/`, `skills/`, `memory/`, `.langgraph_api/`) — the same `.env`-leak instinct as §4's fixture, applied to the image. Your keys and your local memory never accidentally get baked into a published container.

---

## 8. CI: four workflows, four different questions

When you open a PR, four GitHub Actions workflows run. They are deliberately *separate* rather than one big job, because each answers a distinct question and a contributor should be able to see at a glance *which* kind of thing broke — a style nit, a behavior regression, a packaging failure, or a container problem. Here is what each gates:

| Workflow | Trigger | Question it answers | What it runs |
|---|---|---|---|
| **lint** (`lint.yml`) | push to `main`, all PRs | "Is the code style-clean and consistently formatted?" | `uv sync --dev` → `ruff check --output-format=github .` → `ruff format --check .` (5 min) |
| **test** (`test.yml`) | push to `main`, all PRs | "Does it behave correctly, on every supported OS and Python?" | matrix `{ubuntu, windows} × {3.11, 3.12}`, `fail-fast: false`, → `uv run pytest -v --timeout=30` (15 min) |
| **build** (`build.yml`) | push to `main`, all PRs | "Does it still package into a valid wheel + sdist?" | `uv build` → upload `dist/` artifact (no publish) |
| **docker** (`docker.yml`) | push to `main`, `v*` tags, Docker-path PRs, manual | "Does the multi-arch container build (and, on release, publish)?" | QEMU + Buildx → `linux/amd64,arm64` → push to ghcr on non-PR |

Four points deserve elaboration, because each is a real design decision rather than boilerplate.

**The test matrix and `fail-fast: false`.** The test workflow runs the suite four times — the cross product of two operating systems and two Python versions — because "passes on my Linux/3.11" is not the same claim as "passes on Windows/3.12," and EvoScientist genuinely behaves differently across those (recall the Windows event-loop fix of Ch 14, issue #283). The `fail-fast: false` setting (`test.yml:16-17`) is the interesting choice, and its comment is another mini-postmortem: by default, GitHub cancels the *whole* matrix the instant *one* cell fails. While the Windows leg was being stabilized (issue #207), that meant a Windows flake would cancel the Linux runs, hiding whether Linux was actually fine — "playing whack-a-mole one failure at a time." Turning fail-fast off lets all four cells report independently, so a contributor sees the *complete* picture in one run. The `--timeout=30` on pytest (via `pytest-timeout`) is a second guard: a hung stream — precisely the failure mode `stream_v3_fakes`'s `HangingV3Agent` (§3) tests deliberately — should fail as a 30-second timeout, not hang the whole CI job for 15 minutes.

**Build gates packaging separately from behavior.** It is entirely possible for tests to pass while the *wheel* fails to build — a missing package-data glob, a broken `pyproject.toml`. The build workflow (`build.yml:18-19`) runs `uv build` and uploads the artifact, but deliberately does *not* publish. It proves "this could be released" on every PR, decoupling "is it packageable?" from "should it be released?" — the actual PyPI push happens only on a version tag.

**Docker's supply-chain hygiene: SHA-pin third-party, SemVer first-party.** Look closely at how the docker workflow references its actions:

```yaml
      - uses: actions/checkout@93cb6efe...  # v5.0.1
      - uses: docker/setup-qemu-action@c7c53464...  # v3.7.0
      - uses: docker/build-push-action@10e90e36...  # v6.19.2
```
`docker.yml:33-59`

Every *third-party* action is pinned by full **commit SHA**, with the human-readable version in a trailing comment. A git tag like `v6` is *mutable* — whoever controls the action's repo can repoint it at new code — so a compromised or hijacked action could inject a malicious step into your build that has write access to your registry. A commit SHA is *immutable*: pin to it and you run exactly the reviewed code forever. This is the same "pin to content, not names" principle as the Docker digests and `uv.lock`, applied to your *build supply chain*. (EvoScientist's own first-party actions, and Astral's `setup-uv`, are trusted more loosely by SemVer tag — a judgment call about which suppliers you trust to not rewrite tags.) The workflow also uses QEMU + Buildx to cross-compile for both `amd64` and `arm64` from one Linux runner, is path-filtered so it only fires when Docker-relevant files change (`docker.yml:8-14`), and uses `concurrency: cancel-in-progress` so a new push cancels an in-flight build of the same ref.

Together the four form a gate with a clear failure signature: a red **lint** means "run ruff"; a red **test** on one matrix cell means "you broke Windows/3.12 specifically"; a red **build** means "you broke packaging"; a red **docker** means "you broke the container." One conflated job could not tell a contributor any of that.

---

## 9. A coda on humility: `update_check.py`, and "the book is a guide, the repo is the law"

Two final notes close the chapter, and both are about honesty — of code, and of documentation.

First, a small file that models a design attitude the whole project shares. `update_check.py` checks PyPI to tell a user a newer EvoScientist exists. It is a pure nice-to-have, and the code is written from one conviction: **a nice-to-have must never crash the thing it decorates.** Every fallible operation is wrapped so failure is invisible:

```python
    try:
        import urllib.request
        req = urllib.request.Request(PYPI_URL, headers={...})
        with urllib.request.urlopen(req, timeout=3) as resp:
            latest: str = json.loads(resp.read())["info"]["version"]
    except Exception:
        logger.debug("Failed to fetch latest version from PyPI", exc_info=True)
        return None
```
`EvoScientist/update_check.py:48-59`

The cache read, the network fetch, the cache write, and the version comparison are each in their own `try/except Exception` that logs at `debug` and returns a safe default (`:39-45`, `:48-59`, `:62-69`, `:86-91`). If PyPI is down, if the cache file is corrupt, if the version string is unparseable — the CLI starts anyway, silently, as if the check never happened. There is even a dependency-avoidance decision inside: the fetch uses stdlib `urllib` "to avoid extra deps" (`:47`) rather than the `httpx` the project already ships, because a *background update check* is not worth widening the dependency surface. This is the same fail-soft, minimize-blast-radius instinct as §6's getattr-guarded monkeypatch and §4's isolation fixtures — a project-wide temperament, visible here in ninety lines.

Second, a moment of the book's own discipline. `CONTRIBUTING.md:71` cheerfully advertises the test suite as "~890 tests across 36 files." That number is **stale**: the `tests/` directory today holds 121 `test_*.py` files (125 Python files in total), roughly 56,000 lines. The doc was written once and the code kept moving. This is not a criticism of the project — every fast-moving repo's prose lags its code — but it is a perfect illustration of the rule this book states in every Sources table: **the book is a guide; the repo is the law.** If this chapter and the repository ever disagree about how the build works, do not trust this chapter. Open the file, read the code, and believe *that*. The prose you are reading, like `CONTRIBUTING.md`, is a snapshot; the `pyproject.toml`, the Dockerfile, and the workflows are the running truth, and they update themselves every time someone commits.

---

## 要点 / Takeaways

- **uv** is Astral's Rust-based package/project manager that unifies dependency resolution, virtualenv creation, running, and building. `pyproject.toml` is the human spec; `uv.lock` is the machine-pinned resolution; `uv sync --frozen` refuses to drift and is what CI and Docker rely on. **Extras** (`optional-dependencies`, for users) and **dependency groups** (for maintainers) are distinct, and `dev` appears in both to serve `pip` users and `uv` users alike.
- **The crown jewel:** EvoScientist tests its agent by subclassing LangChain's `FakeMessagesListChatModel` into a scripted `_ToolCallingFakeModel` and feeding it to the *real* `create_deep_agent()`. This exercises the genuine LangGraph state machine, ToolNode, `interrupt_on` HITL, and `InMemorySaver` end-to-end — hermetic, fast, free — replacing only the one nondeterministic paid component, the model. The HITL test (`test_stream_events.py:689-741`) is the canonical "follow one request end-to-end."
- Two more fakes cover the boundaries: `stream_v3_fakes.py` hand-builds raw v3 events beneath the Ch 15 translator; `fakes.py` stands in for the SDK/gateway world around the graph.
- **Test isolation is written down as fixtures.** `_isolate_dotenv` (issue #322) stops a real `.env` from leaking into `os.environ`; `_restore_dangerous_env` defends against the `monkeypatch`-delenv-of-absent gotcha. An autouse fixture is where a project records what it learned the hard way about global state.
- **Pinning is an incident log.** Loose comments (`openrouter<0.11`, `textual<8.2.7` — Ch 8) are permanent postmortems next to the code. For a private-API dependency, `~=0.6.12` plus a getattr-guarded, idempotent monkeypatch (`patches.py:1201-1237`) degrades to a no-op instead of crashing. Dependabot automates only the mechanical, low-risk Docker base-image bumps; Python pins are hand-curated.
- The **Dockerfile** teaches digest-pinned bases, copy-only Node vendoring, a uv builder split for layer caching with BuildKit cache mounts and `--frozen`, and a non-root runtime fronted by `tini` for correct signal handling.
- **CI is four workflows** — lint, test (matrix `ubuntu+windows × 3.11+3.12`, `fail-fast: false` per issue #207, `--timeout=30`), build (packaging only, no publish), docker (multi-arch, third-party actions SHA-pinned) — split so a red check names *which* kind of thing broke.
- `update_check.py` swallows every exception and uses stdlib `urllib` to avoid deps — a nice-to-have must never crash the CLI. And `CONTRIBUTING.md`'s "~890 tests across 36 files" is stale (now 121 test files): **when the book and the code disagree, the code wins.**

---

## Sources

*The book is a guide; the repo is the law. When this chapter and the code disagree, believe the code.*

| Topic | Authoritative file(s) |
|---|---|
| uv, dependencies, extras vs. groups, build backend, package-data | `pyproject.toml` |
| The exact resolved dependency versions | `uv.lock` |
| Fake-model testing (the crown jewel) + HITL end-to-end test | `tests/test_stream_events.py` (`:41-43`, `:689-741`) |
| Fakes for the graph boundaries | `tests/stream_v3_fakes.py`, `tests/fakes.py` |
| Provider-registry contract tests | `tests/test_llm.py` |
| Test isolation fixtures + captured deepagents originals | `tests/conftest.py` (`:107-171`), `tests/test_config.py` (`:33-48`) |
| Dependency pins as incident log | `pyproject.toml:28-29`, `:48-49` (see Ch 8) |
| Getattr-guarded defensive monkeypatch | `EvoScientist/llm/patches.py:1201-1237` |
| Dependabot scoping | `.github/dependabot.yml` |
| ruff / pre-commit configuration | `pyproject.toml` (`[tool.ruff]`), `.pre-commit-config.yaml` |
| Multi-stage Docker image | `Dockerfile`, `.dockerignore`, `docker-compose.yml` |
| CI workflows | `.github/workflows/{lint,test,build,docker}.yml` |
| Defensive optional check | `EvoScientist/update_check.py` |
| Stale test-count claim (contradiction #3) | `CONTRIBUTING.md:71` vs. actual `tests/` directory |
