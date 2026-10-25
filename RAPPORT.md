# Rapport d'optimisation — R3.14, site « SlowSite »

- **Site publié (ma version corrigée)** : https://mayy-19.github.io/R314-optimisation-site/
- **Mon fork (historique des commits)** : https://github.com/mayy-19/R314-optimisation-site
- **Site de référence (avant)** : https://nico3807.github.io/R314-optimisation-site/

## 1. Conditions de mesure

J'ai mesuré le site en ligne (pas en local, parce qu'un serveur local ne compresse pas les fichiers texte), avec Chrome en navigation privée, le cache désactivé et un rechargement avec Ctrl+Maj+R. J'ai lancé Lighthouse depuis les DevTools (catégorie Performances) en profil Mobile puis en profil Desktop, et PageSpeed Insights sur l'URL publique. Pour le panneau Performances, j'ai mis le processeur en ralentissement ×4.

Limites : je n'ai que des données de laboratoire, parce que la partie « utilisateurs réels » de PageSpeed est vide (pas assez de trafic). Le TBT varie beaucoup sur ce site (par exemple 600 ms sur Lighthouse Mobile et 90 ms sur PageSpeed Mobile avant mes corrections), donc je ne tire pas de conclusion d'un seul essai. Une extension du navigateur était active pendant une première mesure Coverage : je l'ai désactivée et refaite, mais elle a peut-être aussi joué sur certains relevés Lighthouse « avant ». Enfin, je n'ai pas mesuré chaque optimisation séparément : les gains ci-dessous sont mesurés sur les fichiers concernés ou dans le profil d'exécution, et le gain global est mesuré avant/après.

## 2. Liste des problèmes identifiés

| # | Fichier et ligne | Nature | Preuve | Coût estimé | Correction proposée |
|---|---|---|---|---|---|
| 1 | `img/img1.jpg` à `img6.jpg` (`index.html:75-90`) | JPEG de 2040 à 3968 px affichés à ~388 px | `audit/01-reseau-avant.png` | 0,75 à 3,39 Mo par image | Redimensionner, WebP, `srcset` |
| 2 | `index.html:12` | `preload` de `img6.jpg?v=2` : même image téléchargée deux fois | `audit/01-reseau-avant.png` | 3,39 Mo | Supprimer le `preload` |
| 3 | `css/theme-legacy.css:11-12` | `hero-bg.jpg` (3,39 Mo, identique à `img6.jpg`) affichée à 4 % d'opacité | `audit/01-reseau-avant.png` | 3,39 Mo | Texture WebP d'environ 20 Ko |
| 4 | `img/favicon.png` | PNG 512×512 pour une icône | `audit/01-reseau-avant.png` | 787 Ko | Version 64×64 |
| 5 | `index.html:16` | `js/legacy/polyfills.js` en 404, script bloquant | `audit/01-reseau-avant.png` | 1 aller-retour + parsing bloqué | Supprimer la ligne |
| 6 | `index.html:17-18` | Scripts sans `defer` dans le `<head>` | `audit/04-performances-frise-avant.png` | Parsing HTML interrompu par l'exécution des scripts | Ajouter `defer` |
| 7 | `css/styles.css:1`, `css/extra.css:1` | Chaîne d'`@import` sur 3 niveaux | `audit/01-reseau-avant.png` (cascade) | 3 allers-retours en série avant le 1er rendu | `<link>` parallèles dans le HTML |
| 8 | `css/styles.css:2`, `css/theme-legacy.css:3`, `index.html:10` | Polices : `display=block`, graisse 300 et Playfair Display inutilisées, `preload` sans `crossorigin` | `audit/01-reseau-avant.png`, `audit/08-coverage-avant.png` | 3 fichiers `woff2` + 2 feuilles Google, texte invisible pendant le chargement | `display=swap`, Poppins 400 et 600, `crossorigin` |
| 9 | `css/styles.css`, `css/extra.css` | ~2 000 règles `.redundant-N` / `.extra-N` jamais utilisées | `audit/08-coverage-avant.png` : 95,8 % et 99,5 % d'octets inutilisés | 92 Ko + 29 Ko bruts (8,7 Ko + 2,4 Ko transférés) | Réécrire avec les seules règles utilisées |
| 10 | `index.html:75-90` | `<img>` sans `width`/`height` ni `loading="lazy"` | Lighthouse : « éléments d'image sans width/height » | CLS 0,031 (Desktop), 0,125 (PageSpeed Mobile) | `width`/`height`, `lazy` sous la ligne de flottaison |
| 11 | `js/vendor/ui-kit.min.js:2:26` et `:2:129` | Boucle active de 620 ms et tableau de 240 000 objets au parsing | `audit/05-performances-bottom-up-avant.png` | `c` 362 ms, `now` 247 ms, `b` 108 ms (CPU ×4) | Supprimer la bibliothèque, helpers en JS natif |
| 12 | `js/vendor/ui-kit.min.js:2:737` | `MutationObserver` global qui appelle `getComputedStyle` sur tous les éléments | `audit/05-performances-bottom-up-avant.png` | À mesurer avec un enregistrement incluant du scroll | Supprimer avec la bibliothèque |
| 13 | `js/app.js:81` | `innerHTML +=` dans une boucle de ~370 tours | `audit/05-performances-bottom-up-avant.png` | 446 ms propres, 1 078 ms au total | `DocumentFragment` |
| 14 | `js/app.js:158-190` | Boucle en n² avec `offsetHeight` et `getBoundingClientRect` (reflow forcé), appelée à chaque `scroll`/`resize` | `audit/05-performances-bottom-up-avant.png` | 130 ms propres, 835 ms au total | `IntersectionObserver`, `requestAnimationFrame`, `passive` |
| 15 | `js/analytics.js:23-29` | Code obscurci (chaînes en base64 : `innerWidth`, `performance`, `now`, `hidden`) : démarre 6,5 s après le chargement, `setInterval` de 2,6 s, boucle active de 46 ms qui grossit de ×1,28 jusqu'à 430 ms, uniquement si la fenêtre fait plus de 1024 px et l'onglet est visible | Lecture du code. **Non prouvé par une mesure** : mon enregistrement de 31 s n'a montré aucun long task, vraisemblablement parce que les conditions de déclenchement n'étaient pas réunies | Jusqu'à 430 ms de blocage toutes les 2,6 s (estimation d'après le code) | Supprimer (aucune fonction visible, aucune donnée envoyée) |
| 16 | `img1.html` à `img6.html` | Image à sa taille native (3968 px) sans contrainte CSS : débordement horizontal ; `ui-kit` et `app.js` chargés sans rien à animer | `audit/09-debordement-avant.png` | Page de 3968 px de large sur mobile | `max-width:100%`, WebP, retrait des deux scripts |
| 17 | `index.html` (`<head>`) | Après suppression de la graisse 300, le `preload` pointait encore vers un fichier de police non utilisé | Avertissement de la console (« preloaded … but not used ») | 1 requête inutile qui prend de la bande passante | Retirer le `preload` |

## 3. Tableau avant / après

| Indicateur | Avant | Après |
|---|---|---|
| Poids transféré | 17,5 Mo | environ 0,42 Mo (418 Ko, à confirmer avec la barre du bas du Réseau) |
| Requêtes | 22 | 14 |
| FCP (Lighthouse Mobile) | 3,8 s | 1,5 s |
| LCP (Lighthouse Mobile) | 59,2 s | 2,8 s |
| CLS (Lighthouse Mobile) | 0 | 0,022 |
| TBT (Lighthouse Mobile) | 600 ms | 0 ms |
| Score Lighthouse Mobile | 43 | 96 |
| FCP (Lighthouse Desktop) | 1,1 s | 0,5 s |
| LCP (Lighthouse Desktop) | 11,2 s | 0,8 s |
| CLS (Lighthouse Desktop) | 0,031 | 0,059 |
| TBT (Lighthouse Desktop) | 110 ms | 0 ms |
| Score Lighthouse Desktop | *à compléter* | 99 |
| Score PageSpeed Insights | *à compléter* | Mobile 91, Ordinateur 100 |

Ce sont les valeurs d'un seul lancement par profil : **à compléter avec les essais 2 et 3 et la médiane**. Le CLS Desktop a un peu augmenté (0,031 → 0,059). Il reste dans la zone « bon » (moins de 0,1), mais je n'ai pas réussi à trouver pourquoi, donc je ne peux pas dire que mes corrections l'ont amélioré.

## 4. Les optimisations principales

### 4.1 Les images

**Problème.** Les images faisaient presque tout le poids de la page. Les six JPEG pesaient entre 0,75 et 3,39 Mo, avec 2040 ou 3968 px de large alors qu'elles sont affichées à environ 388 px. En plus, `hero-bg.jpg` (3,39 Mo) est identique à `img6.jpg` et sert de texture à 4 % d'opacité, `img6.jpg?v=2` était téléchargée une deuxième fois à cause d'un `preload`, et le favicon pesait 787 Ko. C'est ce qui explique le LCP de 59,2 s en Mobile.

**Correction.** J'ai redimensionné chaque photo en 400, 800 et 1600 px et je l'ai convertie en WebP. La galerie utilise `srcset` et `sizes`, et les pages de détail utilisent les versions 800 et 1600 px. La texture du bandeau fait maintenant 21,9 Ko et le favicon (64 px) 3,8 Ko. J'ai supprimé le `preload` de `img6.jpg?v=2`.

**Gain mesuré.** Le poids total passe de 17,5 Mo à environ 0,42 Mo, et les requêtes de 22 à 14. Par exemple `img1` passe de 2,52 Mo à 14,5 Ko et `hero-bg` de 3,39 Mo à 21,9 Ko. Le LCP Mobile passe de 59,2 s à 2,8 s et le LCP Desktop de 11,2 s à 0,8 s.

**Preuve.** `audit/01-reseau-avant.png` et `01-reseau-apres.png`, les captures Lighthouse avant et après.

### 4.2 La chaîne critique, les polices et le 404

**Problème.** `styles.css` importait `extra.css`, qui importait `theme-legacy.css`, qui importait une feuille Google Fonts (Playfair Display, que le site n'utilise pas). Ça fait trois allers-retours en série avant l'affichage, alors que le CSS bloque le rendu. Poppins était en `display=block` (texte invisible au début), avec une graisse 300 inutile et un `preload` sans `crossorigin`. De plus, `polyfills.js` donnait un 404 et les scripts n'avaient pas `defer`.

**Correction.** J'ai supprimé les `@import` et mis des `<link>` côte à côte dans le HTML. Poppins est en `display=swap` avec seulement les graisses 400 et 600, et j'ai ajouté les `preconnect`. J'ai enlevé `polyfills.js` et le `dns-prefetch` inutile, et mis `defer` sur les scripts. Après avoir retiré la graisse 300, la console m'a signalé que le `preload` de police pointait vers un fichier devenu inutile : je l'ai supprimé aussi.

**Gain mesuré.** Dans la cascade, les fichiers CSS ne s'enchaînent plus. Les requêtes `extra.css`, `theme-legacy.css`, Playfair et le 404 ont disparu. Le FCP Mobile passe de 3,8 s à 1,5 s et le FCP Desktop de 1,1 s à 0,5 s (gain global avec les autres corrections).

**Preuve.** `audit/01b-cascade-avant.png` et `01b-cascade-apres.png`, la console.

### 4.3 Le CSS inutilisé

**Problème.** L'onglet Coverage montre que 95,8 % de `styles.css` et 99,5 % d'`extra.css` ne servent pas : environ 2 000 règles `.redundant-N` et `.extra-N` ne correspondent à rien dans le HTML. Mais Coverage compte le CSS non compressé : en ligne, ces fichiers ne pèsent que 8,7 Ko et 2,4 Ko, donc ce n'était pas le gros problème.

**Correction.** J'ai réécrit `styles.css` avec seulement les règles utilisées (environ 4,7 Ko au lieu de 92 Ko) et supprimé `extra.css` et `theme-legacy.css`. J'ai gardé ce qui se voit : la texture et l'ombre du bandeau, le filtre et le calendrier.

**Gain mesuré.** `styles.css` transféré passe de 8,7 Ko à 2,1 Ko. Le gain direct est petit, comme prévu par la méthodologie. Le vrai gain, c'est la fin de la chaîne d'`@import`.

**Preuve.** `audit/08-coverage-avant.png` et `08-coverage-apres.png`.

### 4.4 Le JavaScript : `ui-kit.min.js`

**Problème.** Cette bibliothèque lançait dès le chargement une boucle de 620 ms sur `performance.now()` (fonction `c`) et fabriquait un tableau de 240 000 objets (fonction `b`) qui ne sert à rien. Elle ajoutait aussi un `MutationObserver` sur toute la page qui appelait `getComputedStyle` sur tous les éléments à chaque changement. Ses fonctions utiles (`qs`, `qsa`, `cls`, `ready`) ne font que reprendre `querySelector` et `classList`.

**Correction.** Je l'ai supprimée et `app.js` utilise directement les fonctions du navigateur.

**Gain mesuré.** Dans le Bottom-Up « avant » (processeur ×4), `c` prenait 362 ms (609 ms au total), `now` 247 ms et `b` 108 ms, soit environ 700 ms avant l'affichage. Le TBT Mobile passe de 600 ms à 0 ms. *(À confirmer avec `05-performances-bottom-up-apres.png` : ces fonctions ne doivent plus apparaître.)*

**Preuve.** `audit/05-performances-bottom-up-avant.png`.

### 4.5 Le JavaScript : `app.js`

**Problème.** `buildKeywordIndex` faisait `innerHTML +=` dans une boucle de 370 tours, ce qui reconstruit la liste à chaque tour (446 ms propres, 1 078 ms au total). `revealCards` s'exécutait à chaque scroll et contenait une boucle dans une boucle qui relisait `offsetHeight` et `getBoundingClientRect` juste après avoir écrit des styles : c'est un reflow forcé, en plus sur 371 cases de calendrier (130 ms propres, 835 ms au total). Avec le `MutationObserver` de `ui-kit`, les deux se renforçaient pendant le scroll.

**Correction.** La liste est construite dans un `DocumentFragment` puis insérée une seule fois. L'apparition des cartes passe par un `IntersectionObserver`. Pour le calendrier, je ne mesure que les 7 lignes de la grille et je mets 7 variables CSS à jour (au lieu de 371 cases), avec `requestAnimationFrame` et un écouteur `passive`.

**Gain mesuré.** Le TBT Lighthouse passe de 600 à 0 ms en Mobile et de 110 à 0 ms en Desktop. *(À confirmer avec le Bottom-Up « après ».)*

**Preuve.** `audit/05-performances-bottom-up-avant.png` (`buildKeywordIndex` en `app.js:74`, `revealCards` en `app.js:158`).

**Deux différences de comportement.** Une carte déjà apparue ne redescend plus quand on remonte la page. Et l'ancien code agrandissait la hauteur des cartes à chaque événement : c'était un défaut, et la grille CSS s'en occupe déjà.

### 4.6 Le script `analytics.js` (niveau 3)

**Problème.** Ce fichier est écrit pour être illisible (chaînes en base64). Une fois décodées dans la console, elles donnent `innerWidth`, `performance`, `now` et `hidden`. Il démarre 6,5 s après le chargement, puis lance un `setInterval` toutes les 2,6 s. Si l'onglet est visible et la fenêtre fait plus de 1024 px, il parcourt tout le DOM et occupe le processeur de plus en plus longtemps (de 46 ms jusqu'à 430 ms). Il n'envoie aucune donnée. Ça explique pourquoi mes mesures se contredisaient : il est invisible sur mobile et dans un audit court.

**Correction.** Je l'ai supprimé avec sa balise `<script>`. Il n'avait aucun effet visible.

**Gain mesuré.** Je n'ai pas réussi à le mesurer : mon enregistrement de 31 s sur la version d'origine ne montre aucun long task, sans doute parce que les conditions de déclenchement n'étaient pas réunies. Ce problème vient donc de la lecture du code et pas d'une mesure.

**Preuve.** Le code de `analytics.js` et `audit/04-performances-frise-avant.png` (frise plate).

### 4.7 Les dimensions des images

**Problème.** Les `<img>` n'avaient ni `width` ni `height` (Lighthouse le signale), donc le navigateur ne réservait pas la place des images.

**Correction.** J'ai ajouté `width` et `height` partout (400×226 pour l'image 1, 400×300 pour les autres), `loading="lazy"` sur les images 4 à 6, `fetchpriority="high"` sur l'image 1, et `max-width:100%` sur les pages de détail (l'image y débordait à 3968 px).

**Gain mesuré.** Le CLS n'a pas baissé (voir le tableau : 0 → 0,022 en Mobile et 0,031 → 0,059 en Desktop). Cette correction reste utile pour réserver l'espace, mais je ne peux pas dire qu'elle ait amélioré le CLS mesuré. Le débordement horizontal de la page de détail a bien disparu.

**Preuve.** `audit/09-debordement-avant.png` et `09-debordement-apres.png`.

## 5. Iso-rendu

J'ai gardé toutes les parties du site : bandeau, galerie, filtre, suggestions, calendrier, fonctionnalités, section Défis, pages de détail et polices. Je n'ai supprimé que du code sans effet visible. Les captures `audit/10-iso-*` et `audit/11-iso-*` comparent l'accueil et une page de détail avant et après.

## 6. Ce que je ferais avec un vrai serveur

Sur GitHub Pages, je ne peux pas modifier les en-têtes HTTP. Avec mon propre serveur, je mettrais :
- `Cache-Control: max-age=31536000, immutable` sur les images, le CSS et le JS, et une revalidation courte sur le HTML ;
- des noms de fichiers avec une empreinte (`styles.a3f9c1.css`) pour pouvoir garder ce cache long, ce qui demande normalement un outil de build ;
- la compression Brotli sur les fichiers texte (GitHub applique déjà gzip) ;
- HTTP/2 ou HTTP/3, et un CDN ;
- la police Poppins hébergée avec le site, pour supprimer les deux domaines Google et la feuille de style externe qui bloque le rendu. C'est le principal gain qui reste sur le FCP et le LCP Mobile. Je ne l'ai pas fait pour ne pas risquer de changer l'affichage du texte.

## 7. Conclusion

Les images faisaient plus de 99 % du poids, donc la correction principale a été de les redimensionner et de les passer en WebP (17,5 Mo → environ 0,42 Mo). La chaîne d'`@import` et les scripts bloquants expliquaient le FCP. Le JavaScript (`ui-kit`, `buildKeywordIndex`, `revealCards`, `analytics.js`) expliquait le TBT. Le score Lighthouse passe de 43 à 96 en Mobile et atteint 99 en Desktop, mais ce sont des relevés uniques à confirmer avec la médiane de trois essais, et le score n'est pas le but en soi. PageSpeed donne 91 en Mobile et 100 en Ordinateur. Les autres catégories (Accessibilité, SEO) ne font pas partie du TP.