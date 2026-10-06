---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
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

```{code-cell} ipython3
URL = "https://data.seattle.gov/api/views/65db-xm6k/rows.csv?accessType=DOWNLOAD"
```

```{code-cell} ipython3
import pandas as pd
```

```{code-cell} ipython3
# on a déja le fichier en local
local_file = "data/fremont.csv"
```

pour information, voici le code qu'on a utilisé pour aller chercher la donnée

```{code-cell} ipython3
# ceci nécessite alors
# %pip install requests
```

```{code-cell} ipython3
from pathlib import Path

if Path(local_file).exists():
    print(f"le fichier {local_file} est déjà là")
else:
    print(f"allons chercher le fichier {local_file}")

    import requests
    req = requests.get(URL)

    # doit afficher 200
    print(req.status_code)

    # on sauve tel quel dans le fichier local
    with open(local_file, 'w') as writer:
        writer.write(req.text)
```

```{code-cell} ipython3
!head $local_file
```

## chargement

```{code-cell} ipython3
%pip install seaborn
```

```{code-cell} ipython3
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
```

```{code-cell} ipython3
# version naïve
data = pd.read_csv(local_file); data.shape
```

```{code-cell} ipython3
data.head()
```

## doublons

+++

en fait ce qu'il se passe c'est que c'est un peu le bazar ce dataset, et que les données sont principalement présentes en deux exemplaires !

```{code-cell} ipython3
!grep '01/01/2014 12:00:00 AM' $local_file
```

du coup on nettoie

```{code-cell} ipython3
data.drop_duplicates(inplace=True)
```

```{code-cell} ipython3
data.shape
```

## parser les dates

```{code-cell} ipython3
# intéressant aussi, pour voir notamment les points manquants
data.info();
```

```{code-cell} ipython3
# ou tout simplement
data.dtypes
```

bref le point ici c'est que les dates sont **des chaines et pas des dates**

```{code-cell} ipython3
# la version lente

# avec cette forme, on demanderait à read_csv:
# de mettre la date comme index,
# et de parser les dates
# mais on ne va pas le faire car
# 1. c'est peu sûr
# 2. c'est trop lent

# data = pd.read_csv(local_file, index_col='Date', parse_dates=True); data.head()
```

```{code-cell} ipython3
# une meilleure idée, pour améliorer ces deux points d'un coup
# est de fournir nous-même le format des dates

data.index = pd.to_datetime(data.Date, format="%m/%d/%Y %I:%M:%S %p")
del data['Date']
data.head()
```

## renommons les colonnes

```{code-cell} ipython3
:lines_to_next_cell: 2

# les noms de colonne ne sont pas pratiques du tout
data.columns = ['Total', 'West', 'East']
```

## données manquantes et extension types

+++

de manière totalement optionnelle, mais on remarque que les nombres ont été convertis en flottants

et ça c'est parce qu'il y a eu quelques interruptions de service, apparemment, avec le système de récolte de l'information

```{code-cell} ipython3
data[data['Total'].isna()]
```

```{code-cell} ipython3
# ou encore si on préfère

data[data.isna().any(axis=1)].shape
```

on pourrait nettoyer, mais ici on va choisir d'ignorer ces données manquantes; à la place on va les remettre sous la forme d'entiers

```{code-cell} ipython3
# ceci va nous permettre d'avoir des colonnes d'entiers - avec des nan

data = data.convert_dtypes(convert_integer=True)
data.head()
```

```{code-cell} ipython3
# ça ressemble à ceci

data.dtypes
```

```{code-cell} ipython3
# on a toujours les n/a, mais ce n'est pas grave
data[data.isna().any(axis=1)].shape
```

## à quoi ça ressemble

```{code-cell} ipython3
%matplotlib inline

# no longer working ?
# plt.style.use('seaborn')

sns.set(rc={'figure.figsize': (12, 4)})
#plt.rcParams["figure.figsize"] = (12, 4)
```

```{code-cell} ipython3
# un premier jet, pas terrible du tout

data[['East', 'West']].plot();
```

## ajustement

```{code-cell} ipython3
# c'est plus lisible avec seulement un point par semaine
# on pourrait faire la moyenne aussi bien sûr,
# ça donnerait le même dessin mais avec les Y divisés par 7

# le point c'est qu'on a quelques années de plus que sur la vidéo

data.resample('W').sum().plot(); 
```

juste pour être en phase (pouvoir vérifier nos résultats par rapport à ceux de la vidéo), on va s'arrêter à la fin de 2017

(un détail à noter aussi, les données de la vidéo ne contenaient pas la colonne 'total'...)

```{code-cell} ipython3
# c'est facile de couper, la date correpond à l'index de la df
# et grâce au type 'datetime' on peut simplement faire une comparaison

data = data[data.index.year <= 2017]
```

````{admonition} quiz

ici on s'en sort bien car on coupe au début d'une année  
mais comment ferait-on pour couper au 12 Février à 14h32:30 ?

````

```{code-cell} ipython3
data.tail(3)
```

```{code-cell} ipython3
data.resample('W').sum().plot();
```

## `resample()` ?

+++

décortiquons un peu cette histoire de `resample()`

```{code-cell} ipython3
data.shape
```

```{code-cell} ipython3
# la forme du resample() est de:
data.resample('1W').sum().shape
```

```{code-cell} ipython3
# on vérifie que la version resamplée a bien 
# 7 * 24 = 168 fois moins d'entrées que la version brute
# puisqu'on a une mesure par heure et qu'on ré-échatillonne sur une semaine

(full, _), (resampled, _) = data.shape, data.resample('1W').sum().shape

full / resampled , 7 * 24
```

## reprenons

```{code-cell} ipython3
data.resample('1W').sum().plot();
```

```{code-cell} ipython3
# la somme sur une fenêtre tournante d'un an

# mais : méfiez-vous de l'échelle des Y

data.resample('1D').sum().rolling(365).sum().plot();
```

```{code-cell} ipython3
:lines_to_next_cell: 2

# on fait en sorte que le bas de l'échelle des Y soit bien 0
# pour eviter l'effet de loupe

ax = data.resample('1D').sum().rolling(365).sum().plot()
ax.set_ylim(0, None);
```

```{code-cell} ipython3
# regardons la tendance des profils journaliers en moyenne

data.groupby(data.index.time).mean().plot();
```

```{code-cell} ipython3
# mais pour y voir un peu mieux on veut afficher les jours 
# individuellement les uns des autres

# on veut donc dessiner autant de courbes que de jours
# et chaque courbe a en X l'heure de la journée et en Y le nombre (total) de passages

# pour ça on calcule une pivot table
# une courbe par jour: les colonnes sont les jours
# en X les heures: les lignes sont les heures

# ça pourrait se faire à coups de groupby/unstack
# mais c'est quand même plus simple comme ceci

pivoted = data.pivot_table('Total', index=data.index.time, columns=data.index.date)
```

```{code-cell} ipython3
# regardons le coin en haut à gauche

pivoted.iloc[:5, :5]
```

```{code-cell} ipython3
# on confirme les dimensions 

pivoted.shape
```

```{code-cell} ipython3
# du coup on n'a plus qu'à dessiner
# l'astuce qui tue c'est alpha=0.01 pour éviter les gros patés

pivoted.plot(legend=False, alpha=0.01);
```

## classification

+++

ici il s'agit de classifier les jours en deux familles, qu'on voit très distinctement sur la figure

on veut faire une ACP sur un tableau qui aurait 

* les 24 heures en colonnes
* les jours en lignes

et donc c'est presque exactement `pivoted`, sauf que c'est sa transposée !

```{code-cell} ipython3
pivoted.T.shape
```

```{code-cell} ipython3
# ! pip install sklearn
from sklearn.decomposition import PCA
```

```{code-cell} ipython3
# on transpose, et on remplit les trous avec des 0
pca_input = pivoted.T.fillna(0)

# on utilise le PCA de scikit-learn comme une boite noire
pca_output = PCA(2, svd_solver='full').fit_transform(pca_input)
```

```{code-cell} ipython3
# on obtient un tableau numpy avec deux colonnes seulement
# car on a demandé les deux premières composantes principales

type(pca_output), pca_output.shape
```

```{code-cell} ipython3
# on voit effectivement que cet ACP semble bien séparer deux clusters

plt.scatter(pca_output[:, 0], pca_output[:, 1], marker='.', color='green');
```

```{code-cell} ipython3
# pour les trouver ces deux clusters, Jake utilise une GaussianMixture

from sklearn.mixture import GaussianMixture
gmm = GaussianMixture(2)

# c'est ici que tout se passe
labels = gmm.fit(pca_input).predict(pca_input)


# la sortie est une association jour -> type
labels.shape, labels
```

```{code-cell} ipython3
# cette prédiction est bien en phase avec les deux clusters de tout à l'heure

plt.scatter(pca_output[:, 0], pca_output[:, 1], c=labels, cmap='rainbow')
plt.colorbar();
```

### première famille : `label==0`

```{code-cell} ipython3
# pour vérifier notre classification on peut redessiner
# les jours classés label==0 

# ça correspond donc aux jours de la semaine (à moins que ce soit l'inverse..)

pivoted.loc[:, labels==0].plot(legend=False, alpha=0.01);
```

### deuxième famille `label==1`

```{code-cell} ipython3
# et les jours classés label==1

pivoted.loc[:, labels==1].plot(legend=False, alpha=0.01);
```

### les deux clusters avec le jour de la semaine

+++

essayons de vérifier que les deux clusters correspondent bien à l'intuition de départ

pour ça on redessine les deux clusters avec une couleur qui indique le jour de la semaine

```{code-cell} ipython3
# notre index horizontal n'est pas de type DatetimeIndex
pivoted.columns
```

```{code-cell} ipython3
# un index qui contient toutes nos dates et de type DatetimeIndex
dates = pd.DatetimeIndex(pivoted.columns)
```

```{code-cell} ipython3
# ceci nous calcule un index sur les jours
# mais avec comme valeur 0 pour le lundi, ... et 6 pour le dimanche

dayofweek = pd.DatetimeIndex(pivoted.columns).dayofweek

dayofweek.shape, dayofweek
```

```{code-cell} ipython3
# qu'on va utiliser pour mettre les jours en couleur
# les jours de weekend sont en orange et rouge

plt.scatter(pca_output[:, 0], pca_output[:, 1], c=dayofweek, cmap='rainbow')
# pour la légende
plt.colorbar();
```

### les moutons noirs

```{code-cell} ipython3
# on remarque dans le cluster rouge-orange
# des jours d'une couleur qui jure

# pour comprendre à quoi ils correspondent 

odd_index = (labels == 1) & (dayofweek < 5)
odd_index.shape, odd_index
```

```{code-cell} ipython3
# afficher les 48 jours qui sont dans cette catégorie

# comme on peut s'y attendre
# on y retrouve les jours fériés (4 juillet, Noel, ...)

odd_dates = dates[odd_index]
odd_dates, len(odd_dates)
```

```{code-cell} ipython3
# pour rafficher seulement ces jours-là

pivoted[odd_dates].plot(legend=False, alpha=0.3);
```
