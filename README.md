# Prédiction des particules fines (PM) à Montréal

L'objectif de ce projet est de prédire la concentration journalière de particules fines (`PM`, exprimée en IQA) pour les 7 stations de mesure de Montréal sur les 365 jours de 2025, soit 2 555 prédictions. La métrique du concours est le RMSE. Tout le travail (exploration, nettoyage, modèle, prédictions) est dans un seul notebook Julia : `main.ipynb`.

Ce projet a été réalisé dans le cadre du cours MTH3302 (Méthodes probabilistes et statistiques pour l'I.A.) à Polytechnique Montréal en hiver 2026.

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Résultat

Le modèle retenu, `nolag`, est une régression linéaire (GLM.jl) ajustée sur 2007-2024 avec 23 variables. Il n'utilise volontairement aucune variable construite à partir du PM passé (lags, moyennes mobiles du PM) : les prédictions reposent sur la météo, le NO2, l'O3 et la saison, ce qui évite qu'une erreur se propage d'un jour à l'autre.

| Mesure | Valeur |
|---|---|
| Score Kaggle (2025) | 18,25 (RMSE, tel que rapporté dans le notebook) |
| RMSE sur 2024 (en échantillon) | 9,09 |
| RMSE 2024, jours normaux / jours de fumée | 7,17 / 25,43 |

Le RMSE de 2024 est un contrôle de cohérence seulement, car 2024 fait partie des données d'entraînement. Le modèle sous-estime nettement les épisodes de fumée (prédictions plafonnées vers 45 alors que les observations dépassent 80), ce qui est attendu d'un modèle linéaire sur une variable à queue lourde.

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Données

Quatre fichiers CSV dans `data/` :

- `qualite-de-lair_{train,test}.csv` : PM, NO2, O3, SO2 par station et par jour. L'entraînement couvre 2007-2024, le test 2025.
- `meteo_{train,test}.csv` : météo de la seule station de l'aéroport (YUL), appliquée aux 7 stations (pluie, neige, vent, visibilité, humidité, température, etc.).

Les deux sources sont jointes sur `Date`.

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Démarche

1. **Chargement** et jointure air + météo.
2. **Analyse exploratoire** : données manquantes, distribution du PM par station, valeurs extrêmes, corrélations entre polluants, relations PM ~ météo, saisonnalité, variables temporelles.
3. **Variables dérivées** : moyennes mobiles (NO2, pluie, visibilité, température), vent, humidité, sinus/cosinus du mois, interactions température × mois, O3², jours consécutifs sans pluie.
4. **Valeurs manquantes** : interpolation avant/arrière par station ; `neige_au_sol` forcée à 0 de mai à septembre ; SO2 (mesuré par 4 stations sur 7) traité par une stratégie dédiée ; lignes sans PM supprimées plutôt qu'imputées.
5. **Modèle** : GLM linéaire, validation par séparation temporelle (jamais de mélange aléatoire).
6. **Prédiction 2025** : mêmes variables sur le jeu test, en collant les derniers jours de 2024 devant 2025 pour que les moyennes mobiles n'aient pas de trous en janvier.

Choix de nettoyage à noter : une ligne du 2024-12-31 sans `stationId` est retirée, ainsi que les lignes avec `PM >= 500`. Les valeurs de PM supérieures à 80 sont conservées, car elles correspondent à de vrais épisodes (feux de forêt de 2023 notamment).

### Pistes abandonnées

Quatre variantes ont été testées avant de garder `nolag` (section « Bilan » du notebook) :

| Approche | RMSE 2024 | RMSE 2023 |
|---|---|---|
| Poids plus élevé pour les années récentes | 9,43 | 25,33 |
| Un modèle par station | 10,26 | n/d |
| Séparation jours normaux / jours de fumée | 9,21 | 24,27 |
| Sélection de variables (VIF + AIC) | 9,87 | 24,76 |

Aucune n'a été assez meilleure pour remplacer `nolag` au classement Kaggle.

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Lancer le projet

Prérequis : Julia 1.11 avec le noyau IJulia, et les paquets CSV, DataFrames, Dates, Gadfly, GLM, Statistics et Plots. Il n'y a pas de `Project.toml`, il faut donc les installer à la main :

```julia
using Pkg
Pkg.add(["CSV", "DataFrames", "Gadfly", "GLM", "Plots", "IJulia"])
```

Ouvrir ensuite le notebook depuis la racine du dépôt (les chemins sont relatifs, `data/...`) :

```julia
using IJulia
notebook(dir=pwd())
```

Exécuter toutes les cellules dans l'ordre : les sections dépendent de l'état des précédentes. L'exécution écrit `benchmark_predictions.csv` à la racine (colonnes `ID` = `(date, stationId)` et `PM`). Ce fichier, `grille_evaluation.md` et `submission/` sont ignorés par git.

Il n'y a pas de tests automatisés ni d'intégration continue. Le notebook est versionné avec ses sorties (environ 21 Mo), donc pour le lire en ligne de commande il vaut mieux passer par un outil qui comprend le JSON plutôt que de l'ouvrir brut.

Si vous ajoutez une variable, il faut la construire à deux endroits : section 3 (entraînement) et section 6.2 (test).

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Limites et pistes d'amélioration

- Le modèle linéaire n'arrive pas à suivre les pics de fumée ; un modèle non linéaire ou des données externes (feux de forêt, trajectoires de panaches) aideraient.
- La météo d'une seule station sert pour toute l'île, hypothèse jugée raisonnable dans l'analyse mais non vérifiée station par station.
- L'évaluation sur 2024 est en échantillon : le modèle final est ajusté sur 2007-2024, donc aucune mesure hors échantillon propre n'est disponible, hormis le score Kaggle.
- Le code gagnerait à être sorti du notebook dans des fonctions testables, avec un `Project.toml` pour figer l'environnement.

<img src="assets/rainbow-line.svg" width="100%" height="6" alt="">

## Auteurs

Tidiane Cissé, Felix Paille Dowell, Alex Tham, Jérémy Trudel, Sami Mostfa.
