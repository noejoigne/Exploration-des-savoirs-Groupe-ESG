ATTENTION CETTE PAGE EST ENCORE EN COURS DE REDACTION !!!


Nous décrirons ici les données et méthodes utilisées dans le cadre du rendu intermédiaire.

Liens vers les différentes parties :
- [Corpus des méthodologies des notations ESG](https://github.com/noejoigne/Exploration-des-savoirs-Groupe-ESG/blob/main/Rendu%20interm%C3%A9diaire.md#donn%C3%A9es-du-corpus-des-m%C3%A9thodologies-des-notations-esg)
- [AJOUTER PARTIE FREDDY]

## Données du corpus des méthodologies des notations ESG
**1) Format des données**

L'objectif est ici de faire une ACP (Analyse en Composante Principale) sur les scores donnés par plusieurs agences pour une même entreprise (pour mesurer la similarité/déviation des notations sur plusieurs cas réels).  

Les données que l'on recherche doivent donc avoir le format suivant :  
| | Entreprise A | Entreprise B | Entreprise C | ... |
|:-----|:-----------:|:-----------:|:-----------:|:-----------:|
| Agence de Notation 1| Note de l'entreprise A pour l'agence 1 | Note de l'entreprise B pour l'agence 1 | ... | ... |
| Agence de Notation 2| Note de l'entreprise A pour l'agence 2 | ... | ... | ... |
| Agence de Notation 3| ... | ... | ... | ... |


**2) Source des données**

Nous utilisons une base de données gratuite créé par Jennifer Kirschnick Duffy rassemblant les notations ESG faites par S&P, Sustainalytics et MSCI d'environ 700 entreprises, ainsi que leurs informations boursières.
Ce corpus est disponible via [ce lien](https://www.kaggle.com/datasets/jenniferaduffy/industrial-sector-esg-ratings-and-stock-market-data).

**3) Tri et nettoyage des données** 

Le problème de cette base de données est qu'elle contient des entreprises qui ne sont pas notées par les 3 agences de notations. Il nous faut donc supprimer les lignes/entreprises où certaines données manquent.

Pour cela nous avons utilisé dans un tableur le script suivant pour éviter de faire ce tri manuellement :


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
