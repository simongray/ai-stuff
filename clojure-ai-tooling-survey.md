# AI-Assisted Clojure Development in 2026: A Survey

*Prepared for Simon Gray, 13 August 2026. Current setup: Claude Desktop + clojure-mcp (configured ~August 2025, largely unchanged since). New factor: Claude Max subscription. All claims are sourced; links throughout and a full source list at the end. Where a claim rests on a single source or couldn't be verified, that's flagged.*

---

## TL;DR

The single biggest change since you set up your stack is that **the center of gravity has moved from Claude Desktop + a heavyweight MCP server to Claude Code (the CLI) + lightweight hooks, CLI tools, and skills** — and the person who moved it is Bruce Hauman himself. His follow-up project **clojure-mcp-light** opens with the line *"This is not an MCP,"* and the clojure-mcp README now explicitly tells CLI users to *start* with clojure-mcp-light and only optionally add the full server. The economic argument that originally made Claude Desktop attractive (subscription vs. per-token API billing) evaporated once Claude Code was included in Pro/Max subscriptions with a shared usage pool.

Second, **the paren problem is mostly a solved problem** — partly because frontier models got better ("Opus 4.6 … almost never struggles with the parentheses" — maxweber on ClojureVerse, March 2026), and partly because the residual errors are now fixed mechanically and token-free by parinfer-based hooks that intercept edits before they hit disk.

Third, **the customization layer that matters in 2026 is Claude Code's native one**: CLAUDE.md, hooks, skills, plugins, subagents, and (since February 2026) automatic memory and MCP tool search. A small ecosystem of Clojure-specific skills and plugins has grown around it.

Fourth, on the trend you asked about specifically: **third-party AI memory databases over MCP are the most hyped and least vindicated part of the landscape.** The benchmark wars between vendors have been mutually discrediting, practitioner reports consistently favor plain markdown files, and there is essentially no evidence of adoption in the Clojure community. Your existing `PROJECT_SUMMARY.md` habit is, as it turns out, the approach the wider community converged on. The considered recommendation is to skip them.

Finally: the **Clojurians Zulip does mirror the Slack**, including full history, and the archive question has a concrete answer (§8).

---

## 1. The big shift: from fat MCP server to thin hooks

A year ago the canonical setup was exactly yours: Claude Desktop (for subscription economics and a chat-style workflow) speaking MCP to Bruce Hauman's clojure-mcp, which supplied everything Claude lacked — REPL eval, structure-aware editing, paren repair, project file access. Two things eroded that arrangement over the year.

**The economics inverted.** In the May 2025 Hacker News thread on clojure-mcp's launch, the discussion was dominated by token costs ("a simple query cost $0.48" — user rads) and Claude Desktop was recommended precisely because it avoided API billing. Claude Code's inclusion in Max (and later Pro) subscriptions, with claude.ai, the desktop app, and Claude Code drawing from one shared usage pool, removed the reason to prefer Desktop. In mid-2026 reporting, session limits were also raised substantially (5-hour limits doubled, weekly limits +50% around May 2026 — secondary source: ccforeveryone.com). Every 2026 first-person Clojure report found in this survey — jwr, mpenet, TacticalCoder, drob518, Ivan Willig, the Clean Coders team — uses Claude Code or another CLI harness, not Desktop.

**The UI/architecture argument emerged.** MCP-based editing tools bypass Claude Code's native diff display. Hauman's stated rationale for clojure-mcp-light, verbatim from the README:

> "With MCP-based editing tools, you lose Claude Code's native UI integration — tool calls are poorly formatted and difficult to read. Hooks let Claude Code operate normally with its native Edit/Write tools, preserving the clean diff UI you're used to, while transparently fixing delimiter errors behind the scenes."

The broader Claude Code community hit a parallel problem — MCP tool-schema bloat consuming tens of thousands of context tokens before any work began (an Anthropic engineer cited ~67k tokens for a 7-server setup). Anthropic's answer, **MCP Tool Search** (rolled out ~Jan–Feb 2026), lazy-loads tool schemas on demand and blunts this considerably, which matters if you choose to keep the full clojure-mcp in the loop (§2).

The result is a spectrum of setups rather than one canonical one, but with a clear default. In rough order of current popularity among the Clojure reports surveyed:

| Setup | What provides Clojure-awareness | Who it suits |
|---|---|---|
| **Claude Code + clojure-mcp-light** | Hooks (paren repair) + `clj-nrepl-eval` CLI + a skill | The 2026 default; recommended starting point in clojure-mcp's own README |
| **Claude Code + light + full clojure-mcp (`:cli-assist`/`:cli-assist-full` profile)** | Above, plus structural editing, `deps_*` tools, multi-REPL | People who want structure-aware edits as a fallback (e.g. jwr on HN swears by the structural tools) |
| **Claude Desktop + full clojure-mcp** | The full server, as you run it now | Chat-style, conversation-first sessions; still documented, no longer the default |
| **Editor-integrated (Calva Backseat Driver, ECA, Emacs packages)** | Editor exposes REPL/state to the agent | People who want the agent inside the editor rather than beside it |

---

## 2. clojure-mcp today

The project matured considerably over your year away. Highlights from the changelog, August 2025 → August 2026 (latest release **0.5.1, 19 June 2026**):

- **License change AGPL → EPL 2.0** and the "alpha" label dropped (0.1.12, Nov 2025).
- **Global install via Clojure tools**: `clojure -Ttools install-latest :lib io.github.bhauman/clojure-mcp :as mcp`, then `clojure -Tmcp start` from any project — no more per-project alias plumbing. `:start-nrepl-cmd` can auto-start your nREPL with port discovery, and 0.5.0 added opt-in `:fallback-nrepl` auto-spawn.
- **Profiles**: `:cli-assist` (slim, for use alongside a CLI's native tools) and `:cli-assist-full` (0.5.0, promotes structure-aware editing to first-class), plus CLI-level tool filtering (`:enable-tools` / `:disable-tools`) — direct responses to the context-bloat complaint.
- **New tools**: `paren_repair` as an MCP tool (parinfer-based), `deps_list` / `deps_read` / `deps_grep` (search and read inside dependency jars — genuinely useful against hallucinated APIs), `list_nrepl_ports` and multi-REPL support (a `port` parameter on `clojure_eval` — relevant to your clj + shadow-cljs dual-REPL workflow), `dry_run` on edit tools.
- **Dialect support**: Babashka, Basilisp, Scittle, and (0.4.0) ClojureDart; `:shadow-cljs-repl-message` config for ClojureScript.
- Modernized **Streamable HTTP transport** (0.5.0) and MCP tool annotations; 65 built-in model configs for the agent tools.

Persistent caveats: the `dispatch_agent` / `architect` agent tools run on LangChain4j with **separate API keys** — they are not covered by your Max subscription. And the full server is still a JVM process with a large tool surface; the mitigation is profiles + tool filtering + Claude Code's tool search rather than any slimming of the server itself.

Community standing remains high — clojure-mcp received Toyokumo's "Thanks OSS Award" in June 2026, and it is the reference point every other tool defines itself against.

## 3. clojure-mcp-light: what it is and why it exists

Released November 2025, current version 0.2.2 (March 2026). It is **three CLI tools plus Claude Code hooks plus a skill**, installed via `bbin install https://github.com/bhauman/clojure-mcp-light.git`:

1. **`clj-nrepl-eval`** — a command-line nREPL client the agent calls through ordinary bash. Persistent sessions, automatic discovery of nREPL ports **including shadow-cljs**, delimiter auto-repair before eval, heredoc-friendly.
2. **`clj-paren-repair-claude-hook`** — PreToolUse/PostToolUse hooks that intercept Claude Code's native Write/Edit operations and fix delimiter errors *before they reach the filesystem*, using parinfer (with a pure-Clojure fallback). This runs outside the LLM loop: **zero tokens**. It exists to kill what the README names the "Paren Edit Death Loop" — the agent repeatedly failing to fix its own delimiter errors. Hauman's own claim: "In my usage these Hooks have fixed 100% of the errors detected."
3. **`clj-paren-repair`** — the same repair as a standalone command, for harnesses without hooks (Gemini CLI, Codex CLI).

Plus a **`clojure-eval` skill** teaching Claude Code the heredoc REPL workflow, CLAUDE.md snippets, and slash commands. Everything else — reading, editing, grep, diffs — is deliberately left to Claude Code's native tools.

Adoption signals are real rather than hypothetical: HN commenters describe it as "pretty popular" (drob518, Aug 2026: the paren repair "really does the trick"); Max Penet (mpenet): "If you use clojure-mcp/clojure-mcp-light that problem goes away … It's night and day." It has spawned wrappers and imitators — `basic-tools-mcp` (a Babashka MCP server wrapping it), `brepl` (licht1stein's independent take on the same hooks-plus-CLI philosophy, with its own bracket fixer and installers for both Claude Code and ECA), Tony Kay's `fulcrologic/clojure-claude-sandbox` Docker image ships it preinstalled, and Ivan Willig's `clojure-skills` builds on `clj-nrepl-eval`.

One nuance worth knowing: light and full clojure-mcp are **complements, not rivals**, per Bruce's own README — start with light; add the full server with the `:cli-assist` profile if you want structural editing and the `deps_*`/multi-REPL machinery on top.

## 4. The Claude Code layer: hooks, skills, plugins, subagents

Since skills (Oct 2025) and plugins/marketplaces landed, a Clojure-specific ecosystem has grown on top of Claude Code's native extension points. The pieces most worth knowing:

**Hooks as guardrails.** The pattern the community converged on: deterministic checks run by the harness, not left to the model's discretion. The clearest articulation is Clean Coders' June 2026 post ("Teaching Your Coding Agent to Write Clean, Secure Clojure" — Alex Root-Roatch): "A hook is a script the Claude Code harness runs automatically... Claude doesn't choose to run a hook." Their `cleancoders/agent-plugins` marketplace ships two installable plugins: a **clojure** plugin (7 skills plus a PostToolUse hook that runs `cljfmt fix` on every edit, with a shared cljfmt.edn "so Claude, CI, and humans grade against identical standards") and a **clojure-security** plugin (clj-kondo lint on edit, a turn-end scan that blocks unsafe patterns like `read-string` RCE, gitleaks commit-blocking, `/security-audit`; uses clj-kondo, clj-holmes, clj-watson, gitleaks, Semgrep). Install via `/plugin marketplace add cleancoders/agent-plugins`.

**Skills as crystallized knowledge.** Notable collections: Hauman's own `clojure-eval` skill (ships with light); **iwillig/clojure-skills** (78 skills across 29 categories, with the design philosophy "Better 75 excellent skills than 200 mediocre ones" — built for OpenCode but Claude-compatible, built around `clj-nrepl-eval`); hugoduncan's library-skills (including a clj-kondo skill); and the most conceptually interesting entry, **clj-native-agent** (Laurence Chen, May 2026) — rewrite-clj + Babashka skills for *structural* operations: `clj-replace` (S-expression matching insensitive to formatting), `clj-debug` (REPL inspection instead of println logging), `clj-refactor`, plus a validation framework measuring whether a skill actually changes agent behavior ("A skill's value is measured by behavioral change, not knowledge transfer").

**Subagents.** Metabase (a large Clojure codebase) published their production setup in April 2026: ten domain-expert subagents in `.claude/agents/`, each front-loading domain knowledge into its system prompt. This is the pattern for larger teams/codebases; probably overkill for a solo project until it isn't.

**The index for all of this** is Ivan Willig's `awesome-clojure-llm`, which the community treats as the canonical list (it also names #ai-assisted-coding on Clojurians Slack as the hub, confirming your instinct about where the discussion lives).

## 5. Editors and alternatives

**Calva Backseat Driver** (Peter Strömberg/pez, BetterThanTomorrow) is the mainstream VS Code option and is very actively maintained (roughly biweekly releases through 2026). It exposes eight tools to agents — REPL eval via Calva's connection, bracket-balancing (parinfer), top-level-form structural editing, symbol/ClojureDocs lookup, file creation with auto-balanced brackets — both through VS Code's native Copilot tool API and as an MCP server for Claude/Cursor. Since June 2026 it's also a **Cursor extension** with zero-config tool registration, and since July it auto-registers its MCP server with ECA. There's a companion plugin marketplace (`awesome-backseat-driver`) for REPL-first agents/skills, and a Clojure interactive-programming agent made it into GitHub's official awesome-copilot catalog. User report of note: HN's chamomeal — Claude via Backseat Driver produced "10x more complex programs with zero errors" thanks to the live REPL loop.

**ECA (Editor Code Assistant)** — ericdallo (the clojure-lsp maintainer), written in Clojure, Apache-2.0, ~900 stars, very active (v0.144.0, July 2026). An editor-agnostic, LSP-like server: your editor spawns `eca server` and gets chat, agent/subagent orchestration, inline completion, full MCP client support, skills/rules/hooks — with clients for **Emacs, VS Code, Neovim, and IntelliJ**, plus multi-provider auth including Copilot and Anthropic. Notable because it's the Clojure world's own answer to "an agent in every editor," and because clojure-lsp itself has deliberately gained no AI features — Dallo's AI energy goes here. Since you work in IntelliJ/Cursive, ECA is the main path to in-editor agent integration on your side of the fence (Claude Code itself is editor-agnostic and just runs in a terminal next to Cursive).

**Emacs** (for completeness): claude-code-ide.el (manzaltu, 1.6k stars) is the most popular; stevemolitor's claude-code.el + monet pair implements Claude Code's IDE protocol; eca-emacs is on MELPA. CIDER itself has no AI integration — CIDER users attach clojure-mcp/light to their existing nREPL session instead (Yann Esposito's Doom Emacs + gptel + mcp.el + clojure-mcp writeup is the canonical description).

**Other REPL MCP servers**: hugoduncan's `mcp-clj` (minimal, self-contained) is alive; the various 2025-era `nrepl-mcp-server` projects (JohanCodinha's TypeScript one, etc.) are stale — nothing else approaches clojure-mcp, and the momentum is toward CLI-tools-plus-hooks anyway.

**Non-Claude harnesses on Clojure.** Cursor without Clojure tooling remains poor ("could not get the parentheses right: more than half the time" — Julien Bille, Sep 2025), but works well with clojure-mcp or, since June 2026, Backseat Driver. Ivan Willig's longitudinal report lands on open harnesses (opencode, then the pi agent) + clojure-mcp-light + a custom Clojure system prompt, partly because Claude Code "attempts to hide what the LLM is doing." Codex CLI and Gemini CLI are accommodated (light's standalone `clj-paren-repair`, AGENTS.md support) but no substantial first-person Clojure reports on them surfaced. On open-weight models, `awesome-clojure-llm` currently recommends Kimi K2.5, MiniMax-M2.1, and Qwen3-Coder-Next; a fine-tuning experiment (nibzard, Apr 2026) found a Qwen3-30B tuned on Clojure hits 83.8% best-of-16 with a REPL/test verifier — reinforcing that the verifier loop, not the model, is the active ingredient.

## 6. Lived experience: what a year of reports actually says

**Adoption is now the norm**: the State of Clojure 2025 survey (published Feb 2026) found **70% of Clojure developers use AI tools for development**, 12% considering, 18% content without.

**The sentiment arc** over the period runs roughly: 2024 skepticism about chat-based assistants ("the code it outputs is usually 5x as long as you need it to be and messy") → the mid-2025 clojure-mcp moment, when REPL-grounded agents flipped many skeptics → late-2025 consolidation into daily drivers plus the first disillusionment reports about architectural rot and token burn → 2026 maturation and polarization: hooks/skills displace fat MCP servers, parens stop being the problem, and the debate moves up the stack to architecture and process.

**What reliably works**, per multiple independent reports: giving the agent a persistent nREPL connection and making it evaluate before it writes ("Clojure MCP fundamentally altered my belief in the ability of LLMs to write effective Clojure code" — Ivan Willig, whose one-year report is the best single document in this genre); paren-repair hooks; small composable functions; clean codebases; cheap targeted REPL probes instead of full test-suite runs; and treating the REPL as the verifier in a generate-verify loop.

**What still fails**: agents don't refactor architecture unprompted, so "the Agent will reproduce your mistakes continuously" and tech debt compounds (Bille); Claude retains an imperative bias under pressure (`doseq` + atoms where a functional style belongs — Willig); costs balloon on messy codebases; context rot sets in on long sessions; and unattended "vibe coding" produces things like a Python→Clojure port whose core logic quietly migrated into 2,000+ lines of bash (Flexiana's experiment, $134.79 for one week). Gene Kim's resurrect-a-dead-13k-LOC-app experiment ($42 of API tokens, two days) is the positive counterpart, with the same moral: "Your brain stays constantly engaged. You need it to make sure the AI is 'coloring inside the lines.'"

**The philosophical splits** worth knowing, since they'll shape what you read in #ai-assisted-coding: interactive pair-programming (the Calva "backseat driver" ethos) vs. autonomous agents (vibe coding, Agent-o-rama); big-MCP vs. small-hooks (both positions held by Bruce Hauman six months apart); "Clojure is token-efficient and ideal for LLMs" (Felix Barbalet's economics essay) vs. "Lisp is AI-resistant because training data is scarce" (Dan Haskin's widely-discussed lament, whose HN thread is where several of the "actually it works fine now with tooling" quotes above come from). And a notable abstention: the Clojure core team doesn't use AI — Fogus's February 2026 "LLMe" essay is the thoughtful insider-skeptic document ("Trading mentorship for prompt-craft is not a neutral efficiency gain").
## 7. The memory-database trend: mostly hype, verdict "skip"

This is the trend you flagged, and it got a deliberately skeptical review. The landscape: **mem0** (fact extraction into vector+graph stores; its local-first OpenMemory MCP product was deprecated and replaced mid-cycle), **Letta** (ex-MemGPT — an agent runtime, not a bolt-on; adopting it means abandoning Claude Code's loop), **Zep/Graphiti** (temporal knowledge graphs; Zep killed its self-hostable Community Edition in 2025, leaving DIY-Graphiti-plus-Neo4j or paid cloud), **Cognee**, **basic-memory** (markdown-files-with-wikilinks — the most Clojure-temperament of the lot), **claude-self-reflect** (semantic search over your past Claude Code transcripts; small but interesting niche), and a long tail of 2026 SaaS entrants best understood as SEO marketing.

Three findings matter:

**First, the evidence base is rotten.** The vendors' benchmark claims have been mutually discrediting — Zep published a rebuttal showing a plain full-context baseline beat mem0's best configuration; a third party then reproduced Zep's own headline 84% at 58.4%; and an April 2026 analysis found 6.4% of the underlying benchmark's answer key is simply wrong. None of these benchmarks measure coding outcomes anyway. The most quotable line from the HN threads (user gbnwl): for memory tools "there's never any evidence or even attempt at measuring any metric of performance improved by it." That claim went unrefuted in every thread reviewed.

**Second, churn is structural.** OpenMemory sunset, Zep CE end-of-life, Letta's repo split, claude-self-reflect's full rewrite — adopting a 2025 memory server meant at least one forced migration by mid-2026.

**Third, the native alternatives got good, and they are plain files.** Claude Code now has: the CLAUDE.md hierarchy (global → project → local, with `@path` imports and path-scoped `.claude/rules/`), **auto memory** (on by default since Feb 2026: Claude maintains its own `MEMORY.md` index plus topic files per project), skills as load-on-demand procedural memory, and claude.ai/Desktop memory for Pro/Max. Anthropic's own memory architecture — including the API memory tool — is markdown files plus file tools, not a vector database, which rather settles the argument. Practitioner consensus in the threads: deliberate, inspectable, git-versioned notes beat automatic capture ("automatic memory is not learning"); CLAUDE.md works when it's short (<~200 lines) and curated.

**Clojure angle**: no evidence of any Clojure-community adoption of these tools was found — no ClojureVerse or Clojurians threads on mem0/Graphiti/basic-memory at all. The community pattern is exactly what you already do: clojure-mcp's `PROJECT_SUMMARY.md` plus scratch_pad, i.e., curated plain files in the repo. The only Datomic/DataScript MCP servers out there are for querying application databases, not agent memory; nobody has built (or apparently missed) a Datalevin-backed memory store.

Legitimate residual niches, if you ever feel the gap: **claude-self-reflect** for "how did we solve this last month?" transcript search; **beads** (Steve Yegge's issue-tracker-as-agent-memory, 26k stars, the community's organic favorite in this space) for multi-session task state; and team-shared memory, which is the one gap even skeptical reviews concede — irrelevant for a solo project.

## 8. Reading #ai-assisted-coding again

Your Slack-archive problem has a concrete answer. The Clojurians **Zulip** (clojurians.zulipchat.com) runs a one-way sync bot that mirrors Slack channels into a stream called **#slack-archive, with one topic per Slack channel** — and Zulip keeps full history, unlike free-plan Slack's 90-day window. The direct URL to try:

`https://clojurians.zulipchat.com/#narrow/channel/180378-slack-archive/topic/ai-assisted-coding`

*Confirmed working (13 Aug 2026):* the #slack-archive stream is web-public — the URL above opens in a logged-out browser, the `ai-assisted-coding` topic exists, and messages mirror from Slack essentially live. A free Zulip account additionally gets you full-text search. Section 10 below is a digest read directly from it. Because it's a JS app, Google indexes none of it, which is why the discussion feels invisible from the outside. Separately, the old ClojureVerse log site (clojurians-log.clojureverse.org) is still logging into 2026, though #ai-assisted-coding wasn't confirmed among its channels. And per `awesome-clojure-llm`, #ai-assisted-coding remains the acknowledged hub — it even spawned a live meetup series co-organized by Bruce Hauman.

## 9. What I'd set up in your position

You have Claude Max, IntelliJ/Cursive, a REPL-first surgical-edit philosophy, a project already carrying a tuned CLAUDE.md, an LLM_CODE_STYLE.md, a PROJECT_SUMMARY.md, and an `:nrepl` alias on port 7888. That's most of the work already done. Concretely:

**Baseline (the 2026 default, ~an hour of setup):**

1. **Adopt Claude Code as the primary harness.** It's included in Max, draws from the same usage pool as Desktop, and is where all the Clojure ecosystem investment is going. It runs in a terminal beside Cursive; your editor workflow doesn't change.
2. **Install clojure-mcp-light** (`bbin install https://github.com/bhauman/clojure-mcp-light.git` — needs babashka + bbin) and enable its hooks and the `clojure-eval` skill. This gives you token-free paren repair on Claude Code's native edits (keeping the diff UI) and REPL eval via `clj-nrepl-eval`, which auto-discovers both your JVM nREPL and shadow-cljs ports — it should slot straight into your existing browser-REPL debugging recipe.
3. **Carry your context files over as-is.** Your CLAUDE.md already encodes the right instincts (REPL-first, surgical edits, report anomalies immediately). Consider converting LLM_CODE_STYLE.md into a skill (or `@`-importing it from CLAUDE.md) so it loads on demand rather than sitting in every prompt; keep maintaining PROJECT_SUMMARY.md exactly as before.
4. **Add one guardrail hook**: clj-kondo lint on edit (via the cleancoders clojure-security plugin, or a small custom PostToolUse hook). Given your `{:cljfmt false}` clojure-mcp config, skip the cljfmt-on-every-edit hooks the Clean Coders plugin bundles, or configure around them.
5. **Skip the memory databases.** Auto memory (on by default) + CLAUDE.md + PROJECT_SUMMARY.md is the vindicated stack.

**Optional second layer, when you miss the old tools:** install full clojure-mcp globally (`clojure -Ttools install-latest :lib io.github.bhauman/clojure-mcp :as mcp`) and add it to Claude Code with the `:cli-assist` profile. That restores structure-aware editing as a fallback, plus the genuinely new things you never had: `deps_read`/`deps_grep` (read dependency sources instead of hallucinating APIs) and multi-REPL eval (clj and shadow-cljs simultaneously via the `port` parameter). MCP tool search means the context cost is far lower than it would have been a year ago.

**Keep Claude Desktop in the loop for what it's good at** — chat-style design discussions and architecture sessions against the same clojure-mcp setup you have now. Nothing forces a migration; the same subscription covers both.

**Watch list, no action needed:** ECA if you ever want the agent inside IntelliJ rather than beside it; clj-native-agent's structural skills; Backseat Driver only if you drift toward VS Code; open-weight models (Kimi K2.5 et al.) only if cost or independence starts to matter.

---

## 10. Addendum: what #ai-assisted-coding is actually discussing (read 13 Aug 2026, covering ~Aug 4–13)

After the main survey was written, the Zulip mirror of the channel turned out to be fully readable without a login, so this section is primary source material: a digest of the last ten days of discussion, read directly from the archive. It is a small window, but it lands squarely on the survey's conclusions — and updates a few things the web sources were too slow to catch.

**The REPL-first thesis is now the channel's background assumption, not a debate.** maxweber: "Surprise a REPL gives an LLM superpowers 😄". didibus, reacting to Nathan Marz's latest results: "REPL driven development wins again." yogthos (Dmitri Sotnikov): "I use the repl as the default workflow and the turn around is just so much faster … lisp workflow is absolutely amazing for llms" — he has agents run for hours on his veriframe project while a supervising agent "watches the harness and patches it via the repl when it's getting stuck." The showpiece of the week was pez (Peter Strömberg) 3D-modelling his own house by giving an agent *two simultaneous REPLs* — Basilisp inside Blender plus his Epupp browser-userscript REPL pulling elevation data from the Swedish land survey — "elevation tracing [became] a 10 minute thing that the agent can handle itself."

**Paren repair via hooks is likewise settled.** yogthos, on Steve Yegge-style mega-harnesses: "Why have an inference engine figure out how to do stuff like balancing parens when you can do that work mechanically." When didibus asked whether he'd tried a hook for it, the answer was yes — he has since built balancing directly into his own harness ("dirge"). Nobody in ten days argued the models should handle delimiters themselves.

**Memory practice on the ground = markdown protocols, exactly as §7 concluded.** The channel has grown its own family of plain-file memory conventions: michael819's "mementum" protocol (a state.md plus a debated INDEX.md/queue.md — chosen because "it's greppable"), hugod (Hugo Duncan) running "munera" for task tracking and "ramora" for controlling markdown file size. chromalchemy on mementum: "Reliably documents state and pickups up threads in new sessions!" Meanwhile, when ognivo asked "what to read about Clojure-aware memory MCPs", pez's reply was "What's a memory MCP?" — about as clean a confirmation as you could ask that memory databases have no established practice in this community. (The tool ognivo surfaced, DeusData/codebase-memory-mcp, did get an impressed reaction — file it under curiosity, not adoption.) Related niche: eighttrigrams' `us-vs-them`, tracking *provenance* — which edits were made by a human vs. an agent — so agents treat human-authored code with more caution.

**The model landscape has moved since the survey's sources.** Fresh releases got mixed reviews: michael819 finds "Sonnet 5 is not as bad, but Opus 5 is a regression … Opus 4.6 and 4.8 seem to do a much better job … I would rather have the consistency of the older models." The just-released DeepSeek v4 is being called frontier-level ("fable levels" — yogthos), continuing the channel's strong budget/open-weight current (DeepSeek and local qwen for yogthos; Grok 4.5 via Cursor as the bargain of the moment for pez; Kimi 2.5 in Cursor; OpenRouter recommended for model-shopping). Worth knowing when picking your default model in Claude Code — and a reminder that these takes are single-user impressions, not benchmarks.

**Harness talk has moved up-stack, matching §6's trajectory.** The interesting argument was Yegge's "Gas Town" harness reportedly breaking with stronger models while Jeffrey Emanuel's continuously-improved one "just grew stronger"; didibus pressed the skeptical question of what $20k/month of autonomous agents actually produces beyond replicas of existing software. cormacc shared "herdr-orch," a harness-agnostic subagent-orchestration *skill* he runs across pi, Claude and Codex — "I like the pattern of portable skill + babashka cli for tool-call efficiency," which is precisely the bhauman/iwillig pattern from §4. Nathan Marz's Red Planet Labs report #3 (one-shotting complex backends) landed the week's best line on verification: "making the LLM test its designs empirically … was the key to funneling it to the right design." And borkdude announced choq — CLJS on QuickJS in a 5MB binary with sandbox allow-lists and nREPL, aimed at safely running "AI-ish (unsafe) things." Also flagged by ericdallo: Clojure+AI talks are on the program at the upcoming Conj.

The window also carried a steady hum of economics/bubble talk and one sobering yogthos anecdote for the judgment file: Claude implemented stack traces via a shadow stack costing 20x performance, "then it tried to gaslight me that this was the best solution possible. … if you don't have domain knowledge then it becomes very difficult to know when you should push against it."

Net effect on the recommendations in §9: none — this window independently reproduces the survey's three main conclusions (REPL access, mechanical paren repair, plain-file memory) from the mouths of the people who live in the channel.

---

## Sources

**Core tooling**: [clojure-mcp](https://github.com/bhauman/clojure-mcp) · [clojure-mcp CHANGELOG](https://github.com/bhauman/clojure-mcp/blob/main/CHANGELOG.md) · [clojure-mcp-light](https://github.com/bhauman/clojure-mcp-light) · [light CHANGELOG](https://github.com/bhauman/clojure-mcp-light/blob/main/CHANGELOG.md) · [brepl](https://github.com/licht1stein/brepl) · [mcp-clj](https://github.com/hugoduncan/mcp-clj) · [clojure-claude-sandbox](https://github.com/fulcrologic/clojure-claude-sandbox) · [ai-investigator](https://github.com/bhauman/ai-investigator)

**Skills/plugins/practices**: [awesome-clojure-llm](https://github.com/iwillig/awesome-clojure-llm) · [clojure-skills](https://github.com/iwillig/clojure-skills) · [clj-native-agent](https://github.com/humorless/clj-native-agent) + [essay](https://humorless.github.io/posts-output/agent-skill) · [Clean Coders: Teaching Your Coding Agent to Write Clean, Secure Clojure](https://cleancoders.com/blog/2026-06-22-teaching-claude-clean-secure-clojure) · [Metabase: ten custom subagents](https://www.metabase.com/blog/ten-custom-subagents) · [MCP Tool Search in Claude Code](https://tessl.io/blog/anthropic-brings-mcp-tool-search-to-claude-code/) · [Claude Code memory docs](https://code.claude.com/docs/en/memory)

**Editors**: [calva-backseat-driver](https://github.com/BetterThanTomorrow/calva-backseat-driver) · [awesome-backseat-driver](https://github.com/BetterThanTomorrow/awesome-backseat-driver) · [ECA](https://github.com/editor-code-assistant/eca) / [eca.dev](https://eca.dev/) · [claude-code-ide.el](https://github.com/manzaltu/claude-code-ide.el) · [claude-code.el](https://github.com/stevemolitor/claude-code.el) · [monet](https://github.com/stevemolitor/monet) · [Yann Esposito's Emacs setup](https://yannesposito.com/posts/0029-ai-assistants-in-doom-emacs-31-on-macos-with-clojure-mcp-server/index.html)

**Lived experience**: [Ivan Willig: One year of LLM usage with Clojure](https://www.iwillig.me/blog/one-year-of-llm-usage-with-clojure/) · [Julien Bille: Cursor and Clojure-MCP](https://medium.com/@_jba/my-experience-with-cursor-and-clojure-mcp-6e323b90a6f3) · [Gene Kim: resurrecting a Clojure app with Claude Code](https://itrevolution.com/articles/resurrecting-my-trello-management-tool-and-data-pipeline-with-claude-code-using-vibe-coding-part1/) · [Flexiana: Experience with Claude Code](https://medium.com/@flexianadevgroup/experience-with-claude-code-ae1647bc6f17) · [Sean Corfield: My AI Usage Statement](https://corfield.org/blog/2026/08/02/ai/) · [Fogus: LLMe](https://blog.fogus.me/meta/LLMe.html) · [yogthos: Managing Complexity with Mycelium](https://yogthos.net/posts/2026-02-25-ai-at-scale.html) · [Barbalet: Simple Made Inevitable](https://felixbarbalet.com/simple-made-inevitable-the-economics-of-language-choice-in-the-llm-era/) · [Haskin: Writing Lisp is AI resistant](https://blog.djhaskin.com/blog/writing-lisp-is-ai-resistant-and-im-sad/) · [Daniel Janus: translating codebases with Claude](https://blog.danieljanus.pl/2026/03/26/claude-nlp/) · [State of Clojure 2025](https://clojure.org/news/2026/02/18/state-of-clojure-2025) · [HN: clojure-mcp launch](https://news.ycombinator.com/item?id=44086062) · [HN: paren thread 2026](https://news.ycombinator.com/item?id=47384631) · [ClojureVerse: #_ marker thread](https://clojureverse.org/t/as-a-semantic-block-closing-marker-for-llm-friendly-clojure/14851) · [Metosin on clojure-mcp](https://www.metosin.fi/blog/2025-05-27-bruce-hauman-has-done-it-again)

**Memory**: [mem0](https://github.com/mem0ai/mem0) · [Letta](https://github.com/letta-ai/letta) · [Graphiti](https://github.com/getzep/graphiti) · [Zep OSS strategy change](https://blog.getzep.com/announcing-a-new-direction-for-zeps-open-source-strategy/) · [Zep vs mem0 benchmark rebuttal](https://blog.getzep.com/lies-damn-lies-statistics-is-mem0-really-sota-in-agent-memory/) · [zep-papers reproduction issue](https://github.com/getzep/zep-papers/issues/5) · [Penfield benchmark critique](https://penfieldlabs.substack.com/p/proposal-a-new-benchmark-for-long) · [basic-memory](https://github.com/basicmachines-co/basic-memory) · [claude-self-reflect](https://github.com/ramakay/claude-self-reflect) · [beads](https://github.com/steveyegge/beads) · [HN: Recall thread](https://news.ycombinator.com/item?id=48622590) · [HN: ENSUE thread](https://news.ycombinator.com/item?id=46426624) · [Supermemory hands-on eval](https://adventuresinclaude.ai/posts/2026-02-17-supermemory-evaluation/) · [Anthropic: effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**Community/archive**: [Clojurians Zulip](https://clojurians.zulipchat.com) · [slack-sync bot](https://gitlab.com/clojurians-zulip/slack-sync) · [clojurians-log](https://clojurians-log.clojureverse.org/) · [Clojure Deref 2025-11-06](https://clojure.org/news/2025/11/06/deref) · [Deref 2026-03-10](https://clojure.org/news/2026/03/10/deref) · [Deref 2026-06-09](https://clojure.org/news/2026/06/09/deref)

*Known gaps: Reddit (r/Clojure, r/ClaudeAI) was unreachable from the research environment; GitHub star counts and two YouTube items (the Flexiana "Clojure Corner" Hauman interview, and "Vibe Coding With Clojure-MCP" with Hauman/Burton/Kim) could not be fully verified; the May 2026 usage-limit increases rest on a secondary source. Claude-side product facts reflect early-2026 documentation and may have moved again.*

---

## Appendix A: Clean-machine setup runbook

The setup from §9, as actually performed and verified in August 2026 — written so it can be replayed on a new machine and in any Clojure project, not just DanNet. Versions are pinned to what was current at the time of writing; check the repos' release pages before replaying much later.

### A.1 Once per machine

**Claude Code + sign-in:**

```bash
curl -fsSL https://claude.ai/install.sh | bash   # or: npm install -g @anthropic-ai/claude-code
claude                                            # then /login → browser OAuth against claude.ai
```

Covered by a Pro/Max subscription; no API key needed. The login reuses an existing claude.ai browser session if one exists.

**Babashka, bbin, and PATH** (note the two different Homebrew taps — babashka is in `borkdude/brew`, bbin in `babashka/brew`):

```bash
brew install borkdude/brew/babashka
brew install babashka/brew/bbin
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && exec $SHELL
```

**The three clojure-mcp-light tools:**

```bash
bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.2

bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.2 \
  --as clj-nrepl-eval --main-opts '["-m" "clojure-mcp-light.nrepl-eval"]'

bbin install https://github.com/bhauman/clojure-mcp-light.git --tag v0.2.2 \
  --as clj-paren-repair --main-opts '["-m" "clojure-mcp-light.paren-repair"]'
```

**Paren-repair hooks** — add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse":  [{"matcher": "Write|Edit", "hooks": [{"type": "command", "command": "clj-paren-repair-claude-hook"}]}],
    "PostToolUse": [{"matcher": "Edit|Write", "hooks": [{"type": "command", "command": "clj-paren-repair-claude-hook"}]}],
    "SessionEnd":  [{"hooks": [{"type": "command", "command": "clj-paren-repair-claude-hook"}]}]
  }
}
```

Decision point: the upstream README appends `--cljfmt` to each hook command, which also reformats every edited file with cljfmt. Leave it off for pure delimiter repair with no reformatting (the choice made here); add it if you want format-on-every-edit.

**Skill and slash commands** (bbin installs only the binaries; these come from a clone):

```bash
git clone https://github.com/bhauman/clojure-mcp-light.git /tmp/cml
mkdir -p ~/.claude/skills ~/.claude/commands
cp -r /tmp/cml/skills/clojure-eval ~/.claude/skills/
cp /tmp/cml/commands/*.md ~/.claude/commands/    # /start-nrepl and /clojure-nrepl
```

**Optional, personal style skill everywhere:** copy a project's style skill (see A.2) to `~/.claude/skills/` to have it apply in all projects.

### A.2 Once per project

**An nREPL entry point** in `deps.edn` (any port works; light auto-discovers it — a fixed port just makes it predictable):

```clojure
:aliases
{:nrepl {:extra-paths ["test"]
         :extra-deps  {nrepl/nrepl {:mvn/version "1.3.1"}}
         :jvm-opts    ["-Djdk.attach.allowAttachSelf"]  ; lets nREPL interrupt runaway evals
         :main-opts   ["-m" "nrepl.cmdline" "--port" "7888"]}}
```

shadow-cljs projects need nothing extra — `clj-nrepl-eval` discovers the shadow nREPL port on its own.

**A `CLAUDE.md`** at the repo root. Generic template distilled from a year of community practice plus this survey's findings:

```markdown
- Clojure is a FUNCTIONAL language: do REPL-based development in a functional style.
  - Test new code with a few relevant function calls in the REPL, not full test-suite runs.
- Be surgical in your edits; favour clean, lean git diffs over large refactors.
- If you encounter something UNUSUAL (weird output, unexpected errors), report back immediately
  instead of going on a long solo tangent.
- After editing CLJ/CLJC files, `(require '[the.namespace] :reload)` so changes take effect
  in the running REPL.
- REPL access: the `clj-nrepl-eval` command evaluates Clojure via nREPL.
  - Discover running servers: `clj-nrepl-eval --discover-ports`
  - Evaluate: `clj-nrepl-eval -p <port> "<clojure-code>"`
  - Sessions persist per port: namespaces and state are maintained between evaluations.
- When writing or editing Clojure code, follow the `clojure-style` skill (loads on demand).
```

**A style skill instead of an always-loaded style file.** Put the house style in `.claude/skills/clojure-style/SKILL.md` (committed, so collaborators' agents get it too). The frontmatter `description` must say when to use it, phrased assertively ("Use whenever writing, editing, refactoring, or reviewing Clojure code…"), because it is the sole trigger; the body then only loads when the agent is actually about to write Clojure — zero context cost the rest of the time. This replaces the older pattern of `@`-importing a style file from CLAUDE.md into every session.

**Smoke test:** start the REPL (`clojure -M:nrepl`), run `claude` in the project, then: `/hooks` should list the three hooks; asking for `clj-nrepl-eval --discover-ports` should surface the port(s) and prompt once for approval — approving "don't ask again for `clj-nrepl-eval *`" whitelists only that command, which is the intended frictionless-REPL trade-off (arbitrary code does run in your JVM). To watch the repair machinery work: `echo '(defn foo [x] (inc x' > /tmp/broken.clj && clj-paren-repair /tmp/broken.clj && cat /tmp/broken.clj`.

### A.3 Optional second layer, per project

Full clojure-mcp alongside light, for structure-aware editing, `deps_read`/`deps_grep`, and multi-REPL eval:

```bash
clojure -Ttools install-latest :lib io.github.bhauman/clojure-mcp :as mcp
claude mcp add clojure-mcp -- clojure -Tmcp start :config-profile :cli-assist
```

Run the `claude mcp add` inside the project directory (it scopes there by default). Use `:cli-assist-full` instead to promote the structural editing tools to first-class. Claude Desktop setups pointing at the same nREPL keep working unchanged — the two clients share a subscription usage pool.

### A.4 What deliberately isn't in this setup

No memory MCP servers (§7 — CLAUDE.md, auto memory, and a maintained PROJECT_SUMMARY.md cover it); no cljfmt-on-every-edit (a per-taste choice, see A.1); no lint hooks by default (add clj-kondo via a PostToolUse hook or the `cleancoders/agent-plugins` marketplace when wanted — noting its clojure plugin bundles the cljfmt behavior). Start minimal; add layers when their absence hurts.
