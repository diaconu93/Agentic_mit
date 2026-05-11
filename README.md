# Agentic_mit

**Privat. Doar pentru uz intern MIT Consulting.**

Marketplace de plugin-uri Claude Code și convenții de "agentic coding" pentru echipa MIT Dev.

## Plugin-uri în acest repo

| Plugin | Status | Scop |
|---|---|---|
| [`mit-dev`](./plugins/mit-dev) | `0.1.0` scaffold | Toolkit Jira pentru devs MIT |

## Cum se adaugă acest marketplace în Claude Code

```
/plugin marketplace add github://diaconu93/Agentic_mit
```

Apoi instalează plugin-urile dorite:

```
/plugin install mit-dev@mit-devs
```

## Structura

```
Agentic_mit/
├── .claude-plugin/
│   └── marketplace.json       # registrul marketplace-ului
├── plugins/
│   └── mit-dev/               # primul plugin
└── README.md
```

## Cum se adaugă un plugin nou

1. `plugins/<plugin-name>/` cu `.claude-plugin/plugin.json` și restul componentelor.
2. Înregistrează plugin-ul în `.claude-plugin/marketplace.json` (entry nou în `plugins[]`).
3. PR contra `main`. Bump versiune la release.

## Cerințe acces

Trebuie să fii collaborator pe repo-ul `diaconu93/Agentic_mit`. Dacă nu ești încă adăugat, scrie owner-ului.
