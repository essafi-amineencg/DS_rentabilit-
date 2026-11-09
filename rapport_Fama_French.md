# Rapport d'analyse --- Facteurs de risque d'un portefeuille : modèle de Fama-French

**Projet F5 --- Finance et science des données**\
**Méthode :** régressions linéaires par moindres carrés ordinaires
(MCO), erreurs-types robustes HAC (Newey-West)\
**Outil :** Python --- pandas, yfinance, statsmodels, matplotlib\
**Portefeuille principal :** ETF IWM (Russell 2000)\
**Date :** à compléter

------------------------------------------------------------------------

## Résumé exécutif

Ce travail étudie les facteurs qui expliquent la rentabilité mensuelle
de plusieurs ETF américains, avec une attention particulière portée à
IWM, qui réplique l'indice Russell 2000 et représente les petites
capitalisations américaines. La question centrale consiste à déterminer
si le risque de marché suffit à expliquer les rendements de ce
portefeuille ou si les facteurs de taille et de valeur du modèle de
Fama-French à trois facteurs apportent une information supplémentaire.

L'analyse compare trois spécifications : une moyenne historique utilisée
comme référence, le CAPM à un facteur et le modèle Fama-French à trois
facteurs (FF3). Les rendements des ETF sont calculés à partir de cours
ajustés, puis transformés en rendements excédentaires en retranchant le
taux sans risque. Les régressions sont estimées par MCO et l'inférence
s'appuie sur des erreurs-types robustes HAC de Newey-West.

Le notebook prévoit également une comparaison hors échantillon
chronologique, des diagnostics de résidus, une analyse de stabilité des
coefficients et une extension au modèle à cinq facteurs. **Le fichier
notebook fourni ne contient toutefois aucun résultat d'exécution : les
coefficients, statistiques de test, intervalles de confiance et
indicateurs de performance ne sont donc pas disponibles à ce stade.**
Ils sont laissés explicitement à compléter après exécution du notebook.
Aucune conclusion empirique chiffrée ne peut être formulée sans ces
résultats.

## 1. Introduction et problématique

La rentabilité d'un portefeuille financier ne dépend pas uniquement de
la performance générale du marché. Elle peut également être associée à
des caractéristiques systématiques des titres détenus, notamment leur
taille et leur style d'investissement. Une performance supérieure à
celle du marché ne signifie donc pas nécessairement que le gestionnaire
a créé une valeur anormale : elle peut provenir d'une exposition à des
facteurs de risque rémunérés.

Le modèle de marché traditionnel, représenté par le CAPM, explique le
rendement excédentaire d'un actif à l'aide d'un seul facteur : le
rendement excédentaire du marché. Le modèle de Fama-French à trois
facteurs enrichit cette approche en ajoutant deux facteurs : SMB, qui
représente l'écart de performance entre petites et grandes
capitalisations, et HML, qui représente l'écart entre titres « value »
et titres « growth ».

### 1.1 Question principale

Sur la période étudiée, la rentabilité mensuelle en excès de l'ETF
**IWM** s'explique-t-elle uniquement par le risque de marché (CAPM), ou
l'ajout des facteurs de taille (SMB) et de valeur (HML) améliore-t-il
l'explication de ses rendements ? Après contrôle de ces expositions,
l'ETF présente-t-il un alpha statistiquement différent de zéro ?

### 1.2 Questions complémentaires

-   Les expositions estimées correspondent-elles aux styles attendus des
    ETF étudiés ?
-   Les coefficients restent-ils relativement stables au cours du temps
    ?
-   Le modèle FF3 réduit-il l'erreur par rapport au CAPM et à une
    moyenne historique lorsqu'il est évalué chronologiquement ?
-   L'ajout des facteurs de profitabilité et d'investissement dans le
    modèle FF5 apporte-t-il une amélioration supplémentaire ?

### 1.3 Hypothèses de recherche

L'hypothèse nulle portant sur l'alpha est :

\[ H_0 : `\alpha `{=tex}= 0 \]

Elle signifie qu'après contrôle des facteurs inclus dans le modèle, le
rendement moyen anormal n'est pas statistiquement différent de zéro.

Les attentes de signe formulées dans le notebook sont les suivantes :

  -----------------------------------------------------------------------
  ETF                     Style représenté        Exposition attendue
  ----------------------- ----------------------- -----------------------
  SPY                     Grandes capitalisations (`\beta`{=tex}) proche
                          / marché large          de 1 ; SMB et HML
                                                  proches de 0

  IWM                     Petites capitalisations SMB positif

  IWD                     Style Value             HML positif

  IWF                     Style Growth            HML négatif
  -----------------------------------------------------------------------

Ces signes sont des attentes théoriques à confronter aux estimations, et
non des résultats déjà établis.

## 2. Cadre théorique : les facteurs de risque

### 2.1 Rendement et rendement excédentaire

Le rendement mensuel simple d'un ETF est calculé à partir de son cours
ajusté :

\[ R\_{i,t}=`\frac{P_{i,t}}{P_{i,t-1}}`{=tex}-1 \]

où (P\_{i,t}) désigne le cours ajusté à la fin du mois (t).
L'utilisation du cours ajusté permet de prendre en compte les
distributions et opérations intégrées dans cette série de prix.

Le rendement excédentaire correspond au rendement de l'ETF diminué du
taux sans risque :

\[ R\_{i,t}\^{ex}=R\_{i,t}-RF_t \]

Cette transformation permet de comparer le rendement de l'actif à une
référence sans risque exprimée sur la même période.

### 2.2 Facteur de marché : Mkt-RF

`Mkt-RF` représente le rendement du marché diminué du taux sans risque.
Il mesure l'exposition de l'ETF au risque général du marché.

Le coefficient (`\beta`{=tex}) indique la sensibilité du rendement
excédentaire de l'ETF aux variations de ce facteur. Un bêta supérieur à
1 correspond à une sensibilité estimée plus forte que celle du marché ;
un bêta inférieur à 1 indique une sensibilité plus faible.

### 2.3 Facteur de taille : SMB

SMB signifie *Small Minus Big*. Il mesure la différence de rendement
entre des portefeuilles de petites capitalisations et des portefeuilles
de grandes capitalisations.

Le coefficient (s) mesure la sensibilité du portefeuille à ce facteur.
Un coefficient positif indique une exposition orientée vers les petites
capitalisations dans le cadre de la construction du facteur. Il ne
représente pas directement le pourcentage du portefeuille investi dans
des petites entreprises.

### 2.4 Facteur de valeur : HML

HML signifie *High Minus Low*. Il mesure la différence de rendement
entre des portefeuilles présentant un ratio valeur comptable/valeur de
marché élevé (« value ») et ceux présentant un ratio faible (« growth
»).

Le coefficient (h) indique l'exposition du portefeuille à ce facteur. Un
coefficient positif est cohérent avec une exposition de style Value ; un
coefficient négatif est cohérent avec une orientation Growth.

### 2.5 Alpha

L'alpha est la constante de la régression. Il représente la partie du
rendement excédentaire moyen qui n'est pas expliquée par les facteurs
présents dans le modèle.

Un alpha positif n'est pas automatiquement une preuve de compétence de
gestion. Il faut vérifier sa significativité statistique, son intervalle
de confiance, sa stabilité et la robustesse du modèle. Un alpha non
significatif ne permet pas de rejeter l'hypothèse selon laquelle il est
nul.

## 3. Données et constitution de l'échantillon

### 3.1 Sources de données

Le notebook utilise deux sources :

1.  **Facteurs Fama-French :** bibliothèque de données de Kenneth
    French, avec des observations mensuelles pour `Mkt-RF`, `SMB`, `HML`
    et `RF`.
2.  **Cours des ETF :** données Yahoo Finance récupérées avec
    `yfinance`, à partir des cours ajustés.

La récupération via Yahoo Finance n'est pas présentée comme une API
officielle et peut évoluer. Le notebook prévoit une mise en cache locale
afin de limiter les téléchargements répétés.

### 3.2 Fonds étudiés

  Symbole   Description dans l'étude      Fonction dans l'analyse
  --------- ----------------------------- ----------------------------------
  IWM       ETF associé au Russell 2000   Portefeuille principal
  SPY       ETF associé au S&P 500        Référence de marché large
  IWD       ETF orienté Value             Vérification de l'exposition HML
  IWF       ETF orienté Growth            Vérification de l'exposition HML

Le portefeuille principal retenu est IWM. Les trois autres ETF servent à
comparer les expositions estimées à des styles d'investissement
différents.

### 3.3 Période et fréquence

Le notebook fixe le début de la période de recherche au **1er janvier
2001** et travaille à fréquence mensuelle. La date de fin dépend des
données effectivement disponibles au moment de l'exécution.

L'échantillon final est déterminé par l'intersection des dates
disponibles pour les ETF et les facteurs. La période effective et le
nombre d'observations doivent être renseignés après exécution :

  Élément                            Valeur à renseigner
  ---------------------------------- -------------------------------------
  Première observation commune       À calculer
  Dernière observation commune       À calculer
  Nombre d'observations mensuelles   À calculer
  Nombre de mois retenus par ETF     Échantillon commun dans le notebook

### 3.4 Préparation des facteurs

Les données Fama-French sont fournies en pourcentage. Le notebook :

-   extrait la section mensuelle du fichier ;
-   exclut la section annuelle située plus bas dans le fichier ;
-   traite les codes `-99.99` et `-999` comme des valeurs manquantes ;
-   convertit les pourcentages en nombres décimaux en divisant par 100.

Cette conversion est essentielle. Par exemple, une valeur de 1,5 % doit
être représentée par 0,015 dans les calculs de rendement.

### 3.5 Calcul des rendements et alignement

Les rendements mensuels sont calculés à partir des cours ajustés :

\[ R\_{i,t}=`\text{pct\_change}`{=tex}(P\_{i,t}) \]

Les séries sont ensuite alignées sur les dates mensuelles communes aux
ETF et aux facteurs. Les observations ne disposant pas de toutes les
variables nécessaires sont retirées de l'échantillon utilisé par les
régressions.

Enfin, le rendement excédentaire de chaque ETF est calculé par :

\[ R\_{i,t}\^{ex}=R\_{i,t}-RF_t \]

Toutes les séries doivent être exprimées en décimal et utiliser la même
fréquence. Un mélange entre pourcentages et décimaux produirait des
coefficients et des statistiques erronés.

## 4. Méthodologie économétrique

### 4.1 Modèle de référence : moyenne historique

Le modèle de référence ne comporte aucun facteur explicatif :

\[ R\_{i,t}\^{ex}=`\alpha`{=tex}+`\varepsilon`{=tex}\_t \]

Il estime simplement le rendement excédentaire moyen de l'ETF. Il sert
de référence minimale pour évaluer l'apport des modèles factoriels.

### 4.2 Modèle CAPM

Le CAPM s'écrit :

\[
R\_{i,t}\^{ex}=`\alpha`{=tex}+`\beta`{=tex}(Mkt-RF)\_t+`\varepsilon`{=tex}\_t
\]

Il mesure la part de la rentabilité associée au risque de marché.
L'alpha représente le rendement moyen qui reste après prise en compte de
cette seule exposition.

### 4.3 Modèle Fama-French à trois facteurs

Le modèle FF3 est formulé comme suit :

\[
R\_{i,t}\^{ex}=`\alpha`{=tex}+`\beta`{=tex}(Mkt-RF)\_t+sSMB_t+hHML_t+`\varepsilon`{=tex}\_t
\]

Il ajoute aux variations du marché les facteurs de taille et de valeur.
Les coefficients (`\beta`{=tex}), (s) et (h) mesurent les sensibilités
du rendement excédentaire de l'ETF à chacun de ces facteurs.

L'intérêt de la comparaison CAPM/FF3 est de déterminer si SMB et HML
apportent une information explicative supplémentaire et si l'estimation
de l'alpha change lorsque ces facteurs sont pris en compte.

### 4.4 Estimation par MCO et erreurs-types HAC

Les coefficients sont estimés par la méthode des moindres carrés
ordinaires (MCO). Le notebook utilise des erreurs-types robustes HAC de
Newey-West, avec un nombre de retards fixé à six.

Cette correction vise à rendre l'inférence statistique plus robuste en
présence d'hétéroscédasticité et d'autocorrélation des résidus. Elle
modifie les erreurs-types, les statistiques de test et les intervalles
de confiance, mais ne remplace pas les coefficients MCO estimés.

### 4.5 Tests statistiques

L'analyse doit notamment examiner :

-   le test individuel de l'alpha : (H_0:`\alpha=0`{=tex}) ;
-   les tests individuels des coefficients (`\beta`{=tex}), (s) et (h) ;
-   les intervalles de confiance à 95 % ;
-   le test conjoint de significativité de SMB et HML.

Pour le test conjoint :

\[ H_0:s=0 `\quad `{=tex}`\text{et}`{=tex} `\quad `{=tex}h=0 \]

Si cette hypothèse est rejetée, les deux facteurs ajoutent conjointement
une information statistiquement significative dans la spécification
étudiée. Cela ne signifie pas nécessairement que chacun des deux
coefficients est individuellement significatif.

La décision statistique est généralement prise au seuil de 5 %. Une
p-value inférieure à 0,05 conduit à rejeter l'hypothèse nulle
correspondante, sous réserve de la validité du modèle et de la procédure
d'inférence.

## 5. Comparaison des modèles et indicateurs

### 5.1 R² ajusté

Le R² mesure la part de la variation de la variable dépendante expliquée
par le modèle dans l'échantillon. Le R² ajusté tient compte du nombre de
variables explicatives et pénalise l'ajout de facteurs qui n'améliorent
pas suffisamment l'ajustement.

Un R² ajusté plus élevé indique un meilleur ajustement relatif dans
l'échantillon, mais ne suffit pas à démontrer une meilleure capacité de
généralisation.

### 5.2 RMSE et MAE

La racine de l'erreur quadratique moyenne (RMSE) pénalise davantage les
erreurs importantes :

\[ RMSE=`\sqrt{\frac{1}{n}\sum_{t=1}^{n}(y_t-\hat y_t)^2}`{=tex} \]

L'erreur absolue moyenne (MAE) mesure l'erreur absolue moyenne :

\[
MAE=`\frac{1}{n}`{=tex}`\sum`{=tex}\_{t=1}\^{n}\|y_t-`\hat `{=tex}y_t\|
\]

Des valeurs plus faibles indiquent des erreurs d'ajustement plus
faibles. Les tableaux du notebook affichent ces métriques en
pourcentage.

### 5.3 R² hors échantillon

Le R² hors échantillon compare les erreurs du modèle à celles d'une
référence fondée sur la moyenne historique. Une valeur positive indique
que le modèle fait mieux que cette référence selon la définition retenue
dans le notebook ; une valeur négative indique qu'il fait moins bien.

Il faut distinguer ce résultat du R² calculé sur l'échantillon
d'estimation.

## 6. Évaluation chronologique

Le notebook évite un découpage aléatoire, inadapté aux séries
temporelles. Il utilise une fenêtre extensible (*expanding window*).

À chaque mois (t), les coefficients sont estimés uniquement sur les
observations antérieures à (t), à partir d'un minimum de 60 mois
d'apprentissage. Les coefficients obtenus sont ensuite appliqués aux
facteurs observés au mois (t).

Cette procédure permet d'évaluer si les relations estimées sur le passé
conservent un pouvoir explicatif au fil du temps. La référence est la
moyenne historique du rendement excédentaire, calculée sur le même
historique disponible.

**Limite d'interprétation importante :** les facteurs du mois (t) sont
supposés connus au moment de l'évaluation. Il s'agit donc d'une
attribution ou d'un *nowcast* conditionnel aux facteurs observés, et non
d'une véritable prévision du rendement réalisée avant le début du mois.
Une prévision ex ante nécessiterait de disposer de prévisions des
facteurs ou d'informations disponibles à la date de décision.

## 7. Diagnostics et stabilité des coefficients

### 7.1 Diagnostic des résidus

Le notebook prévoit plusieurs indicateurs pour examiner les résidus du
modèle FF3 appliqué à IWM :

  -----------------------------------------------------------------------
  Diagnostic              Ce qu'il examine        Lecture générale
  ----------------------- ----------------------- -----------------------
  Durbin-Watson           Autocorrélation de      Une valeur proche de 2
                          premier ordre des       est généralement
                          résidus                 compatible avec une
                                                  faible autocorrélation
                                                  de premier ordre

  Jarque-Bera             Écart à la normalité    Une p-value faible
                                                  indique un rejet de la
                                                  normalité

  Breusch-Pagan           Hétéroscédasticité      Une p-value faible
                                                  indique une variance
                                                  des résidus non
                                                  constante

  VIF                     Multicolinéarité entre  Des valeurs élevées
                          facteurs                signalent une forte
                                                  dépendance linéaire
                                                  entre variables
                                                  explicatives
  -----------------------------------------------------------------------

Ces diagnostics aident à évaluer les limites de la régression. Ils ne
doivent pas être interprétés isolément et ne garantissent pas que toutes
les hypothèses du modèle sont satisfaites.

### 7.2 Analyse par sous-périodes

Le notebook divise l'échantillon en deux moitiés chronologiques et
réestime le modèle FF3 sur chacune. La comparaison des coefficients
permet de repérer des changements d'exposition entre la première et la
seconde partie de la période.

Il faut examiner en particulier :

-   l'évolution du bêta de marché ;
-   la stabilité du coefficient SMB ;
-   la stabilité du coefficient HML ;
-   les changements de signe ou d'ordre de grandeur ;
-   les différences de significativité entre sous-périodes.

Une différence entre deux estimations ne prouve pas à elle seule une
rupture structurelle. Elle constitue un indice à interpréter avec
prudence.

### 7.3 Régressions glissantes

Une régression glissante est également prévue avec une fenêtre de 36
mois. À chaque date, le modèle est estimé sur les 36 mois précédents. La
représentation graphique des coefficients permet d'observer leur
évolution dans le temps.

Des coefficients variables suggèrent que les expositions de l'ETF ne
sont pas constantes. Cela peut être lié à l'évolution de la composition
du fonds, aux conditions de marché ou à des changements de régime
économique.

## 8. Extension : modèle Fama-French à cinq facteurs

Le notebook prévoit une extension FF5 en ajoutant deux facteurs :

-   **RMW (Robust Minus Weak) :** facteur lié à la profitabilité ;
-   **CMA (Conservative Minus Aggressive) :** facteur lié à la politique
    d'investissement.

La spécification générale devient :

\[
R\_{i,t}\^{ex}=`\alpha`{=tex}+`\beta`{=tex}(Mkt-RF)\_t+sSMB_t+hHML_t+rRMW_t+cCMA_t+`\varepsilon`{=tex}\_t
\]

Cette extension permet de vérifier si les facteurs de profitabilité et
d'investissement apportent une information supplémentaire au-delà des
trois facteurs initiaux.

La comparaison doit porter sur les indicateurs d'ajustement et, si les
résultats correspondants sont calculés, sur la performance hors
échantillon. L'ajout de facteurs augmente la complexité du modèle : une
amélioration du R² en échantillon ne suffit pas à justifier leur
inclusion.

## 9. Présentation des résultats empiriques

**Important :** le notebook transmis est un fichier de code non exécuté
et ne contient pas de sorties numériques. Les tableaux ci-dessous
constituent la grille de restitution à compléter après avoir lancé
toutes les cellules. Les valeurs ne doivent pas être estimées ou
inventées.

### 9.1 Expositions FF3 des ETF

  -------------------------------------------------------------------------------------------
  ETF     Style                  Alpha IC 95 %          Bêta        SMB        HML  R² ajusté
                                       alpha          marché                       
  ------- ----------------- ---------- ---------- ---------- ---------- ---------- ----------
  SPY     Marché large      À calculer À calculer À calculer À calculer À calculer À calculer

  IWM     Petites           À calculer À calculer À calculer À calculer À calculer À calculer
          capitalisations                                                          

  IWD     Value             À calculer À calculer À calculer À calculer À calculer À calculer

  IWF     Growth            À calculer À calculer À calculer À calculer À calculer À calculer
  -------------------------------------------------------------------------------------------

**Interprétation à rédiger après exécution :** comparer les signes et la
significativité des coefficients aux attentes théoriques présentées dans
la section 1.3. Pour IWM, vérifier notamment si le coefficient SMB est
positif. Pour IWD et IWF, examiner si les signes de HML correspondent
respectivement aux styles Value et Growth. Pour SPY, examiner si le bêta
est proche de 1 et si les expositions SMB et HML sont proches de zéro.

### 9.2 Comparaison CAPM et FF3 pour IWM

  Indicateur                                          CAPM          FF3
  ------------------------------------------- ------------ ------------
  Alpha estimé                                  À calculer   À calculer
  Erreur-type HAC de l'alpha                    À calculer   À calculer
  Intervalle de confiance à 95 % de l'alpha     À calculer   À calculer
  p-value de l'alpha                            À calculer   À calculer
  Bêta de marché                                À calculer   À calculer
  Coefficient SMB                               Non inclus   À calculer
  Coefficient HML                               Non inclus   À calculer
  R² ajusté                                     À calculer   À calculer

**Interprétation à rédiger après exécution :**

1.  Si l'alpha du CAPM est significatif mais ne l'est plus dans FF3,
    cela peut indiquer qu'une partie de la performance attribuée
    initialement à l'alpha est associée aux facteurs de taille et/ou de
    valeur.
2.  Si l'alpha reste significatif dans FF3, une composante moyenne non
    expliquée par les trois facteurs subsiste dans l'échantillon. Il
    faudra néanmoins examiner l'intervalle de confiance, la stabilité et
    les biais possibles avant de parler de surperformance.
3.  Si l'intervalle de confiance à 95 % contient zéro, les données ne
    permettent pas de conclure que l'alpha est statistiquement différent
    de zéro au seuil de 5 %.

### 9.3 Comparaison globale des modèles

  ------------------------------------------------------------------------
  ETF       Modèle       R² ajusté     RMSE (%)      MAE (%)       R² hors
                                                               échantillon
  --------- --------- ------------ ------------ ------------ -------------
  SPY       Moyenne /   À calculer   À calculer   À calculer    À calculer
            CAPM /                                           
            FF3                                              

  IWM       Moyenne /   À calculer   À calculer   À calculer    À calculer
            CAPM /                                           
            FF3                                              

  IWD       Moyenne /   À calculer   À calculer   À calculer    À calculer
            CAPM /                                           
            FF3                                              

  IWF       Moyenne /   À calculer   À calculer   À calculer    À calculer
            CAPM /                                           
            FF3                                              
  ------------------------------------------------------------------------

Le tableau final doit contenir une ligne distincte pour chaque
combinaison ETF-modèle, conformément aux sorties du notebook. Il ne faut
pas fusionner les trois modèles dans une même cellule lors de la
restitution définitive.

### 9.4 Test conjoint de SMB et HML

  Test                   Statistique      p-value Décision au seuil de 5 %
  -------------------- ------------- ------------ --------------------------
  (H_0:s=0) et (h=0)      À calculer   À calculer À compléter

Si la p-value est inférieure à 0,05, l'hypothèse selon laquelle les deux
coefficients sont simultanément nuls est rejetée. Il existe alors un
apport statistique conjoint de SMB et HML dans le modèle considéré.

### 9.5 Diagnostics et stabilité

  -----------------------------------------------------------------------
  Élément                             Résultat
  ----------------------------------- -----------------------------------
  Durbin-Watson                       À calculer

  Jarque-Bera --- statistique et      À calculer
  p-value                             

  Breusch-Pagan --- statistique et    À calculer
  p-value                             

  VIF des facteurs                    À calculer

  Coefficients sur la première moitié À calculer

  Coefficients sur la seconde moitié  À calculer

  Évolution des coefficients sur      À commenter à partir du graphique
  fenêtre glissante de 36 mois        
  -----------------------------------------------------------------------

## 10. Discussion des limites et des risques d'interprétation

### 10.1 Biais de survie

Les ETF retenus existent encore aujourd'hui. Une étude limitée aux fonds
survivants peut donner une image trop favorable de la performance du
secteur, car les fonds disparus ou liquidés sont absents de
l'échantillon. Le biais serait encore plus important si l'analyse
utilisait uniquement les entreprises qui composent actuellement un
indice pour reconstituer le passé.

### 10.2 Alpha significatif par hasard et tests multiples

Le notebook examine plusieurs ETF et plusieurs modèles. Plus le nombre
de tests augmente, plus le risque d'obtenir au moins un résultat
significatif par hasard augmente. Un alpha significatif isolé doit donc
être interprété prudemment. Une correction pour tests multiples, par
exemple la méthode de Holm, peut être envisagée si plusieurs alphas sont
testés simultanément.

### 10.3 Instabilité des facteurs

Les expositions peuvent évoluer dans le temps. Un coefficient estimé sur
toute la période résume une relation moyenne et peut masquer des
changements de régime ou de composition. Les analyses par sous-périodes
et les régressions glissantes sont donc importantes pour compléter les
résultats globaux.

### 10.4 Frais et coûts de transaction

Les ETF supportent des frais de gestion et peuvent présenter un écart de
réplication par rapport à leur indice. Les facteurs Fama-French sont des
portefeuilles théoriques et ne représentent pas nécessairement les coûts
réellement supportés par un investisseur. Une partie d'un alpha négatif
éventuel peut être liée aux frais et aux coûts de mise en œuvre.

### 10.5 Hypothèses économétriques

Les rendements financiers peuvent présenter des résidus non normaux, une
volatilité variable et une autocorrélation. Les erreurs-types HAC
améliorent la robustesse de l'inférence face à certaines formes
d'hétéroscédasticité et d'autocorrélation, mais elles ne corrigent pas
une mauvaise spécification du modèle ni une variable explicative omise.

### 10.6 Limites de représentativité et de causalité

L'étude porte sur des ETF du marché américain. Les résultats ne peuvent
pas être généralisés automatiquement aux marchés internationaux, aux
fonds moins liquides ou aux actions individuelles. Enfin, une relation
statistique entre un facteur et un rendement décrit une exposition ;
elle ne démontre pas à elle seule un lien de causalité.

## 11. Conclusion générale

Cette étude propose un cadre d'analyse permettant de distinguer
l'exposition d'un portefeuille aux facteurs de risque de sa performance
résiduelle. La comparaison entre la moyenne historique, le CAPM et le
modèle Fama-French à trois facteurs permet d'évaluer l'apport des
facteurs de marché, de taille et de valeur dans l'explication des
rendements excédentaires.

Le choix d'IWM comme portefeuille principal permet de mettre l'accent
sur les petites capitalisations, tandis que SPY, IWD et IWF servent de
points de comparaison pour examiner la cohérence des expositions
estimées. L'utilisation d'erreurs-types HAC, d'une évaluation
chronologique, de diagnostics et de régressions glissantes renforce la
démarche empirique.

Cependant, le notebook fourni n'ayant pas été exécuté, aucune conclusion
chiffrée ne peut encore être tirée sur la significativité de l'alpha
d'IWM, l'apport effectif de SMB et HML ou la supériorité du modèle FF3.
Ces conclusions devront être formulées après l'exécution du code et le
remplissage des tableaux de résultats. La décision finale ne devra pas
reposer uniquement sur le R² : elle devra également tenir compte de la
significativité, des intervalles de confiance, de la performance hors
échantillon, de la stabilité des coefficients et des limites
méthodologiques.

## 12. Procédure de finalisation du rapport

Avant remise, effectuer les opérations suivantes :

1.  Ouvrir le notebook dans Google Colab ou Jupyter avec une connexion
    Internet.
2.  Exécuter toutes les cellules dans l'ordre.
3.  Vérifier que les données mensuelles des facteurs et des ETF ont été
    téléchargées et correctement alignées.
4.  Vérifier les dates et le nombre d'observations de l'échantillon
    commun.
5.  Récupérer les coefficients, erreurs-types HAC, p-values et
    intervalles de confiance.
6.  Compléter les tableaux de la section 9 sans arrondir excessivement
    les résultats.
7.  Insérer et commenter les graphiques de corrélation, d'exposition aux
    facteurs, de comparaison des erreurs et de stabilité des
    coefficients.
8.  Vérifier que les conclusions écrites correspondent aux résultats
    effectivement obtenus.

**Note de reproductibilité :** le notebook prévoit l'export des tableaux
dans le dossier `resultats/` et la génération automatique d'un fichier
`rapport_F5.md` à partir des résultats calculés. Le présent rapport
constitue une version explicative structurée ; les tableaux empiriques
doivent être actualisés à partir de l'exécution réelle du notebook.
