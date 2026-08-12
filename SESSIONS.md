# muralinko-simulator — Session Log

Newest at top. Each entry: what was discussed, what shipped, what is next.

---

## 2026-08-12 — CORRECTION (fin de session)
**Le modele decrit dans l'entree ci-dessous etait PERIME.** `machine_height_combinations.csv` date de mai 2025 ; le modele courant vit dans **`frais impression_v2.xlsx`** (OneDrive\Downloads, 6 onglets, refait en juin 2026), que je n'ai trouve qu'en cherchant une ligne « suspended » signalee par Chokri. Erreur de methode : j'ai pris le CSV pour source de verite sans verifier s'il existait plus recent, et j'ai lu « le dernier V2 » comme `Printable_Area_Calculator_Form_v2.xlsx` parce que le nom collait.

**Le vrai modele :** corps **125** (pas 145) ; intermediaires empilables Barre 1/2 **40** (pas 39,2), 3 = 100, 5 = 10, 6 = 20, 7 = 30, 8 = 40, 9 = 50 ; **4 terminales** dont une obligatoire — Barre 4 = 97,5, Barre 10 = 10, Barre 11 = 20, Barre 12 = 30 ; total plafonne a **400 cm** ; zone morte au sol **48** (pas 54) ; offset plafond 35. 829 assemblages valides → **45 hauteurs distinctes de 135 a 395 cm**. Selection = plus haute hauteur <= hauteur du mur, puis `-48 -35 si plafond`.

**Consequences :** le plancher machine est **1,35 m** et non 2,43 m — le simulateur refusait des murs parfaitement imprimables (mur 220 sous plafond = **132 cm**, pas « impossible » ; mur 200 = 112 cm). La « regle commerciale » notee plus bas est donc fausse, ainsi que tout le tableau d'ecarts de l'entree precedente. Les barres courtes « a garder pour plus tard » etaient deja dans le fichier.

**Refait :** `CONFIGS` genere depuis le catalogue exactement comme le classeur enumere (1 terminale + sous-ensemble des intermediaires, coupe a 400, dedoublonne par hauteur en gardant le moins de barres) — reproduit les 45 hauteurs ligne par ligne. **Ajout du mode panneau suspendu**, absent du simulateur : panneau detache du mur → aucune barre, aucune deduction, toute la surface imprimable ; les 3 cases plafond/murs se desactivent. Verifie sur les saisies memes du classeur (254 x 830, plafond + 2 murs) → **169,5 x 772 cm, 13,09 m², « Corps + Barre 7 + Barre 4 »**, identique a ses cellules ; suspendu 125 x 300 → 3,75 m², conforme a son journal.

**Livre :** `frais impression_v3.xlsx` (a cote du v2 dans OneDrive\Downloads) — copie du v2 avec les libelles **B6/B7 remis a l'endroit** (ils annoncaient Right/Left alors que B6 retire 10 = gauche et B7 retire 48/65 = droite ; formules inchangees) + une ligne 16 ajoutee a l'onglet Claude Log. Non versionne : ce classeur contient les frais d'impression. `Printable_Area_Calculator_v3.xlsx` **retire du depot** — bati sur le modele perime, donc dangereux a laisser trainer (recuperable dans l'historique git).

**Tranche par Chokri :** zone morte au sol = **48** (formule du classeur, deja en place) ; marge droite sur mur haut = **60** et non 65 (valeur du schema Technical). Applique des deux cotes — `SIDE_FAR_WIDE` dans le simulateur, formules B11 et B20 du classeur v3. Verifie sur la bascule a 300 cm : mur 299 x 500 → 9,58 m², mur 300 x 500 → 9,33 m², identique dans les deux outils. Le seuil de bascule reste la **hauteur du mur >= 300 cm** (regle de la formule) et non la hauteur d'assemblage — a corriger si l'intention etait le seuil machine.

**Incident a noter :** l'AutoSave OneDrive a persiste mes valeurs de test dans `frais impression_v3.xlsx` pendant une verification (250x500 au lieu de 254x830). Detecte en relisant l'etat disque au lieu de le supposer, saisies d'origine restaurees. Sur un fichier OneDrive, toute ecriture COM est persistee meme sans `Save()` — travailler sur une copie scratchpad.

---

## 2026-08-12 (entree initiale — modele perime, conservee pour la trace)
**Discussed:** Ecart entre le calculateur Excel « classique » et le simulateur sur la surface imprimable. Audit des 4 fichiers Excel trouves (`Printable_Area_Calculator*.xlsx`, dont un modele 48 cm et un fichier aux formules cassees) + du CSV machine. Redefinition du modele machine avec Chokri.
**Shipped:**
- **Diagnostic** : le simulateur soustrayait les marges **deux fois**. Les colonnes 3/4 de `machine_height_combinations.csv` ont deja le -54 (et le -35) deduits, or `CONFIGS` stockait ces valeurs et `computePrintable` re-soustrayait 35+54. Deuxieme erreur qui la masquait : le filtre comparait la hauteur d'impression a la hauteur du mur au lieu de la hauteur totale machine.
- **Decomposition du CSV** : les 15 combinaisons se recomposent exactement (ecart 0,00) a partir de 4 valeurs — corps **145,00**, barre 1 = barre 2 **+39,20**, barre 3 **+100,00**, barre 4 **+97,50**.
- **`computePrintable` reecrit** ([index.html:890](index.html:890)) sur ce modele : `CONFIGS` genere (barre 4 toujours montee + sous-ensemble libre de {1,2,3} → 6 hauteurs de 242,50 a 420,90), zones mortes nommees (`DEAD_BOTTOM` 54 depuis le sol, `DEAD_CEILING` 35 sous le sommet machine, `SIDE_NEAR` 10 gauche, `SIDE_FAR` 48 droite, `SIDE_FAR_WIDE` 60 au-dela de 4 m d'assemblage). Avec plafond : plus haut assemblage dont la hauteur **totale** rentre. Sans plafond : plus de contrainte -35, bande = `min(mur, machine) - 54`.
- **Overlay corrige** : la bande partait de 35 cm sous le plafond ; elle part maintenant de 54 cm du sol jusqu'au sommet utile de la machine (presque toujours plus bas que le plafond). Suppression du double appel a `computePrintable` et du `10` en dur.
- Alertes reecrites (plancher machine 2,43 m sous plafond, mur plus haut que la machine), arrondi de la marge haute, badge « max » decolle du nom de config.
- `D:\Code\CLAUDE.md` mis a jour (decrivait encore `top:35, bottom:54, left:10, right:48`).
- **Verifie en live** (localhost:3000, images de test du depot) : 18 combinaisons hauteur x plafond conformes au CSV, 6 combinaisons de marges laterales, etape 3 complete (config max → marge droite 60 « stabilisateur », marge haute 44,1 cm sur mur 4,30 m). Zero erreur console.
- **Impact chiffre** : mur 2,50 x 4,00 sous plafond → 6,14 m² au lieu de 5,72 m² (simulateur) et 6,44 m² (Excel).
**Regle commerciale confirmee :** **sans barre 4, pas d'impression.** Un mur sous plafond de moins de **2,43 m** (hauteur de corps + barre 4 = 242,50) n'est donc pas imprimable — cas reel des murs de 220 cm, qui restent imprimables uniquement s'ils ne sont pas bornes par un plafond (220 - 54 = 166 cm). Message de refus explicite dans les deux ecrans, plancher arrondi au cm **superieur** (2,43 et non 2,42, sinon un mur de 2,42 semblerait passer). **A garder pour plus tard :** Chokri a ajoute des barres plus petites capables de remplacer la barre 4 — elles abaisseront ce plancher, modele a revoir quand leurs hauteurs seront mesurees.
**Confirme par Chokri en fin de session :** toutes les combinaisons de barres sont montables (les 6 assemblages du code sont donc valides) ; seul `Printable_Area_Calculator_Form_v2.xlsx` fait foi cote Excel, les 3 autres fichiers sont a ignorer ; les murs de plus de 4 m avec plafond existent et sont couverts (config max au-dela de 4,21 m + alerte). Cas pratique de reference donne par Chokri : mur 250 / machine 242,50 → les 35 cm se retirent **de la hauteur machine**, pas du mur → 153,50 cm imprimables et 42,5 cm de marge haute.
**Livre en fin de session :** `Printable_Area_Calculator_v3.xlsx` (a la racine du depot, versionne) — meme saisie que le v2 mais avec la table des 6 hauteurs machine en lookup, gauche/droite remis dans le bon sens, refus sous 2,43 m avec plafond, et affichage de la config retenue + des 4 marges. Genere sans dependance (script scratchpad `build_xlsx.py`, XML brut). **Verifie via Excel COM sur 9 scenarios : resultats identiques au simulateur** (250x400 plafond → 6,14 m² ; 220 plafond → refus ; 220 sans plafond → 6,64 m² ; 430x500 deux murs → 14,27 m²). Les 2 commits sont sur `dev` (c1177da, 4f3b7ee), pousses.
**Next (ancien) :** Reconstruire `Printable_Area_Calculator_Form_v2.xlsx` sur le modele machine — sa formule continue `mur - 54 - 35` sur-promet de +7 a +38 cm selon la position du mur entre deux paliers, et jusqu'a +79 cm au-dela de 4,21 m ; elle n'est juste qu'a 4,209 m (machine max touchant le plafond). Deployer sur `dev` puis merger. Reste du backlog inchange : feedback client sur l'auto-detect v1, analytics, Tier 1 (texture, AR WebXR).

## 2026-05-18
**Discussed:** Roadmap stratégique pour rendre le simulateur "unique au monde" (5 principes + 3 tiers + métriques). Débat capture de leads sans friction (capturer au pic émotionnel, pas à l'entrée). Itérations features avec workflow dev→preview→validation→merge. Diagnostic honnête: heuristiques pures insuffisantes pour lighting transfer et auto-detect sur photos chaotiques.
**Shipped:**
- Doc technique PDF généré (`Muralinko-Simulator-Documentation-Technique.pdf`, gitignored).
- **Boutons visuels** refondus: tiles avec icônes SVG, hover lift, états actifs/loading/focus.
- **Feature #1 Multi-design** (mergé en prod): 3 emplacements visibles dès le début, switch live étape 3, taille+reco par design, boutons Taille min/max sur étape 3, support SVG (rasterisation auto), décodage async anti-freeze, fix double-dialog.
- **Feature #2 Lighting transfer**: 4 versions testées (soft-light/overlay/full-wall-projection/gradient artificiel) puis **RETIRÉE** — pas assez convaincante sur photos réelles. Recherche consignée dans `FUTURE_IDEAS.md`.
- **Feature #3 Auto-detect mur**: v1→v2→v3 testées, **revert à v1** (color-uniform region from center, échoue gracefully). Plan ML v5 (MediaPipe segmentation ~10MB) différé dans `FUTURE_IDEAS.md`.
- Setup branche `dev` + preview Cloudflare par commit. Merge `dev→main` finalisé (résolu 3 conflits: PDF locké, untracked files, .gitignore divergent). Prod à jour et vérifiée live.
- `FUTURE_IDEAS.md` créé (lighting transfer 4 approches + auto-detect ML v5 + pistes diverses).
**Next:** Recueillir feedback réel client (5+ users) sur calibrateur manuel + auto-detect v1 avant d'investir dans ML v5. Pistes Tier 1 restantes: détection texture, AR mobile WebXR. Tier 2: générateur design AI, galerie curatée, save&share magic-link, estimation prix inline. Instrumenter analytics (Plausible/Umami) — sans data on navigue à vue.

## 2026-05-11 (afternoon)
**Discussed:** Production deployment of muralinko-simulator + Wix CTA placement.
**Shipped:** Cloudflare Workers deploy at `simulateur.muralinko.workers.dev` (Git auto-deploy from `Bodyguard446/muralinko-simulator`, COOP/COEP headers for fast WASM bg removal); renamed Cloudflare workers subdomain `chokri-kefi` → `muralinko`; renamed worker `muralinko-simulator` → `simulateur`; empty default inputs + step gating in simulator; CTA button "Simulez votre projet en 30s →" added to muralinko.be hero (replaced "Découvrez comment ça marche"); 49/49 automated tests passing (printable calc, homography, recommendations, DOM pipeline, edge cases).
**Next:** Optional — custom subdomain `simulateur.muralinko.be` (requires DNS migration to Cloudflare); secondary CTAs on services + devis pages; click-tracking on simulator CTA.
