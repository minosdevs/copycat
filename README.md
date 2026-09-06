# copycat

**Français** · [English](./README.en.md)

Un skill Claude Code qui prend une URL et te rend **le même site**. Pas « dans l'esprit de » : le
même. Mêmes fonts, mêmes couleurs, mêmes espacements, même responsive, mêmes hovers, mêmes
animations, console propre.

```
/copycat https://exemple.com
```

---

## Pourquoi

Quand tu demandes à une IA de « refaire ce site », elle regarde un screenshot, devine une font,
arrondit les espacements, choisit un gris « proche », oublie le hover, et te rend un truc qui
ressemble à 70 %. Les 30 % restants, c'est tout ce qui fait qu'un site a l'air pro.

copycat remplace la devinette par la **mesure** :

| Sans copycat | Avec copycat |
|---|---|
| Un screenshot du haut de page | Full-page + fold par fold sur desktop, tablet, mobile + un crop par section + les hovers |
| « Ça ressemble à Inter » | `document.fonts` + tous les `@font-face`, fichiers `.woff2` téléchargés, graisses réellement utilisées |
| « Un gris clair » | Palette calculée et pondérée par surface, variables CSS de `:root`, gradients et ombres exacts |
| « Environ 40px » | Typo calculée par balise (taille, poids, interligne, tracking, couleur), containers, breakpoints, échelle d'espacements |
| Hover et animations oubliés | États hover mesurés (avant → après), keyframes, transitions, éléments sticky, reveal-on-scroll |
| « Je pense que c'est bon » | Diff pixel original vs clone, zones les plus fausses mappées aux sections, console et réseau du clone à 0 erreur |

---

## Ce que ça fait, concrètement

### 1. `capture.mjs` — photographier et disséquer l'original

```bash
node skills/copycat/scripts/capture.mjs https://exemple.com
```

Produit `copycat/exemple.com/` :

```
REPORT.md                    ← le résumé lisible : stack, fonts, palette, tokens, typo, layout, sections, CTAs, hover, motion, console
manifest.json                ← tout, en JSON
screens/
  desktop-full.png           ← full-page (natif ou stitching des folds si le site révèle au scroll)
  desktop/desktop-fold-01.png…  ← un vrai viewport à chaque position de scroll
  tablet-full.png, mobile-full.png, tablet/, mobile/
  sections/01-header.png…    ← un crop par section détectée
  hover/07-start-for-free-hover.png…
css/                         ← chaque stylesheet telle que servie (+ les <style> inline)
assets/images|fonts|icons|svg  ← tout téléchargé, nommé d'après la source (les proxys /_next/image sont résolus)
content/text.md              ← tout le texte dans l'ordre, titres en markdown, CTAs tagués
content/dom-outline.txt      ← l'arbre DOM avec tailles et hints flex/grid
content/page.html            ← le DOM rendu après hydratation
pages/<slug>/                ← avec --depth 1 : les autres pages du menu
```

Il gère les cas qui font échouer une capture naïve : scroll complet pour déclencher lazy-load et
IntersectionObserver, bannières cookies masquées en CSS (**sans** cliquer « accepter »), pages trop
hautes pour Chromium, `@font-face` relatifs à un CSS cross-origin, SVG inline (logos), sites qui
cassent après une capture full-page native.

Et il lit la console de l'original : erreurs, requêtes échouées, HTTP ≥ 400, hôtes tiers,
frameworks et libs détectés (Next, Nuxt, Webflow, Framer, Tailwind, GSAP, Lenis, Swiper…),
analytics.

### 2. Le skill — reconstruire dans le bon ordre

Le `SKILL.md` impose l'ordre qui marche : lire les folds un par un → fonts + tokens + body +
container + header + meta → sections de haut en bas, responsive écrit en même temps → vérification.
Avec une checklist de 40 détails qui trahissent un clone et des recettes prêtes (header sticky,
reveal-on-scroll, marquee, FAQ, boutons avec hover mesuré, `@font-face` depuis le manifest).

Stack de sortie : HTML/CSS/JS statique par défaut (le plus fidèle), Next.js si tu es dans un repo
Next.js, ce que tu veux si tu le précises.

### 3. `compare.mjs` — mesurer au lieu de croire

```bash
node skills/copycat/scripts/compare.mjs --original copycat/exemple.com --clone http://localhost:4173
```

Recharge le clone exactement comme l'original (mêmes viewports, même scroll, mêmes fonts prêtes) et
sort `compare/REPORT.md` :

- **Verdict A/B/C/D** et % de pixels différents par viewport (A = < 2 % et console propre)
- **Worst zones** : les bandes les plus fausses, mappées aux sections → tu sais quoi corriger d'abord
- **Design diff** : fonts manquantes, couleurs manquantes, typo h1/h2/h3/p/a/button, body, titres et
  CTAs absents, meta, compteurs d'animations, images cassées
- **Console du clone** : erreurs, exceptions, 404 — cible 0
- `desktop-side-by-side.png` (original | clone | diff) et les folds du clone

Tu itères jusqu'au A. En général trois passes.

---

## Installation

### Option 1, plugin (recommandé)

Dans Claude Code :

```bash
/plugin marketplace add minosdevs/copycat
```

```bash
/plugin install copycat@minosdevs-copycat
```

### Option 2, manuelle

```bash
git clone https://github.com/minosdevs/copycat.git
cp -r copycat/skills/copycat ~/.claude/skills/
cp copycat/commands/copycat.md ~/.claude/commands/
```

### Dans les deux cas : les dépendances des scripts (une fois)

```bash
cd ~/.claude/skills/copycat/scripts && npm install && npx playwright install chromium
```

(ou le dossier `scripts/` du plugin). Node ≥ 18. Ça installe Playwright, pixelmatch, pngjs et un
Chromium headless (~150 Mo).

---

## Utilisation

```bash
/copycat https://exemple.com
```

Claude capture, lit le rapport et les screens, construit, lance le clone, compare, corrige, et te
livre : où est le clone, le verdict par viewport, la console, et la liste honnête de ce qui n'est pas
reproduit.

Variantes :

```bash
/copycat https://exemple.com --depth 1
```
capture aussi les pages du menu.

```bash
/copycat https://exemple.com --stack next
```
force la sortie en composants Next.js.

Tu peux aussi lancer les scripts à la main (options : `--viewports desktop,mobile`, `--scale 2`,
`--dark`, `--locale fr-FR`, `--wait 3000`, `--videos`, `--headed`, `--no-hover`, `--no-assets`).

### Sans Playwright : la console DevTools

`skills/copycat/scripts/extract.browser.js` se colle tel quel dans la console de n'importe quel
navigateur, puis :

```js
copy(JSON.stringify(__copycatExtract(), null, 2))
```

Tu as le même JSON de design tokens que `manifest.json → design`. Pratique pour les sites derrière
un login ou une protection anti-bot.

---

## Ce que ça ne fait pas

- Ne reproduit pas le backend : formulaires, auth, paiement, contenu dynamique, widgets tiers
  (chat, analytics). Le skill le dit dans le livrable.
- Ne contourne pas les protections anti-bot ni les logins : il te propose de capturer à la main.
- Ne clique jamais « accepter » sur une bannière cookies : il la masque en CSS.
- Ne t'autorise pas à réutiliser le logo, les photos, les textes ou la marque d'un tiers. Cloner
  pour apprendre, prototyper ou refaire *ton* site, oui. Mettre en ligne le site de quelqu'un d'autre
  avec ses assets, non — le skill remplace les assets de marque par des placeholders si le clone a un
  autre usage que le tien ou l'étude.

---

## Structure

```
copycat/
├── .claude-plugin/          plugin.json, marketplace.json
├── commands/copycat.md      la commande /copycat
├── skills/copycat/
│   ├── SKILL.md             le workflow (capturer → lire → fondations → sections → vérifier → livrer)
│   ├── scripts/
│   │   ├── capture.mjs      capture Playwright complète
│   │   ├── compare.mjs      diff pixel + design diff + console
│   │   ├── extract.browser.js  l'extracteur in-page (aussi collable dans DevTools)
│   │   ├── lib.mjs          helpers partagés (scroll, folds, stitching, consent, PNG)
│   │   └── package.json
│   └── references/
│       ├── fidelity-checklist.md   les 40 détails qui font « faux »
│       ├── rebuild-recipes.md      recettes : fonts, tokens, header, reveal, marquee, FAQ, boutons…
│       └── troubleshooting.md      site bloqué, capture vide, fonts 404, scroll interne, diff élevé
└── README.md / README.en.md / LICENSE
```

---

## Testé sur

- Landing Next.js + Tailwind v4 avec reveal-on-scroll, header fixed, hero animé, carrousel auto,
  bannière cookies différée, fonts self-hosted et images proxifiées (`/_next/image`) : capture 3
  viewports en ~90 s, 11 sections, 74 assets, 0 warning ; auto-comparaison du site contre lui-même à
  1,6 % (animations infinies), grade A.

---

## Licence

MIT — [Minos](https://github.com/minosdevs).
