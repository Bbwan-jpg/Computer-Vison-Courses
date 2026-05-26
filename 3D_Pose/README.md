# 3D Pose Computation

Vision par ordinateur : calcul de pose 3D par homographie, réalité augmentée et tracking 3D.

## Contenu

1. **Prise en main** : chargement et visualisation des images
2. **Calcul d'homographie** : estimation de H par SVD à partir de correspondances 2D-2D
3. **Réalité augmentée** : dessin d'une boîte 3D sur la couverture du livre (Astérix) via la décomposition de H
4. **3D tracking** : propagation de la pose sur une deuxième image via composition d'homographies (ORB + matching)

## Structure du projet

```
├── 3D_Pose.ipynb      # Notebook principal
├── Hsterix1.png             # Image 1 (couverture Astérix visible)
├── Hsterix2.png             # Image 2 (même livre, point de vue différent)
├── Hsterix1_box.png         # Image de référence avec la boîte attendue
└── README.md
```

Les images sont téléchargeables depuis `imagine.enpc.fr` :

```python
!wget "http://imagine.enpc.fr/~lepetitv/triva/Hsterix1.png"
!wget "http://imagine.enpc.fr/~lepetitv/triva/Hsterix2.png"
!wget "http://imagine.enpc.fr/~lepetitv/triva/Hsterix1_box.png"
```

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

## Algorithme de calcul d'homographie

L'homographie H est calculée par SVD à partir de N correspondances entre points du livre (en cm) et pixels dans l'image.

Chaque correspondance (x,y) → (u,v) donne 2 lignes dans la matrice M de taille 2N×9. H est le vecteur singulier associé à la plus petite valeur singulière de M.

```python
UV_image = [[50,258], [213,52], [580,107], [554,412]]  # pixels dans Hsterix1
UV_livre  = [[0,0], [22.5,0], [22.5,29.5], [0,29.5]]  # coins du livre en cm

H = compute_homography(UV_livre, UV_image)
```

### Adapter à d'autres images

Pour utiliser une autre image, il faut mesurer les reprojections des 4 coins de l'objet dans l'image (en pixels) et connaître ses dimensions réelles. Modifier :

```python
UV_image = [[u1,v1], [u2,v2], [u3,v3], [u4,v4]]  # à mesurer dans la nouvelle image
UV_objet = [[0,0], [L,0], [L,H], [0,H]]            # dimensions réelles en cm
```

## Réalité augmentée — Dessin de la boîte 3D

La matrice intrinsèque K utilisée :

```python
K = np.array([[1000, 0, 320],
              [   0, 1000, 240],
              [   0,    0,   1]], dtype=float)
```

La pose se récupère depuis H via `H = K @ [r1 | r2 | t]` :

```python
KinvH = np.linalg.inv(K) @ H
lam = np.linalg.norm(KinvH[:, 0])
r1 = KinvH[:, 0] / lam
r2 = KinvH[:, 1] / lam
t  = KinvH[:, 2] / lam
```

H étant défini à un facteur de signe près, le code gère les deux cas (H et -H) pour que la boîte soit toujours au-dessus du livre.

### Paramètre modifiable

| Paramètre | Description | Valeur typique |
|-----------|-------------|----------------|
| `height` | Hauteur de la boîte 3D au-dessus de la couverture (cm) | 5.0 |

```python
draw_box(I1, H, K, height=5.0)
```

## 3D Tracking — Propagation sur Hsterix2

Pour dessiner la boîte sur la deuxième image sans re-mesurer les coins :

1. Détection de keypoints ORB sur I1 et I2
2. Matching des descripteurs binaires (Brute Force + ratio test)
3. Calcul de l'homographie H12 entre I1 et I2
4. Composition : H2 = H12 @ H1

```python
detector = cv2.ORB_create(2000)
kp1, des1 = detector.detectAndCompute(I1, None)
kp2, des2 = detector.detectAndCompute(I2, None)
```

ORB est utilisé ici plutôt que Harris car il fournit directement des descripteurs binaires rapides à comparer, adaptés au matching entre deux points de vue différents.
