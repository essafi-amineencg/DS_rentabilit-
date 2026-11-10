# Rapport final — Analyse des facteurs de risque d’un portefeuille selon le modèle de Fama-French

**Projet F5 — Finance et science des données**  
**Portefeuille principal :** IWM (Russell 2000 — petites capitalisations américaines)  
**Portefeuilles de comparaison :** SPY, IWD et IWF  
**Période d’analyse :** février 2001 à septembre 2026  
**Fréquence :** mensuelle  
**Méthodes :** régressions MCO, erreurs-types robustes HAC/Newey-West, évaluation chronologique  
**Environnement :** Python / Google Colab  
**Date d’exécution visible dans le notebook :** 9 novembre 2026

---

## Résumé exécutif

Ce projet cherche à déterminer si les rendements excédentaires de l’ETF IWM sont expliqués uniquement par le risque général du marché, comme le suppose le CAPM, ou si les facteurs de taille (SMB) et de valeur (HML) du modèle de Fama-French à trois facteurs améliorent l’explication de ses performances.

L’analyse porte sur quatre ETF : SPY, IWM, IWD et IWF. Les rendements mensuels sont calculés à partir de cours ajustés, puis diminués du taux sans risque. L’échantillon commun comprend **308 observations mensuelles**, de février 2001 à septembre 2026. Les modèles sont estimés par les moindres carrés ordinaires et les tests de significativité utilisent des erreurs-types HAC de Newey-West avec six retards.

Les résultats montrent que le modèle FF3 améliore nettement l’ajustement de l’ETF IWM par rapport au CAPM : le R² ajusté passe de **0,8250 à 0,9841**. Le coefficient SMB est positif et significatif (**0,8631 ; p < 0,0001**), ce qui confirme une forte exposition au facteur des petites capitalisations. Le coefficient HML est également positif et significatif (**0,2167 ; p < 0,0001**). En revanche, l’alpha FF3 d’IWM est négatif et statistiquement significatif : **−0,1198 % par mois**, avec un intervalle de confiance à 95 % de **[−0,2033 % ; −0,0364 %]** et une p-value de **0,0049**. Dans cet échantillon et selon cette spécification, les rendements d’IWM ne révèlent donc pas de surperformance anormale positive après contrôle des trois facteurs.

L’évaluation chronologique confirme l’apport du modèle FF3 : pour IWM, le RMSE hors échantillon passe de **2,3509 % par mois avec le CAPM à 0,7250 % avec FF3**, et le R² hors échantillon atteint **0,9843**. Cette évaluation reste toutefois une attribution conditionnelle aux facteurs observés pendant le mois considéré, et non une prévision ex ante du rendement.

Les diagnostics ne révèlent pas de problème majeur d’autocorrélation de premier ordre, d’hétéroscédasticité ou de multicolinéarité selon les tests réalisés. Les coefficients estimés sur les deux moitiés de l’échantillon sont relativement proches, ce qui suggère une certaine stabilité des expositions. Enfin, l’extension à cinq facteurs augmente légèrement le R² ajusté d’IWM, de **0,9867 à 0,9881** sur l’échantillon utilisé pour cette comparaison, mais l’alpha demeure négatif et significatif (**−0,0877 % par mois ; p = 0,0092**).

Ces résultats sont des associations statistiques sur la période observée. Ils ne constituent pas une recommandation d’investissement et ne démontrent pas une relation causale.

## 1. Introduction et problématique

La performance d’un portefeuille financier dépend de plusieurs sources de risque. Une partie de son rendement peut provenir de l’évolution générale du marché, tandis qu’une autre partie peut être associée à des caractéristiques systématiques des titres détenus, telles que leur taille ou leur style de valorisation.

Il est donc important de distinguer la performance expliquée par des facteurs de risque connus de la performance résiduelle, souvent appelée alpha. Un portefeuille peut afficher un rendement supérieur à celui d’un indice sans que cette différence soit nécessairement due à une compétence particulière de gestion. Elle peut simplement résulter d’une exposition plus importante à des facteurs rémunérés.

Le CAPM explique le rendement excédentaire d’un actif à partir d’un seul facteur : le rendement du marché diminué du taux sans risque. Le modèle de Fama-French à trois facteurs enrichit cette approche en ajoutant le facteur de taille SMB et le facteur de valeur HML.

### 1.1. Question principale

Sur la période étudiée, la rentabilité mensuelle en excès de l’ETF **IWM** s’explique-t-elle uniquement par le risque de marché (CAPM), ou l’ajout des facteurs de taille (SMB) et de valeur (HML) améliore-t-il l’explication de ses rendements ? Après contrôle de ces expositions, IWM présente-t-il un alpha statistiquement différent de zéro ?

### 1.2. Questions complémentaires

- Les expositions estimées correspondent-elles aux styles attendus des ETF étudiés ?
- Les coefficients restent-ils relativement stables au cours du temps ?
- Le modèle FF3 réduit-il l’erreur par rapport au CAPM et à une moyenne historique lors d’une évaluation chronologique ?
- L’ajout des facteurs de profitabilité et d’investissement dans le modèle FF5 apporte-t-il une amélioration supplémentaire ?

### 1.3. Hypothèses de recherche

L’hypothèse nulle relative à l’alpha est :

\[
H_0 : \alpha = 0
\]

Elle signifie qu’après contrôle des facteurs inclus dans le modèle, le rendement moyen anormal n’est pas statistiquement différent de zéro.

Les signes attendus sont les suivants :

| ETF | Style représenté | Exposition attendue |
|---|---|---|
| SPY | Grandes capitalisations / marché large | Bêta proche de 1 ; SMB et HML relativement proches de 0 |
| IWM | Petites capitalisations | SMB positif |
| IWD | Style Value | HML positif |
| IWF | Style Growth | HML négatif |

Ces attentes sont des hypothèses théoriques à confronter aux résultats estimés.

### 1.4. Intérêt financier

Cette analyse présente plusieurs intérêts :

- **Attribution de performance :** distinguer le rendement associé aux facteurs de risque de la performance résiduelle.
- **Évaluation des fonds :** éviter d’attribuer automatiquement à la compétence d’un gestionnaire une performance provenant d’une exposition factorielle.
- **Gestion des risques :** identifier les sensibilités du portefeuille au marché, à la taille et à la valeur.
- **Allocation d’actifs :** mieux comprendre le profil de risque d’un portefeuille avant de le comparer à d’autres placements.

## 2. Cadre théorique : les facteurs de risque

### 2.1. Rendement mensuel

Le rendement mensuel simple d’un ETF est calculé à partir de son cours ajusté :

\[
R_{i,t}=\frac{P_{i,t}}{P_{i,t-1}}-1
\]

où \(P_{i,t}\) désigne le cours ajusté à la fin du mois \(t\). Le notebook utilise les cours ajustés, qui prennent en compte les distributions et opérations intégrées dans cette série de prix.

### 2.2. Rendement excédentaire

Le rendement excédentaire correspond au rendement de l’ETF diminué du taux sans risque :

\[
R_{i,t}^{ex}=R_{i,t}-RF_t
\]

Cette transformation permet de mesurer la rémunération de l’investissement au-delà du rendement sans risque.

### 2.3. Facteur de marché : Mkt-RF

`Mkt-RF` représente le rendement du marché diminué du taux sans risque. Il mesure l’exposition de l’actif au risque général du marché.

Le coefficient \(\beta\) indique la sensibilité du rendement excédentaire de l’ETF aux variations du marché. Un bêta supérieur à 1 traduit une sensibilité estimée plus forte que celle du marché ; un bêta inférieur à 1 traduit une sensibilité plus faible.

### 2.4. Facteur de taille : SMB

SMB signifie *Small Minus Big*. Il mesure la différence de rendement entre des portefeuilles de petites capitalisations et des portefeuilles de grandes capitalisations.

Le coefficient \(s\) mesure la sensibilité du portefeuille à ce facteur. Un coefficient positif indique une exposition orientée vers les petites capitalisations. Il ne représente pas directement le pourcentage du portefeuille investi dans de petites entreprises.

### 2.5. Facteur de valeur : HML

HML signifie *High Minus Low*. Il mesure la différence de rendement entre des portefeuilles présentant un ratio valeur comptable/valeur de marché élevé (« Value ») et des portefeuilles présentant un ratio faible (« Growth »).

Le coefficient \(h\) indique l’exposition du portefeuille à ce facteur. Un coefficient positif est cohérent avec une orientation Value ; un coefficient négatif est cohérent avec une orientation Growth.

### 2.6. Alpha

L’alpha est la constante de la régression. Il représente la partie du rendement excédentaire moyen qui n’est pas expliquée par les facteurs présents dans le modèle.

Un alpha positif et significatif peut signaler une performance résiduelle positive dans le cadre du modèle estimé. Un alpha négatif et significatif indique au contraire une performance résiduelle négative. Un alpha non significatif ne permet pas de conclure qu’il est différent de zéro.

L’alpha n’est pas une preuve automatique de compétence ou d’incompétence : son interprétation dépend de la qualité du modèle, de la période, des coûts, de la stabilité des coefficients et des biais possibles.

## 3. Données et constitution de l’échantillon

### 3.1. Sources de données

Le notebook utilise deux sources principales :

1. **Facteurs Fama-French :** bibliothèque de données de Kenneth French, avec des observations mensuelles pour `Mkt-RF`, `SMB`, `HML` et `RF`.
2. **Cours des ETF :** données Yahoo Finance récupérées avec la bibliothèque Python `yfinance`, à partir des cours ajustés.

Les facteurs sont téléchargés à partir du fichier mensuel Fama-French. Le notebook prévoit également le téléchargement du fichier à cinq facteurs pour l’extension FF5.

### 3.2. ETF étudiés

| Symbole | Description | Rôle dans l’analyse |
|---|---|---|
| SPY | ETF associé au S&P 500 | Référence de marché large |
| IWM | ETF associé au Russell 2000 | Portefeuille principal : petites capitalisations |
| IWD | ETF associé au Russell 1000 Value | Vérification de l’exposition Value |
| IWF | ETF associé au Russell 1000 Growth | Vérification de l’exposition Growth |

IWM constitue le portefeuille principal de l’étude. Les autres ETF permettent de vérifier si les coefficients estimés correspondent aux styles d’investissement attendus.

### 3.3. Période et fréquence

Le téléchargement des cours commence en janvier 2001. Les cours mensuels disponibles couvrent janvier 2001 à octobre 2026, mais les facteurs Fama-French utilisés dans l’échantillon principal s’arrêtent en septembre 2026.

Après calcul des rendements et alignement des dates, l’échantillon commun comprend :

| Indicateur | Résultat |
|---|---:|
| Première observation commune | Février 2001 |
| Dernière observation commune | Septembre 2026 |
| Nombre d’observations mensuelles | **308** |
| Fréquence | Mensuelle |
| Valeurs manquantes dans l’échantillon final | **0** |

Le mois de janvier 2001 sert à calculer le premier rendement mensuel ; la première observation de rendement est donc février 2001.

### 3.4. Préparation des facteurs

Les données Fama-French sont fournies en pourcentage. Le notebook :

- extrait la section mensuelle du fichier ;
- exclut la section annuelle située plus bas dans le fichier ;
- traite les codes `−99,99` et `−999` comme des valeurs manquantes ;
- convertit les pourcentages en nombres décimaux en divisant par 100.

La base des facteurs téléchargée contient **1 203 observations mensuelles**, de juillet 1926 à septembre 2026, pour quatre variables : `Mkt-RF`, `SMB`, `HML` et `RF`. Aucune valeur manquante n’est signalée dans cette base après lecture.

### 3.5. Calcul des rendements et alignement

Les rendements mensuels sont calculés à partir des cours ajustés :

\[
R_{i,t}=\frac{P_{i,t}}{P_{i,t-1}}-1
\]

Les séries sont ensuite alignées sur les dates communes aux ETF et aux facteurs. Les observations ne disposant pas de toutes les variables nécessaires sont exclues de l’échantillon final.

Le rendement excédentaire est enfin calculé par :

\[
R_{i,t}^{ex}=R_{i,t}-RF_t
\]

Toutes les séries sont exprimées en décimal et à fréquence mensuelle. Cette cohérence des unités est essentielle : une confusion entre pourcentage et décimal fausserait les coefficients et les statistiques.

### 3.6. Statistiques descriptives annualisées

Le notebook annualise les moyennes en multipliant les moyennes mensuelles par 12 et les volatilités en multipliant l’écart-type mensuel par \(\sqrt{12}\). Le ratio de Sharpe est calculé comme le rapport entre la moyenne annualisée du rendement excédentaire et sa volatilité annualisée.

| Série | Moyenne annualisée (%) | Volatilité annualisée (%) | Ratio de Sharpe |
|---|---:|---:|---:|
| Mkt-RF | 8,3225 | 15,5295 | 0,5359 |
| SMB | 1,0827 | 8,9825 | 0,1205 |
| HML | 1,2990 | 10,9930 | 0,1182 |
| SPY excédentaire | 7,9058 | 15,0742 | 0,5245 |
| IWM excédentaire | 8,2298 | 19,9532 | 0,4125 |
| IWD excédentaire | 7,2051 | 15,1974 | 0,4741 |
| IWF excédentaire | 8,4968 | 17,0314 | 0,4989 |

**Interprétation :** IWM présente la volatilité annualisée la plus élevée parmi les quatre ETF étudiés, à **19,9532 %**. Son ratio de Sharpe est de **0,4125**, inférieur à celui de SPY (0,5245), IWD (0,4741) et IWF (0,4989). Sur cette période et selon cette mesure, IWM a donc présenté un rendement excédentaire annualisé moyen de 8,2298 % accompagné d’une volatilité relativement importante.

Ces statistiques descriptives ne tiennent pas compte de la structure factorielle des rendements. C’est précisément le rôle des régressions qui suivent.

### 3.7. Contrôle des valeurs extrêmes

Le notebook calcule des scores standardisés et repère les observations dont la valeur absolue dépasse 4 écarts-types.

| Variable | Nombre d’observations avec \(|z|>4\) |
|---|---:|
| Mkt-RF | 0 |
| SMB | 0 |
| HML | 2 |
| SPY excédentaire | 0 |
| IWM excédentaire | 0 |
| IWD excédentaire | 2 |
| IWF excédentaire | 0 |

Deux observations extrêmes sont détectées pour HML et deux pour le rendement excédentaire d’IWD. Le notebook les signale mais ne les supprime pas automatiquement. Cette décision est prudente : les valeurs extrêmes des séries financières peuvent correspondre à de véritables épisodes de marché et leur suppression pourrait modifier artificiellement les résultats.

## 4. Méthodologie économétrique

### 4.1. Modèle de référence : moyenne historique

Le modèle de référence ne comporte aucun facteur explicatif :

\[
R_{i,t}^{ex}=\alpha+\varepsilon_t
\]

Il estime le rendement excédentaire moyen de l’ETF. Il sert de référence minimale pour évaluer l’apport des modèles factoriels.

### 4.2. Modèle CAPM

Le CAPM s’écrit :

\[
R_{i,t}^{ex}=\alpha+\beta(Mkt-RF)_t+\varepsilon_t
\]

Il mesure la part de la rentabilité associée au risque de marché. L’alpha représente le rendement moyen qui reste après prise en compte de cette seule exposition.

### 4.3. Modèle Fama-French à trois facteurs

Le modèle FF3 est formulé comme suit :

\[
R_{i,t}^{ex}=\alpha+\beta(Mkt-RF)_t+sSMB_t+hHML_t+\varepsilon_t
\]

Il ajoute aux variations du marché les facteurs de taille et de valeur. Les coefficients \(\beta\), \(s\) et \(h\) mesurent les sensibilités du rendement excédentaire de l’ETF à chacun de ces facteurs.

L’intérêt de la comparaison CAPM/FF3 est de déterminer si SMB et HML apportent une information explicative supplémentaire et si l’estimation de l’alpha change lorsque ces facteurs sont pris en compte.

### 4.4. Estimation par MCO et erreurs-types HAC

Les coefficients sont estimés par la méthode des moindres carrés ordinaires (MCO). Le notebook utilise des erreurs-types robustes HAC de Newey-West, avec **six retards**.

Cette correction vise à rendre l’inférence statistique plus robuste en présence d’hétéroscédasticité et d’autocorrélation des résidus. Elle modifie les erreurs-types, les statistiques de test et les intervalles de confiance, mais ne remplace pas les coefficients MCO estimés.

### 4.5. Tests statistiques

L’analyse examine :

- le test individuel de l’alpha : \(H_0:\alpha=0\) ;
- les tests individuels des coefficients \(\beta\), \(s\) et \(h\) ;
- les intervalles de confiance à 95 % ;
- le test conjoint de significativité de SMB et HML.

Pour le test conjoint :

\[
H_0:s=0 \quad \text{et} \quad h=0
\]

Si cette hypothèse est rejetée, les deux facteurs ajoutent conjointement une information statistiquement significative dans la spécification étudiée. Cela ne signifie pas nécessairement que chacun des deux coefficients est individuellement significatif.

Le seuil de significativité retenu est de **5 %**. Une p-value inférieure à 0,05 conduit à rejeter l’hypothèse nulle correspondante.

## 5. Résultats des régressions

### 5.1. Coefficients du modèle FF3 pour les quatre ETF

Le tableau suivant présente les estimations obtenues sur l’échantillon commun de 308 mois. Les coefficients de facteurs sont des sensibilités ; ils ne représentent pas des proportions investies. Les p-values affichées comme 0,0000 par le notebook sont présentées ici comme **p < 0,0001**, car une p-value n’est pas exactement égale à zéro.

| ETF | Alpha (%/mois) | p-value alpha | Bêta marché | p-value bêta | SMB | p-value SMB | HML | p-value HML | R² ajusté |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| SPY | −0,0163 | 0,2613 | 0,9892 | < 0,0001 | −0,1388 | < 0,0001 | 0,0149 | 0,0042 | 0,9956 |
| IWM | −0,1198 | 0,0049 | 1,0155 | < 0,0001 | 0,8631 | < 0,0001 | 0,2167 | < 0,0001 | 0,9841 |
| IWD | −0,0771 | 0,1079 | 0,9262 | < 0,0001 | −0,0495 | 0,0137 | 0,3661 | < 0,0001 | 0,9592 |
| IWF | 0,0102 | 0,8142 | 1,0740 | < 0,0001 | −0,1042 | < 0,0001 | −0,3476 | < 0,0001 | 0,9728 |

**Lecture des résultats :**

- **SPY :** son bêta de 0,9892 est proche de 1, ce qui correspond à une forte sensibilité au marché. Son alpha est négatif mais non significatif (p = 0,2613). Son coefficient SMB est négatif et significatif, tandis que HML est positif et significatif mais de faible amplitude.
- **IWM :** son bêta de 1,0155 indique une sensibilité au marché légèrement supérieure à 1. Son coefficient SMB de 0,8631 est fortement positif et significatif, ce qui confirme son exposition aux petites capitalisations. Son coefficient HML de 0,2167 est également positif et significatif : l’exposition estimée comporte donc aussi une composante Value.
- **IWD :** son coefficient HML de 0,3661 est positif et significatif, conformément au style Value attendu. Son coefficient SMB est légèrement négatif et significatif. Son alpha n’est pas significatif au seuil de 5 %.
- **IWF :** son coefficient HML de −0,3476 est négatif et significatif, conformément au style Growth attendu. Son coefficient SMB est également négatif et significatif. Son alpha est très proche de zéro et non significatif.

Les résultats de HML pour IWD et IWF sont particulièrement cohérents avec les styles annoncés de ces ETF. IWM présente bien une forte exposition positive au facteur de taille.

### 5.2. Alpha : comparaison CAPM et FF3

Les intervalles de confiance ci-dessous sont ceux calculés avec les erreurs-types HAC du notebook. Les alphas sont exprimés en pourcentage par mois.

| ETF | Modèle | Alpha (%/mois) | IC 95 % bas | IC 95 % haut | t(alpha) | p-value alpha | R² ajusté |
|---|---|---:|---:|---:|---:|---:|---:|
| SPY | CAPM | −0,0108 | −0,0617 | 0,0400 | −0,4167 | 0,6769 | 0,9894 |
| SPY | FF3 | −0,0163 | −0,0448 | 0,0122 | −1,1233 | 0,2613 | 0,9956 |
| IWM | CAPM | −0,1239 | −0,3926 | 0,1449 | −0,9032 | 0,3664 | 0,8250 |
| IWM | FF3 | **−0,1198** | **−0,2033** | **−0,0364** | **−2,8141** | **0,0049** | **0,9841** |
| IWD | CAPM | −0,0398 | −0,2407 | 0,1611 | −0,3883 | 0,6978 | 0,8894 |
| IWD | FF3 | −0,0771 | −0,1711 | 0,0169 | −1,6075 | 0,1079 | 0,9592 |
| IWF | CAPM | −0,0208 | −0,2101 | 0,1685 | −0,2151 | 0,8297 | 0,9179 |
| IWF | FF3 | 0,0102 | −0,0750 | 0,0954 | 0,2351 | 0,8142 | 0,9728 |

### 5.3. Interprétation détaillée de l’alpha d’IWM

Avec le CAPM, l’alpha estimé d’IWM est de **−0,1239 % par mois**. Cependant, sa p-value est de **0,3664**, supérieure à 0,05, et son intervalle de confiance à 95 % contient zéro \([−0,3926 ; 0,1449]\). On ne peut donc pas conclure que l’alpha CAPM est statistiquement différent de zéro.

Avec le modèle FF3, l’alpha est de **−0,1198 % par mois**. Sa p-value est de **0,0049**, inférieure au seuil de 5 %. Son intervalle de confiance à 95 % \([−0,2033 ; −0,0364]\) est entièrement négatif et ne contient pas zéro.

Cette différence est importante : après prise en compte des facteurs de taille et de valeur, l’alpha d’IWM devient statistiquement significatif et négatif. Selon le modèle FF3 et la période étudiée, IWM présente donc une performance résiduelle mensuelle négative par rapport à ce que prédisent ses expositions aux facteurs.

Il ne faut pas interpréter ce résultat comme la preuve que l’ETF est systématiquement mauvais ou qu’il perd de l’argent. IWM a enregistré un rendement excédentaire annualisé moyen positif de 8,2298 % dans l’échantillon. L’alpha est une mesure relative à la performance expliquée par les facteurs : un alpha négatif signifie que le rendement observé a été inférieur à celui associé aux expositions estimées, et non que le rendement total a nécessairement été négatif.

Pour SPY, IWD et IWF, les alphas FF3 ne sont pas significatifs au seuil de 5 %. Les données ne permettent donc pas de conclure à une performance anormale différente de zéro pour ces trois ETF selon le modèle estimé.

### 5.4. Test conjoint de SMB et HML

Le notebook effectue un test de Wald de l’hypothèse :

\[
H_0:s=0 \quad \text{et} \quad h=0
\]

| ETF | p-value du test conjoint | Décision au seuil de 5 % |
|---|---:|---|
| SPY | < 0,0001 | Rejet de H0 |
| IWM | < 0,0001 | Rejet de H0 |
| IWD | < 0,0001 | Rejet de H0 |
| IWF | < 0,0001 | Rejet de H0 |

Pour les quatre ETF, l’hypothèse selon laquelle SMB et HML sont simultanément nuls est rejetée. Les facteurs de taille et de valeur apportent donc conjointement une information statistique dans les régressions FF3. Ce test conjoint ne signifie pas que les deux coefficients ont la même importance, ni qu’ils sont tous deux individuellement significatifs dans chaque portefeuille.

## 6. Comparaison de la qualité des modèles

### 6.1. R² ajusté, RMSE, MAE et R² hors échantillon

L’évaluation hors échantillon utilise une fenêtre extensible. Les 60 premières observations servent de période minimale d’apprentissage. Les prédictions sont ensuite calculées sur les mois suivants, ce qui donne **248 mois évalués**, de février 2006 à septembre 2026.

| ETF | Modèle | R² ajusté | RMSE (%) | MAE (%) | R² hors échantillon |
|---|---|---:|---:|---:|---:|
| SPY | Moyenne historique | 0,0000 | 4,3749 | 3,3523 | 0,0000 |
| SPY | CAPM | 0,9894 | 0,4247 | 0,3300 | 0,9906 |
| SPY | FF3 | 0,9956 | 0,2643 | 0,1980 | 0,9963 |
| IWM | Moyenne historique | 0,0000 | 5,7945 | 4,4175 | 0,0000 |
| IWM | CAPM | 0,8250 | 2,3509 | 1,8158 | 0,8354 |
| IWM | FF3 | **0,9841** | **0,7250** | **0,5496** | **0,9843** |
| IWD | Moyenne historique | 0,0000 | 4,5005 | 3,3999 | 0,0000 |
| IWD | CAPM | 0,8894 | 1,4986 | 1,0816 | 0,8891 |
| IWD | FF3 | 0,9592 | 0,9031 | 0,6716 | 0,9597 |
| IWF | Moyenne historique | 0,0000 | 4,8155 | 3,7588 | 0,0000 |
| IWF | CAPM | 0,9179 | 1,3814 | 1,0596 | 0,9177 |
| IWF | FF3 | 0,9728 | 0,8202 | 0,6174 | 0,9710 |

### 6.2. Interprétation pour IWM

Le passage du CAPM au modèle FF3 améliore fortement les indicateurs d’IWM :

- le R² ajusté augmente de **0,8250 à 0,9841** ;
- le RMSE hors échantillon diminue de **2,3509 % à 0,7250 % par mois** ;
- le MAE diminue de **1,8158 % à 0,5496 % par mois** ;
- le R² hors échantillon passe de **0,8354 à 0,9843**.

La réduction du RMSE est d’environ **69,2 %** par rapport au CAPM. Cela signifie que, dans le protocole d’évaluation du notebook, l’ajout des facteurs SMB et HML réduit nettement l’erreur d’explication des rendements d’IWM.

La même tendance apparaît pour les trois autres ETF : le modèle FF3 présente un R² ajusté et un R² hors échantillon plus élevés, ainsi qu’un RMSE et un MAE plus faibles que le CAPM.

### 6.3. Limite de l’évaluation hors échantillon

Le notebook estime les coefficients uniquement à partir des observations antérieures au mois évalué, ce qui respecte l’ordre chronologique. Toutefois, il utilise les facteurs réalisés du mois \(t\) pour expliquer le rendement du même mois \(t\).

Il s’agit donc d’une évaluation de la relation factorielle conditionnelle aux facteurs observés, utile pour l’attribution de performance, mais pas d’une prévision ex ante disponible avant le début du mois. Une véritable prévision ex ante nécessiterait également de prévoir les facteurs ou de disposer de leurs valeurs anticipées.

## 7. Diagnostics économétriques

Les diagnostics ci-dessous sont calculés pour IWM avec le modèle FF3. Le notebook utilise une régression MCO classique non robuste pour les tests de diagnostic, alors que les tests de significativité des coefficients présentés précédemment reposent sur des erreurs-types HAC.

| Diagnostic | Résultat | Interprétation |
|---|---:|---|
| Durbin-Watson | 2,0233 | Valeur proche de 2 : pas d’indice marqué d’autocorrélation de premier ordre |
| Breusch-Pagan — p-value | 0,8780 | Non-rejet de l’hypothèse d’homoscédasticité |
| Jarque-Bera — p-value | 0,1653 | Non-rejet de l’hypothèse de normalité des résidus |
| VIF Mkt-RF | 1,0973 | Pas de multicolinéarité préoccupante |
| VIF SMB | 1,1039 | Pas de multicolinéarité préoccupante |
| VIF HML | 1,0064 | Pas de multicolinéarité préoccupante |

### 7.1. Autocorrélation

La statistique de Durbin-Watson est de **2,0233**, très proche de 2. Cela ne met pas en évidence une autocorrélation de premier ordre importante des résidus selon ce diagnostic.

### 7.2. Hétéroscédasticité

Le test de Breusch-Pagan donne une p-value de **0,8780**, supérieure à 0,05. L’hypothèse d’homoscédasticité n’est donc pas rejetée. Le test ne fournit pas d’indice statistique d’une variance non constante des résidus dans cette spécification.

### 7.3. Normalité

Le test de Jarque-Bera donne une p-value de **0,1653**, supérieure à 0,05. L’hypothèse de normalité des résidus n’est pas rejetée au seuil de 5 %.

### 7.4. Multicolinéarité

Les trois VIF sont proches de 1 et très inférieurs au seuil indicatif de 5 utilisé dans le notebook. Les facteurs explicatifs ne présentent donc pas de multicolinéarité préoccupante selon cet indicateur.

Ces tests sont des diagnostics, non des garanties absolues de validité. Le recours aux erreurs-types HAC reste utile pour renforcer l’inférence statistique face à d’éventuelles formes d’hétéroscédasticité et d’autocorrélation.

## 8. Stabilité des coefficients dans le temps

### 8.1. Comparaison par sous-périodes

Le notebook divise les 308 observations en deux sous-périodes de 154 mois chacune et réestime le modèle FF3 pour IWM.

| Sous-période | N | Alpha (%/mois) | R² ajusté | Bêta marché | SMB | HML |
|---|---:|---:|---:|---:|---:|---:|
| Février 2001 – novembre 2013 | 154 | −0,1006 | 0,9857 | 1,0146 | 0,8562 | 0,1870 |
| Décembre 2013 – septembre 2026 | 154 | −0,1303 | 0,9824 | 1,0198 | 0,8657 | 0,2341 |

### 8.2. Interprétation

Les coefficients de marché sont très proches dans les deux sous-périodes : 1,0146 puis 1,0198. Le bêta d’IWM apparaît donc relativement stable autour de 1.

Le coefficient SMB reste également proche : 0,8562 dans la première moitié et 0,8657 dans la seconde. L’exposition d’IWM au facteur de taille est ainsi positive et relativement stable.

Le coefficient HML augmente légèrement, de 0,1870 à 0,2341, tout en restant positif. L’exposition Value d’IWM semble donc un peu plus forte dans la seconde sous-période.

L’alpha reste négatif dans les deux périodes : −0,1006 % puis −0,1303 % par mois. Le R² ajusté reste élevé, à 0,9857 et 0,9824. Ces comparaisons suggèrent une certaine stabilité des expositions, mais elles ne constituent pas un test formel de rupture structurelle. Les p-values des coefficients par sous-période ne sont pas présentées dans le tableau généré.

### 8.3. Régressions glissantes

Le notebook complète cette comparaison par des régressions glissantes sur une fenêtre de **36 mois**. À chaque date, le modèle FF3 est estimé sur les 36 mois précédents.

Cette méthode permet de visualiser l’évolution des sensibilités au marché, à SMB et à HML. Elle est utile pour repérer d’éventuelles variations des expositions qui pourraient être masquées par une estimation unique sur toute la période.

Le graphique généré par le notebook doit être intégré au rapport final sous le titre : **« Évolution des coefficients FF3 d’IWM — fenêtre glissante de 36 mois »**. Les résultats numériques ponctuels de chaque fenêtre ne sont pas reproduits dans le tableau de sortie textuelle ; leur analyse est donc principalement graphique.

## 9. Extension au modèle Fama-French à cinq facteurs

Le notebook estime également une extension FF5 qui ajoute deux facteurs au modèle initial :

- **RMW (Robust Minus Weak) :** facteur lié à la profitabilité ;
- **CMA (Conservative Minus Aggressive) :** facteur lié à la politique d’investissement.

La spécification générale devient :

\[
R_{i,t}^{ex}=\alpha+\beta(Mkt-RF)_t+sSMB_t+hHML_t+rRMW_t+cCMA_t+\varepsilon_t
\]

Les résultats de l’extension sont les suivants :

| ETF | R² ajusté FF3 | R² ajusté FF5 | Alpha FF3 (%/mois) | Alpha FF5 (%/mois) | p-value alpha FF5 |
|---|---:|---:|---:|---:|---:|
| SPY | 0,9955 | 0,9958 | −0,0147 | −0,0324 | 0,0217 |
| IWM | 0,9867 | 0,9881 | −0,1328 | −0,0877 | 0,0092 |
| IWD | 0,9587 | 0,9619 | −0,0763 | −0,1258 | 0,0057 |
| IWF | 0,9740 | 0,9759 | 0,0109 | 0,0509 | 0,1774 |

**Attention à la comparaison des échantillons :** cette table est calculée dans la partie FF5 du notebook à partir de la base commune aux facteurs FF5 et aux ETF. Les valeurs FF3 de cette table sont donc recalculées sur cet échantillon propre à l’extension. Elles diffèrent légèrement des valeurs FF3 de la section 5, qui sont estimées sur l’échantillon principal de 308 mois. Pour comparer FF3 et FF5, il faut privilégier les deux colonnes de R² calculées dans cette même table plutôt que comparer directement le R² FF5 avec le R² de la table principale.

### 9.1. Interprétation pour IWM

Pour IWM, le R² ajusté passe de **0,9867 avec FF3 à 0,9881 avec FF5**. L’amélioration est positive mais limitée : elle représente un gain de 0,0014 en R² ajusté.

L’alpha passe de −0,1328 % par mois dans la version FF3 recalculée sur l’échantillon FF5 à −0,0877 % par mois dans FF5. Il demeure négatif et statistiquement significatif, avec une p-value de **0,0092**.

L’ajout de RMW et CMA améliore donc légèrement l’ajustement d’IWM et réduit en valeur absolue l’alpha négatif, sans le rendre non significatif. Les résultats disponibles ne permettent toutefois pas de mesurer ici la significativité individuelle de RMW et CMA, car leurs coefficients et p-values ne figurent pas dans le tableau exporté.

Pour les autres ETF, le R² ajusté augmente également légèrement. Les alphas FF5 de SPY et IWD sont significatifs au seuil de 5 % selon les p-values affichées, tandis que celui d’IWF ne l’est pas.

## 10. Discussion générale

Les résultats répondent à la question centrale du projet. Le risque de marché seul ne suffit pas à décrire les rendements d’IWM aussi précisément que le modèle FF3. L’ajout des facteurs de taille et de valeur améliore considérablement l’ajustement statistique et la performance de l’évaluation chronologique.

Le coefficient SMB élevé et positif d’IWM est cohérent avec son exposition aux petites capitalisations. Le coefficient HML positif montre que l’ETF présente également une sensibilité au facteur Value dans la période étudiée. Ces expositions expliquent une part importante des variations de ses rendements excédentaires.

Le résultat relatif à l’alpha mérite une attention particulière. Sous le CAPM, l’alpha d’IWM n’est pas significatif. Sous FF3, il devient négatif et significatif. Cela signifie que la prise en compte des facteurs de taille et de valeur modifie l’interprétation de la performance résiduelle : une fois les expositions contrôlées, l’alpha estimé est inférieur à zéro.

Les résultats hors échantillon vont dans le même sens concernant le pouvoir explicatif : FF3 réduit les erreurs par rapport au CAPM pour chacun des quatre ETF. Il faut néanmoins rappeler que cette procédure utilise les facteurs réalisés pendant le mois évalué. Elle mesure donc une relation conditionnelle et ne doit pas être présentée comme une stratégie de prévision disponible à l’avance.

Enfin, la relative stabilité des coefficients d’IWM entre les deux moitiés de l’échantillon renforce l’idée que ses expositions de marché et de taille sont persistantes sur la période. La légère hausse du coefficient HML dans la seconde moitié indique toutefois que l’exposition Value n’est pas parfaitement constante.

## 11. Limites et risques d’interprétation

### 11.1. Biais de survie

Les ETF retenus existent encore aujourd’hui. Une étude limitée aux fonds survivants peut donner une image trop favorable de la performance du secteur, car les fonds disparus ou liquidés peuvent être absents de l’échantillon. Il faut distinguer ce biais de la sélection des entreprises actuellement présentes dans un indice : le notebook étudie des ETF et ne reconstitue pas directement les rendements historiques à partir des seuls constituants actuels.

### 11.2. Alpha significatif par hasard et tests multiples

Le projet examine plusieurs ETF et plusieurs modèles. Plus le nombre de tests augmente, plus le risque d’obtenir au moins un résultat significatif par hasard augmente. Un alpha significatif isolé doit donc être interprété avec prudence. Une correction pour tests multiples, par exemple la méthode de Holm, pourrait être envisagée si plusieurs alphas sont testés simultanément.

### 11.3. Instabilité des facteurs

Les expositions peuvent évoluer dans le temps. Un coefficient estimé sur toute la période résume une relation moyenne et peut masquer des changements de régime économique ou de composition de l’ETF. Les analyses par sous-périodes et les régressions glissantes complètent donc utilement les résultats globaux.

### 11.4. Frais et coûts de transaction

Les ETF supportent des frais de gestion et peuvent présenter un écart de réplication par rapport à leur indice. Les facteurs Fama-French sont des portefeuilles théoriques et ne représentent pas nécessairement les coûts réellement supportés par un investisseur. Une partie d’un alpha négatif éventuel peut être liée aux frais et aux coûts de mise en œuvre.

### 11.5. Hypothèses économétriques

Les rendements financiers peuvent présenter des résidus non normaux, une volatilité variable et une autocorrélation. Les erreurs-types HAC améliorent la robustesse de l’inférence face à certaines formes d’hétéroscédasticité et d’autocorrélation, mais elles ne corrigent pas une mauvaise spécification du modèle ni une variable explicative omise.

### 11.6. Limite de représentativité

L’étude porte sur des ETF du marché américain. Les résultats ne peuvent pas être généralisés automatiquement aux marchés internationaux, aux fonds moins liquides ou aux actions individuelles.

### 11.7. Association statistique et causalité

Les coefficients factoriels mesurent des sensibilités statistiques. Ils ne démontrent pas à eux seuls un lien de causalité entre les facteurs et les rendements.

### 11.8. Données et reproductibilité

Les cours proviennent de Yahoo Finance via `yfinance`, une source dont la disponibilité et le format peuvent évoluer. Les résultats dépendent également de la date de téléchargement, des données ajustées disponibles et de la version des fichiers Fama-French. Pour assurer la reproductibilité, il est recommandé de conserver les fichiers de données téléchargés, le notebook exécuté et la date d’exécution.

## 12. Conclusion générale

L’objectif de ce projet était de déterminer si la rentabilité d’un portefeuille pouvait être expliquée par le marché seul ou si les facteurs de taille et de valeur apportaient une information supplémentaire. L’étude a comparé le CAPM et le modèle Fama-French à trois facteurs sur quatre ETF américains, en mettant l’accent sur IWM.

Les résultats empiriques montrent que le modèle FF3 explique nettement mieux les rendements d’IWM que le CAPM. Son R² ajusté atteint **0,9841**, contre **0,8250** pour le CAPM. Le facteur SMB présente un coefficient positif et élevé (**0,8631**), confirmant l’exposition d’IWM aux petites capitalisations. Le facteur HML est également positif (**0,2167**), indiquant une sensibilité supplémentaire au style Value.

L’analyse de l’alpha apporte un résultat important : l’alpha CAPM d’IWM n’est pas statistiquement significatif, alors que l’alpha FF3 est négatif et significatif (**−0,1198 % par mois ; p = 0,0049**). Son intervalle de confiance à 95 % est entièrement négatif. Ainsi, après contrôle du marché, de la taille et de la valeur, les données ne mettent pas en évidence une surperformance anormale positive d’IWM sur la période étudiée ; elles indiquent au contraire une performance résiduelle négative selon le modèle.

L’évaluation chronologique confirme l’intérêt explicatif de FF3 : le R² hors échantillon d’IWM est de **0,9843**, et son RMSE est de **0,7250 % par mois**, nettement inférieur aux **2,3509 %** du CAPM. Les diagnostics statistiques ne signalent pas de problème majeur selon les tests réalisés, et les coefficients restent relativement proches entre les deux sous-périodes.

L’extension FF5 apporte une amélioration modeste du R² ajusté d’IWM, de **0,9867 à 0,9881** sur l’échantillon propre à cette comparaison. L’alpha reste néanmoins négatif et significatif (**−0,0877 % par mois ; p = 0,0092**).

En conclusion, le modèle Fama-French à trois facteurs constitue une spécification plus pertinente que le CAPM pour expliquer les rendements des ETF étudiés, en particulier IWM. Il permet de mieux distinguer les expositions systématiques de la performance résiduelle. Ces résultats doivent toutefois être interprétés en tenant compte des limites liées aux ETF survivants, aux tests multiples, à la stabilité temporelle, aux coûts et à la nature conditionnelle de l’évaluation hors échantillon.

## 13. Sources et références

### Sources de données

- **Kenneth R. French Data Library — facteurs Fama-French à trois facteurs :**  
  https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_Factors_CSV.zip
- **Kenneth R. French Data Library — facteurs Fama-French à cinq facteurs :**  
  https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_5_Factors_2x3_CSV.zip
- **Yahoo Finance — cours historiques des ETF :** données téléchargées dans le notebook avec la bibliothèque Python `yfinance`.

### Références théoriques

- Sharpe, W. F. (1964). *Capital Asset Prices: A Theory of Market Equilibrium under Conditions of Risk*. The Journal of Finance, 19(3), 425–442.
- Fama, E. F., & French, K. R. (1993). *Common risk factors in the returns on stocks and bonds*. Journal of Financial Economics, 33(1), 3–56.
- Fama, E. F., & French, K. R. (2015). *A five-factor asset pricing model*. Journal of Financial Economics, 116(1), 1–22.
- Newey, W. K., & West, K. D. (1987). *A Simple, Positive Semi-definite, Heteroskedasticity and Autocorrelation Consistent Covariance Matrix*. Econometrica, 55(3), 703–708.

---

## Annexe — Synthèse des principaux résultats à retenir

| Question | Résultat |
|---|---|
| L’échantillon est-il suffisamment documenté ? | 308 mois communs, février 2001 à septembre 2026, aucune valeur manquante |
| SMB est-il important pour IWM ? | Oui : coefficient 0,8631 ; p < 0,0001 |
| HML est-il important pour IWM ? | Oui : coefficient 0,2167 ; p < 0,0001 |
| L’alpha CAPM d’IWM est-il significatif ? | Non : alpha −0,1239 %/mois ; p = 0,3664 |
| L’alpha FF3 d’IWM est-il significatif ? | Oui, négatif : −0,1198 %/mois ; p = 0,0049 |
| FF3 améliore-t-il l’ajustement d’IWM ? | Oui : R² ajusté de 0,8250 à 0,9841 |
| FF3 améliore-t-il l’évaluation chronologique d’IWM ? | Oui : R² hors échantillon 0,9843 ; RMSE 0,7250 % |
| Les diagnostics signalent-ils un problème majeur ? | Aucun problème majeur détecté par les tests présentés |
| Les expositions d’IWM sont-elles stables ? | Relativement stables entre les deux sous-périodes |
| FF5 améliore-t-il légèrement FF3 pour IWM ? | Oui : R² ajusté de 0,9867 à 0,9881 sur l’échantillon FF5 |
| L’alpha FF5 d’IWM devient-il non significatif ? | Non : −0,0877 %/mois ; p = 0,0092 |

**Note finale :** les valeurs numériques de ce rapport proviennent des sorties enregistrées dans le notebook exécuté transmis. Les résultats FF3 de l’échantillon principal et ceux de la comparaison FF3/FF5 sont présentés séparément, car cette dernière utilise l’échantillon commun propre aux facteurs FF5.
