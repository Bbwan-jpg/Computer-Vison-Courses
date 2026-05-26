# Keypoint Detection & Matching

Vision par ordinateur : détection de points d'intérêt (Harris corner detector) et mise en correspondance de keypoints entre deux images.

## Contenu

1. **Traitement d'image de base** : conversion niveaux de gris, bruit gaussien, convolution, filtres de Sobel
2. **Détecteur de coins Harris** : implémentation from scratch (sans `cv2.harrisCorner`)
3. **Matching de keypoints** : 3 versions progressivement améliorées
4. **Challenge** : localisation d'un tableau de Van Gogh dans 8 images via propagation de matches

## Structure du projet

```
├── Triva_Keypoints.ipynb        # Notebook principal
├── mini_graffiti1.png           # Image test (téléchargée automatiquement)
├── asterix1.png                 # Image 1 pour le matching
├── asterix2.png                 # Image 2 pour le matching
├── vangogh0.png ... vangogh7.png  # Images du challenge
└── README.md
```

Les images sont téléchargées automatiquement depuis `imagine.enpc.fr` au lancement du notebook.

## Dépendances

```
numpy
matplotlib
opencv-python
```

Installation dans un venv :

```bash
python3 -m venv venv
source venv/bin/activate
pip install numpy matplotlib opencv-python ipykernel
python -m ipykernel install --user --name venv --display-name "Python venv"
```

## Algorithme de Harris

### Pipeline complet

1. **Gradients** : convolution avec les filtres de Sobel pour obtenir Ix et Iy
2. **Produits** : calcul de Ix², Iy², et Ix·Iy
3. **Lissage** : flou gaussien sur les trois images de produits (σ = 1.0 par défaut)
4. **Score Harris** : `det(Q) - 0.04 * trace(Q)²` où Q est la matrice de structure locale
5. **Maxima locaux** : sélection des maxima au-dessus d'un seuil

### Paramètres modifiables

| Paramètre | Description | Valeur typique |
|-----------|-------------|----------------|
| `sigma` | Écart-type du flou gaussien sur les produits de gradients | 1.0 |
| `threshold` | Seuil sur le score Harris pour garder un point | 0.5 |

```python
points = detect_harris_points(image, sigma=1.0, threshold=0.5)
```

Plus `sigma` est grand, moins de détails sont détectés mais les points sont plus stables. Un `threshold` plus élevé donne moins de points mais plus fiables.

## Matching de keypoints

### Descripteurs

Chaque keypoint est décrit par une fenêtre 7×7 de valeurs d'intensité autour du point. Simple mais efficace pour des images peu déformées.

### 3 versions de matching

| Fonction | Stratégie | Résultat typique |
|----------|-----------|-----------------|
| `match_V1` | Nearest neighbor global (SSD sur descripteurs) | Beaucoup de faux matches |
| `match_V2` | Nearest neighbor dans une région 40×40 autour du point | Moins de faux matches |
| `match_V3` | Mutual nearest neighbor (les deux points se choisissent mutuellement) | ~2 faux matches seulement |

```python
D1 = extract_descriptors(I1, P1)
D2 = extract_descriptors(I2, P2)
M = match_V3(P1, D1, P2, D2)
show_matches(I1, I2, M, P1, P2)
```

### Adapter à d'autres images

1. Charger les deux images et les convertir en niveaux de gris :
```python
img = plt.imread('mon_image.png')
I = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
```

2. Détecter les points Harris — ajuster `threshold` si trop/pas assez de points :
```python
P = detect_harris_points(I, sigma=1.0, threshold=0.5)
```

3. Pour des images très différentes en orientation ou échelle, `match_V2` peut être moins pertinent (la région 40×40 suppose peu de déplacement entre les images). Dans ce cas, utiliser `match_V1` + `match_V3` directement.

## Challenge Van Gogh

L'objectif est de localiser un tableau dans 8 photos différentes. La stratégie :

1. Les coins du tableau sont connus dans l'image 0 : `[[100,210], [394,424]]`
2. On propage la localisation d'une image à la suivante via `match_V3`
3. Pour chaque match, on estime la transformation affine des coins

Visualisation des résultats :
```python
show_vangogh([[[x_TL, y_TL], [x_BR, y_BR]]])
```
