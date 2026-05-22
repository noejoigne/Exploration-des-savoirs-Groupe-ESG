# 🚧 Page en cours de rédaction

Ce dépôt GitHub est mis à jour régulièrement. La finalisation complète de la documentation est prévue pour le **24 mai 2026**.

### 🔍 Aperçu du projet
En attendant la version finale, nous vous invitons à consulter les données déjà disponibles :

👉 **[Consulter le Rendu intermédiaire](https://github.com/noejoigne/Exploration-des-savoirs-Groupe-ESG/blob/main/Rendu%20interm%C3%A9diaire.md)**

*Dernière mise à jour : Mai 2026*

---

## Analyse du corpus des méthodologies des notations ESG
### OBJECTIF 1 : Montrer la divergence
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

Les résultats détaillés (figures et analyse) sont présentés dans le rendu final (Figures 1, 2 et 3 de l'annexe).  
Pour obtenir l’intégralité des données ou les scripts complets, vous pouvez me contacter : [noe.joigne@sciencespo.fr](mailto:noe.joigne@sciencespo.fr).

### OBJECTIF 2 : Comprendre la divergence
**1) Codage sociotechnique**

L'objectif est ici d’identifier les critères et méthodes employés dans les modèles de notation, de comparer les approches des différentes agences afin d’en dégager un socle commun et d’identifier précisément quels critères produisent ce clivage et quels mécanismes institutionnels le soutiennent. Le choix retenu pour l’étude de ce corpus repose sur une Analyse en Composante Principale (ACP), une Analyse en Composantes Multiples (ACM) et l'algorithme de partitionnement K-Means.

Cependant, les documents de ce corpus (des grilles de notations textuelles) ne nous permettent pas de réaliser de manière automatique les traitements mentionnés ci-dessus ; pour cela, il nous faut transformer ces critères en un matériau compréhensible pour la machine. Nous avons ainsi effectué le codage sociotechnique suivant : 
| Bloc / Dimension | Indicateur / Critère | Modalités de Codage (Valeurs admises) |
| :--- | :--- | :--- |
| I. Bloc "Cadrage Épistémologique" | Type de Matérialité | 1 = Financière / 2 = Double matérialité |
| I. Bloc "Cadrage Épistémologique" | Unité de mesure finale | 1 = Performance, Score [0-100] / 2 = Risque monétisé ou absolu / 3 = Note alphabétique |
| I. Bloc "Cadrage Épistémologique" | Horizon Temporel | 1 = Risques immédiats, Controverses / 2 = Stratégie long terme, Transition |
| I. Bloc "Cadrage Épistémologique" | Approche de la Notation | 1 = Best-in-class [comparaison sectorielle] / 2 =  Absolue |
| I. Bloc "Cadrage Épistémologique" | Transparence des coefficients | 1 = Boîte noire, Poids cachés / 2 = Poids publics par secteur |
| I. Bloc "Cadrage Épistémologique" | Existence d'un "Score de Transparence" | 1 = Non / 2 = Oui, l'agence pénalise l'absence de réponse |
| II. Bloc "Contenu Environnemental" | Émissions GES Scope 1 & 2 | 1 = Non / 2 = Oui |
| II. Bloc "Contenu Environnemental" | Émissions GES Scope 3 | 1 = Ignoré / 2 = Partiel / 3 = Complet, Exigé |
| II. Bloc "Contenu Environnemental" | Analyse de Scénarios Climatiques | 1 = Non mentionné / 2 = Exigé |
| II. Bloc "Contenu Environnemental" | Gestion de l'Eau et Stress hydrique | 1 = Absent / 2 = Présent |
| II. Bloc "Contenu Environnemental" | Biodiversité & Services Écosystémiques | 1 = Absent / 2 = Présent |
| II. Bloc "Contenu Environnemental" | Économie Circulaire / Gestion des Déchets | 1 = Absent /  2 = Présent |
| II. Bloc "Contenu Environnemental" | Pollution de l'Air (NOx, SOx) | 1 = Absent / 2 = Présent |
| II. Bloc "Contenu Environnemental" | Consommation d'Énergie Totale | 1 = Absent / 2 = Présent |
| II. Bloc "Contenu Environnemental" | Part des Énergies Renouvelables | 1 = Absent / 2 = Présent |
| II. Bloc "Contenu Environnemental" | Innovation Environnementale & Produits verts | 1 = Absent / 2 = Valorisé |
| II. Bloc "Contenu Environnemental" | Impact de la Chaîne d'Approvisionnement | 1 = Non / 2 = Oui |
| II. Bloc "Contenu Environnemental" | Risques Physiques (Inondations, Tempêtes) | 1 = Absent / 2 = Présent |
| III. Bloc "Sourcing et Traitement de la Donnée" | Utilisation de Données Estimées/Modélisées | 1 = Non, 2 = Oui, si l'entreprise ne répond pas |
| III. Bloc "Sourcing et Traitement de la Donnée" | Usage de l'IA/Web Scraping pour la donnée brute | 1 = Non / 2 = Oui |
| III. Bloc "Sourcing et Traitement de la Donnée" | Périodicité de mise à jour | 1 = Annuelle fixe / 2 = Temps réel, Continu |
| III. Bloc "Sourcing et Traitement de la Donnée" | Droit de réponse de l'entreprise | 1 = Non / 2 = Processus formel de vérification |
| III. Bloc "Sourcing et Traitement de la Donnée" | Audit externe de la donnée source exigé | 1 = Non / 2 = Bonus si la donnée est auditée |
| III. Bloc "Sourcing et Traitement de la Donnée" | Normalisation par le Chiffre d'Affaire | 1 = Données absolues / 2 = Données intensives/ratio |
| IV. Bloc "Traitement des Controverses" | Présence d'un score de controverse dédié | 0 = Non / 1 = Oui |
| IV. Bloc "Traitement des Controverses" | Pénalité maximale des controverses | 1 = Pas de plafond / 0 = Plafond fixe |
| IV. Bloc "Traitement des Controverses" | Type de sources pour les controverses | 0 = Presse seule / 1 = ONG + Presse + Syndicats |
| IV. Bloc "Traitement des Controverses" | Échelle de gravité des incidents | 0 = Binaire / 1 = Graduée |
| V. Bloc "Gouvernance de l'Environnement" | Rémunération des dirigeants liée au E | 0 = Non / 1 = Oui, critère de notation |
| V. Bloc "Gouvernance de l'Environnement" | Présence d'un Comité Environnement au Conseil | 0 = Absent / 1 = Présent |
| V. Bloc "Gouvernance de l'Environnement" | Certification ISO 14001 | 0 = Non valorisée / 1 = Valorisée comme preuve d'action |
| V. Bloc "Gouvernance de l'Environnement" | Adhésion à des standards internationaux (SBTi, TCFD) | 0 = Non / 1 = Oui |
| VI. Bloc "Divergences Normatives" (Les biais potentiels) | Biais Géographique | 0 = Neutre / 1 = Valorise les régulations européennes |
| VI. Bloc "Divergences Normatives" (Les biais potentiels) | Poids du Secteur (Homogénéité) | 0 = Tous les critères sont les mêmes pour tous / 1 = Les critères E changent radicalement selon le secteur |
| VI. Bloc "Divergences Normatives" (Les biais potentiels) | Exclusion sectorielle automatique (Charbon, Pétrole) | 0 = Non, la note peut être bonne malgré le secteur / 1 = Exclusion/Sanction automatique |
| VI. Bloc "Divergences Normatives" (Les biais potentiels) | Traitement du "Greenwashing" | 0 = Pas de filtre spécifique / 1 = Algorithme de détection de cohérence entre discours et données |

Ce codage est constitué d’un ensemble de variables décrivant à la fois le cadrage épistémologique, le contenu environnemental, les modalités de traitement des données et les mécanismes institutionnels associés aux notations.

**2) Données et nettoyage**

Il nous a ensuite fallu nettoyer les données : 
```python
import pandas as pd
import numpy as np

# Dictionnaire de données brutes 
data = {
    "Agence": ["MSCI","Sustainalytics","EcoVadis","EthiFinance","SP","Moodys","LSEG","ISS"],
    1:[0,0,1,1,0,0,0,1], 2:[2,1,0,0,0,0,0,2], 3:[1,1,1,1,1,1,1,1], 4:[0,1,1,1,0,1,1,1],
    7:[1,1,1,1,1,0,1,1], 8:[0,0,1,1,0,0,0,1], 9:[1,1,1,1,1,1,1,1], 10:[1,1,2,1,2,1,1,1],
    11:[1,1,0,0,1,1,0,0], 12:[1,1,1,1,1,1,1,1], 13:[1,1,1,1,1,1,1,1], 14:[1,1,1,1,1,1,1,1],
    15:[1,1,1,1,1,1,1,1], 16:[1,1,1,1,1,1,1,1], 17:[1,1,1,1,1,1,1,1], 18:[1,0,1,1,1,1,1,1],
    19:[1,1,1,1,1,1,1,1], 20:[1,1,0,1,1,1,1,1], 21:[1,1,1,1,1,1,1,1], 22:[1,1,1,1,1,1,1,1],
    23:[1,1,0,0,0,0,1,0], 24:[1,1,1,1,1,1,1,1], 25:[0,0,1,1,0,0,0,0], 26:[1,1,0,1,1,1,1,1],
    27:[1,1,1,1,1,1,1,1], 28:[0,1,0,0,0,0,0,0], 29:[1,1,1,1,1,1,1,1], 31:[1,1,1,1,1,1,1,1],
    32:[1,1,0,0,1,1,1,0], 33:[1,1,0,0,1,1,1,0], 34:[1,1,1,1,1,1,1,1], 35:[1,1,1,1,1,1,1,1],
    36:[0,0,1,1,0,1,0,0], 37:[1,1,1,1,1,1,1,1], 38:[0,0,0,0,0,0,0,0], 39:[0,0,0,0,0,0,0,0],
    40:[158,140,200,40,60,40,150,70]
}

def get_cleaned_dataframe():
    """Génère et nettoie la version par défaut (data sans la colonne quantitative 40)."""
    df = pd.DataFrame(data)
    df = df.drop(columns=[40])
    df.set_index("Agence", inplace=True)
    return df
```

**3) Algorithmes d'analyse**

Nous avons finalement pu appliquer nos algorithmes de traitement qui sont détaillés ci-dessous. 

- Algorithme de l'Analyse en Composantes Principales :
```python
# 01_acp_analyse.py
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from data_loader import get_cleaned_dataframe

# 1. DATA
print("Step 1 - DATA")
df = get_cleaned_dataframe()

# 2. IMPUTATION
print("Step 2 - IMPUTATION")
imputer = SimpleImputer(strategy="mean")
X_imputed = imputer.fit_transform(df)

# 3. STANDARDISATION
print("Step 3 - STANDARDISATION")
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_imputed)

# 4. ACP
print("Step 4 - ACP")
pca = PCA()
X_pca = pca.fit_transform(X_scaled)
print("Variance expliquée :", pca.explained_variance_ratio_)

# 5. PLOT INDIVIDUS
plt.figure(figsize=(8,6))
for i, name in enumerate(df.index):
    plt.scatter(X_pca[i,0], X_pca[i,1])
    plt.text(X_pca[i,0]+0.1, X_pca[i,1]+0.1, name)
plt.xlabel("PC1")
plt.ylabel("PC2")
plt.title("ACP - Agences ESG")
plt.axhline(0, color='black', linewidth=0.5)
plt.axvline(0, color='black', linewidth=0.5)
plt.show()

# 6. PLOT VARIABLES (cercle des corrélations)
plt.figure(figsize=(8,8))
for i in range(len(df.columns)):
    plt.arrow(0, 0, pca.components_[0,i], pca.components_[1,i])
    plt.text(pca.components_[0,i]*1.1, pca.components_[1,i]*1.1, str(df.columns[i]))
plt.xlim(-1,1)
plt.ylim(-1,1)
plt.axhline(0, color='black', linewidth=0.5)
plt.axvline(0, color='black', linewidth=0.5)
plt.title("Cercle des corrélations")
plt.show()

# 7. LOADINGS (Contribution des variables)
loadings = pd.DataFrame(pca.components_.T[:, :2], columns=["PC1", "PC2"], index=df.columns)
contrib = loadings**2
contrib["PC1 (%)"] = 100 * contrib["PC1"] / contrib["PC1"].sum()
contrib["PC2 (%)"] = 100 * contrib["PC2"] / contrib["PC2"].sum()

contrib_PC1 = contrib.sort_values(by="PC1 (%)", ascending=False)
contrib_PC2 = contrib.sort_values(by="PC2 (%)", ascending=False)

print("\nTop contributions PC1 :")
print(contrib_PC1[["PC1 (%)"]].head(10))
print("\nTop contributions PC2 :")
print(contrib_PC2[["PC2 (%)"]].head(10))

# Graphiques des contributions
contrib_PC1.head(10)["PC1 (%)"].plot(kind="bar")
plt.title("Top contributions PC1")
plt.show()

contrib_PC2.head(10)["PC2 (%)"].plot(kind="bar")
plt.title("Top contributions PC2")
plt.show()
```

- Algorithme de l'Analyse en Composantes Multiples :
```python
# 02_mca_analyse.py
import pandas as pd
import prince
import matplotlib.pyplot as plt
from data_loader import get_cleaned_dataframe

# 1. Préparer les données
df = get_cleaned_dataframe()
df_mca = df.copy().astype(str)

# 2. MCA (2 composantes principales uniquement)
mca = prince.MCA(n_components=2, random_state=42)
mca = mca.fit(df_mca)
coords = mca.row_coordinates(df_mca)

# 3. Plan factoriel : Axes 1 et 2 uniquement
plt.figure(figsize=(10, 7))
for i, name in enumerate(coords.index):
    plt.scatter(coords.iloc[i, 0], coords.iloc[i, 1])
    plt.text(coords.iloc[i, 0] + 0.02, coords.iloc[i, 1] + 0.02, name)
plt.xlabel(f"Dim 1 ({mca.eigenvalues_summary.iloc[0, 1]})")
plt.ylabel(f"Dim 2 ({mca.eigenvalues_summary.iloc[1, 1]})")
plt.title("MCA - Plan Principal 1-2 : Agences ESG")
plt.axhline(0, color='grey', linestyle='--', linewidth=1)
plt.axvline(0, color='grey', linestyle='--', linewidth=1)
plt.grid(alpha=0.3)
plt.show()

# 4. Rapport de corrélation (ETA2) pour les 2 axes
def calculate_eta2(variable, scores):
    grand_mean = scores.mean()
    total_var = ((scores - grand_mean)**2).sum()
    category_means = scores.groupby(variable).mean()
    category_counts = scores.groupby(variable).count()
    between_var = (category_counts * (category_means - grand_mean)**2).sum()
    return between_var / total_var if total_var != 0 else 0

eta2_results = []
for col in df_mca.columns:
    eta2_dim1 = calculate_eta2(df_mca[col], coords[0])
    eta2_dim2 = calculate_eta2(df_mca[col], coords[1])
    eta2_results.append({'Variable': col, 'Dim 1': eta2_dim1, 'Dim 2': eta2_dim2})

df_eta2 = pd.DataFrame(eta2_results).set_index('Variable')

# Affichage de l'importance des variables sur les 2 axes
plt.figure(figsize=(10, 8))
for var in df_eta2.index:
    x = df_eta2.loc[var, 'Dim 1']
    y = df_eta2.loc[var, 'Dim 2']
    plt.arrow(0, 0, x, y, head_width=0.015, head_length=0.02, color='crimson', alpha=0.6)
    plt.text(x + 0.01, y + 0.01, str(var), fontsize=9, fontweight='bold')
plt.xlim(0, 1.1)
plt.ylim(0, 1.1)
plt.axhline(0, color='black', linewidth=0.8)
plt.axvline(0, color='black', linewidth=0.8)
plt.xlabel(f"Dimension 1 ({mca.eigenvalues_summary.iloc[0, 1]})")
plt.ylabel(f"Dimension 2 ({mca.eigenvalues_summary.iloc[1, 1]})")
plt.title("MCA - Importance des Variables (Correlation Ratio $\eta^2$)\nÉquivalent du cercle des corrélations")
plt.grid(alpha=0.3, linestyle='--')
plt.show()

# 5. Contributions des MODALITÉS à la Dimension 1
contrib_modalites = mca.column_contributions_ * 100
top_15_dim1 = contrib_modalites[0].sort_values(ascending=False).head(15)

plt.figure(figsize=(10, 6))
top_15_dim1.plot(kind='bar', color='skyblue', edgecolor='navy')
seuil = 100 / len(contrib_modalites)
plt.axhline(y=seuil, color='red', linestyle='--', label=f"Seuil moyen ({seuil:.2f}%)")
plt.title("MCA - Top 15 des contributions (Modalités) à la Dimension 1")
plt.ylabel("Contribution (%)")
plt.xticks(rotation=45, ha='right')
plt.legend()
plt.tight_layout()
plt.show()

# 6. Visualisation des Modalités dans le plan factoriel (Axes 1 et 2)
col_coords = mca.column_coordinates(df_mca)
plt.figure(figsize=(12, 9))
plt.scatter(col_coords[0], col_coords[1], c='lightgrey', alpha=0.5, s=20)

top_total = contrib_modalites.sum(axis=1).sort_values(ascending=False).head(20).index
for mod in top_total:
    plt.scatter(col_coords.loc[mod, 0], col_coords.loc[mod, 1], color='darkblue', s=60)
    plt.text(col_coords.loc[mod, 0] + 0.01, col_coords.loc[mod, 1] + 0.01, mod, fontsize=10)
plt.axhline(0, color='black', linewidth=1, alpha=0.5)
plt.axvline(0, color='black', linewidth=1, alpha=0.5)
plt.title("MCA - Carte des Modalités (Loadings)\nPosition des réponses types dans l'espace des 2 axes")
plt.xlabel("Dimension 1")
plt.ylabel("Dimension 2")
plt.grid(alpha=0.2)
plt.show()
```

- Algorithme de partitionnement K-Means :
```python
# 03_kmeans_clustering.py
import numpy as np
import pandas as pd
import prince
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.cluster import KMeans
from matplotlib.patches import Ellipse
from data_loader import get_cleaned_dataframe

# 1. Extraction préliminaire des coordonnées factorielles MCA
df = get_cleaned_dataframe()
df_mca = df.copy().astype(str)
mca = prince.MCA(n_components=2, random_state=42)
mca = mca.fit(df_mca)
coords = mca.row_coordinates(df_mca)

def generate_cluster_plot(n_clusters, colors, title):
    """Fonction modulaire pour calculer le K-means et afficher les points et ellipses de confiance."""
    kmeans = KMeans(n_clusters=n_clusters, random_state=42, n_init=10)
    clusters = kmeans.fit_predict(coords.iloc[:, :2])
    
    plt.figure(figsize=(12, 8))
    for cluster_id in range(n_clusters):
        points = coords.iloc[clusters == cluster_id, :2]
        plt.scatter(points.iloc[:, 0], points.iloc[:, 1], s=100, label=f'Groupe {cluster_id + 1}', color=colors[cluster_id])
        
        for i in range(len(points)):
            plt.text(points.iloc[i, 0] + 0.02, points.iloc[i, 1] + 0.02, points.index[i], fontsize=9)

        # Calcul et tracé de l'ellipse englobante
        if len(points) > 1:
            mean = np.mean(points, axis=0)
            width = (np.max(points.iloc[:, 0]) - np.min(points.iloc[:, 0])) + 0.2
            height = (np.max(points.iloc[:, 1]) - np.min(points.iloc[:, 1])) + 0.2
            ellipse = Ellipse(xy=mean, width=width, height=height, edgecolor=colors[cluster_id], fc=colors[cluster_id], alpha=0.1)
            plt.gca().add_patch(ellipse)
        elif len(points) == 1:
            circle = plt.Circle((points.iloc[0, 0], points.iloc[0, 1]), 0.1, color=colors[cluster_id], fill=True, alpha=0.1)
            plt.gca().add_patch(circle)

    plt.xlabel(f"Dim 1 ({mca.eigenvalues_summary.iloc[0, 1]})")
    plt.ylabel(f"Dim 2 ({mca.eigenvalues_summary.iloc[1, 1]})")
    plt.title(title)
    plt.axhline(0, color='grey', linestyle='--', alpha=0.5)
    plt.axvline(0, color='grey', linestyle='--', alpha=0.5)
    plt.legend()
    plt.grid(alpha=0.2)
    plt.show()
    return clusters

# 2. CLUSTERING N=3
colors_3 = ['#1f77b4', '#ff7f0e', '#2ca02c']
generate_cluster_plot(3, colors_3, "Classification des Agences ESG (Clusters K-Means & MCA)")

# 3. CLUSTERING N=4
colors_4 = ['#1f77b4', '#ff7f0e', '#2ca02c', '#d62728']
clusters_4 = generate_cluster_plot(4, colors_4, "Classification des Agences ESG (4 Clusters K-Means & MCA)")

# 4. PROFILING & HEATMAP (Version N=4)
df_dummies = pd.get_dummies(df_mca)
df_dummies['Cluster'] = clusters_4 + 1
heatmap_data = df_dummies.groupby('Cluster').mean()

# Filtrage des variables discriminantes (> 80% de fréquence)
important_cols = [col for col in heatmap_data.columns if (heatmap_data[col] > 0.8).any()]
heatmap_filtered = heatmap_data[important_cols]

plt.figure(figsize=(16, 6))
sns.heatmap(heatmap_filtered, annot=True, cmap="YlGnBu", fmt=".1f", cbar_kws={'label': 'Fréquence'})
plt.title("Profil caractériel des 4 Clusters d'Agences ESG (Fréquences dominantes)")
plt.ylabel("Clusters (Groupes)")
plt.xlabel("Modalités caractéristiques")
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
```

**6) ACP réalisée sur les données normalisées**

Les résultats détaillés (figures et analyse) sont présentés dans le rendu final (Figures 4 à 10 de l'annexe).  
Pour obtenir l’intégralité des données ou les scripts complets, vous pouvez me contacter : [noe.joigne@sciencespo.fr](mailto:noe.joigne@sciencespo.fr).







