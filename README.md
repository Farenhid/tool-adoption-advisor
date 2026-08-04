# Tool Adoption Advisor

An AI agent prompt that answers one question well: **should I actually add this tool/library/repo
to my project, or will it just add complexity I don't need?**

Every day there's a new interesting repo, library, or framework. Most of them are not worth
adopting — not because they're bad, but because your project already has something that does
the job, or the added surface area (new dependency, new config format, new mental model for
contributors) costs more than the tool is worth. This agent exists to be the skeptical second
opinion that grounds that decision in your *actual* project, instead of in how exciting the
tool's README sounds.

It defaults to **rejecting**. A clear REJECT backed by real evidence is just as valuable an
outcome as an ADOPT — the agent's job is to protect your project's coherence, not to be a
cheerleader for every new thing you show it.

## What it does

Give it a URL, README, or description of a tool. It will:

1. Restate what the tool actually claims to do (and verify against its real docs/source if it
   can reach them — it doesn't take marketing copy at face value).
2. Investigate your actual project: it looks for `CLAUDE.md`, `AGENTS.md`, `.cursorrules`,
   `CONTRIBUTING.md`, or infers conventions directly from your code and dependencies if none of
   those exist. It never assumes a specific project structure.
3. Classify the **cost** as *shape* or *infrastructure*, because the two have different answers:

   - **Shape cost** — foreign language, awkward API, its own config format. This is interface
     cost, and an adapter genuinely removes it: your project talks to one narrow contract and
     never sees the foreign shape.
   - **Infrastructure cost** — a service to keep running, a database to provision, an API key
     that bills per call. An adapter hides this from the caller but not from the operator, so it
     stays a legitimate reason to reject.

   A foreign runtime *on its own* is never sufficient grounds for REJECT. Without this step the
   agent reliably rejects useful tools for being written in the wrong language — the single most
   common failure mode in practice.
4. Deliver **exactly one verdict**, always backed by a specific fact from your project:

   | Verdict | Meaning |
   |---|---|
   | **REJECT** | Not worth it. States the specific overlap or conflict with what you already have. |
   | **REJECT, BUT LEARN FROM IT** | Skip the tool, but one idea in it is worth borrowing — named precisely, with where to apply it in your existing setup. |
   | **ADOPT BEHIND AN ADAPTER** | Worth having, but the cost is shape. Brings it in as a separate module behind one narrow contract, and names what the adapter does *not* solve. |
   | **REPLACE** | This should replace something you already have. Says what to remove, not just what to add. |
   | **ADOPT** | Genuinely worth adding. Proposes the smallest-footprint way to bring it in, matched to your existing conventions. |

5. Proposes the concrete next step implied by the verdict (which file to edit, what to remove,
   how to integrate) and **asks before making any change** — it never edits your project
   silently off the back of an opinion.
6. Keeps a running decision log (`.tool-adoption-log.md`) in your project so past verdicts aren't
   forgotten or re-litigated every time a similar tool comes up.

## Worked example

**Pitch:** "Found this all-in-one AI agent config framework — ships 280 skills, 67 subagents,
hooks, a security scanner, and a cross-tool memory format. Should we add it?"

**Investigation:** the project already has a handful of narrowly-scoped subagents defined by
hand, a lightweight `CLAUDE.md`, and 5-6 MCP servers already enabled.

**Verdict: REJECT, BUT LEARN FROM IT.**
Adopting the whole framework would duplicate the subagents and rules you've already built by
hand, and its own docs warn that stacking installs is "the most common breakage." Its security
scanner (audits agent config for exposed secrets and hook injection), however, is a real, narrow
idea worth having — recommend running it once as a standalone one-off audit, not adopting the
framework it ships inside of.

**Follow-up proposed:** "Want me to run `<security-scanner-cli>` against this repo's `.claude/`
config as a one-time check? No files need to change to do that."

That's the shape of every response: one verdict, grounded evidence, and a concrete next step —
never a vague "it depends" or a silent file edit.

## Installation

### Claude Code — automatic sub-agent

Claude Code has a native sub-agent system: drop the file in, and Claude Code automatically
routes matching requests to it based on its description — no manual invocation needed most of
the time.

```bash
mkdir -p .claude/agents
curl -o .claude/agents/tool-adoption-advisor.md \
  https://raw.githubusercontent.com/Farenhid/tool-adoption-advisor/main/claude-code/tool-adoption-advisor.md
```

Then just paste a tool/repo link into a Claude Code chat and ask "should we add this?" — Claude
Code should route to the agent on its own. You can also explicitly say "use the
tool-adoption-advisor agent" if it doesn't route automatically.

### Other AI coding tools (Cursor, Codex, Copilot Chat, and similar)

These tools don't currently have Claude Code's automatic sub-agent routing. The same underlying
prompt still works — just as a prompt you paste into a chat, or save wherever your tool loads
custom/reusable instructions from (for example, a saved prompt, a custom instructions file, or a
pinned message). The verdicts and logic are identical; only how you invoke it differs by tool
and may improve over time as these tools add their own agent features.

```bash
curl -o tool-adoption-advisor.prompt.md \
  https://raw.githubusercontent.com/Farenhid/tool-adoption-advisor/main/generic-prompt/tool-adoption-advisor.prompt.md
```

Paste the contents into a new chat along with the tool you want evaluated.

## Files

- `tool-adoption-advisor.md` — the canonical prompt. Harness-agnostic, no dependency on any
  specific MCP server, framework, or project-config format. This is the source of truth; the
  other files are thin wrappers around it.
- `claude-code/tool-adoption-advisor.md` — Claude Code sub-agent format (adds the required
  frontmatter: name, description with trigger examples, model, color).
- `generic-prompt/tool-adoption-advisor.prompt.md` — plain copy of the canonical prompt, for
  pasting into any chat-based tool.

## Design principles

- **Grounded, not generic.** Every verdict must cite a real file or fact from your project.
  Vague "it depends" answers are treated as a failure mode, not an acceptable output.
- **Default to no.** Its job is to protect project coherence. If the evidence for adopting
  something is thin, it says so plainly rather than hedging toward a soft yes.
- **Discovers your conventions, doesn't assume them.** No hardcoded file names or structures —
  it checks common convention files, falls back to reading your actual code if none exist.
- **Proposes, then asks.** It never silently edits your prompts, config, or dependencies off the
  back of its own opinion. A verdict comes with a concrete next step, and execution needs your
  go-ahead.
- **One tool, one job.** It doesn't do product/feature strategy, code review, or security
  auditing — it answers exactly one question well. Combine it with other agents/tools for those.

## License

MIT
