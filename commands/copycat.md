---
description: Clone un site a l'identique depuis une URL (capture complete, extraction des design tokens, reconstruction, verification par diff pixel)
argument-hint: "<url> [--depth 1] [--stack static|next] [--out dossier]"
---

Clone le site suivant **a l'identique** : $ARGUMENTS

Utilise le skill `copycat` et suis-le dans l'ordre, sans sauter d'etape.

Rappels critiques :

1. **Mesure, ne devine pas.** Lance `capture.mjs` d'abord ; fonts, couleurs, tailles, espacements,
   breakpoints, hover et animations viennent de `REPORT.md` / `manifest.json` / `css/`, jamais de ton
   intuition. Le texte vient de `content/text.md`, copie-colle.
2. **Lis les folds un par un** (`screens/desktop/`, puis `screens/mobile/`) avant d'ecrire une ligne.
3. **Fondations avant sections** : `@font-face` + tokens `:root` + body + container + header + meta,
   puis les sections de haut en bas en regardant chaque crop, responsive ecrit en meme temps.
4. **Stack** : si `--stack` n'est pas donne, `next` seulement si le repo courant est un projet Next.js,
   sinon HTML/CSS/JS statique.
5. **Verifie avec `compare.mjs`** contre le clone servi localement (`preview_start`, jamais un
   serveur dans Bash). Cible : grade A (< 2 % d'ecart) et **0** erreur console / requete echouee.
   Corrige la premiere « worst zone », relance, jusqu'a A ou ecart explique. Teste hover et menu
   mobile a la main dans le Browser pane.
6. **Livrable** : ou est le clone et comment le lancer, verdict `compare` par viewport, console,
   liste honnete de ce qui n'est pas reproduit (formulaires sans backend, tiers, fonts remplacees),
   et la note d'usage : logo, photos et textes appartiennent au site original.

Charge `references/fidelity-checklist.md` avant de coder et relis-la avant de livrer.
