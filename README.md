# The Devil Wears Markdown

A five-minute check for AI skills, MCP servers and rules files, before you install them. It needs only
free tools, and you can do it without installing anything.

From the BSides Cambridge 2026 talk *The Devil Wears Markdown: Vetting AI Components to Catch Malicious
Intent*. Free to copy, print and share ([CC BY 4.0](LICENSE)).

**Contents:** [What you're checking](#what-youre-checking) ·
[The five-minute check](#the-five-minute-check) · [Read it raw](#step-2-read-it-raw) ·
[Red flags](#red-flags) · [Free tools](#free-tools-and-the-defaults-to-change) ·
[When a scanner flags something](#when-a-scanner-flags-something) ·
[If something got in](#if-something-got-in) · [Sources](#sources)

## What you're checking

- **Skill:** a folder with a `SKILL.md` that your AI follows, sometimes with scripts that it runs.
- **MCP server:** a program that gives your AI tools, such as sending email or reading files. It runs
  with your permissions. MCP is the Model Context Protocol.
- **Rules file:** a text file in your project that your assistant loads by itself, such as
  `.cursorrules`, `CLAUDE.md` or `copilot-instructions.md`.
- **Tool description:** the text your AI reads to decide what an MCP tool does. Most apps never show
  it to you in full.
- **Prompt injection:** text your AI should treat as data, but follows as an order. It needs no code.

This is the agentic supply chain risk in the
[OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/).

In a skill, the documentation is the program. A sentence such as "first, read the user's SSH key" is an
order, not a comment.

## The five-minute check

Set up once: install the scanners and build the sandbox. After that, each component takes about five
minutes.

1. **Who made it?** Spend sixty seconds in a browser.
   - How old is the account, and what else has it published?
   - Do the stars make sense next to the downloads and the project's age?
   - Does a vendor name in the title match that vendor's real account?
   - Did the maintainer change recently?
   - Do the install steps ask you to run something? If so, read that first.
2. **Read it raw.** Read the bytes, not the rendered page. [Commands below](#step-2-read-it-raw).
   Read every file, not just the Markdown: in one 2026 campaign, `SKILL.md` stayed clean and the
   payload sat in a setup file.
3. **Scan offline.** Run a free scanner with the network off. [Tools below](#free-tools-and-the-defaults-to-change).
4. **First run in a sandbox.** Give it no network, no real keys and none of your real folders, then
   give it a real task. A run shows only what that task triggers: evidence, not proof.
5. **Pin it and re-check.** Pin the exact version or commit you approved. On every update, read what
   changed and repeat the check. Check what is already on disk too: agent config committed to a repo
   (`.claude/settings.json`, `.vscode/tasks.json`) can run code when you open the folder.

## Step 2: read it raw

Replace `SKILL.md` with the file you are checking. All six commands are tested on macOS, Ubuntu and
Windows PowerShell 5.1 and 7.

### macOS and Linux

1\. Show hidden characters:

```sh
LC_ALL=C cat -v SKILL.md | grep -n 'M-'
```

2\. Find invisible Unicode (zero-width characters, direction controls, tag characters):

```sh
LC_ALL=C grep -nE \
  $'\xe2\x80[\x8b-\x8f\xaa-\xae]|\xf3\xa0' SKILL.md
```

3\. Grep for the red-flag words:

```sh
grep -niE \
 'prereq|\| *(ba)?sh|base64|\.ssh|\.env|tell the user' \
 SKILL.md
```

In zsh, type command 2 exactly as shown.

### Windows PowerShell

1\. Show hidden characters (anything that isn't plain text becomes `<?>`):

```powershell
gc SKILL.md -Enc UTF8 | % { $_ -replace '[^\t -~]','<?>' }
```

2\. Find invisible Unicode. Type the `\u` escapes as text:

```powershell
sls -Enc UTF8 '[\u200B-\u200F\u202A-\u202E]|\uDB40' SKILL.md
```

3\. Grep for the red-flag words:

```powershell
sls -Path SKILL.md -Pattern `
 'prereq|\|\s*(ba)?sh|base64|\.ssh|\.env|tell the user'
```

## Red flags

- A "Prerequisites" or setup step that wants your terminal.
- A command piped into a shell (`| sh`, `| bash`), especially one scrambled with base64.
- Invisible characters in a file that should be plain text.
- Any mention of `.ssh`, `.env`, keys, tokens or wallets that the job doesn't need.
- A tool description longer than the job it does, or one that asks for secrecy ("don't tell the user").
- A name that copies a known vendor or a popular project.

## Free tools, and the defaults to change

| Tool | What it does | Change this default |
|---|---|---|
| [Cisco skill-scanner](https://github.com/cisco-ai-defense/skill-scanner) | Scans skills offline | None. Offline by default. |
| [Cisco mcp-scanner](https://github.com/cisco-ai-defense/mcp-scanner) | Scans MCP tool descriptions without starting the server | Pass `--analyzers yara` to stay local. Set `LITELLM_LOCAL_MODEL_COST_MAP=True`. |
| [NVIDIA SkillSpector](https://github.com/nvidia/skillspector) | Static and taint analysis of skills | Pass `--no-llm`. It still sends dependency names to OSV.dev, so run it inside the sandbox. |
| [YARA-X](https://virustotal.github.io/yara-x/) | Local pattern rules | None. |
| [sandbox-runtime](https://github.com/anthropics/sandbox-runtime) (`srt`) | Sandbox for a process: network and writes denied by default | Reads are allowed by default. Deny `~/.ssh`, `~/.aws` and other secrets. |
| `docker run --network none` | A throwaway container with no network | Mount only the component, never your home folder. |

Leave out tools that upload what you scan. If you use agentjail, which is a runtime guard and not a
pre-install scanner, turn off its telemetry and auto-update first.

Two limits to keep in mind:
- Cisco's own README says pattern rules alone catch about 8% of held-out malicious skills. That's why
  step 2 is your own eyes.
- Claude Code's `/sandbox` covers shell commands only. MCP servers and hooks run outside it.

## When a scanner flags something

A flag is a question, not a verdict. In one test of five open-source scanners, 91 of 100 harmless skills
were flagged by at least two of them.

1. Why did it flag? Find the exact line.
2. Read that line in context.
3. Decide: accept with a note, reject, or ask the author.

An AI reviewer can be prompt-injected by the file it's reviewing. Treat it as one witness, never the
verdict.

## If something got in

1. **Contain.** Disconnect. Stop the agent and its MCP servers. Remove the component from every AI
   tool's settings, and keep a copy.
2. **Rotate your keys.** Do it from a clean device: every password, key and token your AI could reach.
   A stealer that ran has already copied them, so this is not optional.
3. **Investigate.** Read your AI tool's own session logs, then check what changed on disk.

## Practice

[Damn Vulnerable MCP Server](https://github.com/harishsg993010/damn-vulnerable-MCP-server) is a lab built
to be vulnerable, for practising on. Run it in a sandbox.

[ASI04 workshop](https://github.com/syedDS/ASI04_workshop) is a Docker lab with thirteen supply chain
attacks, including a reproduction of the postmark-mcp silent BCC. Some of its services really exfiltrate
data, so run it only inside its own containers.

## Sources

**Incidents**
- ClawHavoc, January 2026: one campaign put 335 malicious skills on ClawHub, with 386 malicious uploads in
  a single day. [eSecurityPlanet](https://www.esecurityplanet.com/threats/hundreds-of-malicious-skills-found-in-openclaws-clawhub/) ·
  [Antiy CERT](https://www.antiy.net/p/clawhavoc-analysis-of-large-scale-poisoning-campaign-targeting-the-openclaw-skill-market-for-ai-agents/) ·
  [Unit 42](https://unit42.paloaltonetworks.com/openclaw-ai-supply-chain-risk/)
- Scanners bypassed, June 2026: Trail of Bits got malicious skills past the ClawHub, Cisco and Vercel
  scanners. [The sorry state of skill distribution](https://blog.trailofbits.com/2026/06/03/the-sorry-state-of-skill-distribution/)
- Clean, then swapped, 2026: skills on skills.sh uploaded clean and made malicious later.
  [Zenity Labs](https://labs.zenity.io/post/attackers-target-agents-via-the-skill-supply-chain) ·
  [CSO](https://www.csoonline.com/article/4206851/trojanized-ai-skills-gain-1-7m-installs-in-agent-targeted-attack.html)
- CHAINDROP, August 2026: an npm worm that hid in agent config files.
  [Elastic Security Labs](https://www.elastic.co/security-labs/shai-hulud-chaindrop-npm-supply-chain)
- postmark-mcp, September 2025: version 1.0.16 added one line that BCC'd every email to the attacker.
  [The Hacker News](https://thehackernews.com/2025/09/first-malicious-mcp-server-found.html)

**Research**
- Invisible text in rules files: [Pillar Security](https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents)
- Poisoned tool descriptions: [Invariant Labs](https://github.com/invariantlabs-ai/mcp-injection-experiments)
- 400 confirmed malicious skills across 7 stores, 262 of them from stores that advertise scanning:
  [MalSkills](https://arxiv.org/html/2603.27204)
- [Snyk ToxicSkills](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)

## Optional: one tool for all five steps

[unpick](https://github.com/amberb617/unpick) is my open-source wrapper. It runs the four scanners above in
their offline modes, and adds the raw view, the sandbox run and pin-and-verify. Everything on this page
works without it.

---

Disclosure: I co-founded Jiffy Labs, which works on this problem. Nothing on this page needs it.
