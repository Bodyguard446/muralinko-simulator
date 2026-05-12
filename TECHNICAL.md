# Muralinko Simulator — Documentation technique

> Simulateur d'impression murale pour muralinko.be. Permet à un client d'uploader une photo de son mur + un design, de calibrer la perspective, et de visualiser le rendu imprimé avec calcul de zone imprimable, recommandations de taille et suppression de fond.

**URL de production :** `https://simulateur.muralinko.workers.dev`
**Repository :** `Bodyguard446/muralinko-simulator`
**Source local :** `D:\Code\muralinko-simulator\`

---

## 1. Stack & contraintes

- **Un seul fichier HTML** (`index.html`, ~1500 lignes) : HTML5, CSS3, vanilla JS ES2020.
- **Zéro dépendance build.** Aucun bundler, aucun framework, aucun package.json.
- **Une seule dépendance runtime**, chargée à la demande via CDN :
  `@imgly/background-removal@1.7.0` (~30 MB, ISNet ML model en WASM).
- **JS encapsulé dans une IIFE** pour isoler le scope global.
- **Langue UI :** Français.

Cette contrainte "single file" est délibérée : déploiement zero-config, édition rapide, pas de pipeline à maintenir.

---

## 2. Flow utilisateur (wizard 3 étapes)

### Étape 1 — Votre mur
- Upload d'une photo du mur (drag & drop ou clic).
- Saisie des dimensions réelles du mur : **largeur (m)** + **hauteur (m)**.
- 3 checkboxes : `Plafond` / `Mur perpendiculaire gauche` / `Mur perpendiculaire droite`.
- **Calibrateur de perspective** : 4 poignées draggables (haut-gauche, haut-droite, bas-droite, bas-gauche) avec **loupe 3× sur drag**, à positionner sur les coins réels du mur dans la photo.
- **Card "Surface imprimable max"** mise à jour en temps réel : montre la config machine sélectionnée, dimensions max, et warnings éventuels.
- Bouton **Continuer** gated : disabled tant que (photo absent OU largeur vide OU hauteur vide).

### Étape 2 — Votre design
- Upload du design (jpg/png — fond transparent idéal mais pas requis).
- **Card "Limites d'impression"** : affiche `Largeur max`, `Hauteur max`, et 2 recommandations :
  - **Recommandé min (sans déformation)** : taille qui respecte le ratio naturel du design dans la zone imprimable.
  - **Recommandé max (étirement toléré)** : recommandé min × 1.5 sur la dimension non contrainte, capped à la max printable.
- Inputs **Largeur (m)** + **Hauteur (m)** auto-remplis avec la reco min, clampés par les max printable.
- 2 boutons `Taille min` / `Taille max` pour appliquer une reco en un clic.
- Bouton **Voir l'aperçu** gated sur design uploadé + dimensions > 0.

### Étape 3 — Aperçu
- Rendu DOM avec perspective réelle :
  - `<img>` du mur en background.
  - `<img>` du design avec `mix-blend-mode: multiply` (simule l'encre sur surface) et `transform: matrix3d(...)` calculé par homographie 4 points.
- **Drag-to-position** : le design se déplace dans le plan du mur (coordonnées (u, v) normalisées 0..1) via inverse homography.
- **Règle d'échelle perspective-correct** : ligne horizontale blanche avec ticks 10 cm et labels "1 m / 2 m / 3 m" projetée sur le bas du mur — donne au client un anchor visuel pour vérifier que la taille est crédible.
- **Polygone zone imprimable** (dashed mint) projeté en perspective.
- **Sidebar** :
  - Card "Votre impression" : dimensions, surface m².
  - Card "Surface imprimable" : config machine, breakdown des marges (haut/bas/gauche/droite cm), zone, surface max.
  - Toggles **Comparer (mur seul)** et **Retirer le fond**.
  - Bouton **Télécharger l'aperçu** (PNG résolution naturelle du mur, avec watermark muralinko.be).

---

## 3. Algorithmes critiques

### 3.1. Calcul de zone imprimable

Données machine (hardcoded) — 8 configurations de barres, chacune avec une hauteur max différente selon présence de plafond :

```js
const CONFIGS = [
  { name: "Corps + Barre 4",                      hC: 153.5, hN: 188.5 },
  { name: "Corps + Barre 1 + Barre 4",            hC: 192.7, hN: 227.7 },
  { name: "Corps + Barre 2 + Barre 4",            hC: 192.7, hN: 227.7 },
  { name: "Corps + Barre 3 + Barre 4",            hC: 253.5, hN: 288.5 },
  { name: "Corps + Barres 1+2 + Barre 4",         hC: 231.9, hN: 266.9 },
  { name: "Corps + Barres 1+3 + Barre 4",         hC: 292.7, hN: 327.7 },
  { name: "Corps + Barres 2+3 + Barre 4",         hC: 292.7, hN: 327.7 },
  { name: "Corps + Barres 1+2+3 + Barre 4 (max)", hC: 331.9, hN: 366.9 }
];
```

**Algorithme :**

1. Filtrer les configs dont la hauteur ≤ hauteur mur (avec ou sans plafond selon checkbox).
2. Sélectionner la plus haute valide. Si aucune ne fit, prendre la plus petite et flagger `tooShort`.
3. Détecter `isMax` (config la plus haute) — déclenche le stabilisateur.
4. Marges (cm) :
   - Haut : 35 si plafond, sinon 0
   - Bas : 54 toujours
   - Gauche : 10 si mur perpendiculaire, sinon 0
   - Droite : 0 si pas de mur, 48 si mur (config standard), 60 si mur + config max (stabilisateur)
5. Zone imprimable : `(configH - top - bottom) × (wallW - left - right)`, clampé à 0.

Voir `computePrintable()` dans le code.

### 3.2. Homographie 4-points (perspective)

Cœur du rendu perspective. Étant donnés 4 points source et 4 points destination, calcule la matrice 3×3 H telle que H × src = dst (en coords homogènes).

Résolu par système linéaire 8×8 (Gauss avec pivot partiel) :

```js
function homography(src, dst) {
  // Pour chaque point i : 2 équations linéaires en les 8 inconnues (a..h, i=1)
  // [sx, sy, 1, 0, 0, 0, -dx*sx, -dx*sy] · [a b c d e f g h]ᵀ = dx
  // [0, 0, 0, sx, sy, 1, -dy*sx, -dy*sy] · [a b c d e f g h]ᵀ = dy
  // ...résoudre 8×8.
}
```

**Conversion en CSS `matrix3d`** : la matrice 3×3 H est embedée dans une 4×4 (z=0 plane) puis sérialisée dans l'ordre column-major attendu par CSS :

```js
function homographyToMatrix3d(h) {
  return `matrix3d(${h[0]},${h[3]},0,${h[6]}, ${h[1]},${h[4]},0,${h[7]}, 0,0,1,0, ${h[2]},${h[5]},0,${h[8]})`;
}
```

**Pipeline de rendu :**

1. Calculer H_wall : unit square `[(0,0),(1,0),(1,1),(0,1)]` → quad mur en pixels du stage.
2. Pour la position normalisée (u, v) du design, calculer ses 4 coins en (u, v) du plan mur.
3. Projeter ces 4 coins via H_wall → 4 points cibles en pixels.
4. Calculer H_design : rect natif du design → 4 points cibles.
5. Appliquer via `transform: matrix3d(...)` sur l'`<img>` du design.

### 3.3. Drag du design en coordonnées plan-mur

Sur mousedown/mousemove, convertir la position curseur (pixels) en (u, v) normalisé via **inverse homography** (`homography(quad, unit_square)`). Permet un drag qui respecte la perspective : 1 cm de déplacement sur le mur réel = 1 cm de drag, peu importe l'angle de la photo.

### 3.4. Recommandations de taille

```js
function reco(natW, natH, maxW, maxH) {
  const ratio = natW / natH;
  let recH = maxH, recW = recH * ratio, constrained = 'h';
  if (recW > maxW) { recW = maxW; recH = recW / ratio; constrained = 'w'; }
  const STRETCH_MAX = 1.5;
  let mrW = recW, mrH = recH;
  if (constrained === 'h') mrW = Math.min(recW * STRETCH_MAX, maxW);
  else mrH = Math.min(recH * STRETCH_MAX, maxH);
  return { recW, recH, mrW, mrH };
}
```

**Min** = ratio naturel respecté, occupe la plus grande dim possible.
**Max** = stretch 1.5× sur la dim non contrainte (limite empirique au-delà de laquelle la déformation devient visible).

### 3.5. Suppression de fond ML

Lazy-load à la demande du module `@imgly/background-removal` (~30 MB, ISNet model). Exécution 100% client en WASM. Sans threads (single-thread), ~30 s sur premier run. Avec COOP/COEP headers (multi-thread), ~8 s.

**Auto-scale du sujet** : après suppression, on détecte la bbox du contenu non transparent. Si le sujet n'occupe que 70% de la frame originale, on multiplie la taille d'affichage du design par `1/0.7` pour que le sujet conserve sa taille physique perçue sur le mur.

```js
function detectContentBbox(img) {
  // scan ImageData pour trouver min/max x,y où alpha > 8
  return { w, h, fullW, fullH };
}
// bgScaleX = fullW / w; bgScaleY = fullH / h;
// fW = Math.min(1, (designWcm * bgScaleX) / wallWcm);
```

### 3.6. Export canvas avec perspective (download)

Le rendu écran utilise `matrix3d` (GPU). Pour le PNG téléchargé, on ne peut pas extraire la composition CSS — on refait le rendu sur un canvas en utilisant la **décomposition en 2 triangles affines** :

```js
function drawTexturedTriangle(ctx, img, srcTri, dstTri) {
  // Calcule la transformation affine 2D qui mappe les 3 points source aux 3 destinations
  // clip + setTransform + drawImage
}
```

Chaque triangle d'un quad est affine-exact (3 points définissent une affine unique). Les deux triangles partagent la diagonale donc la jointure est invisible.

L'export se fait à la résolution naturelle de la photo du mur, avec watermark "muralinko.be" en bas à droite.

---

## 4. État global (variables clés)

```js
// IIFE scope
let wallImg = null;          // HTMLImageElement de la photo mur
let designImg = null;        // HTMLImageElement du design original
let designImgNoBg = null;    // version sans fond (lazy)

let wallHeightM, wallWidthM; // dimensions réelles mur (m)
let hasCeiling, hasLeftWall, hasRightWall;
let designWcm, designHcm;    // dimensions souhaitées du design (cm)

let wallCorners;             // [{x,y}×4] fractions 0..1 dans l'image
let designU = 0.5, designV = 0.5;  // position du design dans plan-mur

let bgOn = false, bgScaleX = 1, bgScaleY = 1;
let compareOn = false;       // toggle "mur seul"
```

---

## 5. Structure fichiers

```
D:\Code\muralinko-simulator\
├── index.html         # tout le code
├── wrangler.jsonc     # config Cloudflare Workers
├── _headers           # COOP/COEP pour WASM rapide
├── .gitignore
└── TECHNICAL.md       # ce fichier
```

`wrangler.jsonc` :

```jsonc
{
  "name": "simulateur",
  "compatibility_date": "2025-01-01",
  "assets": {
    "directory": "./",
    "not_found_handling": "single-page-application"
  }
}
```

`_headers` :

```
/*
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  Cache-Control: public, max-age=300, must-revalidate
```

---

## 6. Déploiement

**Auto-deploy sur push** vers `Bodyguard446/muralinko-simulator` (branche `main`) → Cloudflare Workers lance `npx wrangler deploy` → publié sous ~30 s sur `simulateur.muralinko.workers.dev`.

Workflow itération :

```bash
cd D:\Code\muralinko-simulator
# édit index.html
git add . && git commit -m "..." && git push
# attendre ~30 s, hard-reload Cloudflare URL
```

Pour développer en local :

```bash
npx serve D:/Code/muralinko-simulator -l 3456
# ouvre http://localhost:3456
```

---

## 7. Tests automatisés

49 tests couvrants :

- **Calculateur de zone imprimable** (9) : 4 cas de spec officiels + edge cases (mur trop bas, config max, stabilisateur).
- **Recommandations de taille** (4) : carré / portrait / paysage extrême / très haut.
- **Homographie** (3) : identité, round-trip H · H⁻¹, mapping exact des coins.
- **Pipeline DOM** (14) : upload, calibration, rendu perspective, drag, toggle comparaison, navigation entre étapes.
- **Cohérence UI** (5) : unités en mètres partout, ordre des inputs largeur/hauteur cohérent, clamping.
- **Réactivité live preview** (5) : input + checkboxes + badge max + warnings.
- **Calibrateur drag** (2) : drag corner + magnifier hide.
- **Reset après re-upload** (1).
- **Navigation back/forward** (1).
- **Resize window** (1).
- **Clamping input design** (1).

Lancés via `mcp__Claude_Preview__preview_eval` en injectant des images canvas générées + simulant des événements MouseEvent. Voir transcripts session 2026-05-10.

---

## 8. Limitations connues

- **Suppression de fond approximative** sur fonds complexes (ML model ISNet est généraliste, pas spécifique aux murs). Le toggle permet au client de comparer et choisir.
- **Pas de correction d'éclairage** : si le mur est sombre/textué, le design en multiply peut paraître muddy. Une future amélioration serait de sampler une luminance map du mur et la ré-appliquer sur le design.
- **Magnifier sur mobile** : la loupe se positionne à 90 px du doigt — peut être hors écran sur très petits viewports. À tester sur iPhone SE.
- **Pas de redimensionnement par poignées** dans l'étape 3 (volontairement supprimé car incompatible avec la perspective). La taille se change uniquement via les inputs de l'étape 2.
- **`workers.dev` dans l'URL** : pas de custom subdomain `simulateur.muralinko.be` car les DNS de muralinko.be sont gérés chez Wix (Cloudflare Workers Custom Domains nécessite que la zone DNS soit chez Cloudflare).

---

## 9. Spécifications source

Le calculateur de zone imprimable suit un spec produit fourni par le client (4 cas de test validés) :

| Test | Mur (h×l cm) | Plafond | Gauche | Droite | Config attendue | Surface |
|------|--------------|---------|--------|--------|-----------------|---------|
| 1 | 240×330 | ✓ | ✓ | ✓ | Barres 1+2+4 | 3.89 m² |
| 2 | 300×500 | — | — | — | Barre 3+4 | 11.73 m² |
| 3 | 400×300 (dépasse max) | ✓ | — | ✓ | MAX + stabilisateur | 5.83 m² |
| 4 | 400×250 (dépasse max) | — | ✓ | ✓ | MAX + stabilisateur | 5.63 m² |

Tous validés.

---

## 10. Intégration Wix

Bouton CTA "Simulez votre projet en 30s →" sur :

- **Page d'accueil** (hero) : aux côtés de "Demandez un devis".
- **Page Explications** (hero) : aux côtés de "Prendre rendez-vous".

Lien : `https://simulateur.muralinko.workers.dev` (ouvre dans un nouvel onglet).

Le simulateur n'est **pas embedé en iframe** dans Wix — c'est une page autonome, raison : (1) la lib bg-removal a besoin de COOP/COEP qui sont bloqués en iframe Wix, (2) l'upload de photo / fullscreen UX est plus simple en standalone, (3) le simulateur peut évoluer indépendamment du site Wix.
