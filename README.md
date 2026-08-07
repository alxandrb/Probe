# PROBE

Convertisseur de textures vers environnements 360 prêts pour Unity. Page HTML autonome, rendu WebGL2, aucun serveur : les images ne quittent jamais la machine.

## Démarrer

Ouvrir `probe-360.html` dans un navigateur récent (Chrome, Edge, Firefox, Safari 16+). Aucune installation, aucune dépendance — sauf JSZip, chargé depuis un CDN uniquement au moment de cliquer sur « Tout télécharger (.zip) ». Sans connexion, les téléchargements un par un fonctionnent quand même.

Déposer une image, vérifier l'orientation dans la vue 360, cocher les sorties voulues, générer.

## Entrées reconnues

Le format est déduit du ratio de l'image, et peut être forcé dans le menu déroulant.

| Format | Ratio détecté | Notes |
|---|---|---|
| Équirectangulaire (lat-long) | 2:1 | le cas le plus courant, HDRI de banque d'images |
| Croix horizontale | 4:3 | cubemap déplié |
| Croix verticale | 3:4 | le −Z est lu tourné à 180° |
| Bande horizontale | 6:1 | ordre `+X −X +Y −Y +Z −Z` |
| Bande verticale | 1:6 | même ordre, de haut en bas |
| Angular map / light probe | — | centre de l'image = `+Z`, bord du cercle = `−Z` |
| Boule miroir | — | ball photographiée depuis `−Z` |
| Fisheye 180° | — | dôme au zénith, hémisphère inférieur miroité |
| Texture plate, étirée | tout | l'image est tendue sur la sphère entière |
| Texture plate, en tuiles | tout | répétition réglable, avec miroir pour tuer les coutures |

Fichiers acceptés : PNG, JPG, WebP, et **Radiance `.hdr`** (RLE et non compressé). Un `.hdr` est décodé en flottant et reste linéaire dans toute la chaîne.

## Sorties

- **Équirectangulaire** — 1024 à 8192 px de large, ratio 2:1.
- **6 faces séparées** — nommées `posx_right`, `negx_left`, `posy_up`, `negy_down`, `posz_front`, `negz_back`.
- **Bande horizontale** — la disposition « 6 Frames Layout » de Unity.
- **Croix horizontale (4×3)** et **croix verticale (3×4)** — le −Z de la croix verticale est écrit tourné à 180°.

Formats : PNG 8 bits, JPEG qualité 92, ou Radiance `.hdr`. La sortie `.hdr` n'est proposée que depuis une source `.hdr`, exige `EXT_color_buffer_float`, et est plafonnée à 4096 px de large côté lat-long (au-delà, la lecture GPU dépasse le gigaoctet).

## Conventions

Tout est exprimé dans le repère de Unity : main gauche, `X` à droite, `Y` en haut, `Z` en avant.

**Lat-long.** Le centre horizontal de l'image correspond à `+X`, le haut à `+Y`, et la longitude augmente vers `−Z`. C'est exactement ce que fait le shader `Skybox/Panoramic` de Unity. La plupart des HDRI de banque d'images placent leur centre ailleurs : le slider « Rotation autour de Y » sert à ça.

**Cubemap.** Faces D3D/Unity :

```
+X → ( 1,  v, -u)      +Y → ( u,  1, -v)      +Z → ( u,  v,  1)
−X → (-1,  v,  u)      −Y → ( u, -1,  v)      −Z → (-u,  v, -1)
```

avec `u` vers la droite de l'image et `v` vers le haut. Ces mêmes fichiers conviennent à un asset Cubemap et à un matériau `Skybox/6 Sided` — les deux partagent cette orientation.

**Vérification.** La boussole de la vue 360 affiche `+Z / +X / −Z / −X` au fil de la rotation. Si un repère connu de la source ne tombe pas sur le bon axe, corriger avec la rotation Y ou les miroirs avant d'exporter, pas après.

## Import dans Unity

**Lat-long en cubemap** — `Texture Shape → Cube`, `Mapping → Latitude-Longitude Layout (Cylindrical)`. `Convolution Type: None` pour un skybox, `Specular` pour une réflexion.

**Bande ou croix en cubemap** — `Texture Shape → Cube`, `Mapping → 6 Frames Layout (Cubic Environment)`.

**6 faces séparées** — matériau `Skybox/6 Sided` : `_RightTex`=posx, `_LeftTex`=negx, `_UpTex`=posy, `_DownTex`=negy, `_FrontTex`=posz, `_BackTex`=negz. Passer les six en `Wrap Mode: Clamp`, sinon des lignes apparaissent aux arêtes du cube.

**Fichiers .hdr** — importés nativement. Activer `High Dynamic Range`, laisser la compression en `BC6H` pour une réflexion ou une lightmap.

**Éclairage** — `Window → Rendering → Lighting → Environment → Skybox Material`, puis `Generate Lighting` pour que l'ambiante suive la nouvelle sphère.

## Détails d'implémentation

Un seul contexte WebGL2 et un seul fragment shader font tout le travail. Le shader calcule une direction monde à partir du pixel de sortie (lat-long, face de cube, ou rayon caméra pour l'aperçu), applique l'orientation, puis échantillonne la source selon son interprétation. Les exports passent par des framebuffers hors écran, `RGBA8` pour le 8 bits et `RGBA32F` pour le `.hdr`.

Deux précautions contre les artefacts de filtrage :

- La couture au méridien ±180° d'une lat-long casse les dérivées d'écran et produit une ligne floue. Le shader calcule les dérivées sur deux paramétrisations décalées d'un demi-tour et garde la plus petite.
- Pour une source en croix ou en bande, les dérivées sont calculées dans l'espace local de la face, ce qui garde le mip correct partout sauf sur la ligne d'arête.

Les mipmaps de la source sont générées quand le pilote l'accepte, ce qui évite l'aliasing quand une grande texture est réduite vers une petite face.

## Limites connues

- L'export `.hdr` n'écrit pas de compression RLE : les fichiers sont plus gros que ceux d'un outil dédié, mais restent parfaitement lisibles.
- Les `.hdr` dont l'en-tête déclare une orientation autre que `-Y … +X …` sont refusés.
- Un `.exr` ne peut pas être lu (le décodage OpenEXR demanderait une dépendance externe).
- Le point de silhouette d'une boule miroir n'a pas d'information exploitable : une source de ce type laisse toujours une zone dégénérée derrière la caméra.
- Une sortie 8192×4096 en PNG demande environ 500 Mo de mémoire pendant l'encodage. Sur une machine juste, rester à 4096.
