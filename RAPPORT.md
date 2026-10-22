# Liste des problèmes identifiés

| # | Fichier et ligne | Nature | Preuve | Coût estimé | Correction proposée |
|---|---|---|---|---|---|
| 1 | `img/img1.jpg` à `img6.jpg` (`index.html:75-90`) | JPEG de 2040 à 3968 px affichés à ~388 px | `audit/01-reseau-avant.png` | 0,75 à 3,39 Mo par image | Redimensionner, WebP, `srcset` |
| 2 | `index.html:12` | `preload` de `img6.jpg?v=2` : même image téléchargée deux fois | `audit/01-reseau-avant.png` | 3,39 Mo | Supprimer le `preload` |
| 3 | `css/theme-legacy.css:11-12` | `hero-bg.jpg` (3,39 Mo, identique à `img6.jpg`) affichée à 4 % d'opacité | `audit/01-reseau-avant.png` | 3,39 Mo | Texture WebP d'environ 20 Ko |
| 4 | `img/favicon.png` | PNG 512×512 pour une icône | `audit/01-reseau-avant.png` | 787 Ko | Version 64×64 |
| 5 | `index.html:16` | `js/legacy/polyfills.js` en 404, script bloquant | `audit/01-reseau-avant.png` | 1 aller-retour + parsing bloqué | Supprimer la ligne |
| 6 | `index.html:17-18` | Scripts sans `defer` dans le `<head>` | `audit/04-performances-frise-avant.png` | Parsing HTML interrompu par l'exécution des scripts | Ajouter `defer` |
| 7 | `css/styles.css:1`, `css/extra.css:1` | Chaîne d'`@import` sur 3 niveaux | `audit/01-reseau-avant.png` (cascade) | 3 allers-retours en série avant le 1er rendu | `<link>` parallèles dans le HTML |
| 8 | `css/styles.css:2`, `css/theme-legacy.css:3`, `index.html:10` | Polices : `display=block`, graisse 300 et Playfair Display inutilisées, `preload` sans `crossorigin` | `audit/01-reseau-avant.png` | 3 fichiers `woff2` + 2 feuilles Google, texte invisible pendant le chargement | `display=swap`, Poppins 400 et 600, `crossorigin` |
| 9 | `css/styles.css`, `css/extra.css` | ~2 000 règles `.redundant-N` / `.extra-N` jamais utilisées | `audit/01-reseau-avant.png` | 92 Ko + 29 Ko bruts | Réécrire avec les seules règles utilisées |
| 10 | `index.html:75-90` | `<img>` sans `width`/`height` ni `loading="lazy"` | Lighthouse : « éléments d'image sans width/height » | CLS 0,031 (Desktop), 0,125 (PSI Mobile) | `width`/`height`, `lazy` sous la ligne de flottaison |
| 11 | `js/vendor/ui-kit.min.js:2:26` et `:2:129` | Boucle active de 620 ms et tableau de 240 000 objets au parsing | `audit/05-performances-bottom-up-avant.png` | `c` 362 ms, `now` 247 ms, `b` 108 ms (CPU ×4) | Supprimer la bibliothèque, helpers en JS natif |
| 12 | `js/vendor/ui-kit.min.js:2:737` | `MutationObserver` global qui appelle `getComputedStyle` sur tous les éléments | `audit/05-performances-bottom-up-avant.png` | À mesurer avec un enregistrement incluant du scroll | Supprimer avec la bibliothèque |
| 13 | `js/app.js:81` | `innerHTML +=` dans une boucle de ~370 tours | `audit/05-performances-bottom-up-avant.png` | 446 ms propres, 1 078 ms au total | `DocumentFragment` |
| 14 | `js/app.js:158-190` | Boucle en n² avec `offsetHeight` et `getBoundingClientRect` (reflow forcé), appelée à chaque `scroll`/`resize` | `audit/05-performances-bottom-up-avant.png` | 130 ms propres, 835 ms au total | `IntersectionObserver`, `requestAnimationFrame`, `passive` |
| 15 | `js/analytics.js:23-29` | Code obscurci : démarre 6,5 s après le chargement, `setInterval` de 2,6 s, boucle active jusqu'à 430 ms, uniquement si la fenêtre fait plus de 1024 px | **À prouver** : enregistrement Performances de 30 s | Jusqu'à 430 ms de blocage toutes les 2,6 s | Supprimer (aucune fonction visible) |