# Architecture — Pipeline SEO Manager in Motion

## Contexte

Pipeline automatisé de production de contenu SEO hebdomadaire pour **Manager in Motion**, cabinet de management de transition opérant principalement en Espagne et au Portugal.

Chaque semaine, le pipeline génère automatiquement un article SEO complet en espagnol, prêt pour relecture et publication sur le site Joomla.

---

## Vue d'ensemble

```
[Claude Code Routine]
       │ Déclenché chaque lundi à 8h00 (Madrid)
       ▼
┌─────────────────┐     output/agent1_report.md
│    Agent 1      │ ──────────────────────────────┐
│  Inteligencia   │                               │
│  de Mercado     │                               ▼
└─────────────────┘              ┌─────────────────┐     output/agent2_brief.md
                                 │    Agent 2      │ ──────────────────────────┐
                                 │  Estratega SEO  │                           │
                                 └─────────────────┘                           ▼
                                                          ┌─────────────────────────┐
                                                          │       Agent 3           │
                                                          │  Redactor de Contenido  │
                                                          └──────────┬──────────────┘
                                                                     │
                                                          output/agent3_article.docx
                                                                     │
                                                          ┌──────────▼──────────┐
                                                          │   Git push → GitHub  │
                                                          └──────────┬──────────┘
                                                                     │
                                                          ┌──────────▼──────────┐
                                                          │  Relecture (Arnaud)  │
                                                          │  + Publication Joomla│
                                                          └─────────────────────┘
```

---

## Composants

### Orchestration : Claude Code Routines

L'orchestration repose sur les **Claude Code Remote Routines** (CCR), des sessions isolées dans le cloud Anthropic.

| Paramètre | Valeur |
|-----------|--------|
| Routine ID | `trig_01Jpywe6jncJJaQjGoYRiVCP` |
| URL de gestion | https://claude.ai/code/routines/trig_01Jpywe6jncJJaQjGoYRiVCP |
| Environnement | `env_01Pbb5KLFZ8niXzRiCrWBxDf` (Default) |
| Modèle | `claude-sonnet-4-6` |
| Planning | Chaque lundi à 6h00 UTC (`0 6 * * 1`) = 8h00 Madrid (CEST) |
| Dépôt source | `https://github.com/arnaudatwork-hub/mimo-seo` |

> **Note heure hivernale :** En CET (UTC+1, novembre–mars), le pipeline tourne à 7h00 Madrid au lieu de 8h00. Ajuster le cron à `0 7 * * 1` sur https://claude.ai/code/routines si nécessaire.

### Recherche web : Tavily MCP

| Paramètre | Valeur |
|-----------|--------|
| Connecteur | Tavily (`connector_uuid: 373b2714-657a-45e8-bfdf-0dff100326a9`) |
| URL | `https://mcp.tavily.com/mcp` |
| Usage | Agent 1 uniquement — scan hebdomadaire des tendances |

### Stockage et versioning : GitHub

| Paramètre | Valeur |
|-----------|--------|
| Repo | `https://github.com/arnaudatwork-hub/mimo-seo` |
| Branche principale | `master` |
| Fichiers de sortie | `output/` (commités automatiquement à chaque run) |

---

## Les trois agents

### Agent 1 — Inteligencia de Mercado

**Fichier :** `.claude/agents/prompt-agent-1.md`

Scanne les sources publiques via Tavily pour identifier les 5 principales tendances hebdomadaires liées à la gestion intérimaire en Espagne et en Europe. Produit un rapport structuré incluant le contenu observé chez les concurrents, les FAQ pertinentes, et une recommandation de sujet pour l'article de la semaine.

- **Input :** recherche web temps réel (Tavily)
- **Output :** `output/agent1_report.md`
- **Langue :** espagnol

### Agent 2 — Estratega SEO

**Fichier :** `.claude/agents/prompt-agent-2.md`

Transforme le rapport de l'Agent 1 en une stratégie SEO complète pour un article. Sélectionne la meilleure opportunité selon le potentiel B2B, l'intention de recherche et l'adéquation avec le positionnement Manager in Motion (audience cible : CEOs, CHROs, CFOs, COOs, investisseurs PE, leaders de transformation).

- **Input :** `output/agent1_report.md`
- **Output :** `output/agent2_brief.md`
- **Langue :** espagnol

### Agent 3 — Redactor de Contenido SEO

**Fichier :** `.claude/agents/prompt-agent-3.md`

Rédige l'article SEO complet (~2 000 mots) en espagnol à partir du brief de l'Agent 2. Génère ensuite le fichier Word `.docx` avec styles formatés (titres H1/H2/H3 colorés, police Arial, format A4) via la librairie Node.js `docx`. Commit et push le résultat sur GitHub.

- **Input :** `output/agent2_brief.md`
- **Output :** `output/agent3_article.docx`
- **Langue :** espagnol
- **Ton :** niveau exécutif, 90 % valeur, 10 % positionnement commercial

---

## Structure du dépôt

```
mimo-seo/
├── .claude/
│   ├── agents/
│   │   ├── prompt-agent-1.md   ← Agent 1 : Inteligencia de Mercado
│   │   ├── prompt-agent-2.md   ← Agent 2 : Estratega SEO
│   │   └── prompt-agent-3.md   ← Agent 3 : Redactor de Contenido
│   └── settings.local.json
├── output/
│   ├── agent1_report.md        ← Rapport hebdomadaire (écrasé chaque semaine)
│   ├── agent2_brief.md         ← Brief SEO (écrasé chaque semaine)
│   └── agent3_article.docx     ← Article final (écrasé chaque semaine)
├── .gitignore
└── ARCHITECTURE.md
```

> Les fichiers `output/` sont écrasés à chaque run. L'historique des articles est conservé via l'historique git (chaque push hebdomadaire crée un commit daté).

---

## Flux de données

```
Tavily (web)
     │
     ▼
Agent 1 → output/agent1_report.md
                    │
                    ▼
          Agent 2 → output/agent2_brief.md
                              │
                              ▼
                    Agent 3 → output/agent3_article.docx
                                          │
                                          ▼
                                   git commit + push
                                          │
                                          ▼
                              GitHub (mimo-seo/output/)
```

---

## Workflow de publication

1. **Lundi ~8h15** — Le pipeline termine. Un nouveau commit apparaît sur `github.com/arnaudatwork-hub/mimo-seo`
2. **Relecture** — Arnaud télécharge `output/agent3_article.docx` depuis GitHub
3. **Vérification** — Relecture du contenu, ajustements éventuels
4. **Publication** — Copier-coller dans l'interface d'administration **Joomla**

---

## Contraintes connues

| Contrainte | Description |
|------------|-------------|
| Pas d'interface de relecture | Accès au .docx uniquement via GitHub |
| Heure d'été / heure d'hiver | Le cron UTC ne s'adapte pas automatiquement (voir note planning) |
| Historique non structuré | Les articles passés sont dans l'historique git, pas dans une base de données |
| Dépendance Anthropic Cloud | Si les Routines Claude sont indisponibles, le pipeline ne tourne pas |
| Pas de régénération partielle | Impossible de rejouer un seul agent sans relancer toute la chaîne |

---

## Évolutions futures envisagées

- **Intégration Joomla API** pour publication automatique après validation
