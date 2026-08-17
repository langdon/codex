> ## ⚠️ This fork is retired — read this before using anything here
>
> This is a personal fork of [openai/codex](https://github.com/openai/codex) that
> existed to prototype **signal-driven control of a running Codex session**: sending
> a signal to an already-running agent to inject instructions or reload config,
> without killing the session and losing its context.
>
> **The work is complete and it works. It is not going anywhere, for two reasons:**
> maintaining a fork of a fast-moving upstream for one feature is not a cost worth
> paying, and **upstream declined the feature.**
>
> ### Upstream verdicts
>
> | Request | Outcome |
> |---|---|
> | [openai/codex#16060](https://github.com/openai/codex/issues/16060) — SIGUSR1 mid-task instruction injection via inbox file | **NOT_PLANNED** (2026-07-01) |
> | [openai/codex#16061](https://github.com/openai/codex/issues/16061) — SIGUSR2 live config/rules reload | **Declined** — closed for insufficient upvotes (the `COMPLETED` state reason on that issue is misleading; read the closing comment) |
> | [anthropics/claude-code#36849](https://github.com/anthropics/claude-code/issues/36849) — the same SIGUSR1 ask, filed against Claude Code | **NOT_PLANNED** (2026-04-18) |
> | [anthropics/claude-code#36847](https://github.com/anthropics/claude-code/issues/36847) — SIGHUP config reload | **NOT_PLANNED** (2026-05-27) |
>
> Two vendors independently declined every request that writes into a running
> agent's instruction stream, while shipping the adjacent *notification* features.
> That reads as a deliberate product boundary rather than an oversight — so this
> approach has no upstream path, in either engine.
>
> ### What is actually here
>
> | Branch | PR | What it does |
> |---|---|---|
> | `codex/implement-maildir-inbox-for-sigusr1` | [#9](../../pull/9) (merged) | SIGUSR1 + maildir inbox: atomic rename, crash recovery, inotify watcher removed |
> | `feat/sigusr2-config-reload` | [#11](../../pull/11) | SIGUSR2 reload of model, base instructions, provider and exec rules from `config.toml`. Does **not** restart MCP servers |
> | `feat/sigusr1-and-sigusr2` | [#6](../../pull/6) | Both handlers combined |
> | `codex/add-sigusr1-signal-handler-to-codex` | [#3](../../pull/3) | Earlier SIGUSR1 launcher plumbing |
> | `codex/add-sigusr1-inbox-injection-to-codex` | [#4](../../pull/4) | Earlier SIGUSR1 core plumbing |
> | `codex/add-sigusr2-config-reload-support` | [#5](../../pull/5) | Earlier SIGUSR2 attempt |
>
> `docs/inbox-injection.md` on the maildir branch is the design doc, and is the best
> starting point if you want the reasoning rather than the diff.
>
> ### Where the idea went
>
> Since the runtime cannot be modified without a fork and upstream will not take the
> change, the capability belongs one layer out — in a message bus between agents
> rather than inside any single agent process. Worth knowing if you are solving the
> same problem: **a bus gives you transport, identity and signing, but it does not
> wake an idle session sitting at a prompt.** Those are separate problems, and this
> fork only ever solved the second one.
>
> PRs here are closed deliberately. Nothing is in flight.

---

<p align="center"><code>npm i -g @openai/codex</code><br />or <code>brew install --cask codex</code></p>
<p align="center"><strong>Codex CLI</strong> is a coding agent from OpenAI that runs locally on your computer.
<p align="center">
  <img src="https://github.com/openai/codex/blob/main/.github/codex-cli-splash.png" alt="Codex CLI splash" width="80%" />
</p>
</br>
If you want Codex in your code editor (VS Code, Cursor, Windsurf), <a href="https://developers.openai.com/codex/ide">install in your IDE.</a>
</br>If you want the desktop app experience, run <code>codex app</code> or visit <a href="https://chatgpt.com/codex?app-landing-page=true">the Codex App page</a>.
</br>If you are looking for the <em>cloud-based agent</em> from OpenAI, <strong>Codex Web</strong>, go to <a href="https://chatgpt.com/codex">chatgpt.com/codex</a>.</p>

---

## Quickstart

### Installing and running Codex CLI

Install globally with your preferred package manager:

```shell
# Install using npm
npm install -g @openai/codex
```

```shell
# Install using Homebrew
brew install --cask codex
```

Then simply run `codex` to get started.

<details>
<summary>You can also go to the <a href="https://github.com/openai/codex/releases/latest">latest GitHub Release</a> and download the appropriate binary for your platform.</summary>

Each GitHub Release contains many executables, but in practice, you likely want one of these:

- macOS
  - Apple Silicon/arm64: `codex-aarch64-apple-darwin.tar.gz`
  - x86_64 (older Mac hardware): `codex-x86_64-apple-darwin.tar.gz`
- Linux
  - x86_64: `codex-x86_64-unknown-linux-musl.tar.gz`
  - arm64: `codex-aarch64-unknown-linux-musl.tar.gz`

Each archive contains a single entry with the platform baked into the name (e.g., `codex-x86_64-unknown-linux-musl`), so you likely want to rename it to `codex` after extracting it.

</details>

### Using Codex with your ChatGPT plan

Run `codex` and select **Sign in with ChatGPT**. We recommend signing into your ChatGPT account to use Codex as part of your Plus, Pro, Team, Edu, or Enterprise plan. [Learn more about what's included in your ChatGPT plan](https://help.openai.com/en/articles/11369540-codex-in-chatgpt).

You can also use Codex with an API key, but this requires [additional setup](https://developers.openai.com/codex/auth#sign-in-with-an-api-key).

## Docs

- [**Codex Documentation**](https://developers.openai.com/codex)
- [**Contributing**](./docs/contributing.md)
- [**Installing & building**](./docs/install.md)
- [**Open source fund**](./docs/open-source-fund.md)

This repository is licensed under the [Apache-2.0 License](LICENSE).
