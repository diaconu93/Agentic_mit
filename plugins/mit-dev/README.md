# mit-dev

**Privat. Doar pentru uz intern MIT Consulting. Nu redistribui.**

Plugin Claude Code pentru developerii MIT Consulting. Bundle de MCP-uri și skill-uri care încapsulează conveniile companiei (cloudId Jira, host, JQL standard) și expun workflow-uri standardizate.

**Scope-ul actual e exclusiv Jira.** YouTrack e explicit exclus — dacă va fi nevoie, va fi un plugin separat.

## Status

| Versiune | Conținut |
|---|---|
| `0.1.0` (current) | Scaffold + Atlassian MCP. Spec aprobat pentru `mit:log-hours` (implementare în 0.2.0). |

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

| Skill | Scop | Status |
|---|---|---|
| `mit:log-hours` | Backfill worklog Jira din commit-uri | Design aprobat ([spec](./docs/specs/2026-05-11-mit-log-hours-design.md)) |
| `mit:hours-report` | Raport săptămânal worklog per persoană | Planificat |
| `mit:bookstack-doc` | Wrap pages create/update BookStack | Planificat |
| `mit:standup` | Yesterday/today/blockers din worklog | Planificat |
| `mit:pr-link` | Auto-comment Jira cu PR URL din current branch | Planificat |

---

## Cum folosești `mit:log-hours`

> ⚠️ În `0.1.0` skill-ul e doar în spec — implementarea vine în următorul release. Documentația de mai jos descrie cum se va folosi după release.

### Ce face

Scanează commit-urile tale din **repo-ul curent (cwd)**, le mapează la issue-urile Jira după cheia din mesaj (ex. `AIC-215`), estimează 8h/zi distribuite proporțional pe commit-uri, și postează worklog-urile cu confirmare per-zi.

Pentru zile lucrătoare (Lu-Vi) din ultima lună **fără worklog Jira**:
- Dacă ai commit-uri → propune o distribuție 8h pe issue-urile atinse.
- Dacă NU ai commit-uri → te întreabă pe ce ai lucrat (listă issue-uri active assignate).

### Comenzi

**Backfill ultima lună (default):**
```
/mit:log-hours
```

**Window custom:**
```
/mit:log-hours --since 2026-04-15 --until 2026-05-10
```

**Dry-run** (nu face POST, doar afișează planul):
```
/mit:log-hours --dry-run
```

### Reguli (firm)

| Regulă | Detaliu |
|---|---|
| Max 8h/zi | Algoritmul nu generează > 8h. User input la zile fără commit e validat ≤ 8h. |
| Doar Lu–Vi | Weekend-urile sunt ignorate complet. Sărbători legale nu sunt detectate. |
| Skip ziua deja logată | Dacă ai **orice** worklog (chiar 1m) pe ziua aia, skîll-ul sare peste — nu se face top-up. |
| Per repo | Skill-ul se uită doar la `cwd`. Pentru altă bază de cod, `cd` în alt repo și rulează din nou. |
| Per-zi confirmare | Vezi preview cu issue-uri + ore + comment înainte ca POST să se întâmple. |
| Comment = lista commit-urilor | Worklog-ul postat are subject + hash scurt pentru fiecare commit al zilei. |

### Algoritm de estimare (rezumat)

```
T = total commit-uri în zi (commit-urile multi-key contează 1)
Pentru fiecare issue atins în acea zi:
  W = suma fracțiunilor (1/k pentru fiecare commit unde issue-ul apare alături de k-1 alte chei)
  hours = 8 × (W / T), rotunjit la 15min
Ultimul issue absoarbe drift-ul de rotunjire ca suma zilei să fie exact 8h.
```

Exemplu: 3 commit-uri într-o zi: 2 pe `AIC-215`, 1 pe `AIC-215`+`BRON-499` (multi-key).
- `AIC-215`: `(1 + 1 + 0.5) / 3 × 8 ≈ 6.67h → 6h 45m`
- `BRON-499`: `0.5 / 3 × 8 ≈ 1.33h → 1h 15m` (cu drift absorbit ca suma să fie 8h)

### Flow practic

1. Rulezi `/mit:log-hours` într-un repo MIT (după ce ai făcut commit-urile zilei).
2. Skill afișează scan summary: `42 commits in 18 workdays, 7 days already logged (skip), 11 days to review`.
3. Pentru fiecare zi netagged, în ordine cronologică:
   - **Zi cu commit-uri**: vezi tabel `Issue | Hours | Comment`. Confirm / Edit / Skip.
   - **Zi fără commit**: alegi din lista issue-urilor active, introduci orele manual, sau "Skip — zi liberă".
4. La final, raport: ce s-a postat, ce s-a sărit, ce a eșuat (cu motive).

### Edge cases comune

- **Commit fără cheie Jira** (`refactor utils`) → ignorat la estimare, listat la final.
- **Cheie inexistentă în MIT Jira** (`FAKE-999`) → exclus, raportat la final.
- **`git config user.email` lipsă** → skill întreabă inline.
- **Cwd nu e git repo** → eroare imediată, sugerează `cd`.
- **OAuth Atlassian expirat** → mesaj clar, te trimite la `/plugin atlassian` pentru reauth.

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
