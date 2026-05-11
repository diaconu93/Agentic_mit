# mit-dev

**Privat. Doar pentru uz intern MIT Consulting. Nu redistribui.**

Plugin Claude Code pentru developerii MIT Consulting. Bundle de MCP-uri și skill-uri care încapsulează conveniile companiei (cloudId Jira, host, JQL standard) și expun workflow-uri standardizate.

**Scope-ul actual e exclusiv Jira.** YouTrack e explicit exclus — dacă va fi nevoie, va fi un plugin separat.

## Status

| Versiune | Conținut |
|---|---|
| `0.1.0` (current) | Scaffold + Atlassian MCP. Fără skill-uri încă. |

Roadmap: vezi secțiunea de mai jos.

## Cerințe

1. **Acces la repo-ul privat** `diaconu93/Agentic_mit` (collaborator pe repo).
2. **GitHub CLI autentificat** sau cheie SSH configurată — Claude Code clonează repo-ul în spatele tău:
   ```bash
   gh auth login
   ```
3. **Claude Code v2+** cu sistemul de plugin-uri activat.

## Install

```
/plugin marketplace add github://diaconu93/Agentic_mit
/plugin install mit-dev@mit-devs
```

Verifică:
```
/plugin list
```
Așteptat: `mit-dev v0.1.0  enabled`.

## OAuth Atlassian

La prima interacțiune cu Jira, Claude Code va declanșa OAuth în browser. Autentifică-te cu Google SSO MIT (`@mit-dev.com`). E one-time per mașină.

Plugin-ul **nu stochează credențiale** — token-ul OAuth e gestionat de Claude Code, nu de cod-ul plugin-ului.

## Skills

Niciun skill în `0.1.0`. Roadmap-ul scurt:

| Skill | Scop | Status |
|---|---|---|
| `mit:log-hours` | Logare worklog în Jira | În design |
| `mit:hours-report` | Raport săptămânal worklog per persoană | Planificat |
| `mit:bookstack-doc` | Wrap pages create/update BookStack | Planificat |
| `mit:standup` | Yesterday/today/blockers din worklog | Planificat |
| `mit:pr-link` | Auto-comment Jira cu PR URL din current branch | Planificat |

## Update

```
/plugin marketplace update mit-devs
```

## Troubleshooting

**`Could not clone repo`** la `/plugin marketplace add`:
- Verifică `gh auth status` — trebuie să fii logat.
- Verifică că ești collaborator pe repo-ul `diaconu93/Agentic_mit`.
- Verifică SSH/HTTPS access: `git clone git@github.com:diaconu93/Agentic_mit.git /tmp/test && rm -rf /tmp/test`.

**`OAuth required`** la primul call Jira:
- E normal prima dată. Completează în browser.
- Dacă persistă, deautentifică-te din `/plugin atlassian` și relogheză-te.

**`Skill not found`** la `/mit:log-hours`:
- Verifică `/plugin list` — `mit-dev` trebuie să fie enabled.
- Skill-urile din `skills/` se descoperă automat. Dacă lipsesc, plugin-ul e probabil în starea `0.1.0` (scaffold only) — așteaptă următorul release.

## Contribuții

1. Branch din `main`: `feat/<nume-scurt>` sau `fix/<nume-scurt>`.
2. Test local înainte de PR:
   ```
   /plugin install /Users/<tu>/repos/agentic-coding-standards/plugins/mit-dev
   ```
3. PR contra `main`. Code owners din echipa MIT Dev review-uiesc.
4. Bump versiune în `plugin.json` + `marketplace.json` pentru orice schimbare user-visible.

Skill nou:
- Folder: `plugins/mit-dev/skills/<nume>/SKILL.md`.
- Frontmatter cu `name: mit:<nume>` și `description:` clar (triggere de invocare).
- Urmează modelul plugin-ului oficial `atlassian` din claude-plugins-official.

## Filing issues

Bug-uri și feature requests: issues în acest repo (`diaconu93/Agentic_mit`).

**Probleme de securitate**: NU pe GitHub. Trimite intern pe canalul `#security` în Slack MIT.
