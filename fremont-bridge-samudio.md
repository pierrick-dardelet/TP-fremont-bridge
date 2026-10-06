---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
  formats: md:myst
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
language_info:
  name: python
  nbconvert_exporter: python
  pygments_lexer: ipython3
nbhosting:
  title: "les v\xE9los sur le pont de Fremont"
---

# Les vélos sur le pont de Fremont

pour exécuter ce code localement sur votre ordi,
{download}`commencez par télécharger le zip<./ARTEFACTS-fremont-bridge.zip>`

+++

````{admonition} lien vers la version originale

Voir la version originale de ce code - par Jake Vanderplas - sur Youtube

<https://www.youtube.com/watch?v=_ZEWDGpM-vM&list=PLYCpMb24GpOC704uO9svUrihl-HY1tTJJ>
````

+++

On part des données publiques qui décrivent le trafic des vélos [sur
le pont de Fremont (à Portland - Oregon)](https://www.google.com/maps/place/Fremont+Bridge/@45.5166602,-122.7147124,12.31z/data=!4m5!3m4!1s0x0:0x9014fe26b76a82db!8m2!3d45.5379639!4d-122.6830729)

```{admonition} l'objectif du TP
l'idée centrale de ce TP est d'apprendre à exploiter un **index temporel** (`DatetimeIndex`) pandas :  
resampling, filtrage par date, extraction d'attributs (heure, jour de semaine), pivot...  
c'est ce qui permet ensuite de repérer des motifs (jours de semaine vs week-end) sans écrire de boucle
```

```{code-cell} ipython3
URL = "https://data.seattle.gov/api/views/65db-xm6k/rows.csv?accessType=DOWNLOAD"
```

```{code-cell} ipython3
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
```

```{code-cell} ipython3
# on a déja le fichier en local
local_file = "data/fremont.csv"
```

pour information, voici le code qu'on a utilisé pour aller chercher la donnée

```{code-cell} ipython3
from pathlib import Path

if Path(local_file).exists():
    print(f"le fichier {local_file} est déjà là")
else:
    print(f"allons chercher le fichier {local_file}")

    import requests
    req = requests.get(URL)
    print(req.status_code)

    with open(local_file, 'w') as writer:
        writer.write(req.text)
```

```{code-cell} ipython3
!head $local_file
```

## chargement

```{code-cell} ipython3
# version naïve
data = pd.read_csv(local_file)
data.shape
```

```{code-cell} ipython3
data.head()
```

## doublons

en fait ce qu'il se passe c'est que c'est un peu le bazar ce dataset, et que les données sont principalement présentes en deux exemplaires !

```{code-cell} ipython3
!grep '01/01/2014 12:00:00 AM' $local_file
```

**Exercice :** débarrassez-vous des lignes en double, et vérifiez la nouvelle taille du tableau.

<details>
<summary><b>Indice</b></summary>

il existe une méthode `DataFrame` dédiée à ça, qui peut s'appliquer "surplace" ou `inplace` c'est à dire en modifiant directement le dataframe sans en créer de nouveau.
</details>

## échauffement : le `DatetimeIndex`

avant d'aller plus loin, familiarisons-nous avec les objets temporels de pandas ; c'est l'outil qu'on va exploiter du début à la fin de ce TP

```{code-cell} ipython3
sample_dates = pd.to_datetime([
    "2024-01-15 08:00:00",
    "2024-01-20 17:30:00",
    "2024-02-03 12:00:00",
])
```

**Énoncé :**
1. Quel est le type de `sample_dates` ?
2. Que retournent `sample_dates.date`, `sample_dates.time` et `sample_dates.dayofweek` ?
3. Comment ne garder que les dates postérieures ou égales au 20 janvier 2024 ?
4. Comment ne garder que les dates de janvier 2024 ?

<details>
<summary><b>Indice 1</b></summary>

`type(sample_dates)` répond directement à la question 1
</details>

<details>
<summary><b>Indice 2</b></summary>

`.date` et `.time` séparent la partie calendaire de la partie horaire  
`.dayofweek` retourne un entier : 0 pour lundi, ..., 6 pour dimanche
</details>

<details>
<summary><b>Indice 3 et 4</b></summary>

un `DatetimeIndex` se compare directement à une chaine de caractères ou à un entier (`.year`, `.month`) comme s'il s'agissait de vraies dates  
ceci permet d'indexer un tableau avec un masque booléen, exactement comme pour n'importe quel autre `Series`/`DataFrame`
</details>

## parser les dates

```{code-cell} ipython3
# intéressant aussi, pour voir notamment les points manquants
data.info()
```

```{code-cell} ipython3
data.dtypes
```

bref le point ici c'est que les dates sont **des chaines et pas des dates**, et qu'elles ont un format particulier :
`mois/jour/année heure:minute:seconde AM/PM`

**Exercice :** transformez la colonne `Date` en un vrai `DatetimeIndex`, utilisez-le comme index du `DataFrame`, puis supprimez la colonne `Date` devenue inutile.

```{admonition} pourquoi ne pas utiliser read_csv(parse_dates=True) ?
on pourrait demander à `read_csv` de parser les dates directement  
mais c'est 1. peu fiable (pandas doit deviner le format) et 2. lent sur un gros fichier  
il est préférable de fournir soi-même le format exact
```

<details>
<summary><b>Indice</b></summary>

`pd.to_datetime(colonne, format=...)` accepte une chaine de format à la `strftime`  
`%m` mois, `%d` jour, `%Y` année, `%I` heure sur 12h, `%M` minutes, `%S` secondes, `%p` AM/PM
</details>

## renommons les colonnes

les noms de colonne ne sont pas pratiques du tout

```{code-cell} ipython3
data.columns = ['Total', 'West', 'East']
```

## données manquantes et extension types

de manière totalement optionnelle, mais on remarque que les nombres ont été convertis en flottants

et ça c'est parce qu'il y a eu quelques interruptions de service, apparemment, avec le système de récolte de l'information

**Exercice :**
1. Affichez les lignes qui contiennent au moins une valeur manquante (deux façons possibles : via `Total` seule, ou via toutes les colonnes).
2. On choisit de ne pas les supprimer, mais de les remettre sous forme d'entiers (avec des `NA`) plutôt que de flottants. Quelle méthode `DataFrame` permet ça ?

<details>
<summary><b>Indice 1</b></summary>

`data['Total'].isna()` retourne un masque booléen ; pour "au moins une colonne", pensez à `.any(axis=...)` sur `data.isna()`
</details>

<details>
<summary><b>Indice 2</b></summary>

cherchez du côté de `convert_dtypes`, qui a un paramètre pour forcer la conversion vers des entiers "nullable"
</details>

## à quoi ça ressemble

```{code-cell} ipython3
%matplotlib inline

sns.set(rc={'figure.figsize': (12, 4)})
```

```{code-cell} ipython3
# un premier jet, pas terrible du tout
data[['East', 'West']].plot()
```

## `resample()` : changer la fréquence temporelle

la courbe brute est illisible : une mesure par heure, sur plusieurs années  
`resample()` permet d'agréger un index temporel sur une fréquence plus grossière ('D' jour, 'W' semaine, 'M' mois, 'Y' année), un peu comme un `groupby` mais sur le temps

**Exercice :**
1. Tracez le nombre total de passages, agrégé par semaine.
2. Vérifiez que le nombre de lignes obtenu est bien environ `7 * 24` fois plus petit que le nombre de lignes original (une mesure par heure, sept jours par semaine).

<details>
<summary><b>Indice 1</b></summary>

`data.resample("1W")` retourne un objet intermédiaire ; il faut lui appliquer une agrégation (ici, on veut le total, donc une somme) avant de pouvoir tracer
</details>

<details>
<summary><b>Indice 2</b></summary>

comparez `data.shape[0]` et `data.resample("1W").sum().shape[0]`
</details>

## découpage temporel

juste pour être en phase (pouvoir vérifier nos résultats par rapport à ceux de la vidéo), on va s'arrêter à la fin de 2017

(un détail à noter aussi, les données de la vidéo ne contenaient pas la colonne 'total'...)

**Exercice :** ne gardez que les données dont l'année est inférieure ou égale à 2017.

<details>
<summary><b>Indice</b></summary>

grâce au `DatetimeIndex`, on peut comparer directement `data.index.year` à un entier, et s'en servir comme masque booléen sur `data`
</details>

````{admonition} quiz
ici on s'en sort bien car on coupe au début d'une année  
mais comment ferait-on pour couper au 12 février 2017 à 14h32:30 ?
````

<details>
<summary><b>Réponse</b></summary>

```python
data = data[data.index <= "2017-02-12 14:32:30"]
```

grâce au `DatetimeIndex`, la comparaison avec une simple chaine de caractères fonctionne directement, sans avoir besoin de la convertir explicitement
</details>

## évolution annuelle

on veut maintenant une courbe qui lisse les variations saisonnières : pour chaque jour, la somme des passages sur les 365 jours précédents

**Exercice :** construisez cette courbe, en veillant à ce que l'axe des Y démarre bien à 0 (sinon l'oeil se laisse facilement tromper par un effet de "zoom").

<details>
<summary><b>Indice 1 : l'enchainement</b></summary>

il faut enchainer trois opérations : un `resample` journalier avec une somme, puis un `rolling(365)` avec une nouvelle somme
</details>

<details>
<summary><b>Indice 2 : l'axe des Y</b></summary>

`.plot()` retourne l'objet `Axes` utilisé ; on peut ensuite appeler une méthode dessus pour fixer les bornes de l'axe Y
</details>

## profil journalier moyen

on veut maintenant la tendance moyenne du trafic au fil des heures d'une journée, tous jours confondus

**Exercice :** tracez ce profil moyen.

<details>
<summary><b>Indice</b></summary>

on ne veut pas grouper par jour, mais par heure de la journée, indépendamment du jour  
`data.index.time` donne justement l'heure seule, sans la date ; c'est cette valeur qu'il faut utiliser comme clé de `groupby`
</details>

## une courbe par jour : la `pivot_table`

mais pour y voir un peu mieux on veut afficher les jours individuellement les uns par rapport aux autres : une courbe par jour, avec en X l'heure de la journée et en Y le nombre de passages

pour ça on calcule une pivot table : une colonne par jour, une ligne par heure

**Exercice :** construisez ce tableau, qu'on appellera `pivoted`, à partir de la colonne `'Total'`.

<details>
<summary><b>Indice</b></summary>

`data.pivot_table(values, index=..., columns=...)` retourne un tableau où :
- `index` détermine les lignes → ici on veut une ligne par heure, donc `data.index.time`
- `columns` détermine les colonnes → ici on veut une colonne par jour, donc `data.index.date`
- `values` est la colonne à répartir dans le tableau → ici `'Total'`

ça pourrait aussi se faire à coups de `groupby`/`unstack`, mais `pivot_table` fait ça en un seul appel
</details>

**Exercice :** dessinez maintenant toutes ces courbes superposées, sans légende (il y en aurait bien trop), et en jouant sur la transparence pour distinguer les motifs qui se répètent souvent.

<details>
<summary><b>Indice</b></summary>

`plot()` accepte un paramètre `legend` et un paramètre `alpha` (essayez une valeur assez faible, comme `0.01`)
</details>

## classification des jours

ici il s'agit de classifier les jours en deux familles, qu'on voit très distinctement sur la figure précédente

on veut faire une ACP (analyse en composantes principales) sur un tableau qui aurait

- les 24 heures en colonnes
- les jours en lignes

et donc c'est presque exactement `pivoted`, sauf que c'est sa transposée !

**Exercice :** construisez `pca_input`, la transposée de `pivoted`, en remplaçant les valeurs manquantes par 0.

<details>
<summary><b>Indice</b></summary>

`.T` transpose ; `.fillna(0)` remplace les `NaN`  
pourquoi remplacer par 0 plutôt que supprimer ces jours ? car un jour incomplet (interruption de service) reste un jour qu'on veut classer
</details>

### ACP

```{code-cell} ipython3
from sklearn.decomposition import PCA
```

**Exercice :** réduisez `pca_input` à 2 dimensions avec `PCA`, et affichez le nuage de points obtenu.

<details>
<summary><b>Indice</b></summary>

on utilise `PCA` comme une boite noire ici : `PCA(n_components=2).fit_transform(...)`  
le résultat est un tableau numpy à deux colonnes, une par composante
</details>

### `GaussianMixture`

pour identifier ces deux groupes de façon automatique, on utilise un modèle de mélange gaussien, qui va estimer à quel groupe appartient chaque jour

```{code-cell} ipython3
from sklearn.mixture import GaussianMixture
```

**Exercice :** entrainez un `GaussianMixture` à 2 composantes sur `pca_input`, et récupérez un label (0 ou 1) par jour. Affichez à nouveau le nuage de points, coloré cette fois par label.

<details>
<summary><b>Indice</b></summary>

`GaussianMixture(n_components).fit_predict(data)` fait en un seul appel ce que ferait `.fit(data).predict(data)`  
pour la couleur, `plt.scatter` accepte un paramètre `c` qui peut être un tableau de labels, associé à une `cmap`
</details>

### comparons chaque famille aux profils journaliers

**Exercice :** redessinez séparément les profils journaliers (comme plus haut, avec `pivoted`) pour `label==0` puis pour `label==1`. Qu'observez-vous ?

<details>
<summary><b>Indice</b></summary>

`pivoted` a une colonne par jour, dans le même ordre que les lignes de `pca_input`, donc que `labels`  
`pivoted.loc[:, labels == 0]` sélectionne les colonnes correspondantes
</details>

### vérifions avec le jour de la semaine

essayons de vérifier que les deux clusters correspondent bien à l'intuition de départ, en coloriant cette fois le nuage de points par jour de semaine réel plutôt que par label

**Exercice :** à partir de `pivoted.columns` (qui ne sont pas encore un vrai `DatetimeIndex`), obtenez le jour de la semaine de chaque colonne (0 = lundi, ..., 6 = dimanche), et redessinez le nuage de points avec cette couleur.

<details>
<summary><b>Indice</b></summary>

on a vu dans l'échauffement comment reconstruire un `DatetimeIndex` à partir d'une liste de dates (`pd.DatetimeIndex(...)`), et comment en extraire le jour de la semaine
</details>

### les moutons noirs

on remarque dans le cluster "week-end" des jours d'une couleur qui jure, c'est-à-dire des jours de semaine classés comme des jours de week-end

**Exercice :** identifiez ces jours atypiques, puis affichez leurs profils.

<details>
<summary><b>Indice 1 : combiner deux conditions</b></summary>

on veut : classé dans le cluster 1, **et** dont le jour de semaine est un jour ouvré (`dayofweek < 5`)  
en pandas/numpy, la combinaison de deux masques booléens se fait avec `&` (pas `and`), chaque condition étant entre parenthèses
</details>

<details>
<summary><b>Indice 2 : retrouver les dates</b></summary>

`dates[odd_index]` retourne les dates correspondantes ; `pivoted[odd_dates]` sélectionne les colonnes de ces jours-là
</details>

## bonus : k-means, une troisième catégorie de jours ?

Le `GaussianMixture` à 2 composantes a séparé bien jours de semaine et week-ends ; on se demande si les données contiennent une structure qu'on n'a pas vue.

**Exercice :**
1. Appliquez `KMeans` avec `n_clusters=3` sur `pca_input`, puis affichez le nuage de points de la PCA coloré par label.
2. Croisez les labels avec le jour de la semaine. Un groupe correspond-il aux week-ends ?
3. Affichez les labels sur un calendrier : une bande par année, un trait par jour. À quoi correspondent les deux autres groupes ?
4. Superposez le profil horaire moyen de chaque cluster. Les deux groupes de jours ouvrés ont-ils la même forme ?

<details>
<summary><b>Indice 1 : KMeans</b></summary>

`KMeans(n_clusters=3, n_init=10).fit_predict(...)` s'utilise comme `GaussianMixture`  
l'ordre des labels change d'une exécution à l'autre : ne vous fiez pas aux couleurs d'une exécution précédente
</details>

<details>
<summary><b>Indice 2 : labels et jour de la semaine</b></summary>

`pca_input.index` contient des objets `date` : reconstruisez un `DatetimeIndex` avec `pd.DatetimeIndex(...)` pour accéder à `.dayofweek`  
`pd.crosstab(labels, dayofweek)` croise les deux
</details>

<details>
<summary><b>Indice 3 : la bande calendaire</b></summary>

`plt.subplots(n, 1, sharex=True)` crée `n` graphiques alignés, un par année  
pour chaque année, sélectionnez les jours avec un masque sur `dates.year`, puis utilisez `dayofyear` en abscisse et `marker='|'` pour un trait par jour  
fixez `vmin` et `vmax` dans `scatter` pour que les couleurs soient identiques d'une année à l'autre
</details>

<details>
<summary><b>Indice 4 : profils moyens</b></summary>

`pca_input[labels == k].mean()` donne un profil de 24 valeurs, prêt pour `.plot()`  
comparez l'amplitude et la forme : k-means sépare-t-il des types de jours ou des niveaux de trafic ?
</details>