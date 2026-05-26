# Stereo Matching with Loopy Belief Propagation

Vision par ordinateur : estimation de carte de disparité par stereo matching avec l'algorithme Loopy Belief Propagation (LBP).

## Contenu

1. **Théorie LBP** : formules des messages, complexité, beliefs et labels MAP
2. **Modèle de Potts** : formule efficace réduisant la complexité de O(d²) à O(d)
3. **Implémentation** : calcul de la carte de disparité sur une paire stéréo
4. **Analyse** : rôle de `normalize_msg`, effet du paramètre λ sur les résultats

## Structure du projet

```
├── Stereo_LBP_Assignment.ipynb    # Notebook principal
├── images stéréo gauche/droite    # Images d'entrée
└── README.md
```

## Dépendances

```
numpy
matplotlib
```

Installation dans un venv :

```bash
python3 -m venv venv
source venv/bin/activate
pip install numpy matplotlib ipykernel
python -m ipykernel install --user --name venv --display-name "Python venv"
```

## Algorithme LBP pour le stereo matching

### Formulation générale

Le message que le nœud `p` envoie au voisin `q` à l'itération `t` :

```
m_t(p→q)(l_q) = min_{l_p} [ D_p(l_p) + V(l_p - l_q) + sum_{r ∈ N(p)\q} m_{t-1}(r→p)(l_p) ]
```

Le **belief** du pixel `q` à l'itération `t` :

```
b_q(l_q) = D_q(l_q) + sum_{r ∈ N(q)} m_t(r→q)(l_q)
```

Le label MAP : `l*_q = argmin_{l_q} b_q(l_q)`

### Formule efficace avec le modèle de Potts

Le modèle de Potts : `V(x) = 0` si `x = 0`, `λ` sinon.

Cela permet de simplifier le calcul du message :

```
m_t(p→q)(l_q) = min( h_p(l_q),  min_{l_p} h_p(l_p) + λ )
```

où `h_p(l_p) = D_p(l_p) + sum_{r ∈ N(p)\q} m_{t-1}(r→p)(l_p)`.

Le minimum global est calculé **une seule fois** (scalaire), puis appliqué vectoriellement → complexité **O(d)** par arête au lieu de O(d²).

## Paramètre λ — Guide de réglage

λ contrôle l'équilibre entre fidélité aux données et lissage spatial :

| Valeur | Effet | Énergie à convergence |
|--------|-------|-----------------------|
| λ = 0.1 | Quasi pur data term, carte très bruitée | ~500 (converge en ~5 iter) |
| λ = 1 | Données dominantes, bruit poivre-et-sel visible | ~195 500 (iter ~30) |
| **λ = 10** | **Meilleur compromis, carte cohérente** | **~343 000 (iter ~30)** |
| λ = 100 | Régularisation forte, détails fins perdus | ~1 300 000 (iter ~30) |
| λ = 1000+ | Sur-lissage, grandes zones uniformes, convergence lente | >> (pas convergé à 60 iter) |

**Valeur recommandée : λ = 10** — assez fort pour lisser le bruit, assez faible pour préserver les structures fines.

### Adapter à une autre paire stéréo

- Si les images ont beaucoup de textures et peu de zones uniformes → λ plus faible (~5)
- Si les images ont de grandes zones homogènes → λ plus fort (~20-50)
- Si la convergence est trop lente → réduire λ ou le nombre d'itérations

```

Pour le PDF, ouvrir le HTML dans un navigateur et `Cmd+P` → Enregistrer en PDF.
