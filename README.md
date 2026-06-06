# Globe Chronos

Globe terrestre 3D interactif construit avec Three.js — première étape du projet Globe Chronos.

## Rendu

- Globe terrestre avec shader jour/nuit (blend smooth sur le terminateur)
- Couche nuages séparée (rotation légèrement plus rapide que la Terre)
- Halo atmosphérique bleu (double couche : BackSide outer + FrontSide inner rim)
- Fond étoilé généré procéduralement (4 500 points)
- Auto-rotation lente, drag souris/tactile avec inertie

## Stack

- [Three.js r128](https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js) via cdnjs
- Vanilla JS / HTML5 / CSS3 — zéro dépendance npm
- Google Fonts : Playfair Display, Cormorant Garamond, DM Mono

## Textures

Chargées depuis [Solar System Scope](https://www.solarsystemscope.com/textures/) :

| Fichier | Usage |
|---|---|
| `2k_earth_daymap.jpg` | Surface terrestre (côté jour) |
| `2k_earth_nightmap.jpg` | Lumières des villes (côté nuit) |
| `2k_earth_clouds.jpg` | Couche nuages (AdditiveBlending) |

> **Note CORS** : si les textures ne se chargent pas depuis `file://`, téléchargez-les localement et mettez à jour les chemins dans `index.html` (variable `BASE`), ou servez le projet avec un serveur local (`npx serve .` ou équivalent).

## Lancer le projet

```bash
# Option 1 — ouverture directe (peut bloquer les textures selon le navigateur)
open index.html

# Option 2 — serveur local recommandé
npx serve .
# ou
python -m http.server 8080
```

## Design system

```css
--ink:       #0d0c0a   /* fond spatial */
--parchment: #f5f0e8   /* textes clairs */
--gold:      #c9a84c   /* accents dorés */
```

## Prochaines étapes

- Timeline historique interactive
- Points de civilisation géolocalisés
- Panel latéral de détail
- Filtres par époque / thème
