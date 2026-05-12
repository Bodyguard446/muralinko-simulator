# Future Ideas — Muralinko Simulator

Pistes notées au fur et à mesure. Pas encore implémentées ou retirées en attendant une meilleure approche.

---

## Lighting transfer (retiré 2026-05-12)

**Objectif** : le design hérite des variations de lumière du mur photographié, pour qu'il ait l'air "imprimé" plutôt que "collé".

**Approches déjà testées et retirées** :

### v1 — Wall-clipped blurred overlay (`soft-light` blend)
- Une copie du mur, fortement floutée + grisée, clippée au quad du design via `clip-path`, mix-blend-mode `soft-light`.
- **Problème** : effet trop subtil sur la plupart des photos.

### v2 — Stronger blend (`overlay`)
- Même approche en `overlay` au lieu de `soft-light`, opacity 0.85.
- **Problème** : effet plus marqué mais "localisé au centre" du design — l'overlay agit principalement sur les midtones, et avec un design type Ferrari (ciel rouge + voitures noires) les extrêmes ne bougent quasiment pas.

### v3 — Full-wall projection
- Au lieu de clipper, projeter le mur entier en perspective dans le quad du design via matrix3d (même technique que pour le design lui-même).
- **Idée** : faire en sorte que le design hérite du gradient global de la pièce, pas juste de la portion locale derrière le design.
- **Problème** : sur des photos chaotiques (warehouse avec câbles, ombres irrégulières), il n'y a pas de gradient lumineux clair à transférer. Donc l'effet reste imperceptible.

### v4 — Artificial directional gradient (`multiply`)
- Abandonner le transfer depuis le mur. Générer un gradient artificiel `linear-gradient(white→gray)` avec 4 boutons directionnels (haut/bas/gauche/droite) + slider d'intensité.
- **Problème** : effet visible et prévisible, mais pas réaliste. L'utilisateur jugé que ça ne valait pas le coup d'avoir une feature qui n'est pas physiquement liée à la photo du mur.

---

## Pistes à explorer pour une vraie solution

1. **Détection automatique de la direction de lumière du mur**
   - Analyser le mur photo : sampler les luminances par zone (haut/bas/gauche/droite)
   - Calculer un vecteur de "direction de lumière" dominant
   - Appliquer un gradient artificiel orienté dans cette direction
   - **Avantage** : effet automatique + cohérent avec la photo. Pas de toggle direction à choisir.

2. **Match Color algorithm (LAB color space)**
   - Convertir design + zone mur visible en LAB
   - Faire matcher l'histogramme du L (luminance) du design avec celui du mur derrière
   - Reconvertir en RGB
   - **Avantage** : approche photographique professionnelle. La luminance du design "épouse" celle du mur de manière statistiquement correcte.
   - **Inconvénient** : implementation complexe en pur canvas/JS.

3. **Shadow/ambient occlusion fait main**
   - Détecter les bords du design (où il rencontre le mur)
   - Ajouter une légère ombre portée au bord supérieur (simule un débord d'épaisseur d'encre/papier)
   - **Avantage** : effet "tangible" du design comme objet physique sur le mur, plus que comme une projection.
   - Plus simple à implémenter, effet de réalisme distinct.

4. **Inférence ML via une petite vision lib**
   - Modèle léger qui estime une "lighting map" d'une photo (ex: Mono-Depth + analyse de gradient)
   - Appliquer la lighting map sur le design via multiply
   - **Avantage** : vraiment basé sur la photo, marche sur tous types d'images
   - **Inconvénient** : encore un modèle à charger (~5-30 MB), latence

5. **Approche "carte d'ombre" simplifiée**
   - Demander à l'utilisateur de marquer 1-2 sources de lumière sur la photo (clic sur "fenêtre", "lampe")
   - Générer un gradient radial ou directionnel à partir de ces points
   - Appliquer sur le design
   - **Avantage** : intuitif, contrôle manuel mais guidé par la photo réelle
   - **Inconvénient** : friction UX supplémentaire

---

## Autres idées (pas testées)

- **Tint matching** : adoucir le design avec un wash de la couleur dominante du mur (warm/cool unification). Effet subtle mais ancre le design dans l'ambiance de la pièce.
- **Edge integration** : flouter légèrement les bords du design pour qu'ils se fondent moins agressivement dans le mur.
- **Texture map** : appliquer une texture noisy du mur sur le design (mur en brique = design "imprimé sur brique"). Demande de la perspective + matching de texture.
