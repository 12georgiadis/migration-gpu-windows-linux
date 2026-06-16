# Piloter plusieurs agents Claude Code sur Linux (cmux est macOS-only)

**Note, juin 2026.** Beaucoup pilotent Claude Code via **cmux** (l'app de manaflow-ai), très ergonomique. Mais cmux est **macOS uniquement**. Si on passe son poste de pilotage sous Linux, il faut un équivalent. Voici l'état des lieux et les alternatives, vérifiés.

---

## Pourquoi cmux ne passe pas sur Linux

`cmux` (manaflow-ai) est une **app macOS native** : écrite en **Swift + AppKit**, moteur terminal **Ghostty** (libghostty), sous licence GPL-3. Le dépôt n'annonce **aucun support Linux/Windows ni roadmap de portage**.

**Le port serait-il facile ? Non.** AppKit, la couche d'interface de macOS, **n'existe pas sur Linux**. Swift, lui, tourne sur Linux (serveur/CLI), mais toute l'interface de cmux (onglets verticaux, panneau de notifications, rendu natif) devrait être **réécrite** en GTK ou Qt. C'est une réécriture, pas un portage. Seul **Ghostty**, le terminal sous-jacent, est déjà multiplateforme.

---

## Nuance d'architecture (importante)

Si on pilote des machines GPU Linux **headless** (sans écran), le multiplexer vit sur la **machine de contrôle** (le poste de travail), pas sur les serveurs GPU. Ceux-ci sont des cibles SSH. Donc :

- Migrer ses **rigs GPU** vers Linux **ne touche pas** au multiplexer de pilotage : il reste sur le poste de contrôle (Mac ou Linux).
- La question "cmux sur Linux" ne se pose **que** si on veut aussi mettre son **poste de pilotage** sous Linux.

---

## Les alternatives Linux

| Outil | Ce que c'est | Linux | Proximité cmux |
|---|---|---|---|
| **Claude Squad** | TUI qui gère N agents (Claude Code, Codex, Gemini, Aider, OpenCode), chacun dans son **git worktree** isolé. Nécessite `tmux` et `gh`. | Oui (natif) | Le plus proche en terminal |
| **Zellij** | Multiplexer terminal moderne en Rust : panes, tabs, layouts, sessions, UI découvrable. | Oui (natif) | Le "tmux en mieux" pour empiler des agents |
| **craigsc/cmux** | Un **autre** projet homonyme : wrapper **bash** autour des git worktrees, qui lance un agent Claude par branche (`cmux new <branche>`, `cmux merge`, `cmux rm`). Pas une GUI ni un multiplexer. Pur bash (git + Claude CLI). | macOS **et Linux** | `curl ... install.sh \| sh` |
| **tmux** | Le multiplexer historique, socle fiable et scriptable. | Oui (natif) | La base |
| **vibe-kanban / Nimbalyst** (ex-Crystal) | UI **web** kanban pour orchestrer plusieurs sessions d'agents. | Cross-platform | Pour qui préfère le visuel au terminal |
| **Agent Teams** (Claude Code natif) | Orchestration multi-agents intégrée, sans app tierce. | Partout | Complément, pas un multiplexer |

---

## Recommandation

Sur Linux, le combo qui se rapproche le plus de l'ergonomie cmux :

**Zellij** (multiplexer ergonomique) **+ Claude Squad** (orchestration multi-agents par worktree) **+ Ghostty** comme terminal.

Installation type :
```bash
# Zellij (multiplexer)
cargo install zellij        # ou: paquet de la distro / binaire GitHub

# Claude Squad (orchestrateur d'agents, binaire 'cs')
curl -fsSL https://raw.githubusercontent.com/smtg-ai/claude-squad/main/install.sh | bash
# prérequis : tmux + gh installés
```

Ces deux outils tournent **à l'identique sur macOS et Linux** : on peut tester l'ergonomie sur un poste macOS aujourd'hui, elle transférera telle quelle sur un poste Linux.

---

## Licence

Texte sous CC BY-NC-ND 4.0, extraits de code sous PolyForm Noncommercial 1.0.0. Voir `LICENSE`.

*Sources : github.com/manaflow-ai/cmux · github.com/craigsc/cmux · github.com/smtg-ai/claude-squad · zellij.dev · ghostty.org*
