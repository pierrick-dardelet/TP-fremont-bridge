---
jupytext:
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  name: python3
  display_name: Python 3 (ipykernel)
  language: python
language_info:
  name: python
  pygments_lexer: ipython3
  nbconvert_exporter: python
---

# n-body problem

+++

pour faire cette activité sur votre ordi localement, {download}`commencez par télécharger le zip<ARTEFACTS-n-body.zip>`

dans ce TP on vous invite à écrire un simulateur de la trajectoire de n corps qui interagissent entre eux au travers de leurs masses, pour produire des sorties de ce genre

```{image} media/init3-1.png
:align: center
:width: 600px
```

+++

on suppose:

- on se place dans un monde en 2 dimensions
- on fixe au départ le nombre de corps N
- chacun est décrit par une masse constante
- et au début du monde chacun possède une position et une vitesse

```{admonition} la 3D
en option on vous proposera, une fois votre code fonctionnel en 2D, de passer à la 3D  
ça peut valoir le coup d'anticiper ça dès le premier jet, si vous vous sentez de le faire comme ça
```

```{admonition} l'objectif du TP
l'objectif principal est de **ne pas écrire de boucle `for` sur les corps** pour calculer les forces  
tout doit se faire avec des opérations sur des tableaux, en s'appuyant sur le **broadcasting** de numpy
```

+++

## imports

on pourra utiliser le mode `ipympl` de `matplotlib`

```{code-cell} ipython3
import numpy as np
import matplotlib.pyplot as plt

%matplotlib ipympl
```

## échauffement : le broadcasting

avant de commencer, assurez-vous d'avoir bien en tête les règles du broadcasting

```{code-cell} ipython3
a = np.array([1, 2, 3])
```

**Énoncé :**
1. Quelle est la forme de `a[:, None]` ? et de `a[None, :]` ?
2. Quelle sera la forme de `a[:, None] - a[None, :]` ? Que contient l'élément `[i, j]` ?
3. Même question avec un tableau `b` de forme `(2, 3)` et l'expression `b[:, None, :] - b[:, :, None]`

<details>
<summary><b>Indice : règles du broadcasting</b></summary>

numpy aligne les formes **par la droite** ; deux dimensions sont compatibles si elles sont égales ou si l'une des deux vaut 1  
une dimension de taille 1 est alors "répétée" autant de fois que nécessaire
</details>

<details>
<summary><b>Correction</b></summary>

```python
a[:, None].shape                 # (3, 1)
a[None, :].shape                 # (1, 3)

d = a[:, None] - a[None, :]      # (3, 1) - (1, 3) -> (3, 3)
# d[i, j] = a[i] - a[j]

b = np.arange(6).reshape(2, 3)
e = b[:, None, :] - b[:, :, None]   # (2, 1, 3) - (2, 3, 1) -> (2, 3, 3)
# e[k, i, j] = b[k, j] - b[k, i]
```
</details>

## initialisation aléatoire

en fixant arbitrairement des limites dans l'espace des positions, des vitesses et des masses, la fonction `init_problem()` tire au hasard une configuration de départ pour la simulation

```{code-cell} ipython3
# les bornes pour le tirage au sort initial
mass_max = 3.
    
x_min, x_max = -10., 10.
y_min, y_max = -10., 10.

speed_max = 1.
```

```{code-cell} ipython3
:tags: [level_basic]

# votre code

def init_problem(N):
    """
    retourne un tuple masses, positions, speeds
    de formes resp.   (N,)    (2, N)     (2, N)
    tiré au sort dans les bornes définies ci-dessus
    """
    return None, None, None
```

<details>
<summary><b>Indice 1 : les masses</b></summary>

attention à la borne inférieure : une masse nulle poserait problème plus tard (on divisera par la masse)
</details>

<details>
<summary><b>Indice 2 : les positions</b></summary>

`np.random.uniform(low, high, size)` accepte des `low` et `high` qui sont eux-mêmes des tableaux, tant qu'ils sont *broadcastables* avec `size`  
quelle forme donner à `low` et `high` pour tirer directement un tableau `(2, N)` ?
</details>

<details>
<summary><b>Correction</b></summary>

```python
def init_problem(N):
    masses = np.random.uniform(0.1, mass_max, N)

    low = np.array([[x_min], [y_min]])      # (2, 1)
    high = np.array([[x_max], [y_max]])     # (2, 1)
    positions = np.random.uniform(low, high, (2, N))

    speeds = np.random.uniform(-speed_max, speed_max, (2, N))
    return masses, positions, speeds
```
</details>

```{code-cell} ipython3
:tags: [level_intermediate]

# pour tester

# normalement vous devez pouvoir faire ceci

masses, positions, speeds = init_problem(10)

# et ceci devrait afficher OK
try:
    assert masses.shape == (10,) and positions.shape == speeds.shape == (2, 10)
    print("OK")
except:
    print("KO")
```

## initialisation reproductible

par commodité on vous donne la fonction suivante qui crée 3 objets:

- le premier - pensez au soleil - de masse 3, an centre de la figure, de vitesse nulle
- et deux objets de masse 1, disposés symétriquement autour du soleil  
  - position initiale (5, 1) et vitesse initiale (-1, 0)
  - symétrique en     (-5, -1) et vitesse initiale (1, 0)

```{code-cell} ipython3
# for your convenience

def init3():
    # first element is sun-like: heavy, at the center, and no speed
    masses = np.array([3, 1, 1], dtype=float)
    positions = np.array([
        [0, 5, -5], 
        [0, 1, -1]], dtype=float)
    speeds = np.array([
        [0, -1, 1], 
        [0, 0, 0]], dtype=float)
    return masses, positions, speeds
```

## les forces

à présent, on va écrire une fonction qui va calculer les influences de toutes les particules entre elles, suivant la loi de Newton


$$
\vec{F}_i = \sum_{\substack{j=1 \\ j \neq i}}^N 
   G \, m_i m_j \, \frac{\vec{r}_j - \vec{r}_i}{\lvert \vec{r}_j - \vec{r}_i \rvert^3}
$$

pour cela on se propose d'écrire la fonction suivante

```{code-cell} ipython3
:tags: [level_basic]

# votre code

def forces(masses, positions, G=1.0):
    """
    returns an array of shape (2,  N)
    that contains the force felt by each mass from all the others
    G is the Newton constant and by default is set to 1 
    (we have abstract units anyway)
    """
    pass
```

```{admonition} sans boucle !
on vous suggère de procéder par étapes, chacune produisant un tableau dont vous pouvez vérifier la forme  
1. tous les vecteurs $\vec{r}_j - \vec{r}_i$ pour tous les couples $(i, j)$  
2. toutes les distances $\lvert \vec{r}_j - \vec{r}_i \rvert$  
3. le coefficient $G \, m_i m_j / \lvert \dots \rvert^3$ pour tous les couples  
4. la somme sur $j$
```

<details>
<summary><b>Indice 1 : les vecteurs entre particules</b></summary>

on veut un tableau `diff` de forme `(2, N, N)` tel que `diff[:, i, j]` soit le vecteur $\vec{r}_j - \vec{r}_i$  
repensez à l'échauffement : à partir de `positions` de forme `(2, N)`, comment obtenir par broadcasting une forme `(2, N, N)` ?
</details>

<details>
<summary><b>Indice 2 : les distances</b></summary>

regardez du côté de `np.linalg.norm` et de son paramètre `axis`  
sur quel axe faut-il calculer la norme, et quelle forme obtient-on ?
</details>

<details>
<summary><b>Indice 3 : le cas i = j</b></summary>

que vaut `dist[i, i]` ? que se passe-t-il quand on divise par cette valeur ?  
une particule n'agit pas sur elle-même : trouvez une astuce pour que sa contribution soit nulle  
(pensez à `np.fill_diagonal` et à `np.inf`)
</details>

<details>
<summary><b>Indice 4 : le produit des masses</b></summary>

même idée que dans l'échauffement : comment obtenir un tableau `(N, N)` dont l'élément `[i, j]` vaut `masses[i] * masses[j]` ?
</details>

<details>
<summary><b>Indice 5 : la somme</b></summary>

`coeff` est de forme `(N, N)` et `diff` de forme `(2, N, N)` : leur produit est-il possible ?  
ensuite, sur quel axe faut-il sommer pour obtenir la force sur la particule `i` ?
</details>

<details>
<summary><b>Correction</b></summary>

```python
def forces(masses, positions, G=1.0):
    # diff[:, i, j] = r_j - r_i
    diff = positions[:, None, :] - positions[:, :, None]      # (dim, N, N)
    dist = np.linalg.norm(diff, axis=0)                       # (N, N)
    np.fill_diagonal(dist, np.inf)
    coeff = G * masses[:, None] * masses[None, :] / dist**3   # (N, N)
    return np.sum(coeff * diff, axis=2)                       # (dim, N)
```

le produit `coeff * diff` fonctionne car `(N, N)` est vu comme `(1, N, N)`  
la somme sur `axis=2` additionne bien les contributions de tous les $j$ pour chaque $i$
</details>

```{code-cell} ipython3
:tags: [level_intermediate]

# pour tester, voici les valeurs attendues avec la config prédéfinie

masses, positions, speeds = init3()

f = forces(masses, positions)

# should be true
# np.all(np.isclose(f, np.array([
#     [ 0.        , -0.12257258,  0.12257258],
#     [ 0.        , -0.02451452,  0.02451452]])))
```

## le simulateur

à présent il nous reste à utiliser cette brique de base pour "faire avancer" le modèle depuis son état initial et sur un nombre fixe d'itérations

cela pourrait se passer dans une fonction qui ressemblerait à ceci

```{code-cell} ipython3
:tags: [level_basic]

# votre code

def simulate(masses, positions, speeds, dt=0.1, nb_steps=100):
    """
    should return the positions across time
    so an array of shape (nb_steps, 2, N)
    optional dt is the time step
    """
    pass
```

<details>
<summary><b>Indice 1 : le stockage</b></summary>

allouez dès le départ le tableau résultat, de forme `(nb_steps, dim, N)`  
la dimension `dim` peut se lire sur `positions`, ce qui rend votre code indépendant de la 2D ou de la 3D
</details>

<details>
<summary><b>Indice 2 : de la force à l'accélération</b></summary>

`forces()` retourne des forces, pas des accélérations : $a = F / m$  
`forces` est de forme `(dim, N)` et `masses` de forme `(N,)` : est-ce que la division est possible directement ?
</details>

<details>
<summary><b>Indice 3 : la mise à jour</b></summary>

à chaque pas : accélération, puis vitesses, puis positions (c'est l'approche 1 décrite plus bas)  
$s = s + a.dt$ puis $p = p + s.dt$
</details>

<details>
<summary><b>Correction</b></summary>

```python
def simulate(masses, positions, speeds, dt=0.1, nb_steps=100):
    dim, N = positions.shape
    trajectory = np.empty((nb_steps, dim, N))
    trajectory[0] = positions
    for step in range(1, nb_steps):
        acc = forces(masses, positions) / masses
        speeds = speeds + acc * dt
        positions = positions + speeds * dt
        trajectory[step] = positions
    return trajectory
```

la boucle porte sur le temps, ce qui est inévitable ; il n'y en a aucune sur les corps
</details>

```{code-cell} ipython3
:tags: [level_intermediate]

# pour tester

SMALL_STEPS = 4

s = simulate(masses, positions, speeds, nb_steps=SMALL_STEPS)

try:
    if s.shape == (SMALL_STEPS, 2, 3):
        print("shape OK")
except Exception as exc:
    print(f"OOPS {type(exc)} {exc}")
```

```{code-cell} ipython3
:tags: [level_intermediate]

# pour tester: should be true

# first step
# positions1 = s[1]

# np.all(np.isclose(positions1, np.array([
#     [ 0.        ,  4.89877427, -4.89877427],
#     [ 0.        ,  0.99975485, -0.99975485]
# ])))
```

## dessiner

ne reste plus qu'à dessiner; quelques indices potentiels:

- 1. chaque corps a une couleur; l'appelant peut vous passer un jeu de couleurs, sinon en tirer un au hasard
- 2.a pour l'épaisseur de chaque point, on peut imaginer utiliser la masse de l'objet  
  2.b ou peut-être aussi, à tester, la vitesse de l'objet (plus c'est lent et plus on l'affiche en gros ?)

```{admonition} masses et vitesses ?
j'ai choisi de repasser à `draw()` le tableau des masses à cause de 2.a;  
si j'avais voulu implémenter 2.b il faudrait tripoter un peu plus nos interfaces - car en l'état on n'a pas accès aux vitesses pendant la simulation - mais n'hésitez pas à le faire si nécessaire..
```

```{code-cell} ipython3
:tags: [level_basic]

# votre code

def draw(simulation, masses, colors=None, scale=10.):
    """
    takes as input the result of simulate() above,
    and draws the nb_steps positions of each of the N bodies
    ideally it should return a matplotlib Axes object

    one can provide a collection of N colors to use for each body
    if not provided this is randomized

    also the optional scale parameter is used as a constant
    multiplier to obtain the final size of each dot on the figure
    """
    pass
```

<details>
<summary><b>Indice 1 : extraire un corps</b></summary>

que représente `simulation[:, :, i]` ? quelle est sa forme ?  
et sa transposée, `simulation[:, :, i].T` ?
</details>

<details>
<summary><b>Indice 2 : les arguments de scatter</b></summary>

`ax.scatter(x, y, color=..., s=...)` : `s` est la taille des points  
en Python, `f(*t)` passe les éléments de `t` comme arguments séparés ; avec la bonne forme, cela permet d'écrire un seul appel valable en 2D comme en 3D
</details>

<details>
<summary><b>Correction</b></summary>

```python
def draw(simulation, masses, colors=None, scale=10.):
    nb_steps, dim, N = simulation.shape
    if colors is None:
        colors = np.random.rand(N, 3)

    fig = plt.figure()
    ax = fig.add_subplot(projection='3d' if dim == 3 else None)
    for i in range(N):
        ax.scatter(*simulation[:, :, i].T, color=colors[i], s=masses[i] * scale)
    return ax
```
</details>

## un jeu de couleurs

```{code-cell} ipython3
# for convenience

colors3 = np.array([
    [32, 32, 32],
    (228, 90, 146),
    (111, 0, 255),
]) / 255
```

## on assemble le tout

pour commencer et tester, on se met dans l'état initial reproductible

```{code-cell} ipython3
:tags: [level_intermediate]

# décommentez ceci pour tester votre code

# masses, positions, speeds = init3()
# draw(simulate(masses, positions, speeds), masses, colors3)
```

et avec ces données vous devriez obtenir plus ou moins une sortie de ce genre  
mais [voyez aussi la discussion ci-dessous sur les diverses stratégies possibles](label-n-body-strategies)
```{image} media/init3-1.png
```

+++

`````{grid} 2 2 2 2 
````{card}
après vous avez le droit de vous enhardir avec des scénarii plus compliqués
par exemple avec ce code

```python
m5, p5, s5 = init_problem(5)
sim5 = simulate(m5, p5, s5, nb_steps=1000)
draw(sim5, m5, scale=3);
plt.savefig("random5.png")
```
````
````{card}
j'ai pu obtenir ceci
```{image} media/random5.png
```
````
`````

+++

***
***
***

+++

## partie optionnelle

+++

### option 1: la 3D

modifiez votre code pour passer à une simulation en 3D

<details>
<summary><b>Indice</b></summary>

si vous avez écrit `forces`, `simulate` et `draw` sans jamais supposer `dim == 2`, il n'y a presque rien à changer  
il suffit de fournir des `positions` et `speeds` de forme `(3, N)`  
que doit-on modifier dans `init_problem` pour tirer au sort une troisième coordonnée ?
</details>

<details>
<summary><b>Correction</b></summary>

```python
def init3_3d():
    masses = np.array([5, 1, 1], dtype=float)
    positions = np.array([
        [0, 10, 0],
        [0, 0, 10],
        [0, 0, 0]], dtype=float)
    speeds = np.array([
        [0, -0.5, 0],
        [0, 0, -0.5],
        [0, 0, 0]], dtype=float)
    return masses, positions, speeds

masses, positions, speeds = init3_3d()
draw(simulate(masses, positions, speeds, nb_steps=1000), masses)
```

pour le tirage aléatoire, on peut ajouter un paramètre `dim` et construire `low` et `high` avec une forme `(dim, 1)`
</details>

+++

### option 2: un rendu plus interactif

le rendu sous forme de multiples scatter plots donne une idée du résultat mais c'est très améliorable  
voyez un peu si vous arrivez à produire un outil un peu plus convivial pour explorer les résultats de manière interactive; avec genre

- une animation qui affiche les points au fur et à mesure du temps
- qu'on peut controler un peu comme une vidéo avec pause / backward / forward
- l'option de laisser la trace du passé
- et si vous avez un code 3d, la possibilité de changer le point de vue de la caméra sur le monde
- etc etc...

voici une possibilité avec matplotlib; mais cela dit ne vous sentez pas obligé de rester dans Jupyter Lab ou matplotlib, il y a plein de technos rigolotes qui savent se décliner sur le web, vous avez l'embarras du choix...

```{code-cell} ipython3
:tags: [prune-remove-input, remove-input]

# prune-remove-input

# credit: Damien Corral
# with good old matplotlib FuncAnimation

from matplotlib.animation import FuncAnimation
from IPython.display import HTML

def animate(simulation, masses, colors=None, scale=5., interval=50):
    nb_steps, _, N = simulation.shape
    colors = (colors if colors is not None
              else np.random.uniform(0.3, 1., size=(N, 3)))

    fig, ax = plt.subplots()
    ax.set_title(f"we have {N} bodies over {nb_steps} steps")

    ax.set_xlim(simulation[:, 0].min() - 1, simulation[:, 0].max() + 1)
    ax.set_ylim(simulation[:, 1].min() - 1, simulation[:, 1].max() + 1)

    scat = ax.scatter(np.zeros(N), np.zeros(N), c=colors, s=(masses*scale)**2)

    def init():
        scat.set_offsets(np.zeros((nb_steps, N)))
        return scat

    def update(step):
        x, y = simulation[step]
        scat.set_offsets(np.c_[x, y])
        return scat

    animation = FuncAnimation(
        fig, update, frames=nb_steps,
        init_func=init, blit=True, interval=interval
    )
    plt.close()
    return animation

def animate_from_file(filename):
    simulation = np.loadtxt(filename).reshape((100, 2, 3))
    animation = animate(simulation, masses, colors=colors3)
    return HTML(animation.to_jshtml())

animate_from_file("data/init3-simu-1.txt")
```

(label-n-body-strategies)=
## plusieurs stratégies

Pour les curieux, vous avez sans doute observé qu'il y a plusieurs façons possibles d'écrire la fonction `simulate()`

dans ce qui suit, on note l'accélération $a$, la vitesse $s$ et la position $p$

````{admonition} Approche 1
:class: note
dans l'implémentation qui a servi à calculer l'illustration ci-dessus, on a écrit principalement ceci:
- on calcule l'accélération
- ce qui permet d'extrapoler les vitesses  
  $s = s + a.dt$
- et ensuite d'extrapoler les positions  
  $p = p + s.dt$
```{admonition} 1bis
:class: tip
une variante consiste à intervertir les deux, en arguant du fait que la vitesse à l'instant $t$ agit sur la position à l'instant $t$
- on extrapole d'abord les positions
- puis seulement on calcule l'accélération
- enfin on extrapole les vitesses
```
````
````{admonition} Approche 2
:class: tip
dans cette approche plus fine, on utiliserait deux versions de l'accélération (l'instant présent et l'instant suivant), et un dévelopmment du second ordre, ce qui conduirait à
- calculer la position comme $p = p + s.dt + \frac{a}{2}.dt^2$
- calculer les accélérations $a_+$ sur la base de cette nouvelle position
- estimer l'accélération sur l'intervalle comme la demie-somme entre les deux accélérations
  $a_m = (a+a_+)/2$
- mettre à jour les vitesses
  $s = s + a_m.dt$
- ranger $a_+$ dans $a$ pour le prochain instant
````

+++

Bref, vous voyez qu'il y a énormément de liberté sur la façon de s'y prendre  
Ce qui peut expliquer pourquoi vous n'obtenez pas la même chose que les illustrations avec pourtant les mêmes données initiales

D'autant que, c'est bien connu, ce problème des n-corps est l'exemple le plus célèbre de problème instable, et donc la moindre divergence entre deux méthodes de calcul peut entrainer de très sérieux écarts sur les résultats obtenus

+++

Voici d'ailleurs les résultats obtenus avec ces deux approches alternatives, et vous pouvez constater qu'effectivement les résultats sont tous très différents !

+++

### approche 1bis

+++

````{grid} 2 2 2 2 
```{image} media/init3-1bis.png
```
```{code-cell} python
:tags: [remove-input]
animate_from_file("data/init3-simu-1bis.txt")
```
````

+++

### approche 2

+++

````{grid} 2 2 2 2 
```{image} media/init3-2.png
```
```{code-cell} python
:tags: [remove-input]
animate_from_file("data/init3-simu-2.txt")
```
````