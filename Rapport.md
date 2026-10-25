# Rapport d'optimisation — R3.14, site « SlowSite »

- **Mon site corrigé (en ligne)** : https://tonio-cc19.github.io/R314-optimisation-site/
- **Mon fork (historique des commits)** : https://github.com/Tonio-cc19/R314-optimisation-site
- **Site de référence (avant)** : https://nico3807.github.io/R314-optimisation-site/

## 1. Comment j'ai mesuré

J'ai mesuré le site **en ligne** et pas en local, parce qu'un serveur local ne compresse pas les fichiers texte comme GitHub Pages. J'ai utilisé Chrome en navigation privée, avec le cache désactivé (la case est cochée sur mes captures du Réseau) et un rechargement avec Ctrl+Maj+R. J'ai lancé Lighthouse dans les DevTools (catégorie Performances) en Mobile puis en Desktop, et PageSpeed Insights sur l'URL de mon fork (onglets Mobile et Bureau). Pour le panneau Performances, j'ai mis le CPU en ralentissement ×4. Les mesures « avant » et « après » sont faites sur la même URL, avant puis après mes modifs.

Les limites, pour être honnête :
- Je n'ai que des **données de labo**. La partie « utilisateurs réels » de PageSpeed affiche « Aucune donnée » (pas assez de trafic).
- Chaque chiffre vient d'**un seul lancement** par profil. Le TBT varie pas mal sur ce site, donc je ne conclus rien sur de petits écarts.
- Mon enregistrement Performances « avant » ne dure que **4,8 s** (au lieu des 20-30 s demandées). C'est trop court pour voir ce qui démarre après un délai (problème 15).
- Ma capture `audit/coverage_avant.png` montre le panneau Styles et pas le rapport Coverage. Pour le CSS inutilisé « avant », je me base donc sur la taille du fichier et la lecture du code.
- Je n'ai pas mesuré chaque optimisation séparément : les gains par partie sont mesurés sur les fichiers concernés ou dans le profil d'exécution, et le gain global est mesuré avant/après.

## 2. Les problèmes que j'ai trouvés

| # | Fichier et ligne | Le problème | Preuve | Ce que ça coûte | Ma correction |
|---|---|---|---|---|---|
| 1 | `img/img1.jpg` à `img6.jpg` (`index.html:75-90`) | JPEG de 2040 à 3968 px de large, affichés à ~388 px | `audit/reseau_avant.png` | 0,77 à 3,39 Mo par image | Redimensionner, WebP, `srcset` |
| 2 | `index.html:12` | `preload` de `img6.jpg?v=2` : la même image est téléchargée 2 fois | `audit/reseau_avant.png` (3 393 ko chacune) | 3,39 Mo | Supprimer le `preload` |
| 3 | `css/theme-legacy.css:6-12` | `hero-bg.jpg` (3,39 Mo, même taille que `img6.jpg`) affichée à 4 % d'opacité | `audit/reseau_avant.png` | 3,39 Mo | Texture WebP d'environ 20 Ko |
| 4 | `img/favicon.png` | PNG 512×512 pour une simple icône | `audit/reseau_avant.png` | 788 Ko | Version 64×64 |
| 5 | `index.html:16` | `js/legacy/polyfills.js` n'existe pas (404) et bloque le chargement | `audit/reseau_avant.png` (ligne rouge) | 1 requête perdue + blocage | Supprimer la ligne |
| 6 | `index.html:17-18` | Scripts sans `defer` dans le `<head>` | `audit/performances_frise_avant.png` | Le HTML est bloqué le temps que les scripts s'exécutent | Ajouter `defer` |
| 7 | `css/styles.css:1`, `css/extra.css:1` | Chaîne d'`@import` sur 3 niveaux (`styles.css` → `extra.css` → `theme-legacy.css` → Google Fonts) | `audit/reseau_avant.png` (colonne Initiateur) | 3 allers-retours à la suite avant le 1er affichage | Des `<link>` en parallèle dans le HTML |
| 8 | `css/styles.css:2`, `css/theme-legacy.css:3`, `index.html:9-11` | Polices : `display=block`, graisse 300 et Playfair Display jamais utilisées, `preload` de police sans `crossorigin`, `dns-prefetch` inutile | `audit/reseau_avant.png` (3 `woff2`, 2 feuilles Google) | Texte invisible au début, requêtes en trop | `display=swap`, Poppins 400 et 600, `preconnect` |
| 9 | `css/styles.css`, `css/extra.css` | ~2 000 règles `.redundant-N` / `.extra-N` jamais utilisées | Taille des fichiers (git) : 92 048 octets sur 1 261 lignes, et 28 831 octets | 92 Ko + 29 Ko bruts (8,6 Ko + 2,5 Ko transférés) | Réécrire avec seulement les règles utilisées |
| 10 | `index.html:75-90` | `<img>` sans `width`/`height` ni `loading="lazy"` | Lecture du code | Place non réservée, images hors écran chargées d'emblée | `width`/`height`, `lazy` sous la ligne de flottaison |
| 11 | `js/vendor/ui-kit.min.js:2:845` | Boucle de 620 ms (`c`) et tableau de 240 000 objets (`b`) créés au chargement | `audit/performances_bottom_up_avant.png` : appel `ui-kit.min.js:2:845` = 1 348 ms | ~1,35 s dans `readystatechange` (CPU ×4) | Supprimer la bibliothèque, JS natif |
| 12 | `js/vendor/ui-kit.min.js` (fin du fichier) | `MutationObserver` sur toute la page, déclenché à chaque changement du DOM | Lecture du code (**pas mesuré séparément**) | À mesurer avec du scroll | Supprimé avec la bibliothèque |
| 13 | `js/app.js:74-81` | `innerHTML +=` dans une boucle (`buildKeywordIndex`) | `audit/performances_bottom_up_avant.png` : `app.js:74:29` = 758 ms au total | 758 ms (CPU ×4) | `DocumentFragment` |
| 14 | `js/app.js:158-190` | `revealCards` : boucle dans une boucle avec `offsetHeight` et `getBoundingClientRect` (reflow forcé), 371 cases de calendrier relues, appelée à chaque `scroll`/`resize` (lignes 213-214) | Lecture du code, `audit/performances_frise_avant.png` | Reflow à chaque scroll | `IntersectionObserver`, `requestAnimationFrame`, écouteur `passive` |
| 15 | `js/analytics.js` | Code volontairement illisible (chaînes en base64 lues avec `atob` : `innerWidth`, `performance`, `now`, `hidden`). Démarre 6,5 s après le chargement, `setInterval` de 2,6 s, boucle de 46 ms qui grossit de ×1,28 jusqu'à 430 ms, seulement si la fenêtre fait plus de 1024 px et l'onglet est visible | Lecture du code. **Pas prouvé par une mesure** (mon enregistrement « avant » ne dure que 4,8 s) | Jusqu'à 430 ms de blocage toutes les 2,6 s (estimation d'après le code) | Supprimer (aucune fonction visible, aucune donnée envoyée) |
| 16 | `img1.html` à `img6.html` | Image à sa taille native (jusqu'à 3968 px) sans limite CSS : la page déborde. `ui-kit` et `app.js` chargés pour rien | `audit/debordement_avant.png` | Page plus large que l'écran | `max-width:100%`, WebP, retrait des 2 scripts |

## 3. Avant / après

### 3.1 Les chiffres globaux

| Indicateur | Avant | Après | Gain |
|---|---|---|---|
| Poids transféré | 17,5 Mo | ~0,42 Mo (418 ko) | environ ÷ 42 |
| Requêtes | 22 | 14 | −8 |
| **Lighthouse Mobile** | | | |
| Score | 37 | 93 | +56 |
| FCP | 3,5 s | 1,8 s | −1,7 s |
| LCP | 71,2 s | 3,1 s | −68,1 s |
| TBT | 970 ms | 10 ms | −960 ms |
| CLS | 0 | 0 | inchangé |
| Speed Index | 11,5 s | 1,8 s | −9,7 s |
| **Lighthouse Desktop** | | | |
| Score | 66 | 99 | +33 |
| FCP | 0,9 s | 0,5 s | −0,4 s |
| LCP | 11,1 s | 0,9 s | −10,2 s |
| TBT | 50 ms | 0 ms | −50 ms |
| CLS | 0,059 | 0,059 | inchangé |
| Speed Index | 3,3 s | 0,5 s | −2,8 s |
| **PageSpeed Insights** | | | |
| Mobile | 61 | 93 | +32 |
| Ordinateur | 45 | 99 | +54 |

D'où viennent les chiffres : le poids vient de la synthèse du panneau Performances (`performances_frise_avant.png` : 17 524 ko + 25 ko Google Fonts, et `performances_frise_apres.png` : 402 ko + 16,5 ko). Les requêtes viennent du décompte des lignes de `reseau_avant.png` (22) et `reseau_apres.png` (14), parce que je n'ai pas de capture de la barre de statut du bas. Le CLS n'a pas bougé : mes corrections ne l'ont ni amélioré ni dégradé, et il reste dans la zone « bon » (moins de 0,1).

### 3.2 Avant / après pour quelques fichiers

| Fichier | Avant | Après |
|---|---|---|
| `img1` | 2 525 ko (JPEG) | 14,5 ko (WebP 800 px) |
| `img4` | 1 233 ko (JPEG) | 97,6 ko (WebP 800 px) |
| `hero-bg` | 3 393 ko (JPEG) | 21,8 ko (WebP) |
| `favicon.png` | 788 ko (512×512) | 3,8 ko (64×64) |
| `styles.css` (transféré) | 8,6 ko | 2,0 ko |
| `styles.css` (brut) | 92 048 octets | 5 024 octets |
| `extra.css` et `theme-legacy.css` | 2 fichiers | supprimés |
| `ui-kit.min.js`, `analytics.js`, `polyfills.js` | 3 scripts | supprimés |

### 3.3 Avant / après sur le CPU (panneau Performances)

| Mesure | Avant | Après |
|---|---|---|
| Création de script (résumé de la frise) | 1 402 ms | 79 ms |
| `readystatechange` (Bottom-Up) | 1 394 ms | plus aucune fonction JS en tête |
| `buildKeywordIndex` | 758 ms au total | n'apparaît plus |
| Durée de l'enregistrement | 4,8 s | 30 s (sans tâche longue) |

## 4. Mes optimisations principales

### 4.1 Les images

**Le problème.** Les images font presque tout le poids de la page. Les 6 JPEG pèsent entre 0,77 et 3,39 Mo, avec 2040 ou 3968 px de large alors qu'elles sont affichées à environ 388 px. En plus, `hero-bg.jpg` (3,39 Mo) a la même taille qu'`img6.jpg` et ne sert que de texture à 4 % d'opacité, `img6.jpg?v=2` est téléchargée une 2ᵉ fois à cause d'un `preload`, et le favicon fait 788 Ko. C'est ce qui explique le LCP de 71,2 s en Mobile.

**Ce que j'ai fait.** J'ai redimensionné chaque photo en 400, 800 et 1600 px et je l'ai convertie en WebP. La galerie utilise `srcset` et `sizes`, et les pages de détail utilisent les versions 800 et 1600 px. La texture du bandeau fait maintenant 21,6 Ko et le favicon 64×64 fait 3,7 Ko. J'ai supprimé le `preload` de `img6.jpg?v=2` et les anciens JPEG.

**Le gain.** 17,5 Mo → ~0,42 Mo, et 22 → 14 requêtes. Le LCP Mobile passe de 71,2 s à 3,1 s et le LCP Desktop de 11,1 s à 0,9 s.

**Preuve.** `audit/reseau_avant.png`, `audit/reseau_apres.png` et les captures Lighthouse avant/après.

### 4.2 La chaîne critique, les polices et le 404

**Le problème.** `styles.css` importait `extra.css`, qui importait `theme-legacy.css`, qui importait une feuille Google Fonts (Playfair Display, que le site n'utilise même pas). Ça fait 3 allers-retours à la suite avant d'afficher quoi que ce soit, alors que le CSS bloque le rendu. Poppins était en `display=block` (texte invisible au début), avec une graisse 300 inutile et un `preload` sans `crossorigin`. En plus `polyfills.js` donnait un 404 et les scripts n'avaient pas `defer`.

**Ce que j'ai fait.** J'ai supprimé les `@import` et mis des `<link>` côte à côte dans le HTML. Poppins est en `display=swap` avec seulement les graisses 400 et 600, et j'ai ajouté les `preconnect` (avec `crossorigin` pour `fonts.gstatic.com`). J'ai enlevé `polyfills.js`, le `preload` de police et le `dns-prefetch`, et mis `defer` sur le script restant.

**Le gain.** Dans la cascade, les CSS ne s'enchaînent plus et les requêtes `extra.css`, `theme-legacy.css`, Playfair et le 404 ont disparu. Le FCP Mobile passe de 3,5 s à 1,8 s et le FCP Desktop de 0,9 s à 0,5 s (gain global, avec les autres corrections).

**Preuve.** `audit/reseau_avant.png` (colonne Initiateur) et `audit/reseau_apres.png`.

### 4.3 Le CSS inutilisé

**Le problème.** `styles.css` faisait 92 048 octets sur 1 261 lignes et `extra.css` 28 831 octets. Environ 2 000 règles `.redundant-N` et `.extra-N` ne correspondent à rien dans le HTML. Mais ces tailles sont non compressées : en ligne, ils ne pèsent que 8,6 Ko et 2,5 Ko, donc ce n'était pas le plus gros problème.

**Ce que j'ai fait.** J'ai réécrit `styles.css` avec seulement les règles utilisées (5 024 octets) et supprimé `extra.css` et `theme-legacy.css`. J'ai gardé tout ce qui se voit : texture et ombre du bandeau, filtre, calendrier.

**Le gain.** `styles.css` transféré passe de 8,6 Ko à 2,0 Ko. Le gain direct est petit, comme prévu. Le vrai gain, c'est la fin de la chaîne d'`@import`. Dans le Coverage « après », `styles.css` n'a plus que 14,2 % d'octets inutilisés (706 sur 4 956).

**Preuve.** `audit/coverage_apres.png`. Pour l'« avant », voir la limite de la partie 1.

### 4.4 Le JavaScript : `ui-kit.min.js`

**Le problème.** Cette bibliothèque lançait dès le chargement une boucle de 620 ms sur `performance.now()` (fonction `c`) et fabriquait un tableau de 240 000 objets (fonction `b`) qui ne sert à rien. Elle ajoutait aussi un `MutationObserver` sur toute la page, déclenché à chaque changement du DOM. Ses fonctions utiles (`qs`, `qsa`, `cls`, `ready`) ne font que reprendre `querySelector` et `classList`.

**Ce que j'ai fait.** Je l'ai supprimée, et `app.js` utilise directement les fonctions du navigateur.

**Le gain.** Dans le Bottom-Up « avant », `readystatechange` prend 1 394 ms dont 1 348 ms dans l'appel `ui-kit.min.js:2:845`. La « création de script » passe de 1 402 ms à 79 ms. Le TBT Mobile passe de 970 ms à 10 ms. Dans le Bottom-Up « après », il n'y a plus de fonction JS en tête, seulement des tâches du navigateur (recalcul du style, mise en page).

**Preuve.** `audit/performances_bottom_up_avant.png` et `audit/performances_bottom_up_apres.png`.

### 4.5 Le JavaScript : `app.js`

**Le problème.** `buildKeywordIndex` faisait `innerHTML +=` dans une boucle, donc la liste était reconstruite à chaque tour (758 ms au total). `revealCards` tournait à chaque scroll avec une boucle dans une boucle qui relisait `offsetHeight` et `getBoundingClientRect` juste après avoir écrit des styles : c'est un **reflow forcé**, en plus sur 371 cases de calendrier. Avec le `MutationObserver` de `ui-kit`, les deux se renforçaient pendant le scroll.

**Ce que j'ai fait.** La liste est construite dans un `DocumentFragment` puis insérée une seule fois. L'apparition des cartes passe par un `IntersectionObserver`. Pour le calendrier, je ne mesure plus que les 7 lignes de la grille et je mets à jour 7 variables CSS (au lieu de 371 cases), avec `requestAnimationFrame` et un écouteur `passive`.

**Le gain.** Le TBT passe de 970 à 10 ms en Mobile et de 50 à 0 ms en Desktop. Sur la frise « après » (30 s, avec du scroll), il n'y a plus de tâche longue.

**Preuve.** `audit/performances_bottom_up_avant.png` (`buildKeywordIndex` en `app.js:74:29`) et `audit/performances_frise_apres.png`. Pour `revealCards` (`app.js:158`), c'est la lecture du code : je n'ai pas de capture qui l'isole.

**Deux petites différences de comportement.** Une carte déjà apparue ne redescend plus quand on remonte la page. Et l'ancien code agrandissait la hauteur des cartes à chaque événement : c'était un défaut, et la grille CSS gère déjà ça.

### 4.6 Le script `analytics.js` (niveau 3)

**Le problème.** Le fichier est écrit pour être illisible (chaînes en base64). Une fois décodées avec `atob`, elles donnent `innerWidth`, `performance`, `now` et `hidden`. Il démarre 6,5 s après le chargement, puis lance un `setInterval` toutes les 2,6 s. Si l'onglet est visible et que la fenêtre fait plus de 1024 px, il occupe le processeur de plus en plus longtemps (de 46 ms jusqu'à 430 ms). Il n'envoie aucune donnée. Ça peut expliquer pourquoi des mesures se contredisent : il est invisible sur mobile et dans un audit court.

**Ce que j'ai fait.** Je l'ai supprimé avec sa balise `<script>`. Il n'avait aucun effet visible.

**Le gain.** Je n'ai pas réussi à le mesurer : mon enregistrement « avant » dure 4,8 s, donc moins que les 6,5 s de délai avant son démarrage. Ce problème vient de la lecture du code et pas d'une mesure. Sur la frise « après » de 30 s, il n'y a plus de tâche longue.

**Preuve.** Le code de `analytics.js` (dans l'historique du fork) et `audit/performances_frise_apres.png`.

### 4.7 Les dimensions des images

**Le problème.** Les `<img>` n'avaient ni `width` ni `height`, donc le navigateur ne réservait pas la place. Celles sous la ligne de flottaison étaient aussi chargées dès le début.

**Ce que j'ai fait.** J'ai ajouté `width` et `height` partout (400×226 pour l'image 1, 400×300 pour les autres, 1600×903 sur la page de détail 1), `loading="lazy"` et `decoding="async"` sur les images 4 à 6, `fetchpriority="high"` sur l'image 1, et `max-width:100%` sur les pages de détail (l'image y débordait à sa taille native).

**Le gain.** Le CLS n'a pas baissé (0 → 0 en Mobile, 0,059 → 0,059 en Desktop). Cette correction reste utile pour réserver la place, mais je ne peux pas dire qu'elle a amélioré le CLS mesuré. Par contre, le débordement horizontal de la page de détail a bien disparu.

**Preuve.** `audit/debordement_avant.png` et `audit/debordement_apres.png`.

## 5. Iso-rendu

J'ai gardé toutes les parties du site : bandeau, galerie, filtre, suggestions, calendrier, fonctionnalités, section Défis, pages de détail et polices. Je n'ai supprimé que du code qui n'avait aucun effet visible. Les aperçus de page dans mes captures PageSpeed avant et après (`audit/pagespeed_mobile_*.png` et `audit/pagespeed_bureau_*.png`) montrent le même bandeau et la même galerie, et `audit/debordement_apres.png` montre la page de détail bien affichée. Je n'ai pas de capture dédiée qui compare toute la page d'accueil avant et après.

## 6. Ce que je ferais avec un vrai serveur

Sur GitHub Pages, je ne peux pas modifier les en-têtes HTTP. Avec mon propre serveur, je mettrais :
- `Cache-Control: max-age=31536000, immutable` sur les images, le CSS et le JS, et une revalidation courte sur le HTML ;
- des noms de fichiers avec une empreinte (`styles.a3f9c1.css`) pour garder ce cache long, ce qui demande normalement un outil de build ;
- la compression Brotli sur les fichiers texte (GitHub fait déjà du gzip) ;
- HTTP/2 ou HTTP/3, et un CDN ;
- la police Poppins hébergée avec le site, pour supprimer les deux domaines Google et la feuille de style externe qui bloque le rendu. C'est le principal gain qui reste sur le FCP et le LCP Mobile (1,8 s et 3,1 s, encore en orange). Je ne l'ai pas fait pour ne pas risquer de changer l'affichage du texte.

## 7. Conclusion

Les images faisaient plus de 99 % du poids, donc la correction principale a été de les redimensionner et de les passer en WebP (17,5 Mo → ~0,42 Mo). La chaîne d'`@import` et les scripts bloquants expliquaient le FCP. Le JavaScript (`ui-kit`, `buildKeywordIndex`, `revealCards`, `analytics.js`) expliquait le TBT. Le score Lighthouse passe de 37 à 93 en Mobile et de 66 à 99 en Desktop, mais ce sont des relevés uniques à confirmer avec la médiane de 3 essais, et le score n'est pas le but en soi. PageSpeed donne 93 en Mobile (61 avant) et 99 en Ordinateur (45 avant). Sur PageSpeed, Bonnes pratiques passe de 96 à 100, et l'Accessibilité et le SEO restent à 91 : ils ne font pas partie du TP.