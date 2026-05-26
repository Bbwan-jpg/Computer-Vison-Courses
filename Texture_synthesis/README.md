# Texture Synthesis & Inpainting

Vision par ordinateur : synthèse de texture par patches et inpainting (suppression d'objets dans une image).

## Contenu

- **Synthèse de texture** : génération d'une image à partir d'un échantillon de texture (briques, crackers, chocolat, gaze…) en utilisant un algorithme patch-based avec SSD.
- **Inpainting** : suppression de la Tour Montparnasse d'une photo de Paris en remplaçant la zone par du ciel et de la ville synthétisés.

## Structure du projet

```
├── Triva_Texture_Synthesis_4.ipynb   # Notebook principal
├── montparnasse.png                  # Image de Paris avec la tour
├── bricks.png                        # Texture briques
├── crackers.png                      # Texture crackers
├── chocolate.png                     # Texture chocolat
├── gauze2.png                        # Texture gaze
└── README.md
```

## Dépendances

```
numpy
matplotlib
imageio
scipy
```

Installation dans un venv :

```bash
python3 -m venv venv
source venv/bin/activate
pip install numpy matplotlib imageio scipy ipykernel
python -m ipykernel install --user --name venv --display-name "Python venv"
```

## Algorithme de synthèse de texture

1. Extraction de tous les patches possibles depuis la texture source
2. Placement d'un patch aléatoire en haut à gauche du canvas
3. Pour chaque position suivante (balayage ligne par ligne) :
   - Calcul du SSD masqué entre la fenêtre courante et tous les patches source
   - Sélection aléatoire parmi les meilleurs candidats (paramètre `percentage`)
   - Collage avec moyenne sur la zone d'overlap pour lisser les transitions

### Paramètres modifiables

| Paramètre | Description | Valeur typique |
|-----------|-------------|----------------|
| `patch_size` | Taille des patches carrés | 32-64 |
| `overlap` | Recouvrement entre patches adjacents | 10-15 |
| `percentage` | Fraction des meilleurs patches parmi lesquels on tire au hasard | 0.01-0.05 |

## Inpainting — Suppression de la Tour Montparnasse

L'inpainting utilise deux banques de patches séparées (ciel et ville) pour éviter que des patches de ciel n'apparaissent dans la zone ville et inversement.

### Paramètres clés

| Paramètre | Description | Comment l'ajuster |
|-----------|-------------|-------------------|
| `split_source` | Frontière pour la **construction** des banques de patches | Mettre assez bas (~245) pour que la banque ville ne contienne que de la vraie ville |
| `split_fill` | Frontière pour le **remplissage** | Plus haut (~225) pour reconstruire la ligne d'horizon |
| `blend_weight` | Netteté du blending sur l'overlap (ville) | 0.85 = net, 0.5 = flou |
| `blend_weight_sky` | Netteté du blending (ciel) | 0.5 = plus lissé, le ciel supporte bien le flou |

### Zones interdites (`valid_mask`)

Pour éviter la duplication d'éléments distinctifs (dôme des Invalides, grue, bâtiments sombres), on exclut ces zones des patches source :

```python
valid_mask[ymin:ymax, xmin:xmax] = False                                          # tour
valid_mask[max(0,ymin-40):min(H,ymax+20), max(0,xmin-60):min(W,xmax+60)] = False  # marge autour
valid_mask[150:345, 60:160] = False                                                # Invalides
valid_mask[250:300, 150:200] = False                                               # bâtiment sombre
valid_mask[310:345, 0:70] = False                                                  # coin bas-gauche
```

Pour adapter à une autre image, repérer les coordonnées des éléments à exclure avec :

```python
plt.imshow(image[y1:y2, x1:x2])
plt.title(f"y={y1}..{y2}, x={x1}..{x2}")
```

### Post-traitement du ciel

Un flou gaussien + fondu progressif aux bords élimine la ligne de transition visible entre le ciel synthétisé et le ciel original :

```python
from scipy.ndimage import gaussian_filter

sky_zone = result[0:225, x0:x1, :].astype(np.float32)
orig_zone = image[0:225, x0:x1, :].astype(np.float32)
for c in range(3):
    sky_zone[:, :, c] = gaussian_filter(sky_zone[:, :, c], sigma=3)

# Fondu progressif sur 15px aux bords
margin = 15
w_map = np.ones(x1 - x0, dtype=np.float32)
for d in range(margin):
    w = d / margin
    w_map[d] = w
    w_map[-(d+1)] = w
```

## Adapter à une autre image

1. **Définir le rectangle de l'objet à supprimer** : `(xmin, ymin, xmax, ymax)`
2. **Identifier la frontière ciel/ville** : afficher des bandes horizontales à différentes hauteurs pour trouver `split_source` et `split_fill`
3. **Repérer les éléments distinctifs à exclure** des patches source et ajouter les zones dans `valid_mask`
4. **Ajuster les paramètres de blending** : `blend_weight_sky` plus bas pour le ciel, `blend_weight` plus haut pour la ville

## Export

```bash
jupyter nbconvert --to html notebook.ipynb
```

Pour le PDF, ouvrir le HTML dans un navigateur et `Cmd+P` → Enregistrer en PDF.
