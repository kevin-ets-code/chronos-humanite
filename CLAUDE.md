# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

CHRONOS · Humanité — an interactive 3D Earth globe visualizing human history from -300,000 to today, built with Three.js. Zero npm dependencies: a single self-contained `index.html` plus external JSON data files.

- Live: https://chronos-humanite.vercel.app
- Repo: https://github.com/kevin-ets-code/chronos-humanite

## Commands

```bash
# Run locally (textures fail to load via file:// due to CORS — use a server)
npx serve .
# or
python -m http.server 8080

# Deploy: just push to main, Vercel auto-deploys
git push origin main
```

There is no build step, bundler, linter, or test suite — everything runs directly in the browser from `index.html`.

## Structure

- `index.html` — the entire app: HTML, CSS (`<style>`), and all JS (`<script>` blocks). ~3380 lines.
- `js/vendor/` — scripts three.js examples r128 du bloom (`CopyShader`, `LuminosityHighPassShader`, `EffectComposer`, `ShaderPass`, `RenderPass`, `UnrealBloomPass`), embarqués depuis le tag r128 du repo three.js car cdnjs ne sert pas `examples/js` en r128 (404, même cas qu'earcut). Voir la section "Bloom".
- `data/*.json` — event data, split by historical era (`01-prehistoire.json` … `13-xxie.json`), plus `categories.json`, `eras.json` and `territories.json` (GeoJSON snapshots des empires, overlay EMPIRES).
- `data/timeline-config.json` — fenêtre temporelle par zone historique (`windowZones`, chacune avec un `halfWindow` qui définit la largeur de la fenêtre visible autour de l'année centrale).
- `textures/` — Earth day/night/normal/specular/clouds JPGs (+ variantes `2k_*`), et skybox Voie lactée `8k_stars_milky_way.jpg` (desktop) / `2k_stars_milky_way.jpg` (mobile).
- `#border-left`, `#border-top`, `#border-bottom` — fines lignes dorées translucides sur les bords de la fenêtre (identiques visuellement au `::before` de `#side-panel`); `#border-left-glow`, `#border-top-glow` — glows dorés associés, en dégradé depuis le bord. Le conteneur `#timeline` a un `::before` qui projette un glow doré vers le haut, en direction du globe.

## Architecture: the script blocks in index.html

The page wires together seven separate script blocks (in source order), each owning one concern and communicating only through `window._*` globals (see below). When changing one, check whether it reads/writes a global another block depends on.

1. **Globe renderer** (~`L1032-1931`): the largest block. Persistence helper (`window._state` — localStorage + URL query sync), Three.js renderer (ACES Filmic + sRGB, see "Pipeline couleur"), scene/camera, textures, Earth mesh with custom GLSL day/night shader (`earthMat`), cloud layer (fBm shader desktop / JPG mobile), atmospheric halo (**one** BackSide shader sphere), starfield + Milky Way skybox in a shared `skyGroup`, the `#visual-settings` slider factory (`makeVsSlider`), mouse/touch drag with inertia, the bloom desktop (voir section "Bloom"), and the render loop (`(function loop() {...})` — `composer.render()` dès que le bloom est initialisé, sinon rendu direct). Globe auto-rotation speed is `rotSpeed` (slider VITESSE, section GLOBE; min = 0 = immobile — the old play/pause "ROTATION DE LA TERRE" button was removed, and a legacy persisted paused state is migrated to `vs-rot = 0` then purged).
2. **Timeline slider** (~`L2065-2261`): drives a single draggable thumb (`centerVal`, range 0-1000) on `#tl-track`, converts slider position to a year via breakpoints (`BP` / `sliderToYear`), updates the displayed year/era label, pushes `uNightLights` into `earthMat`, and calls `updateTerritories()`. The visible time window around the centered year is data-driven: `getHalfWindow(year)` looks up the matching zone in `data/timeline-config.json` (`windowZones`, each with a `halfWindow`) and returns its half-width; `window.currentYearStart`/`window.currentYearEnd` are then computed as `centerYear ± halfWindow` and read by the sprites block to drive event visibility. Persisted as `'tl-slider-center'` (localStorage) and `?t=<year>` (URL); the older `'tl-slider-left'`, `'tl-slider-right'`, and `'tl-duration-index'` keys are actively cleared on load.
3. **Sun control** (~`L2263-2353`): drag handle UI (`#sun-control` / `#sun-handle`) on a full 360° ring; recomputes the sun angle β and writes `(sin β, 0.3, cos β)` to `earthMat.uniforms.uSun`. Persisted as `'sun-angle'` (degrees) and `?sun=` (URL).
4. **Sprites + hover + filters** (~`L2355-2856`): loads all JSON data (`loadData`/`ERA_FILES`, including `territories.json`), builds one `THREE.Sprite` per event placed via `latLngToVec3`, fades sprite opacity in/out based on the current year (`getOpacity`/`smoothstep`), handles hover tooltips and HTML hit-zones overlaid on the canvas (`#sprite-overlay`), spreads overlapping events with `applyClusterOffsets`, and builds the category filter chips (`#filters-list`, persisted as `'active-filters'` / `?filters=`). Rendu des sprites : `SpriteMaterial` avec `toneMapped = false` (hors ACES, comme le halo atmosphère — sinon le tonemapping désature les couleurs de catégories vers le blanc) et texture canvas en `encoding = THREE.sRGBEncoding` obligatoire (sinon double éclaircissement avec `outputEncoding`). Sur desktop, `applySpriteDim(0.6)` atténue `material.color` par défaut pour rester sous le threshold du bloom (0.85) ; setter console `window._spriteDim(valeur)`, ré-applicable à chaud (couleur de base mémorisée, jamais écrasée).
5. **Side panel + territoires** (~`L2886-3095`): opens `#side-panel` with an event's details (`openPanel`/`closePanel`); then the territories overlay — empire polygons drawn into a 4096×2048 canvas texture mapped on a sphere (`terrSphere`, r=1.004, child of `_earth`), redrawn by `drawTerritoriesCanvas(year)` / `window.updateTerritories`, toggled by the EMPIRES switch (section AFFICHAGE, persisted `'terr-visible'` / `?empires=`).
6. **Territory hover tooltip** (~`L3106-3194`): raycasts the Earth mesh, converts hit point to lat/lng, point-in-polygon against the active territory snapshots, shows `#terr-tooltip`.
7. **Mobile panels** (~`L3196-3252`): `toggleMobilePanel`/`closeMobilePanel` turn `#controls-left` and `#filters` into fullscreen slide-up panels under 768px.

### Cross-block globals (`window._*`)

Because each block is an isolated IIFE (or top-level script), they share state exclusively via globals on `window`:

| Global | Set by | Meaning |
|---|---|---|
| `_state` | renderer | persistence helper: `saveLocal`/`getLocal` (localStorage) + `syncURL`/`getParams` (query string) |
| `_scene`, `_camera`, `_renderer` | renderer | Three.js scene/camera/renderer (`_renderer` exposé pour calibrer `toneMappingExposure` en console) |
| `_earth`, `_earthMat` | renderer | Earth mesh and its `ShaderMaterial` (sprites and `terrSphere` are children of `_earth`; slider/sun write to `_earthMat.uniforms`) |
| `_cloudsMesh`, `_cloudsMat` | renderer | cloud layer (child of `_earth`); ShaderMaterial on desktop, MeshBasicMaterial on mobile |
| `_atmosphereMat` | renderer | halo shader material (`atmMultiplier` driven by the ATMOSPHÈRE slider) |
| `_starsMat` | renderer | starfield ShaderMaterial (calibrage console : `uTwinkleAmp`) |
| `_milkyMat`, `_milkySky` | renderer | Milky Way skybox material (brightness via `color.setScalar`) and mesh (`rotation.y` = position du cœur galactique) |
| `_bloom` | renderer (`initBloom`) | `UnrealBloomPass` du composer desktop (calibrage console : `strength` / `radius` / `threshold`) |
| `_spriteDim` | sprites block | setter d'atténuation des sprites (`_spriteDim(0.6)` desktop par défaut, protection anti-bloom) |
| `_hoverPause` | sprites block | true while hovering a sprite — pauses auto-rotation |
| `_categories`, `_eras`, `_territories` | sprites block (`loadData`) | parsed `categories.json` / `eras.json` / `territories.json` features, read by timeline & panel/territories blocks |
| `currentYearStart`, `currentYearEnd` | timeline | visible time window, read by sprites + territories |
| `updateTerritories`, `_terrSphere` | panel/territories block | redraw function called by the timeline; territories overlay mesh |
| `selectedEvent` | sprites block (`selectEvent`) | currently selected event object, read by the filter logic to keep it visible |
| `getCatColor`, `getCatLabel`, `_closePanel` | sprites/panel blocks | helper functions exposed for cross-block use |

## Pipeline couleur (ACES + sRGB)

Le renderer est configuré avec `toneMapping = ACESFilmicToneMapping` (`toneMappingExposure = 1.0`) et `outputEncoding = sRGBEncoding`. Conséquences pour tout matériau de la scène :

- **ShaderMaterial custom (Terre, nuages)** : doivent terminer leur fragment shader par `#include <tonemapping_fragment>` + `#include <encodings_fragment>` pour passer dans le pipeline. L'auto-décodage `texture.encoding` ne s'applique pas aux samplers de ShaderMaterial : `uDay`/`uNight` sont décodés manuellement (`pow(rgb, 2.2)`) dans le shader ; `uNormalMap`/`uSpecularMap` sont des maps techniques et restent linéaires.
- **Halo atmosphérique** : volontairement HORS pipeline ACES (aucun include) — glow additif dont le rendu calibré serait écrasé par le tonemapping.
- **Étoiles** : `#include <encodings_fragment>` seulement, PAS de tonemapping — l'ACES compressait les hautes lumières au point d'écraser les écarts de teinte entre familles et l'amplitude du scintillement.
- **Sprites événements** : `toneMapped = false` (hors ACES, comme le halo — le tonemapping désaturait les couleurs de catégories vers le blanc) + texture canvas en `encoding = THREE.sRGBEncoding` (le canvas 2D est authoré en sRGB ; sans cet encoding, échantillonné comme linéaire puis ré-encodé en sRGB → double éclaircissement).
- **MeshBasicMaterial (nuages mobile, Voie lactée)** : décodage auto via `texture.encoding = sRGBEncoding` ; la luminosité Voie lactée > 1 surexpose volontairement, l'ACES recompresse.

## Bloom (desktop uniquement)

`EffectComposer` + `UnrealBloomPass` (~`L1916-1970`), activé sur desktop uniquement (même détection `isMobile` que le reste) ; sur mobile — et pendant les premières frames le temps du chargement des scripts — `composer` reste `null` et la boucle garde le rendu direct.

- **Scripts** : les `examples/js` de r128 sont absents de cdnjs (404, même cas qu'earcut) → embarqués dans `js/vendor/` depuis le tag r128 du repo three.js, chargés dynamiquement avec `async = false` (exécution dans l'ordre d'insertion : `UnrealBloomPass` étend `THREE.Pass` défini par `EffectComposer`, etc.) ; `initBloom()` ne s'exécute que quand les 6 scripts sont chargés.
- **Paramètres** : `strength 0.35`, `radius 0.4`, `threshold 0.85`.
- **Anti double tone mapping** : les render targets du composer (`renderTarget1`/`renderTarget2`) sont en `sRGBEncoding` — en r128 l'encoding de sortie d'un matériau suit la texture du render target courant, la scène s'encode donc dans le RT exactement comme sur le canvas (ACES + sRGB pour Terre/nuages, halo/étoiles/sprites hors ACES inchangés). Et `bloom.basic.toneMapped = false` sur le matériau de copie écran du pass : le readBuffer est déjà tonemappé, décodage sRGB + réencodage sRGB = identité.
- Le resize handler appelle aussi `composer.setSize`.
- Console : `window._bloom` (`strength` / `radius` / `threshold`).

## GLSL shaders

### Terre (`earthMat`, ~`L1155-1384`)

Blends day and night textures based on `dot(N, uSun)` smoothstepped across the terminator, modulated by `uNightLights` (driven by the timeline year — city lights only show in eras where they'd exist). La normale `N` est dérivée par finite difference de la bump map (voir « Relief » ci-dessous), TBN construit au vertex. Texture day = NASA Blue Marble juillet topo+bathymetry (8192×4096 desktop / 2048×1024 mobile, mêmes noms de fichiers `earth_daymap.jpg` / `2k_earth_daymap.jpg`). Blocs additionnels, dans l'ordre du fragment shader :

- **Relief (bump map + displacement géométrique)** — bump map Natural Earth III (domaine public, http://www.shadedrelief.com/natural3/pages/extra.html), JPEG grayscale blanc=haut, **8192×4096 desktop + 2K mobile** (réutilise les noms de fichiers `earth_normal_map.jpg` / `2k_earth_normal_map.jpg` ; remplace l'ancienne normal map RGB de Solar System Scope). `uBumpMap` : sampler grayscale, `wrapS = RepeatWrapping` (corrige le seam au méridien 180° — la finite difference et le displacement échantillonnaient hors-bord en U), `wrapT = ClampToEdgeWrapping` (pas de repli des pôles), `generateMipmaps = false` + `minFilter = LinearFilter`. **Désactivation des mipmaps** : au seam la finite difference passe des UV discontinus (`fract()` + saut d'attribut 1→0 à la couture des sommets), ce qui fait spiker les dérivées implicites du mip-LOD et lire le mip 1×1 sur la ligne du méridien 180° → normale plate / strie verticale visible, amplifiée par glint+bloom. Sans mips, plus aucune sélection de LOD, le seam disparaît (le shader échantillonne des texels précis, les mips n'apportaient rien). Sert à deux choses : (1) **displacement géométrique** au vertex shader — `position + normal * texture2D(uBumpMap, uv).r * uDisplacementScale` (défaut **0.030**, amplitude du relief le long de la normale, altitude max ≈ R + 0.030 = 1.030) ; (2) **normale par finite difference** au fragment — 4 voisins à offset 1 texel (`1/4096`), en U via `fract()` pour un wrap continu au seam, en V sans wrap ; `N = normalize(vec3(-(hR-hL), -(hT-hB), 1.0/uNormalStrength))` dans le repère tangent (défaut `uNormalStrength` **2.5** : plus grand = pente plus marquée). TBN **analytique** passé en varying depuis le vertex shader (`tangent = cross(up, normalW)`, `bitangent = cross(normalW, tangent)`, `up` basculé près des pôles) — pas de dérivées d'écran. Sphère Terre densifiée à **512×256 segments desktop / 256×128 mobile** (au lieu de 64×64) pour porter le displacement. La **sphère empires** (`terrSphere`) est aussi densifiée à **512×256 desktop / 256×128 mobile** et **partage les uniforms `uBumpMap` et `uDisplacementScale` de `_earthMat`** (même référence d'objet → tout calibrage console se répercute à la frame suivante) : son vertex shader applique `displaced = position * 1.004 + normal * texture2D(uBumpMap, uv).r * uDisplacementScale` pour suivre exactement le relief, l'offset radial de 0.004 étant maintenu partout (l'overlay épouse la topographie au lieu d'être percé par les montagnes). Calibrage console : `uDisplacementScale`, `uNormalStrength`.

- **Compression océans vers teinte médiane** — `uOceanMid` (défaut `vec3(0.12, 0.12, 0.7)`, teinte cible) + `uOceanContrast` (défaut 0.85, conservation du contraste original ; 1 = pas d'effet, < 1 aplatit vers `uOceanMid`) : `mix(uOceanMid, dayColor, uOceanContrast)` appliqué aux océans uniquement (la bathymétrie Blue Marble est trop contrastée), masqué par la specular map (`specMask`, hissé là puis réutilisé par le sun glint), appliqué après le decode pow 2.2 et avant terminateur / mix jour-nuit (crépuscule et nuit restent cohérents).
- **Atténuation glace polaire** — `uIceLift` (défaut 0.72, facteur multiplicatif) + `uIceTint` (défaut `vec3(0.78, 0.85, 0.92)`, cyan-blanc neigeux) : l'Antarctique ressort cramé en blanc pur (saturé par ACES). Masque glace = forte luminance Rec.601 (`smoothstep(0.65, 0.85, lum)`) ET latitude haute (`smoothstep(0.30, 0.42, abs(vUv.y - 0.5))`, soit > ~60° N/S) ; sur ces pixels `dayColor *= uIceLift` puis désaturation 50% vers `uIceTint`. Appliqué AVANT le mix jour/nuit.
- **Terminateur crépusculaire** — teinte chaude de la lumière rasante. Uniforms `uTwilightWidth` (0.3), `uTwilightColor` (orangé), `uTwilightStrength` (0.85). Masque `1 - smoothstep(0, uTwilightWidth, cosA)` sans coupure côté nuit (la teinte meurt avec la lumière, pas de gap bleu), teinte normalisée à luminance constante (Rec.601) pour ne pas créer d'anneau plus lumineux que le jour.
- **Ombres portées des nuages** (desktop only — `uCloudShadowStrength` forcé à 0 sur mobile) — duplique le bruit du shader nuages (mêmes hash/noise/constantes) avec un fBm dégradé à 3 octaves au lieu de 6, échantillonné avec un offset de parallaxe vers le soleil. Uniforms `uTime` et `uCloudDensity` tenus en phase avec le shader nuages (boucle render + slider DENSITÉ) ; le bruit est évalué en repère LOCAL du globe (`vNormalL`/`vSunL`) pour que l'ombre tourne avec lui. Calibrage : `uCloudShadowStrength` (0.35).
- **Sun glint océanique** — Blinn-Phong deux lobes (voile large exp 100 / poids 0.2 + cœur serré exp 600 / poids 0.4, constantes commentées dans le shader), masqué par la specular map (océans) et `max(cosA, 0)`, atténué par la couverture nuageuse (`pow(1 - cloudShadow·1.5, 3)`). Multiplicateur global : `uGlintStrength`.

Don't touch the uniforms (`uDay`, `uNight`, `uSun`, `uNightLights`, `uTime`, `uCloudDensity`) without understanding how the timeline slider, sun control, density slider and render loop depend on them.

### Nuages (~`L1394-1532`)

Desktop : ShaderMaterial procédural — fBm 6 octaves avec double domain warping, masque latitudinal (zones de Hadley), éclairé par `uSun` (synchronisé sur `earthMat.uniforms.uSun` dans la boucle). Mobile : fallback JPG `MeshBasicMaterial` (perf), opacité = slider DENSITÉ, dérive simulée par rotation du mesh. `cloudsMesh` est enfant de `earth` → hérite de sa rotation.

### Atmosphère (~`L1534-1573`)

**Une seule** sphère ShaderMaterial (r=1.1, `BackSide`, additive) — glow de Fresnel piloté par `atmMultiplier` (slider ATMOSPHÈRE, clé `vs-atm`). Hors pipeline ACES (voir ci-dessus).

## Couches du globe (rayons R)

Empilement radial des coquilles concentriques (rayon local, sphère Terre = 1.0). Depuis l'ajout du displacement, le relief monte jusqu'à ~1.030 (à `uDisplacementScale` 0.030), ce qui contraint l'ordre des couches :

| Couche | R | Note |
|---|---|---|
| Globe surface | 1.000 | relief jusqu'à ~1.030 à `uDisplacementScale` 0.030 |
| Empires overlay (`terrSphere`) | 1.004 + displacement | **suit le relief** (partage `uBumpMap`/`uDisplacementScale` de `_earthMat`, offset radial 0.004 maintenu partout) → jamais percé par les montagnes, épouse la topographie |
| Sprites événements (`latLngToVec3` + `applyClusterOffsets`, **les deux** rayons à garder en phase) | 1.045 | juste sous les nuages |
| Nuages (`cloudsGeo`) | 1.050 | au-dessus de tout |

Sprites en perspective : marge confortable au-dessus du relief, mais les nuages (1.050) peuvent occasionnellement passer **devant** un sprite (1.045) — effet naturel accepté. Les hitzones HTML des sprites suivent automatiquement le rayon (projection de `sprite.getWorldPosition()` via `.project(cam)`, aucun R en dur). Le halo atmosphère (1.1) reste au-dessus de l'ensemble.

**Dépendance matérielle** : le displacement (Terre + empires) repose sur un **vertex texture fetch** (`texture2D` appelé dans le vertex shader). C'est universellement supporté sur les GPU modernes en WebGL1/r128 — pas de fallback prévu.

## Étoiles & Voie lactée (`skyGroup`)

- **Starfield** (~`L1685-1830`) : géométrie générée UNE SEULE FOIS à `STAR_MAX = 100000` ; le slider ÉTOILES ne fait que varier `geometry.drawRange` (aucune réallocation — les positions étant i.i.d. uniformes, tout préfixe reste homogène). Défaut 4500. ShaderMaterial points : tailles `pow(r,3)` (rares grandes étoiles), clamp `gl_PointSize` à 2.5 px + profil gaussien au fragment (anti-shimmer sub-pixel), 3 familles de couleurs (~25% bleu-blanc, ~55% blanc neutre, ~20% jaune-orangé), blending additif. **Scintillement** : désynchronisé au vertex, réservé aux étoiles au-dessus de la taille médiane (~40% d'entre elles → ~20% du total), onde composite lente ~10–25 s par étoile (`uTwinkleAmp` 0.25, `uTime` alimenté par `cloudClock`). Il est **binaire** via `uTwinkleOn`, piloté par le slider ROTATION du ciel : rotation = 0 → `0.0` (étoiles parfaitement figées) ; rotation > 0 → `1.0` (même lueur lente constante, amplitude et vitesse indépendantes de la vitesse de rotation). Initialisé avec la valeur persistée de `skySpeed`.
- **Skybox Voie lactée** (~`L1793-1827`) : sphère `BackSide` rayon 400 (au-delà des étoiles r ≤ 160, dans le far caméra 500), `renderOrder = -1`, texture `8k_stars_milky_way.jpg` desktop / `2k` mobile, anisotropy max. Orientation locale (ordre ZYX, y=1.2, z=45°) qui place le cœur galactique en haut à droite ; luminosité via `milkyMat.color.setScalar` (slider LUMINOSITÉ, défaut 5.0).
- Les deux couches vivent dans un `skyGroup` commun, indépendant du globe, dont `rotation.y` avance de `skySpeed·dt` dans la boucle (slider ROTATION, max 0.015 rad/s ≈ un tour en ~7 min). La rotation du groupe se compose par-dessus l'orientation locale de la skybox.

## Settings (`#controls-left`) & persistance

Sections du panneau de réglages (sliders construits par `makeVsSlider`, valeur persistée à chaque drag) :

| Section | Slider | Clé localStorage | Plage | Effet |
|---|---|---|---|---|
| GLOBE | ATMOSPHÈRE | `vs-atm` | 0–20 | `atmMultiplier` du halo |
| GLOBE | VITESSE | `vs-rot` | 0–0.002 | rotation du globe, **min = 0 = immobile** (remplace l'ancien bouton play/pause) |
| GALAXIE | ÉTOILES | `vs-stars` | 0–100000 | `drawRange` du starfield |
| GALAXIE | LUMINOSITÉ | `vs-milky` | 0–6 | brightness Voie lactée |
| GALAXIE | ROTATION | `vs-sky-rot` | 0–0.015 | vitesse de rotation du `skyGroup` |
| NUAGES | DENSITÉ | `vs-cloud-density` | 0–1 | `uCloudDensity` (nuages + ombres) / opacité JPG mobile |
| NUAGES | VITESSE | `vs-clouds` | 0–1 | vitesse du bruit (desktop) / dérive (mobile) |
| AFFICHAGE | EMPIRES (toggle) | `terr-visible` | 0/1 | visibilité de l'overlay territoires (+ URL `?empires=`) |

Autres clés persistées : `sun-angle` (+ `?sun=`), `tl-slider-center` (+ `?t=`), `active-filters` (+ `?filters=`). Clés legacy purgées au chargement : `tl-slider-left`, `tl-slider-right`, `tl-duration-index`, `rotation-paused` (migré vers `vs-rot = 0` si pause active, idem `?paused=1`).

Typo du panneau : `.ctrl-section-title` 0.7rem, `.ctrl-row-label` 0.6rem — identiques sur tous les écrans (pas d'override dans la media query mobile).

### Calibrage console

Objets exposés sur `window` pour ajuster les valeurs en live : `_renderer` (`toneMappingExposure`), `_earthMat` (`uCloudShadowStrength`, `uTwilight*`, `uGlintStrength`…), `_atmosphereMat`, `_cloudsMat`/`_cloudsMesh`, `_starsMat` (`uTwinkleAmp`, `uTwinkleOn`), `_milkyMat`, `_milkySky` (`rotation.y`), `_bloom` (`strength`/`radius`/`threshold`), `_spriteDim(valeur)`.

## Data — adding an event

Add an entry to the appropriate era file in `data/` (`01-prehistoire.json` … `13-xxie.json`, chosen by date — see `eras.json` for date ranges). Never hardcode event data in `index.html`; `loadData()` fetches and merges all era files plus `categories.json`/`eras.json`/`territories.json` at startup.

Event fields:

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Unique slug |
| `title` | string | Event name shown in the panel/tooltip |
| `year` | number | Start year (negative = BCE) |
| `yearEnd` | number | End year — used for fade-out timing (`getOpacity`); set equal to `year` for instantaneous events |
| `lat`, `lng` | number | Coordinates, converted to a 3D position via `latLngToVec3` |
| `cat` | string | Category id, must match an entry in `categories.json` |
| `region` | string | Display region name |
| `desc` | string | Description shown in the side panel |

## Categories (`data/categories.json`)

| id | label | color |
|---|---|---|
| `sc` | Science | `#5b9bd5` |
| `te` | Technologie | `#a8d46a` |
| `em` | Empires | `#c97a5b` |
| `po` | Politique | `#e08a5a` |
| `ph` | Philosophie | `#d4c06a` |
| `re` | Religion | `#9b7bc4` |
| `ar` | Art & Culture | `#7bc49b` |
| `gu` | Guerre | `#d46a6a` |
| `ex` | Exploration | `#6abed4` |

## Eras (`data/eras.json`)

`aube-humanite` (-300000 → -10000), `neolithique` (-10000 → -3500), `antiquite` (-3500 → -500), `periode-classique` (-500 → 500), `haut-moyen-age` (500 → 1000), `moyen-age` (1000 → 1400), `renaissance` (1400 → 1600), `ere-moderne` (1600 → 1800), `ere-contemporaine` (1800 → 1945), `monde-actuel` (1945 → 2100).

## Design system

```css
--ink:       #0d0c0a   /* background */
--parchment: #f5f0e8   /* light text */
--gold:      #c9a84c   /* accents */
```

Fonts: Playfair Display (titles), DM Mono, Cormorant Garamond (body/labels).

## Règles méthodologiques

- **JAMAIS de validation visuelle via Playwright ou captures headless** — faux positifs systématiques. Seule la validation visuelle de Kevin sur appareil réel fait foi.
- **Tout élément lumineux hors globe** (sprites, halo, UI 3D…) doit être évalué vis-à-vis du pipeline ACES/sRGB : `toneMapped = false` et/ou `texture.encoding = sRGBEncoding` selon le cas (voir "Pipeline couleur").

## Notes

- The core Three.js r128 is loaded from cdnjs — there is no local copy to update. Exception : les scripts `examples/js` du bloom, absents de cdnjs en r128, vivent dans `js/vendor/` (voir "Bloom").
- The slider's year mapping is non-linear (`BP` breakpoints in the timeline block) to give more resolution to recent history; `sliderToYear` is duplicated in both the timeline and sprites blocks — keep them in sync if you change the breakpoints.
- Mobile (`isMobile`, UA + largeur < 768px) dégrade plusieurs effets : nuages JPG au lieu du fBm, ombres de nuages coupées (`uCloudShadowStrength = 0`), Voie lactée 2k au lieu de 8k, pas de bloom (rendu direct, sprites non atténués — `applySpriteDim` n'est pas appelé).
