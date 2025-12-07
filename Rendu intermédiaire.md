Nous décrirons ici les données et méthodes utilisées dans le cadre du rendu intermédiaire.

Liens vers les différentes parties :
- [Corpus des méthodologies des notations ESG](https://github.com/noejoigne/Exploration-des-savoirs-Groupe-ESG/blob/main/Rendu%20interm%C3%A9diaire.md#donn%C3%A9es-du-corpus-des-m%C3%A9thodologies-des-notations-esg)

## Données du corpus des méthodologies des notations ESG
**1) Format des données**

L’objectif est de réaliser une Analyse en Composantes Principales (ACP) (et éventuellement un MDS) sur les scores attribués par plusieurs agences pour une même entreprise, afin de mesurer :
- la similarité des notations,
- la divergence entre méthodologies,
- la structure commune éventuelle des indicateurs ESG.

Les données que l'on recherche doivent donc avoir le format suivant :  
| | Entreprise A | Entreprise B | Entreprise C | ... |
|:-----|:-----------:|:-----------:|:-----------:|:-----------:|
| Agence de Notation 1| Note de l'entreprise A pour l'agence 1 | Note de l'entreprise B pour l'agence 1 | ... | ... |
| Agence de Notation 2| Note de l'entreprise A pour l'agence 2 | ... | ... | ... |
| Agence de Notation 3| ... | ... | ... | ... |


**2) Source des données**

Nous utilisons une base de données gratuite créé par Jennifer Kirschnick Duffy intitulée "Industrial sector ESG ratings and stock market data" qui rassemblent les notations ESG faites par S&P, Sustainalytics et MSCI d'environ 700 entreprises, ainsi que leurs informations boursières.
Ce corpus est disponible via [ce lien](https://www.kaggle.com/datasets/jenniferaduffy/industrial-sector-esg-ratings-and-stock-market-data).



**3) Tri et nettoyage des données** 

Le problème de cette base de données est qu'elle contient des entreprises qui ne sont pas notées par les 3 agences de notations. Pour effectuer l’ACP, il faut conserver uniquement les entreprises ayant un score S&P, un score Sustainalytics et un score MSCI.

Pour cela nous avons utilisé dans un tableur le script VBA suivant pour éviter de faire ce tri manuellement :


```vba
Sub SupprimerLignesSiColonnesVides()
    Dim i As Long
    Dim derniereLigne As Long

    Application.ScreenUpdating = False

    derniereLigne = Cells(Rows.Count, 9).End(xlUp).Row 

    For i = derniereLigne To 2 Step -1 
        If Trim(Cells(i, 9).Value) = "" Or _
           Trim(Cells(i, 10).Value) = "" Or _
           Trim(Cells(i, 11).Value) = "" Then

            Rows(i).Delete
        End If
    Next i

    Application.ScreenUpdating = True
End Sub
```
**4) Normalisation des notes**  

Comme les trois agences n’utilisent pas les mêmes échelles, une normalisation commune est indispensable. Les échelles utilisées sont les suivantes : 
- **S&P** (0–100, 100 = Note ESG élevée)  
Déjà sur la bonne échelle → Pas de transformation nécessaire.
- **Sustainalytics** (0–40, 0 = Note ESG élevée)  
Transformation en score sur 100 où 100 correspond à une note ESG élevée.
```python
Note_Normalisée = 100 − (2.5 × Note_brute)
```
- **MSCI** (échelle qualitative AAA → CCC, AAA = Note ESG élevée)  
Conversion en échelle 0–100 selon la règle suivante :

| Note MSCI | Note Normalisée | Code pour tableur |
|:-----:|:-----------:|:-----------:|
|AAA|92.86|```=ARRONDI(100*13/14;2)```|
|AA|78.58|```=ARRONDI(100*11/14;2)```|
|A|64.29|```=ARRONDI(100*9/14;2)```|
|BBB|50.00|```=ARRONDI(100*7/14;2)```|
|BB|35.71|```=ARRONDI(100*5/14;2)```|
|B|21.43|```=ARRONDI(100*3/14;2)```|
|CCC|7.14|```=ARRONDI(100*1/14;2)```|

Ce choix de normalisation a été fait comme l'expliquent les schémas suivants :

| ![Image_expliquant_choix_normalisation_MSCI](https://github.com/noejoigne/Exploration-des-savoirs-Groupe-ESG/blob/Rendu-interm%C3%A9diaire/Normalisation_MSCI_V2.png)  | ![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSdMREiqmWXzIIL5zVrwSjt-kXM9nlo_pXyN4BTLkMRNC4w5-WI) |
|:-----:|:-----------:|
|Schéma résumant la traduction des notes|Données de traduction des notes fournis par MSCI|


**5) ACP réalisée sur les données normalisées**

Le code utilisé pour l’ACP est le suivant :
```python
import pandas as pd
import numpy as np
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv("ESG_DATA_V4.csv", sep=";", decimal=",")
cols = ["SNP_normalized", "Sustainalytics_normalized", "MSCI_normalized"]
data = df[cols].copy()

scaler = StandardScaler()
X = scaler.fit_transform(data)

pca = PCA(n_components=2)
X_pca = pca.fit_transform(X)

explained = pca.explained_variance_ratio_
print("Variance expliquée (PC1, PC2):", explained)
print("Variance cumulée (2 PC):", explained.sum())

loadings = pca.components_.T  # shape (n_features, n_components)
loadings_df = pd.DataFrame(loadings, index=cols, columns=["PC1","PC2"])
print("\nLoadings (poids des agences dans PC1 et PC2) :\n", loadings_df)

contrib = (loadings**2) / np.sum(loadings**2, axis=0) * 100
contrib_df = pd.DataFrame(contrib, index=cols, columns=["PC1_pct","PC2_pct"])
print("\nContributions relatives des variables (%) :\n", contrib_df.round(1))

plt.figure(figsize=(9,7))
plt.scatter(X_pca[:,0], X_pca[:,1], alpha=0.7)
for i, idx in enumerate(df['ID'].astype(str)):
    plt.text(X_pca[i,0], X_pca[i,1], idx, fontsize=8, alpha=0.8)
arrow_scale = 2.5  
for i, colname in enumerate(cols):
    plt.arrow(0, 0, loadings[i,0]*arrow_scale, loadings[i,1]*arrow_scale,
              color='r', width=0.005, head_width=0.08)
    plt.text(loadings[i,0]*arrow_scale*1.15, loadings[i,1]*arrow_scale*1.15,
             colname, color='r', fontsize=11)
plt.axhline(0, color='gray', linewidth=0.5)
```

**6) ACP réalisée sur les données normalisées**

Les résultats détaillés sont présentés dans le rendu intermédiaire.  
Pour obtenir l’intégralité des données ou les scripts complets, vous pouvez me contacter : [noe.joigne@sciencespo.fr](mailto:noe.joigne@sciencespo.fr).
