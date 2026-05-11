# Agentic_mit

**Privat. Doar pentru uz intern MIT Consulting.**

Marketplace de plugin-uri Claude Code și convenții de "agentic coding" pentru echipa MIT Dev.

## Plugin-uri în acest repo

| Plugin | Status | Scop | Skills active |
|---|---|---|---|
| [`mit-dev`](./plugins/mit-dev) | `0.1.0` | Toolkit Jira pentru devs MIT | 0 (spec aprobat pentru `mit:log-hours` — implementare în `0.2.0`) |

## Quick start

**1. Adaugă marketplace-ul în Claude Code:**
```
/plugin marketplace add github://diaconu93/Agentic_mit
```

**2. Instalează plugin-ul `mit-dev`:**
```
/plugin install mit-dev@mit-devs
```

**3. La primul call Jira**, Claude Code declanșează OAuth în browser. Autentifică-te cu MIT Google SSO. One-time.

**4. Verifică:**
```
/plugin list
```
Trebuie să vezi `mit-dev v0.1.0  enabled`.

## Ce conține `mit-dev` astăzi (v0.1.0)

- **MCP server**: Atlassian (Jira + Confluence) wired via OAuth — fără setup local de token-uri.
- **Skills**: 0 (scaffold). Primul skill — `mit:log-hours` — e specificat și aprobat, [vezi designul aici](./plugins/mit-dev/docs/specs/2026-05-11-mit-log-hours-design.md). Iese în `0.2.0`.

Roadmap full în [plugin README](./plugins/mit-dev/README.md#skills).

## Ce va aduce `mit:log-hours` (0.2.0, în implementare)

Backfill worklog Jira din git commits. Pe scurt:
- Rulezi `/mit:log-hours` în repo-ul tău MIT.
- Skill-ul citește commit-urile tale din ultima lună, le mapează la issue-urile Jira (după cheia din mesaj, ex. `AIC-215`), estimează 8h/zi distribuite proporțional pe commit-uri, și postează worklog-uri cu confirmare per-zi.
- Pentru zile Lu-Vi fără commit-uri și nelogate, te întreabă pe ce ai lucrat (listă issue-uri active).

Detalii complete (algoritm, edge cases, reguli, flow): [`docs/specs/2026-05-11-mit-log-hours-design.md`](./plugins/mit-dev/docs/specs/2026-05-11-mit-log-hours-design.md).

## Structura repo-ului

```
Agentic_mit/
├── .claude-plugin/
│   └── marketplace.json              # registrul marketplace-ului ("mit-devs")
├── plugins/
│   └── mit-dev/                      # primul plugin
│       ├── .claude-plugin/plugin.json
│       ├── .mcp.json                 # Atlassian MCP (remote HTTP, OAuth)
│       ├── README.md                 # user guide al plugin-ului
│       ├── docs/specs/               # design docs aprobate
│       └── skills/                   # SKILL.md per skill (curent gol)
└── README.md
```

## Cum se adaugă un plugin nou în marketplace

1. Folder: `plugins/<plugin-name>/` cu `.claude-plugin/plugin.json`, `.mcp.json` (dacă e cazul), `skills/`, `README.md`.
2. Înregistrează în `.claude-plugin/marketplace.json` (entry nou în `plugins[]` cu `name`, `source`, `description`, `version`).
3. PR contra `main` cu spec design în `plugins/<name>/docs/specs/`. Bump versiune la release.

## Cum se adaugă un skill nou într-un plugin existent

1. Brainstorm + spec → `plugins/<plugin>/docs/specs/YYYY-MM-DD-<skill>-design.md`.
2. Spec aprobat → `plugins/<plugin>/skills/<skill>/SKILL.md` cu frontmatter `name: <prefix>:<skill>` + `description:` clar.
3. Bump versiune plugin în `plugin.json` + `marketplace.json`.
4. PR contra `main`. Testare locală cu `/plugin install <local-path>` înainte de merge.

## Cerințe acces

Trebuie să fii collaborator pe repo-ul `diaconu93/Agentic_mit`. Dacă nu ești încă adăugat, scrie owner-ului (`@diaconu93`).

Pentru auth GitHub: `gh auth login` sau SSH key configurat — Claude Code clonează repo-ul în spatele tău.

## Convenții (agentic coding standards)

Acest repo nu e doar un container de plugin-uri — e și un loc de standarde pentru cum scriem skill-uri/MCP-uri pentru echipa MIT:

- **Spec înainte de cod** — orice skill nou trece prin brainstorming + design doc înainte de implementare.
- **Single-tenant** — cloudId MIT și host hardcoded în skill-uri; plugin-ul nu e gândit să fie redistribuit.
- **Per-day human-in-the-loop** pentru orice POST în Jira — nu postăm fără confirmare explicită.
- **Jira-only** în `mit-dev` — YouTrack și alte sisteme intră în plugin-uri separate dacă va fi nevoie.
