ATTENTION CETTE PAGE EST ENCORE EN COURS DE REDACTION !!!


Nous décrirons ici les données et méthodes utilisées dans le cadre du rendu intermédiaire.

Liens vers les différentes parties :
- [Corpus des méthodologies des notations ESG](https://github.com/noejoigne/Exploration-des-savoirs-Groupe-ESG/blob/main/Rendu%20interm%C3%A9diaire.md#donn%C3%A9es-du-corpus-des-m%C3%A9thodologies-des-notations-esg)
- [AJOUTER PARTIE FREDDY]

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
