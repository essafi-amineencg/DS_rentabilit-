# Facteurs de risque d'un portefeuille : le modèle de Fama-French appliqué à des ETF américains

**Rapport d'analyse — Projet F5, Finance et science des données**

| Rubrique | Détail |
|---|---|
| **Question centrale** | La rentabilité d'un portefeuille s'explique-t-elle par le marché, la taille et la valeur ? |
| **Portefeuille principal** | ETF **IWM** (Russell 2000, petites capitalisations américaines) |
| **Portefeuilles de contrôle** | SPY (marché large), IWD (Value), IWF (Growth) |
| **Méthode** | Régressions linéaires multiples par MCO, erreurs-types robustes HAC (Newey-West, 6 retards) |
| **Modèles comparés** | Moyenne historique (référence) · CAPM · Fama-French à 3 facteurs (FF3) · extension FF5 |
| **Outils** | Python — pandas, yfinance, statsmodels, matplotlib |
| **Livrables** | Notebook exécutable, figures, tableaux CSV, présent rapport |
| **Auteur / Date** | À compléter |

> **Statut du rapport.** Le cadre théorique, la méthode, le protocole d'évaluation, la grille d'interprétation et les limites sont rédigés. Les **résultats empiriques ne sont pas encore disponibles** : le notebook transmis ne contient aucune sortie d'exécution. Toutes les cases marquées *À calculer* doivent être remplies à partir de l'exécution réelle, sans estimation ni valeur approximative.

---

## Sommaire

1. [Résumé exécutif](#résumé-exécutif)
2. [Introduction et problématique](#1-introduction-et-problématique)
3. [Cadre théorique : les facteurs de risque](#2-cadre-théorique--les-facteurs-de-risque)
4. [Données et constitution de l'échantillon](#3-données-et-constitution-de-léchantillon)
5. [Méthodologie économétrique](#4-méthodologie-économétrique)
6. [Indicateurs de comparaison des modèles](#5-indicateurs-de-comparaison-des-modèles)
7. [Évaluation chronologique](#6-évaluation-chronologique)
8. [Diagnostics et stabilité des coefficients](#7-diagnostics-et-stabilité-des-coefficients)
9. [Extension : modèle à cinq facteurs](#8-extension--modèle-à-cinq-facteurs)
10. [Résultats empiriques](#9-résultats-empiriques)
11. [Limites et risques d'interprétation](#10-limites-et-risques-dinterprétation)
12. [Conclusion générale](#11-conclusion-générale)
13. [Procédure de finalisation](#12-procédure-de-finalisation)
14. [Annexes](#annexes)

---

## Résumé exécutif

### Objet de l'étude

Ce travail étudie les facteurs qui expliquent la rentabilité mensuelle de plusieurs ETF américains, avec une attention particulière portée à **IWM**, qui réplique l'indice Russell 2000. Il s'agit de déterminer si le risque de marché suffit à expliquer ses rendements, ou si les facteurs de **taille (SMB)** et de **valeur (HML)** du modèle de Fama-French apportent une information supplémentaire, et si un **alpha** subsiste après contrôle de ces expositions.

### Démarche

| Étape | Contenu |
|---|---|
| 1. Données | Facteurs Ken French (mensuels) et cours ajustés Yahoo Finance, alignés et convertis en rendements excédentaires |
| 2. Modèles | Moyenne historique → CAPM → FF3 (→ FF5 en extension) |
| 3. Inférence | MCO avec erreurs-types HAC, tests individuels et test conjoint SMB = HML = 0 |
| 4. Évaluation | R² ajusté en échantillon ; RMSE, MAE et R² hors échantillon par fenêtre extensible |
| 5. Robustesse | Diagnostics de résidus, deux sous-périodes, régressions glissantes sur 36 mois |
| 6. Limites | Biais de survie, tests multiples, instabilité, frais, hypothèses économétriques |

### État des résultats

| Élément | Statut |
|---|---|
| Cadre théorique et méthode | Rédigés |
| Protocole d'évaluation et diagnostics | Rédigés |
| Coefficients, tests, intervalles de confiance | **À compléter après exécution du notebook** |
| Indicateurs de performance (R², RMSE, MAE) | **À compléter après exécution du notebook** |
| Conclusion empirique chiffrée | **Impossible à ce stade** |

---

## 1. Introduction et problématique

La rentabilité d'un portefeuille ne dépend pas uniquement de la performance générale du marché. Elle peut aussi être associée à des caractéristiques systématiques des titres détenus, notamment leur **taille** et leur **style d'investissement**. Une performance supérieure à celle du marché ne prouve donc pas que le gestionnaire a créé une valeur anormale : elle peut provenir d'une exposition à des facteurs de risque rémunérés.

Le **CAPM** explique le rendement excédentaire d'un actif par un seul facteur, le rendement excédentaire du marché. Le **modèle de Fama-French à trois facteurs** enrichit cette approche par deux facteurs :

- **SMB** (*Small Minus Big*) : écart de performance entre petites et grandes capitalisations ;
- **HML** (*High Minus Low*) : écart entre titres « value » et titres « growth ».

### 1.1 Question principale

> Sur la période étudiée, la rentabilité mensuelle en excès de l'ETF **IWM** s'explique-t-elle uniquement par le risque de marché (CAPM), ou l'ajout des facteurs de taille (SMB) et de valeur (HML) améliore-t-il l'explication de ses rendements ? Après contrôle de ces expositions, l'ETF présente-t-il un alpha statistiquement différent de zéro ?

### 1.2 Questions complémentaires

| N° | Question | Section |
|---|---|---|
| Q1 | Les expositions estimées correspondent-elles aux styles attendus des ETF étudiés ? | 9.1 |
| Q2 | Les coefficients restent-ils relativement stables au cours du temps ? | 7, 9.5 |
| Q3 | Le modèle FF3 réduit-il l'erreur par rapport au CAPM et à la moyenne historique lorsqu'il est évalué chronologiquement ? | 6, 9.3 |
| Q4 | L'ajout des facteurs de profitabilité et d'investissement (FF5) apporte-t-il une amélioration supplémentaire ? | 8 |

### 1.3 Hypothèses de recherche

L'hypothèse nulle portant sur l'alpha est :

$$H_0 : \alpha = 0$$

Elle signifie qu'après contrôle des facteurs inclus dans le modèle, le rendement moyen anormal n'est pas statistiquement différent de zéro.

**Matrice hypothèses — tests — règles de décision**

| Réf. | Hypothèse nulle | Test | Règle de décision (seuil 5 %) | Résultat |
|---|---|---|---|---|
| H1 | $\alpha = 0$ (IWM, FF3) | Student, erreurs HAC | Rejet si $p < 0{,}05$ ou si l'IC 95 % exclut 0 | À compléter |
| H2 | $s = 0$ et $h = 0$ (IWM) | Wald conjoint | Rejet si $p < 0{,}05$ | À compléter |
| H3 | $\beta = 1$ (SPY) | Student sur $\beta - 1$ | Rejet si $p < 0{,}05$ | À compléter |
| H4 | Coefficients constants dans le temps | Comparaison de deux sous-périodes, régressions glissantes | Lecture qualitative (pas de test formel de rupture) | À compléter |
| H5 | FF3 n'améliore pas le CAPM hors échantillon | R², RMSE et MAE hors échantillon | FF3 supérieur si ses erreurs sont plus faibles et son R² hors échantillon plus élevé | À compléter |
| H6 | RMW et CMA n'apportent rien au-delà de FF3 | Comparaison FF3 / FF5 | Lecture conjointe du R² ajusté et de la significativité | À compléter |

**Attentes de signe (validation du modèle)**

| ETF | Style représenté | Exposition attendue |
|---|---|---|
| SPY | Grandes capitalisations / marché large | $\beta$ proche de 1 ; SMB et HML proches de 0 |
| IWM | Petites capitalisations | SMB positif |
| IWD | Style Value | HML positif |
| IWF | Style Growth | HML négatif |

Ces signes sont des **attentes théoriques à confronter aux estimations**, et non des résultats établis. S'ils ne sont pas retrouvés, il faut d'abord soupçonner une erreur de données ou de traitement (unités, décalage de dates) avant d'interpréter IWM.

---

## 2. Cadre théorique : les facteurs de risque

### 2.1 Rendement et rendement excédentaire

Le rendement mensuel simple d'un ETF est calculé à partir de son cours ajusté :

$$R_{i,t} = \frac{P_{i,t}}{P_{i,t-1}} - 1$$

où $P_{i,t}$ désigne le cours ajusté à la fin du mois $t$. Le cours ajusté intègre les distributions et opérations portées par la série de prix.

Le rendement excédentaire est le rendement diminué du taux sans risque :

$$R^{ex}_{i,t} = R_{i,t} - RF_t$$

Cette transformation permet de comparer le rendement de l'actif à une référence sans risque exprimée sur la même période.

### 2.2 Les facteurs et leur interprétation

| Facteur | Définition | Coefficient | Lecture |
|---|---|---|---|
| **Mkt-RF** | Rendement du marché diminué du taux sans risque | $\beta$ | $\beta > 1$ : sensibilité supérieure à celle du marché ; $\beta < 1$ : sensibilité inférieure |
| **SMB** | *Small Minus Big* : petites moins grandes capitalisations | $s$ | $s > 0$ : orientation petites capitalisations (dans la construction du facteur) |
| **HML** | *High Minus Low* : ratio valeur comptable / valeur de marché élevé moins faible | $h$ | $h > 0$ : style Value ; $h < 0$ : style Growth |

> **Précaution.** Un coefficient SMB positif ne représente pas directement le pourcentage du portefeuille investi dans de petites entreprises : il mesure une sensibilité statistique au facteur.

### 2.3 L'alpha

L'alpha est la constante de la régression : la part du rendement excédentaire moyen **non expliquée par les facteurs du modèle**.

- Un alpha positif **n'est pas** automatiquement une preuve de compétence de gestion. Il faut vérifier sa significativité, son intervalle de confiance, sa stabilité et la robustesse du modèle.
- Un alpha non significatif ne permet pas de rejeter l'hypothèse selon laquelle il est nul ; il ne démontre pas non plus qu'il est nul.

### 2.4 Intérêt financier

| Usage | Apport de l'analyse factorielle |
|---|---|
| Attribution de performance | Séparer les expositions rémunérées, reproductibles à faible coût, de la performance résiduelle |
| Évaluation de gérants | Éviter de qualifier de compétence une simple surpondération des petites capitalisations |
| Gestion des risques | Identifier les risques systématiques réellement portés, au-delà de l'étiquette du fonds |
| Coût du capital | Estimer le rendement exigé à partir des bêtas et des primes de risque |

---

## 3. Données et constitution de l'échantillon

### 3.1 Sources

| Source | Contenu | Fréquence | Remarque |
|---|---|---|---|
| **Bibliothèque de Kenneth French** | `Mkt-RF`, `SMB`, `HML`, `RF` (fichier `F-F_Research_Data_Factors_CSV.zip`) | Mensuelle | Valeurs en **pourcentage** ; section annuelle à exclure |
| **Yahoo Finance** (`yfinance`) | Cours ajustés des ETF | Journalière, ramenée au dernier cours du mois | Source non officielle, susceptible d'évoluer ; mise en cache locale prévue |

### 3.2 Fonds étudiés

| Symbole | Description | Fonction dans l'analyse |
|---|---|---|
| **IWM** | ETF associé au Russell 2000 | **Portefeuille principal** |
| SPY | ETF associé au S&P 500 | Référence de marché large |
| IWD | ETF orienté Value | Vérification de l'exposition HML |
| IWF | ETF orienté Growth | Vérification de l'exposition HML |

### 3.3 Période et fréquence

Le notebook fixe le début de la période au **1er janvier 2001** et travaille à fréquence mensuelle. La date de fin dépend des données disponibles à l'exécution. L'échantillon final est l'**intersection** des dates disponibles pour les ETF et les facteurs.

| Élément | Valeur |
|---|---|
| Première observation commune | À calculer |
| Dernière observation commune | À calculer |
| Nombre d'observations mensuelles | À calculer |
| Valeurs manquantes après alignement | À calculer |
| Observations extrêmes ($\lvert z \rvert > 4$) | À calculer |

### 3.4 Préparation des facteurs

Les données Fama-French sont fournies en pourcentage. Le notebook :

- extrait la section mensuelle du fichier ;
- exclut la section annuelle située plus bas ;
- traite les codes `-99.99` et `-999` comme des valeurs manquantes ;
- convertit les pourcentages en décimaux (division par 100).

> **Point critique.** Une valeur de 1,5 % doit être représentée par 0,015. Un mélange entre pourcentages et décimaux produirait des coefficients et des statistiques erronés ; le notebook comporte un contrôle de cohérence des unités.

### 3.5 Calcul des rendements et alignement

Les rendements mensuels sont calculés par variation relative des cours ajustés, puis les séries sont alignées sur les mois communs aux ETF et aux facteurs. Les observations incomplètes sont retirées. Le rendement excédentaire de chaque ETF est ensuite calculé : $R^{ex}_{i,t} = R_{i,t} - RF_t$.

**Chaîne de traitement**

| Étape | Opération | Contrôle |
|---|---|---|
| 1 | Lecture du bloc mensuel Ken French | Exclusion de la section annuelle |
| 2 | Conversion % → décimal, codes manquants | Aucune valeur $-99{,}99$ résiduelle |
| 3 | Dernier cours ajusté de chaque mois | Mois en cours incomplet exclu |
| 4 | Rendements mensuels simples | Absence de rendement aberrant |
| 5 | Jointure sur les mois communs | Nombre d'observations documenté |
| 6 | Rendements excédentaires | Unités cohérentes (aucun $\lvert R \rvert > 60\,\%$) |

---

## 4. Méthodologie économétrique

### 4.1 Les trois spécifications

| Modèle | Équation | Rôle |
|---|---|---|
| **Moyenne historique** | $R^{ex}_{i,t} = \alpha + \varepsilon_t$ | Référence minimale, sans facteur |
| **CAPM** | $R^{ex}_{i,t} = \alpha + \beta\,(Mkt\text{-}RF)_t + \varepsilon_t$ | Modèle à un facteur |
| **FF3** | $R^{ex}_{i,t} = \alpha + \beta\,(Mkt\text{-}RF)_t + s\,SMB_t + h\,HML_t + \varepsilon_t$ | Modèle principal à trois facteurs |

La comparaison CAPM / FF3 vise à déterminer si SMB et HML apportent une information explicative supplémentaire et si l'estimation de l'alpha change lorsque ces facteurs sont pris en compte.

### 4.2 Estimation et erreurs-types HAC

Les coefficients sont estimés par **moindres carrés ordinaires (MCO)**. L'inférence repose sur des **erreurs-types HAC de Newey-West** (6 retards), qui rendent les tests plus robustes à l'hétéroscédasticité et à l'autocorrélation des résidus.

Cette correction modifie les erreurs-types, les statistiques de test et les intervalles de confiance, mais **ne remplace pas** les coefficients MCO estimés.

### 4.3 Tests statistiques

| Test | Hypothèse nulle | Usage |
|---|---|---|
| Student sur l'alpha | $\alpha = 0$ | Existence d'une performance anormale |
| Student sur $\beta$, $s$, $h$ | Coefficient nul | Significativité de chaque exposition |
| Intervalles de confiance à 95 % | — | Précision des estimations |
| Wald conjoint | $s = 0$ **et** $h = 0$ | Apport global de SMB et HML |

Pour le test conjoint :

$$H_0 : s = 0 \quad \text{et} \quad h = 0$$

Le rejet de cette hypothèse signifie que les deux facteurs ajoutent **conjointement** une information significative. Cela n'implique pas que chaque coefficient soit significatif individuellement. La décision est prise au seuil de 5 % : une p-value inférieure à 0,05 conduit au rejet, sous réserve de la validité du modèle et de la procédure d'inférence.

---

## 5. Indicateurs de comparaison des modèles

### 5.1 R² ajusté

Le R² mesure la part de variation expliquée dans l'échantillon. Le **R² ajusté** pénalise l'ajout de facteurs qui n'améliorent pas suffisamment l'ajustement. Un R² ajusté plus élevé indique un meilleur ajustement relatif, mais **ne démontre pas** une meilleure capacité de généralisation.

### 5.2 RMSE et MAE

$$RMSE = \sqrt{\frac{1}{n}\sum_{t=1}^{n}\left(y_t - \hat{y}_t\right)^2}
\qquad
MAE = \frac{1}{n}\sum_{t=1}^{n}\left|y_t - \hat{y}_t\right|$$

Le RMSE pénalise davantage les erreurs importantes ; le MAE mesure l'erreur absolue moyenne. Des valeurs plus faibles indiquent de meilleures erreurs d'ajustement. Le notebook les affiche en pourcentage.

### 5.3 R² hors échantillon

$$R^2_{OOS} = 1 - \frac{\sum_t \left(y_t - \hat{y}^{\,modèle}_t\right)^2}{\sum_t \left(y_t - \hat{y}^{\,moyenne}_t\right)^2}$$

Une valeur **positive** indique que le modèle fait mieux que la moyenne historique ; une valeur **négative**, qu'il fait moins bien. Ce résultat doit être distingué du R² calculé sur l'échantillon d'estimation.

**Tableau de lecture des indicateurs**

| Indicateur | Où il est mesuré | Sens favorable | Ce qu'il ne prouve pas |
|---|---|---|---|
| R² ajusté | En échantillon | Plus élevé | Une bonne généralisation |
| RMSE | Hors échantillon | Plus faible | Une absence d'erreurs extrêmes (sensible aux grosses erreurs) |
| MAE | Hors échantillon | Plus faible | La qualité en cas de crise |
| R² hors échantillon | Hors échantillon | Positif et plus élevé | Une capacité de prévision ex ante (voir section 6) |

---

## 6. Évaluation chronologique

Un découpage aléatoire est inadapté aux séries temporelles : il ferait fuir de l'information du futur vers le passé. Le notebook utilise une **fenêtre extensible** (*expanding window*).

| Élément | Choix |
|---|---|
| Apprentissage minimal | 60 mois |
| À chaque mois $t$ | Coefficients estimés **uniquement** sur les mois antérieurs à $t$ |
| Application | Coefficients appliqués aux facteurs observés au mois $t$ |
| Référence | Moyenne historique du rendement excédentaire, sur le même historique |

Cette procédure vérifie que les relations estimées sur le passé conservent un pouvoir explicatif au fil du temps.

> **Limite d'interprétation importante.** Les facteurs du mois $t$ sont supposés connus au moment de l'évaluation. Il s'agit d'une **attribution** ou d'un *nowcast* conditionnel aux facteurs observés, et **non** d'une véritable prévision du rendement avant le début du mois. Une prévision ex ante exigerait des prévisions des facteurs ou des informations disponibles à la date de décision.

---

## 7. Diagnostics et stabilité des coefficients

### 7.1 Diagnostic des résidus (FF3, IWM)

| Diagnostic | Ce qu'il examine | Lecture générale |
|---|---|---|
| **Durbin-Watson** | Autocorrélation de premier ordre des résidus | Valeur proche de 2 : faible autocorrélation de premier ordre |
| **Jarque-Bera** | Écart à la normalité | P-value faible : rejet de la normalité |
| **Breusch-Pagan** | Hétéroscédasticité | P-value faible : variance des résidus non constante |
| **VIF** | Multicolinéarité entre facteurs | Valeurs élevées : forte dépendance linéaire (seuil usuel : 5) |

Ces diagnostics aident à évaluer les limites de la régression. Ils ne doivent pas être interprétés isolément et ne garantissent pas que toutes les hypothèses du modèle sont satisfaites.

### 7.2 Analyse par sous-périodes

L'échantillon est divisé en **deux moitiés chronologiques** et le modèle FF3 est réestimé sur chacune. Points à examiner :

- l'évolution du bêta de marché ;
- la stabilité du coefficient SMB ;
- la stabilité du coefficient HML ;
- les changements de signe ou d'ordre de grandeur ;
- les différences de significativité entre sous-périodes.

> Une différence entre deux estimations ne prouve pas à elle seule une rupture structurelle : c'est un indice à interpréter avec prudence.

### 7.3 Régressions glissantes

Une régression glissante sur **36 mois** estime le modèle, à chaque date, sur les 36 mois précédents. Le graphique des coefficients montre leur évolution. Des coefficients variables suggèrent que les expositions ne sont pas constantes, en raison de l'évolution de la composition du fonds, des conditions de marché ou de changements de régime économique.

---

## 8. Extension : modèle à cinq facteurs

Le modèle FF5 ajoute deux facteurs :

| Facteur | Signification | Dimension économique |
|---|---|---|
| **RMW** | *Robust Minus Weak* | Profitabilité |
| **CMA** | *Conservative Minus Aggressive* | Politique d'investissement |

$$R^{ex}_{i,t} = \alpha + \beta\,(Mkt\text{-}RF)_t + s\,SMB_t + h\,HML_t + r\,RMW_t + c\,CMA_t + \varepsilon_t$$

Cette extension vérifie si la profitabilité et l'investissement apportent une information supplémentaire au-delà des trois facteurs initiaux. La comparaison doit porter sur les indicateurs d'ajustement et, si disponibles, sur la performance hors échantillon.

> L'ajout de facteurs augmente la complexité du modèle : une amélioration du R² en échantillon ne suffit pas à justifier leur inclusion. Le fichier FF5 possède son propre jeu de facteurs ; la comparaison FF3 / FF5 doit donc être faite sur le même échantillon.

---

## 9. Résultats empiriques

> **Important.** Le notebook transmis est un fichier de code non exécuté. Les tableaux ci-dessous constituent la **grille de restitution** à compléter après exécution de toutes les cellules. Les valeurs ne doivent être ni estimées ni inventées.

### 9.1 Expositions FF3 des ETF

| ETF | Style | Alpha (%/mois) | IC 95 % alpha | Bêta marché | SMB | HML | R² ajusté |
|---|---|---|---|---|---|---|---|
| SPY | Marché large | À calculer | À calculer | À calculer | À calculer | À calculer | À calculer |
| **IWM** | Petites capitalisations | À calculer | À calculer | À calculer | À calculer | À calculer | À calculer |
| IWD | Value | À calculer | À calculer | À calculer | À calculer | À calculer | À calculer |
| IWF | Growth | À calculer | À calculer | À calculer | À calculer | À calculer | À calculer |

*Source : tableau `coefficients_ff3.csv` ; figure 3.*

**Interprétation à rédiger après exécution :** comparer les signes et la significativité des coefficients aux attentes de la section 1.3.

| ETF | Point à vérifier |
|---|---|
| SPY | $\beta$ proche de 1 ? SMB et HML proches de 0 ? |
| IWM | Coefficient SMB positif ? |
| IWD | Coefficient HML positif (Value) ? |
| IWF | Coefficient HML négatif (Growth) ? |

### 9.2 Comparaison CAPM et FF3 pour IWM

| Indicateur | CAPM | FF3 |
|---|---|---|
| Alpha estimé | À calculer | À calculer |
| Erreur-type HAC de l'alpha | À calculer | À calculer |
| Intervalle de confiance à 95 % de l'alpha | À calculer | À calculer |
| P-value de l'alpha | À calculer | À calculer |
| Bêta de marché | À calculer | À calculer |
| Coefficient SMB | Non inclus | À calculer |
| Coefficient HML | Non inclus | À calculer |
| R² ajusté | À calculer | À calculer |

*Source : tableau `modeles_alpha_r2.csv`.*

**Grille d'interprétation de l'alpha**

| Constat | Lecture |
|---|---|
| Alpha significatif dans le CAPM mais plus dans FF3 | Une partie de la performance attribuée à l'alpha provient des facteurs de taille et/ou de valeur |
| Alpha encore significatif dans FF3 | Une composante moyenne non expliquée par les trois facteurs subsiste dans l'échantillon ; examiner l'IC, la stabilité et les biais avant de parler de surperformance |
| IC à 95 % contenant zéro | Les données ne permettent pas de conclure à un alpha différent de zéro au seuil de 5 % |

### 9.3 Comparaison globale des modèles (en échantillon et hors échantillon)

Une ligne par combinaison ETF–modèle, conformément aux sorties du notebook.

| ETF | Modèle | R² ajusté | RMSE (%) | MAE (%) | R² hors échantillon |
|---|---|---|---|---|---|
| SPY | Moyenne | À calculer | À calculer | À calculer | 0 (par construction) |
| SPY | CAPM | À calculer | À calculer | À calculer | À calculer |
| SPY | FF3 | À calculer | À calculer | À calculer | À calculer |
| IWM | Moyenne | À calculer | À calculer | À calculer | 0 (par construction) |
| IWM | CAPM | À calculer | À calculer | À calculer | À calculer |
| IWM | FF3 | À calculer | À calculer | À calculer | À calculer |
| IWD | Moyenne | À calculer | À calculer | À calculer | 0 (par construction) |
| IWD | CAPM | À calculer | À calculer | À calculer | À calculer |
| IWD | FF3 | À calculer | À calculer | À calculer | À calculer |
| IWF | Moyenne | À calculer | À calculer | À calculer | 0 (par construction) |
| IWF | CAPM | À calculer | À calculer | À calculer | À calculer |
| IWF | FF3 | À calculer | À calculer | À calculer | À calculer |

*Source : tableau `evaluation_hors_echantillon.csv` ; figures 4 et 5.*

### 9.4 Test conjoint de SMB et HML

| Test | Statistique | P-value | Décision au seuil de 5 % |
|---|---|---|---|
| $H_0 : s = 0$ et $h = 0$ | À calculer | À calculer | À compléter |

Si la p-value est inférieure à 0,05, l'hypothèse de nullité simultanée est rejetée : SMB et HML apportent un gain statistique conjoint dans le modèle considéré.

### 9.5 Diagnostics et stabilité

| Élément | Résultat |
|---|---|
| Durbin-Watson | À calculer |
| Jarque-Bera — statistique et p-value | À calculer |
| Breusch-Pagan — statistique et p-value | À calculer |
| VIF des facteurs | À calculer |
| Coefficients sur la première moitié | À calculer |
| Coefficients sur la seconde moitié | À calculer |
| Coefficients glissants sur 36 mois | À commenter à partir de la figure 7 |

*Source : tableau `sous_periodes.csv` ; figures 6 et 7.*

### 9.6 Extension FF5 (optionnelle)

| ETF | R² ajusté FF3 | R² ajusté FF5 | Alpha FF3 (%/mois) | Alpha FF5 (%/mois) | P-value alpha FF5 |
|---|---|---|---|---|---|
| SPY | À calculer | À calculer | À calculer | À calculer | À calculer |
| IWM | À calculer | À calculer | À calculer | À calculer | À calculer |
| IWD | À calculer | À calculer | À calculer | À calculer | À calculer |
| IWF | À calculer | À calculer | À calculer | À calculer | À calculer |

*Source : tableau `extension_ff5.csv`.*

---

## 10. Limites et risques d'interprétation

| N° | Risque | Description | Parade dans l'étude |
|---|---|---|---|
| 10.1 | Biais de survie | Les ETF retenus existent encore ; les fonds disparus sont absents, ce qui peut donner une image trop favorable. Le biais serait plus fort avec les seules entreprises actuelles d'un indice | ETF anciens, grands et liquides ; limite explicitement signalée |
| 10.2 | Tests multiples | Plusieurs ETF et modèles : la probabilité d'un alpha significatif par hasard augmente | Lecture de l'IC plutôt que de la seule p-value ; correction de Holm envisageable |
| 10.3 | Instabilité des facteurs | Les expositions évoluent ; un coefficient global est une moyenne | Sous-périodes et régressions glissantes |
| 10.4 | Frais et coûts | Frais de gestion et écart de réplication ; facteurs théoriques sans coûts | Interpréter un alpha négatif avec cette réserve |
| 10.5 | Hypothèses économétriques | Résidus non normaux, volatilité variable, autocorrélation | Erreurs HAC ; diagnostics ; pas de correction d'une mauvaise spécification |
| 10.6 | Représentativité et causalité | Marché américain, ETF liquides ; une association n'est pas une cause | Aucune généralisation automatique |

### 10.1 Biais de survie

Une étude limitée aux fonds survivants peut surestimer la performance du secteur, puisque les fonds disparus ou liquidés n'y figurent pas. Constituer un portefeuille avec les entreprises actuelles d'un indice pour reconstituer le passé aggraverait ce biais.

### 10.2 Alpha significatif par hasard et tests multiples

Plus le nombre de tests augmente, plus la probabilité d'obtenir au moins un résultat significatif par hasard augmente. Pour $m$ tests indépendants au seuil de 5 %, cette probabilité est $1 - 0{,}95^{m}$. Un alpha significatif isolé doit donc être interprété prudemment.

**Correction de Holm.** On classe les $m$ p-values par ordre croissant $p_{(1)} \le \dots \le p_{(m)}$ et on rejette $H_0$ pour le rang $k$ tant que :

$$p_{(k)} \le \frac{0{,}05}{m - k + 1}$$

La procédure s'arrête au premier rang où la condition échoue. Elle contrôle le risque d'erreur globale de façon moins conservatrice que la correction de Bonferroni.

**Précision de l'alpha.** L'erreur-type de l'alpha est approximativement $\sigma_{\varepsilon}/\sqrt{N}$ : plus la volatilité résiduelle est élevée et plus l'échantillon est court, plus l'alpha doit être grand pour être distingué du bruit.

### 10.3 Instabilité des facteurs

Un coefficient estimé sur toute la période résume une relation moyenne et peut masquer des changements de régime ou de composition. Les analyses par sous-périodes et les régressions glissantes complètent les résultats globaux.

### 10.4 Frais et coûts de transaction

Les ETF supportent des frais de gestion et peuvent présenter un écart de réplication. Les facteurs Fama-French sont des portefeuilles théoriques qui ne reflètent pas les coûts réels d'un investisseur : une partie d'un alpha négatif éventuel peut simplement refléter ces frais.

### 10.5 Hypothèses économétriques

Les rendements financiers présentent souvent des résidus non normaux, une volatilité variable et une autocorrélation. Les erreurs HAC améliorent la robustesse de l'inférence, mais ne corrigent ni une mauvaise spécification du modèle ni une variable explicative omise.

### 10.6 Représentativité et causalité

Les résultats ne se généralisent pas automatiquement aux marchés internationaux, aux fonds moins liquides ou aux actions individuelles. Une relation statistique entre un facteur et un rendement décrit une **exposition** ; elle ne démontre pas à elle seule un lien de causalité.

### 10.7 Fuite d'information et biais d'anticipation

L'évaluation hors échantillon n'estime les coefficients que sur le passé, mais suppose les facteurs du mois courant connus (section 6). Les facteurs de Ken French sont publiés avec retard et peuvent être révisés.

---

## 11. Conclusion générale

Cette étude propose un cadre d'analyse permettant de distinguer l'**exposition** d'un portefeuille aux facteurs de risque de sa **performance résiduelle**. La comparaison entre la moyenne historique, le CAPM et le modèle FF3 évalue l'apport des facteurs de marché, de taille et de valeur dans l'explication des rendements excédentaires.

Le choix d'IWM met l'accent sur les petites capitalisations, tandis que SPY, IWD et IWF servent de points de comparaison pour vérifier la cohérence des expositions estimées. Les erreurs-types HAC, l'évaluation chronologique, les diagnostics et les régressions glissantes renforcent la démarche empirique.

Cependant, **le notebook fourni n'ayant pas été exécuté, aucune conclusion chiffrée ne peut encore être tirée** sur la significativité de l'alpha d'IWM, l'apport effectif de SMB et HML ou la supériorité de FF3. La décision finale ne devra pas reposer sur le seul R² : elle devra aussi tenir compte de la significativité, des intervalles de confiance, de la performance hors échantillon, de la stabilité des coefficients et des limites méthodologiques.

**Plan de conclusion à rédiger après exécution (3 à 5 phrases)**

1. Réponse à la question principale : part expliquée par le marché (R² ajusté du CAPM) et par FF3.
2. Apport de SMB et HML : test conjoint et confirmation hors échantillon.
3. Alpha : valeur, intervalle de confiance, lecture nuancée selon la grille de la section 9.2.
4. Validation : signes retrouvés (ou non) sur SPY, IWD et IWF.
5. Réserves principales : tests multiples, stabilité des bêtas, biais de survie.

---

## 12. Procédure de finalisation

| N° | Action | Fait |
|---|---|---|
| 1 | Ouvrir le notebook dans Google Colab ou Jupyter avec une connexion Internet | ☐ |
| 2 | Exécuter toutes les cellules dans l'ordre | ☐ |
| 3 | Vérifier le téléchargement et l'alignement des données (facteurs et ETF) | ☐ |
| 4 | Vérifier les dates et le nombre d'observations de l'échantillon commun | ☐ |
| 5 | Récupérer coefficients, erreurs-types HAC, p-values et intervalles de confiance | ☐ |
| 6 | Compléter les tableaux de la section 9 sans arrondir excessivement | ☐ |
| 7 | Insérer et commenter les figures (corrélations, expositions, erreurs, stabilité) | ☐ |
| 8 | Rédiger la conclusion à partir des seuls résultats obtenus | ☐ |
| 9 | Vérifier que chaque affirmation écrite correspond à un résultat calculé | ☐ |

> **Reproductibilité.** Le notebook exporte les tableaux dans `resultats/` et génère automatiquement un fichier `rapport_F5.md` à partir des résultats calculés. Le présent rapport est une version explicative structurée ; ses tableaux empiriques doivent être actualisés à partir de l'exécution réelle.

---

## Annexes

### Annexe A — Correspondance notebook ↔ rapport

| Élément du notebook | Fichier produit | Section du rapport |
|---|---|---|
| Performance cumulée des ETF et facteurs | `figures/fig1_performance_cumulee.png` | 3 |
| Corrélations et distribution des rendements | `figures/fig2_correlations_distribution.png` | 3 |
| Coefficients FF3 avec IC 95 % | `figures/fig3_coefficients_ff3.png` | 9.1 |
| Gain cumulé d'erreur hors échantillon | `figures/fig4_gain_hors_echantillon.png` | 9.3 |
| Comparaison des modèles | `figures/fig5_comparaison_modeles.png` | 9.3 |
| Diagnostics des résidus | `figures/fig6_diagnostics_residus.png` | 9.5 |
| Bêtas glissants | `figures/fig7_betas_glissants.png` | 9.5 |
| Alpha, IC, R² ajusté par modèle | `resultats/modeles_alpha_r2.csv` | 9.2 |
| Coefficients FF3 par ETF | `resultats/coefficients_ff3.csv` | 9.1 |
| Évaluation hors échantillon | `resultats/evaluation_hors_echantillon.csv` | 9.3 |
| Sous-périodes | `resultats/sous_periodes.csv` | 9.5 |
| Extension FF5 | `resultats/extension_ff5.csv` | 9.6 |

### Annexe B — Paramètres du notebook

| Paramètre | Valeur | Rôle |
|---|---|---|
| `MAIN` | `IWM` | Portefeuille principal |
| `START` | 2001-01-01 | Début de la période de téléchargement |
| `HAC_LAGS` | 6 | Retards des erreurs-types Newey-West |
| `ROLL` | 36 | Fenêtre des régressions glissantes (mois) |
| `MIN_TRAIN` | 60 | Apprentissage minimal pour l'évaluation hors échantillon (mois) |
| `SIGNIF` | 0,05 | Seuil de significativité |

### Annexe C — Glossaire

| Terme | Définition |
|---|---|
| **Alpha** | Part du rendement excédentaire moyen non expliquée par les facteurs du modèle |
| **Bêta** | Sensibilité du rendement excédentaire à un facteur |
| **Rendement excédentaire** | Rendement diminué du taux sans risque |
| **HAC / Newey-West** | Erreurs-types robustes à l'hétéroscédasticité et à l'autocorrélation |
| **R² ajusté** | R² corrigé du nombre de variables explicatives |
| **Fenêtre extensible** | Protocole où l'ensemble d'apprentissage s'agrandit avec le temps, sans jamais utiliser le futur |
| **Nowcast** | Estimation conditionnelle à des facteurs déjà observés, et non prévision ex ante |
| **Biais de survie** | Biais dû à l'exclusion des entités disparues de l'échantillon |
| **VIF** | Facteur d'inflation de la variance, indicateur de multicolinéarité |

### Annexe D — Références

- Sharpe, W. F. (1964). *Capital Asset Prices: A Theory of Market Equilibrium under Conditions of Risk*. Journal of Finance.
- Fama, E. F. et French, K. R. (1993). *Common Risk Factors in the Returns on Stocks and Bonds*. Journal of Financial Economics.
- Fama, E. F. et French, K. R. (2015). *A Five-Factor Asset Pricing Model*. Journal of Financial Economics.
- Newey, W. K. et West, K. D. (1987). *A Simple, Positive Semi-Definite, Heteroskedasticity and Autocorrelation Consistent Covariance Matrix*. Econometrica.
- Holm, S. (1979). *A Simple Sequentially Rejective Multiple Test Procedure*. Scandinavian Journal of Statistics.
- Kenneth R. French, *Data Library* : https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html

---

*Projet pédagogique : ces résultats ne constituent pas un conseil en investissement.*
