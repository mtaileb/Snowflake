# Introduction

Une fonction de fenêtre (window function) est une fonction SQL analytique qui opère sur un groupe de lignes liées appelé une partition. Une partition est généralement un groupe logique de lignes selon une dimension familière, telle que la catégorie de produit, l'emplacement, la période ou l'unité commerciale. Les résultats des fonctions sont calculés pour chaque partition, par rapport à un cadre de fenêtre implicite ou explicite. Un cadre de fenêtre est un ensemble fixe ou variable de lignes relatif à la ligne courante. La ligne courante est une seule ligne d'entrée pour laquelle le résultat de la fonction est actuellement calculé. Les résultats des fonctions sont calculés ligne par ligne dans chaque partition, et chaque ligne du cadre de fenêtre prend à son tour le rôle de ligne courante.

La syntaxe qui définit ce comportement est la clause `OVER` pour la fonction. Dans de nombreux cas, la clause `OVER` distingue une fonction de fenêtre d'une fonction SQL régulière portant le même nom (comme `AVG` ou `SUM`). La clause `OVER` se compose de trois composants principaux :

*   Une clause `PARTITION BY`
*   Une clause `ORDER BY`
*   Une spécification de cadre de fenêtre

![Description de l'image](https://github.com/mtaileb/Snowflake/blob/main/images/Cheat_sheet_SQL_Window_Functions.png?raw=true)

Selon la fonction ou la requête en question, tous ces composants peuvent être facultatifs ; une fonction de fenêtre avec une clause `OVER` vide est valide : `OVER()`. Cependant, dans la plupart des requêtes analytiques, les fonctions de fenêtre nécessitent un ou plusieurs composants explicites de la clause `OVER`. Vous pouvez appeler une fonction de fenêtre dans tout contexte qui prend en charge d'autres fonctions SQL. Les sections suivantes expliquent plus en détail les concepts derrière les fonctions de fenêtre et présentent quelques exemples d'introduction.

## Fonctions de fenêtre vs fonctions d'agrégation

Une bonne façon de commencer à apprendre les fonctions de fenêtre est de comparer les fonctions d'agrégation régulières avec leurs homologues fonctions de fenêtre. Plusieurs fonctions d'agrégation standard, comme `SUM`, `COUNT`, et `AVG`, ont des fonctions de fenêtre correspondantes portant le même nom. Pour distinguer les deux, notez que :
*   Pour une fonction d'agrégation, l'entrée est un groupe de lignes, et la sortie est une ligne.
*   Pour une fonction de fenêtre, l'entrée est chaque ligne dans une partition, et la sortie est une ligne par ligne d'entrée.

Par exemple, la fonction d'agrégation `SUM` renvoie une seule valeur totale pour toutes les lignes d'entrée, alors qu'une fonction de fenêtre renvoie de multiples totaux : un pour chaque ligne (la ligne courante) par rapport à toutes les autres lignes de la partition.

Pour voir comment cela fonctionne, commençons par créer et charger la table `menu_items`, qui contient le coût des marchandises vendues et les prix des articles du menu d'un camion de restauration.

### Création et chargement de la table `menu_items`

Pour créer et insérer des lignes dans la table `menu_items` utilisée dans certains exemples de fonctions, exécutez les commandes SQL suivantes. (Cette table contient 60 lignes. Elle est basée sur, mais pas identique à, la table `menu` de l'exemple de base de données Tasty Bytes.)

Nota: avant d'exéctuer le code, sélectionnez le user ACCOUNTADMIN, et le warehouse COMPUTE_WH.

```sql
-- 1. Créer une base de données dédiée (si elle n'existe pas)
CREATE DATABASE IF NOT EXISTS FOODTRUCK_DB;

-- 2. Utiliser la base de données
USE DATABASE FOODTRUCK_DB;

-- 3. Créer un schéma dédié
CREATE SCHEMA IF NOT EXISTS MENU_DATA;

-- 4. Utiliser le schéma
USE SCHEMA MENU_DATA;

-- 5. Créer la table menu_items
CREATE OR REPLACE TABLE menu_items (
  menu_id INT NOT NULL,
  menu_category VARCHAR(20),
  menu_item_name VARCHAR(50),
  menu_cogs_usd NUMBER(7,2),
  menu_price_usd NUMBER(7,2)
);

-- 6. Insérer les données
INSERT INTO menu_items VALUES
(1,'Beverage','Bottled Soda',0.500,3.00),
(2,'Beverage','Bottled Water',0.500,2.00),
(3,'Main','Breakfast Crepe',5.00,12.00),
(4,'Main','Buffalo Mac & Cheese',6.00,10.00),
(5,'Main','Chicago Dog',4.00,9.00),
(6,'Main','Chicken Burrito',3.2500,12.500),
(7,'Main','Chicken Pot Pie Crepe',6.00,15.00),
(8,'Main','Combination Curry',9.00,15.00),
(9,'Main','Combo Fried Rice',5.00,11.00),
(10,'Main','Combo Lo Mein',6.00,13.00),
(11,'Main','Coney Dog',5.00,10.00),
(12,'Main','Creamy Chicken Ramen',8.00,17.2500),
(13,'Snack','Crepe Suzette',4.00,9.00),
(14,'Main','Fish Burrito',3.7500,12.500),
(15,'Snack','Fried Pickles',1.2500,6.00),
(16,'Snack','Greek Salad',4.00,11.00),
(17,'Main','Gyro Plate',8.00,12.00),
(18,'Main','Hot Ham & Cheese',7.00,11.00),
(19,'Dessert','Ice Cream Sandwich',1.00,4.00),
(20,'Beverage','Iced Tea',0.7500,3.00),
(21,'Main','Italian',6.00,11.00),
(22,'Main','Lean Beef Tibs',6.00,13.00),
(23,'Main','Lean Burrito Bowl',3.500,12.500),
(24,'Main','Lean Chicken Tibs',5.00,11.00),
(25,'Main','Lean Chicken Tikka Masala',10.00,17.00),
(26,'Beverage','Lemonade',0.6500,3.500),
(27,'Main','Lobster Mac & Cheese',10.00,15.00),
(28,'Dessert','Mango Sticky Rice',1.2500,5.00),
(29,'Main','Miss Piggie',2.600,6.00),
(30,'Main','Mothers Favorite',4.500,12.00),
(31,'Main','New York Dog',4.00,8.00),
(32,'Main','Pastrami',8.00,11.00),
(33,'Dessert','Popsicle',0.500,3.00),
(34,'Main','Pulled Pork Sandwich',7.00,12.00),
(35,'Main','Rack of Pork Ribs',11.2500,21.00),
(36,'Snack','Seitan Buffalo Wings',4.00,7.00),
(37,'Main','Spicy Miso Vegetable Ramen',7.00,17.2500),
(38,'Snack','Spring Mix Salad',2.2500,6.00),
(39,'Main','Standard Mac & Cheese',3.00,8.00),
(40,'Dessert','Sugar Cone',2.500,6.00),
(41,'Main','Tandoori Mixed Grill',11.00,18.00),
(42,'Main','The Classic',4.00,12.00),
(43,'Main','The King Combo',12.00,20.00),
(44,'Main','The Kitchen Sink',6.00,14.00),
(45,'Main','The Original',1.500,5.00),
(46,'Main','The Ranch',2.400,6.00),
(47,'Main','The Salad of All Salads',6.00,12.00),
(48,'Main','Three Meat Plate',10.00,17.00),
(49,'Main','Three Taco Combo Plate',7.00,11.00),
(50,'Main','Tonkotsu Ramen',7.00,17.2500),
(51,'Main','Two Meat Plate',9.00,14.00),
(52,'Dessert','Two Scoop Bowl',3.00,7.00),
(53,'Main','Two Taco Combo Plate',6.00,9.00),
(54,'Main','Veggie Burger',5.00,9.00),
(55,'Main','Veggie Combo',4.00,9.00),
(56,'Main','Veggie Taco Bowl',6.00,10.00),
(57,'Dessert','Waffle Cone',2.500,6.00),
(58,'Main','Wonton Soup',2.00,6.00),
(59,'Main','Mini Pizza',null,null),
(60,'Main','Large Pizza',null,null);

-- 7. Vérifier la création
SELECT 
    'Database: FOODTRUCK_DB' as info,
    COUNT(*) as total_rows
FROM menu_items;
```

Utilisez une fonction `AVG` régulière pour trouver le coût moyen des marchandises vendues ("COGS" signifie "Cost Of Goods Sold") pour les articles de menu dans différentes catégories :

```sql
-- 8. Votre requête AVG
SELECT menu_category,
    AVG(menu_cogs_usd) avg_cogs
FROM menu_items
GROUP BY 1
ORDER BY menu_category;
```

```
+---------------+------------+
| MENU_CATEGORY |   AVG_COGS |
|---------------+------------|
| Beverage      | 0.60000000 |
| Dessert       | 1.79166667 |
| Main          | 6.11046512 |
| Snack         | 3.10000000 |
+---------------+------------+
```

Notez que la fonction renvoie un résultat groupé pour `avg_cogs`.

Alternativement, vous pouvez spécifier une clause `OVER` et utiliser `AVG` comme fonction de fenêtre. (Le résultat est limité à 15 lignes sur les 60 de la table.)

```sql
SELECT menu_category,
    AVG(menu_cogs_usd) OVER(PARTITION BY menu_category) avg_cogs
  FROM menu_items
  ORDER BY menu_category
  LIMIT 15;
```

```
+---------------+----------+
| MENU_CATEGORY | AVG_COGS |
|---------------+----------|
| Beverage      |  0.60000 |
| Beverage      |  0.60000 |
| Beverage      |  0.60000 |
| Beverage      |  0.60000 |
| Dessert       |  1.79166 |
| Dessert       |  1.79166 |
| Dessert       |  1.79166 |
| Dessert       |  1.79166 |
| Dessert       |  1.79166 |
| Dessert       |  1.79166 |
| Main          |  6.11046 |
| Main          |  6.11046 |
| Main          |  6.11046 |
| Main          |  6.11046 |
| Main          |  6.11046 |
+---------------+----------+
```

Notez que la fonction renvoie une moyenne pour chaque ligne dans chaque partition et réinitialise le calcul lorsque la valeur de la colonne de partition change. Pour rendre la valeur de la fonction de fenêtre plus apparente, ajoutez une clause `ORDER BY` et un cadre de fenêtre à la définition de la fonction. Renvoyez également les valeurs brutes de `menu_cogs_usd`, en plus des moyennes, pour voir comment les calculs spécifiques fonctionnent. Cette requête est un exemple simple d'une "moyenne mobile", un calcul glissant qui dépend d'un cadre de fenêtre explicite.

```sql
SELECT menu_category, menu_price_usd, menu_cogs_usd,
    AVG(menu_cogs_usd) OVER(PARTITION BY menu_category
      ORDER BY menu_price_usd, menu_cogs_usd ROWS BETWEEN CURRENT ROW and 2 FOLLOWING) avg_cogs
  FROM menu_items
  ORDER BY menu_category, menu_price_usd, menu_cogs_usd
  LIMIT 15;
```

```
+---------------+----------------+---------------+----------+
| MENU_CATEGORY | MENU_PRICE_USD | MENU_COGS_USD | AVG_COGS |
|---------------+----------------+---------------+----------|
| Beverage      |           2.00 |          0.50 |  0.58333 |
| Beverage      |           3.00 |          0.50 |  0.63333 |
| Beverage      |           3.00 |          0.75 |  0.70000 |
| Beverage      |           3.50 |          0.65 |  0.65000 |
| Dessert       |           3.00 |          0.50 |  0.91666 |
| Dessert       |           4.00 |          1.00 |  1.58333 |
| Dessert       |           5.00 |          1.25 |  2.08333 |
| Dessert       |           6.00 |          2.50 |  2.66666 |
| Dessert       |           6.00 |          2.50 |  2.75000 |
| Dessert       |           7.00 |          3.00 |  3.00000 |
| Main          |           5.00 |          1.50 |  2.03333 |
| Main          |           6.00 |          2.60 |  3.00000 |
| Main          |           6.00 |          2.00 |  2.33333 |
| Main          |           6.00 |          2.40 |  3.13333 |
| Main          |           8.00 |          4.00 |  3.66666 |
+---------------+----------------+---------------+----------+
```

Le cadre de fenêtre ajuste les calculs de moyenne de sorte que seules la ligne courante et les deux lignes qui la suivent (dans la partition) sont prises en compte. La dernière ligne d'une partition n'a pas de lignes suivantes, donc la moyenne pour la dernière ligne Beverage, par exemple, est la même que la valeur `menu_cogs_usd` correspondante (0.65). La sortie de la fonction de fenêtre dépend de la ligne individuelle qui est passée à la fonction et des valeurs des autres lignes qui sont éligibles pour le cadre de fenêtre.

> **Note**
> Lorsque vous utilisez des fonctions de fenêtre avec des clauses `ORDER BY`, assurez-vous que l'ordre est déterministe. Si plusieurs lignes ont la même valeur pour les colonnes `ORDER BY`, ajoutez des colonnes supplémentaires comme départageurs pour garantir des résultats cohérents et prévisibles entre les exécutions de requêtes. Dans cet exemple, `menu_cogs_usd` est inclus comme départageur car plusieurs lignes peuvent avoir la même valeur `menu_price_usd`.

## Ordonnancement des lignes pour les fonctions de fenêtre

L'exemple précédent de fonction de fenêtre `AVG` utilise une clause `ORDER BY` dans la définition de la fonction pour s'assurer que le cadre de fenêtre est appliqué à des données triées (par `menu_price_usd` dans ce cas).

Deux types de fonctions de fenêtre nécessitent une clause `ORDER BY` :
*   Les fonctions de fenêtre avec des cadres de fenêtre explicites, qui effectuent des opérations glissantes sur des sous-ensembles de lignes dans chaque partition, comme le calcul de totaux cumulés ou de moyennes mobiles. Sans clause `ORDER BY`, le cadre de fenêtre est dénué de sens ; l'ensemble des lignes "précédentes" et "suivantes" doit être déterministe.
*   Les fonctions de fenêtre de classement, telles que `CUME_DIST`, `RANK` et `DENSE_RANK`, qui renvoient des informations basées sur le "rang" d'une ligne. Par exemple, si vous classez des magasins par ordre décroissant de profit par mois, le magasin avec le profit le plus élevé sera classé 1 ; le deuxième magasin le plus profitable sera classé 2, et ainsi de suite.

La clause `ORDER BY` pour une fonction de fenêtre prend en charge la même syntaxe que la clause `ORDER BY` principale qui trie les résultats finaux d'une requête. Ces deux clauses `ORDER BY` sont séparées et distinctes. Une clause `ORDER BY` dans une clause `OVER` contrôle uniquement l'ordre dans lequel la fonction de fenêtre traite les lignes ; elle ne contrôle pas la sortie de l'ensemble de la requête. Dans de nombreux cas, vos requêtes de fonctions de fenêtre contiendront les deux types de clauses `ORDER BY`.

Les clauses `PARTITION BY` et `ORDER BY` dans la clause `OVER` sont également indépendantes. Vous pouvez utiliser la clause `ORDER BY` sans la clause `PARTITION BY` et vice versa.

Vérifiez la syntaxe des fonctions de fenêtre individuelles avant d'écrire des requêtes. Les exigences de syntaxe pour la clause `ORDER BY` varient selon la fonction :
*   Certaines fonctions de fenêtre nécessitent une clause `ORDER BY`.
*   Certaines fonctions de fenêtre utilisent une clause `ORDER BY` si elle est présente, mais ne la requièrent pas.
*   Certaines fonctions de fenêtre n'autorisent pas de clause `ORDER BY`.
*   Certaines fonctions de fenêtre interprètent une clause `ORDER BY` comme un cadre de fenêtre implicite.

> **Attention**
> De manière générale, SQL est un langage explicite, avec peu de clauses implicites. Cependant, pour certaines fonctions de fenêtre, une clause `ORDER BY` implique un cadre de fenêtre. Pour plus de détails, voir les notes d'utilisation pour les cadres de fenêtre.
> Parce qu'un comportement qui est implicite plutôt qu'explicite peut conduire à des résultats difficiles à comprendre, Snowflake recommande de déclarer les cadres de fenêtre explicitement.

## Utilisation de différents types de cadres de fenêtre

Les cadres de fenêtre sont définis explicitement ou implicitement. Ils dépendent de la présence d'une clause `ORDER BY` dans la clause `OVER` :
*   Pour la syntaxe de cadre explicite, voir `windowFrameClause` sous Syntaxe. Vous pouvez définir des limites ouvertes : du début de la partition à la ligne courante ; de la ligne courante à la fin de la partition ; ou complètement "illimitées" d'un bout à l'autre. Alternativement, vous pouvez utiliser des décalages explicites (inclusifs) qui sont relatifs à la ligne courante dans la partition.
*   Les cadres implicites sont utilisés par défaut lorsque la clause `OVER` n'inclut pas de `windowFrameClause`. Le cadre par défaut dépend de la fonction en question. Voir aussi Notes d'utilisation pour les cadres de fenêtre.

### Cadres de fenêtre basés sur les plages (RANGE) vs basés sur les lignes (ROWS)

Snowflake prend en charge deux principaux types de cadres de fenêtre :

**Basé sur les lignes (ROWS)**
Une séquence exacte de lignes appartient au cadre, basée sur un décalage physique par rapport à la ligne courante. Par exemple, `5 PRECEDING` signifie les cinq lignes précédant la ligne courante. Le décalage doit être un nombre. Le mode `ROWS` est inclusif et est toujours relatif à la ligne courante. Si le nombre spécifié de lignes précédentes ou suivantes dépasse les limites de la partition, Snowflake traite la valeur comme NULL.

Si le cadre a des limites ouvertes plutôt que des limites explicitement numérotées, un décalage physique similaire s'applique. Par exemple, `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` signifie que le cadre se compose de l'ensemble des lignes (zéro ou plus) qui précèdent physiquement la ligne courante et de la ligne courante elle-même.

**Basé sur les plages (RANGE)**
Une plage logique de lignes appartient au cadre, étant donné un décalage par rapport à la valeur `ORDER BY` pour la ligne courante. Par exemple, `5 PRECEDING` signifie les lignes avec des valeurs `ORDER BY` qui ont la valeur `ORDER BY` de la ligne courante, plus ou moins un maximum de 5 (plus pour l'ordre `DESC`, moins pour l'ordre `ASC`). La valeur de décalage peut être un nombre ou un intervalle.

Si le cadre a des limites ouvertes plutôt que numérotées, un décalage logique similaire s'applique. Par exemple, `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` signifie que le cadre se compose de toutes les lignes qui précèdent physiquement la ligne courante, de la ligne courante elle-même, et de toutes les lignes adjacentes qui ont la même valeur `ORDER BY` que la ligne courante. Pour un cadre de fenêtre `RANGE`, `CURRENT ROW` ne signifie pas physiquement la ligne courante ; cela signifie toutes les lignes qui ont la même valeur `ORDER BY` que la ligne physique courante.

Les distinctions entre les cadres de fenêtre `ROWS BETWEEN` et `RANGE BETWEEN` sont importantes car les requêtes de fonctions de fenêtre peuvent renvoyer des résultats très différents, selon l'expression `ORDER BY`, les données dans les tables et la définition exacte du cadre. Les exemples suivants démontrent les différences de comportement.

### Comparaison de `RANGE BETWEEN` et `ROWS BETWEEN` avec des décalages explicites

Un cadre de fenêtre basé sur une plage nécessite une colonne ou une expression `ORDER BY` et une spécification `RANGE BETWEEN`. La limite logique du cadre de fenêtre dépend de la valeur `ORDER BY` (une constante numérique ou un littéral d'intervalle) pour la ligne courante.

Par exemple, une table de séries chronologiques nommée `heavy_weather` est définie comme suit :

```sql
CREATE OR REPLACE TABLE heavy_weather
  (start_time TIMESTAMP, precip NUMBER(3,2), city VARCHAR(20), county VARCHAR(20));
```

Les lignes d'exemple dans cette table ressemblent à ceci :

```
+-------------------------+--------+-------+-------------+
| START_TIME              | PRECIP | CITY  | COUNTY      |
|-------------------------+--------+-------+-------------|
| 2021-12-30 11:23:00.000 |   0.12 | Lebec | Los Angeles |
| 2021-12-30 11:43:00.000 |   0.98 | Lebec | Los Angeles |
| 2021-12-30 13:53:00.000 |   0.23 | Lebec | Los Angeles |
| 2021-12-30 14:53:00.000 |   0.13 | Lebec | Los Angeles |
| 2021-12-30 15:15:00.000 |   0.29 | Lebec | Los Angeles |
| 2021-12-30 17:53:00.000 |   0.10 | Lebec | Los Angeles |
| 2021-12-30 18:53:00.000 |   0.09 | Lebec | Los Angeles |
| 2021-12-30 19:53:00.000 |   0.07 | Lebec | Los Angeles |
| 2021-12-30 20:53:00.000 |   0.07 | Lebec | Los Angeles |
+-------------------------+--------+-------+-------------+
```

Supposons qu'une requête calcule une moyenne mobile de 3 heures (AVG) sur la colonne `precip` (précipitations), en utilisant un cadre de fenêtre ordonné par `start_time` :

```sql
AVG(precip)
  OVER(ORDER BY start_time
    RANGE BETWEEN CURRENT ROW AND INTERVAL '3 hours' FOLLOWING)
```

Étant donné les lignes d'exemple ci-dessus, lorsque la ligne courante est `2021-12-30 11:23:00.000` (la première ligne d'exemple), seules les deux lignes suivantes entrent dans le cadre (`2021-12-30 11:43:00.000` et `2021-12-30 13:53:00.000`). Les horodatages suivants sont supérieurs à 3 heures plus tard.

Cependant, si vous changez le cadre de fenêtre en un intervalle de 1 jour, toutes les lignes d'exemple qui suivent la ligne courante entrent dans le cadre car elles ont toutes des horodatages à la même date (2021-12-30) :

```
RANGE BETWEEN CURRENT ROW AND INTERVAL '1 day' FOLLOWING
```

Si vous deviez changer cette syntaxe de `RANGE BETWEEN` à `ROWS BETWEEN`, le cadre devrait spécifier des limites fixes, qui représentent un nombre exact de lignes : la ligne courante plus le nombre exact suivant de lignes ordonnées, comme 1, 3 ou 10 lignes, indépendamment des valeurs renvoyées par l'expression `ORDER BY`.

Voir aussi l'exemple `RANGE BETWEEN` avec des décalages numériques explicites.

### Comparaison de `RANGE BETWEEN` et `ROWS BETWEEN` avec des limites ouvertes

L'exemple suivant compare les résultats lorsque les cadres de fenêtre suivants sont calculés sur le même ensemble de lignes :

```
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

Cet exemple effectue une sélection sur une petite table nommée `menu_items`. Voir Création et chargement de la table `menu_items`.

La fonction de fenêtre `SUM` agrège les valeurs `menu_price_usd` pour chaque partition `menu_category`. Avec la syntaxe `ROWS BETWEEN`, il est facile de voir comment les totaux cumulés s'accumulent dans chaque partition.

```sql
SELECT menu_category, menu_price_usd,
    SUM(menu_price_usd)
      OVER(PARTITION BY menu_category ORDER BY menu_price_usd
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) sum_price
  FROM menu_items
  WHERE menu_category IN('Beverage','Dessert','Snack')
  ORDER BY menu_category, menu_price_usd;
```

```
+---------------+----------------+-----------+
| MENU_CATEGORY | MENU_PRICE_USD | SUM_PRICE |
|---------------+----------------+-----------|
| Beverage      |           2.00 |      2.00 |
| Beverage      |           3.00 |      5.00 |
| Beverage      |           3.00 |      8.00 |
| Beverage      |           3.50 |     11.50 |
| Dessert       |           3.00 |      3.00 |
| Dessert       |           4.00 |      7.00 |
| Dessert       |           5.00 |     12.00 |
| Dessert       |           6.00 |     18.00 |
| Dessert       |           6.00 |     24.00 |
| Dessert       |           7.00 |     31.00 |
| Snack         |           6.00 |      6.00 |
| Snack         |           6.00 |     12.00 |
| Snack         |           7.00 |     19.00 |
| Snack         |           9.00 |     28.00 |
| Snack         |          11.00 |     39.00 |
+---------------+----------------+-----------+
```

Lorsque la syntaxe `RANGE BETWEEN` est utilisée avec une requête par ailleurs identique, les calculs ne sont pas aussi évidents au premier abord ; ils dépendent d'une interprétation différente de la ligne courante : la ligne courante elle-même plus toutes les lignes adjacentes qui ont la même valeur `ORDER BY` que cette ligne.

Par exemple, les valeurs `sum_price` pour les deuxième et troisième lignes du résultat sont toutes deux 8.00 car la valeur `ORDER BY` pour ces lignes est la même. Ce comportement se produit à deux autres endroits dans l'ensemble de résultats, où `sum_price` est calculé consécutivement comme 24.00 et 12.00.

```sql
SELECT menu_category, menu_price_usd,
    SUM(menu_price_usd)
      OVER(PARTITION BY menu_category ORDER BY menu_price_usd
      RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) sum_price
  FROM menu_items
  WHERE menu_category IN('Beverage','Dessert','Snack')
  ORDER BY menu_category, menu_price_usd;
```

```
+---------------+----------------+-----------+
| MENU_CATEGORY | MENU_PRICE_USD | SUM_PRICE |
|---------------+----------------+-----------|
| Beverage      |           2.00 |      2.00 |
| Beverage      |           3.00 |      8.00 |
| Beverage      |           3.00 |      8.00 |
| Beverage      |           3.50 |     11.50 |
| Dessert       |           3.00 |      3.00 |
| Dessert       |           4.00 |      7.00 |
| Dessert       |           5.00 |     12.00 |
| Dessert       |           6.00 |     24.00 |
| Dessert       |           6.00 |     24.00 |
| Dessert       |           7.00 |     31.00 |
| Snack         |           6.00 |     12.00 |
| Snack         |           6.00 |     12.00 |
| Snack         |           7.00 |     19.00 |
| Snack         |           9.00 |     28.00 |
| Snack         |          11.00 |     39.00 |
+---------------+----------------+-----------+
```

## Cadres de fenêtre pour les calculs cumulatifs et glissants

Les cadres de fenêtre sont un mécanisme très flexible pour exécuter différents types de requêtes analytiques, y compris les calculs cumulatifs et les calculs mobiles. Pour renvoyer des sommes cumulatives, par exemple, vous pouvez spécifier un cadre de fenêtre qui commence à un point fixe et se déplace ligne par ligne à travers toute la partition :

```sql
OVER(PARTITION BY col1 ORDER BY col2 ROWS UNBOUNDED PRECEDING)
```

Un autre exemple de ce type de cadre pourrait être :

```sql
OVER(PARTITION BY col1 ORDER BY col2 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

Le nombre de lignes éligibles pour ces cadres est variable, mais les points de départ et de fin des cadres sont fixes, utilisant des limites nommées plutôt que des limites numériques ou d'intervalle.

Si vous voulez que le calcul de la fonction de fenêtre glisse vers l'avant sur un nombre (ou une plage) spécifique de lignes, vous pouvez utiliser des décalages explicites :

```sql
OVER(PARTITION BY col1 ORDER BY col2 ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING)
```

Dans ce cas, le résultat est un cadre glissant qui se compose d'un maximum de sept lignes (3 + ligne courante + 3). Un autre exemple de ce type de cadre pourrait être :

```sql
OVER(PARTITION BY col1 ORDER BY col2 ROWS BETWEEN CURRENT ROW AND 3 FOLLOWING)
```

Les cadres de fenêtre peuvent contenir un mélange de limites nommées et de décalages explicites.

### Cadres de fenêtre glissants

Un cadre de fenêtre glissant est un cadre de largeur fixe qui "glisse" à travers les lignes de la partition, couvrant une tranche différente de la partition à chaque fois. Le nombre de lignes dans le cadre reste le même, sauf au début ou à la fin d'une partition, où il peut contenir moins de lignes.

Les fenêtres glissantes sont souvent utilisées pour calculer des moyennes mobiles, qui sont basées sur un intervalle de taille fixe (comme un nombre de jours). La moyenne est "mobile" car bien que la taille de l'intervalle soit constante, les valeurs réelles dans l'intervalle changent au fil du temps (ou sur une autre dimension).

Par exemple, les analystes du marché boursier analysent souvent les actions en partie sur la base de la moyenne mobile sur 13 semaines du prix d'une action. La moyenne mobile du prix aujourd'hui est la moyenne du prix à la fin d'aujourd'hui et du prix à la fin de chaque jour au cours des 13 dernières semaines. Si les actions sont négociées 5 jours par semaine, et s'il n'y a pas eu de jours fériés au cours des 13 dernières semaines, la moyenne mobile est le prix moyen sur chacun des 65 derniers jours de négociation (y compris aujourd'hui).

L'exemple suivant montre ce qui arrive à une moyenne mobile de 13 semaines (91 jours) du prix d'une action le dernier jour de juin et les premiers jours de juillet :
*   Le 30 juin, la fonction renvoie le prix moyen du 1er avril au 30 juin (inclus).
*   Le 1er juillet, la fonction renvoie le prix moyen du 2 avril au 1er juillet (inclus).
*   Le 2 juillet, la fonction renvoie le prix moyen du 3 avril au 2 juillet (inclus).

L'exemple suivant utilise une petite fenêtre glissante de 3 jours sur les 7 premiers jours du mois. Cet exemple prend en compte le fait qu'au début de la période, la partition pourrait ne pas être pleine :

![Cadre de fenêtre glissant](https://docs.snowflake.com/en/_images/window-functions-sliding-frame.png)

Comme vous pouvez le voir dans la maquette correspondante d'un résultat de requête, la dernière colonne contient la somme des données de vente des trois jours les plus récents. Par exemple, la valeur de la colonne pour le jour 4 est 36, ce qui est la somme des ventes pour les jours 2, 3 et 4 (11 + 12 + 13) :

```
+--------+-------+---------------+
| Day of | Sales | Most Recent   |
| Month  | Today | 3 Days' Sales |
|--------+-------+---------------+
|      1 |    10 |            10 |
|      2 |    11 |            21 |
|      3 |    12 |            33 |
|      4 |    13 |            36 |
|      5 |    14 |            39 |
|    ... |   ... |           ... |
+--------+-------+---------------+
```

## Fonctions de fenêtre de classement

La syntaxe d'une fonction de fenêtre de classement est essentiellement la même que celle des autres fonctions de fenêtre. Les exceptions incluent :
*   Les fonctions de fenêtre de classement nécessitent la clause `ORDER BY` à l'intérieur de la clause `OVER`.
*   Pour certaines fonctions de classement, comme `RANK` elle-même, aucun argument d'entrée n'est requis. Pour la fonction `RANK`, la valeur renvoyée est basée uniquement sur le classement numérique, tel que déterminé par la clause `ORDER BY` à l'intérieur de la clause `OVER`. Par conséquent, passer un nom de colonne ou une expression à la fonction est inutile.

La fonction de classement la plus simple s'appelle `RANK`. Vous pouvez utiliser cette fonction pour :
*   Classer les commerciaux sur le chiffre d'affaires (ventes), du plus élevé au plus bas.
*   Classer les pays sur la base de leur PIB par habitant (revenu par personne), du plus élevé au plus bas.
*   Classer les pays sur la pollution de l'air, du plus bas au plus élevé.

Cette fonction identifie simplement la position de classement numérique d'une ligne dans un ensemble ordonné de lignes. La première ligne a le rang 1, la seconde a le rang 2, et ainsi de suite. L'exemple suivant montre l'ordre de classement des commerciaux basé sur le Montant Vendu :

```
+-------------+-------------+------+
| Salesperson | Amount Sold | Rank |
|-------------+-------------+------|
| Smith       |        2000 |    1 |
| Jones       |        1500 |    2 |
| Torkelson   |        1200 |    3 |
| Dolenz      |        1100 |    4 |
+-------------+-------------+------+
```

Les lignes doivent déjà être triées avant que les classements puissent être attribués. Par conséquent, vous devez utiliser une clause `ORDER BY` dans la clause `OVER`.

Considérez l'exemple suivant : vous voulez savoir où se classe le profit de votre magasin parmi les succursales de la chaîne (si votre magasin est classé premier, deuxième, troisième, etc.). Cet exemple classe chaque magasin par rentabilité dans sa ville. Les lignes sont mises dans l'ordre décroissant (profit le plus élevé en premier), donc le magasin le plus rentable est classé 1 :

```sql
SELECT city, branch_ID, net_profit,
       RANK() OVER (PARTITION BY city ORDER BY net_profit DESC) AS rank
    FROM store_sales
    ORDER BY city, rank;
```

```
+-----------+-----------+------------+------+
| CITY      | BRANCH_ID | NET_PROFIT | RANK |
|-----------+-----------+------------+------|
| Montreal  |         3 |   10000.00 |    1 |
| Montreal  |         4 |    9000.00 |    2 |
| Vancouver |         2 |   15000.00 |    1 |
| Vancouver |         1 |   10000.00 |    2 |
+-----------+-----------+------------+------+
```

> **Note**
> La colonne `net_profit` n'a pas besoin d'être passée comme argument à la fonction `RANK`. Au lieu de cela, les lignes d'entrée sont triées par `net_profit`. La fonction `RANK` doit simplement renvoyer la position de la ligne (1, 2, 3, etc.) dans la partition.

La sortie d'une fonction de classement dépend de :
*   La ligne individuelle passée à la fonction.
*   Les valeurs des autres lignes dans la partition.
*   L'ordre de toutes les lignes dans la partition.

Snowflake fournit plusieurs fonctions de classement différentes. Pour une liste de ces fonctions, et plus de détails sur leur syntaxe, voir Fonctions de fenêtre.

Pour classer votre magasin par rapport à tous les autres magasins de la chaîne, pas seulement aux autres magasins de votre ville, utilisez la requête ci-dessous :

```sql
SELECT
    branch_ID,
    net_profit,
    RANK() OVER (ORDER BY net_profit DESC) AS sales_rank
  FROM store_sales
```

La requête suivante utilise la première clause `ORDER BY` pour contrôler le traitement par la fonction de fenêtre et la deuxième clause `ORDER BY` pour contrôler l'ordre de sortie de l'ensemble de la requête :

```sql
SELECT
    branch_ID,
    net_profit,
    RANK() OVER (ORDER BY net_profit DESC) AS sales_rank
  FROM store_sales
  ORDER BY branch_ID;
```

## Exemple illustré

Cet exemple utilise un scénario de vente pour illustrer bon nombre des concepts décrits plus tôt dans ce sujet.

Supposons que vous ayez besoin de générer un rapport financier qui montre des valeurs basées sur les ventes de la dernière semaine :
*   Ventes quotidiennes
*   Classement dans la semaine (c'est-à-dire les ventes classées du plus élevé au plus bas pour la semaine)
*   Ventes jusqu'à présent cette semaine (c'est-à-dire la "somme cumulative" pour tous les jours depuis le début de la semaine jusqu'au jour actuel inclus)
*   Total des ventes pour la semaine
*   Moyenne mobile sur trois jours (c'est-à-dire la moyenne sur le jour actuel et les deux jours précédents)

Le rapport pourrait ressembler à ceci :

```
+--------+-------+------+--------------+-------------+--------------+
| Day of | Sales | Rank | Sales So Far | Total Sales | 3-Day Moving |
| Week   | Today |      | This Week    | This Week   | Average      |
|--------+-------+------+--------------+-------------|--------------+
|      1 |    10 |    4 |           10 |          84 |         10.0 |
|      2 |    14 |    3 |           24 |          84 |         12.0 |
|      3 |     6 |    5 |           30 |          84 |         10.0 |
|      4 |     6 |    5 |           36 |          84 |          9.0 |
|      5 |    14 |    3 |           50 |          84 |         10.0 |
|      6 |    16 |    2 |           66 |          84 |         11.0 |
|      7 |    18 |    1 |           84 |          84 |         12.0 |
+--------+-------+------+--------------+-------------+--------------+
```

Le SQL pour cette requête est quelque peu complexe. Plutôt que de montrer l'exemple comme une seule requête, cette discussion décompose le SQL pour les colonnes individuelles.

Dans un scénario réel, vous auriez des années de données, donc pour calculer des sommes et des moyennes pour une semaine spécifique de données, vous devriez utiliser une fenêtre d'une semaine, ou utiliser un filtre similaire à :

```sql
... WHERE date >= start_of_relevant_week and date <= end_of_relevant_week ...
```

Cependant, pour cet exemple, supposons que la table ne contient que les données de la semaine la plus récente.

```sql
CREATE TABLE store_sales_2 (
    day INTEGER,
    sales_today INTEGER
    );
```

```
+-------------------------------------------+
| status                                    |
|-------------------------------------------|
| Table STORE_SALES_2 successfully created. |
+-------------------------------------------+
```

```sql
INSERT INTO store_sales_2 (day, sales_today) VALUES
    (1, 10),
    (2, 14),
    (3,  6),
    (4,  6),
    (5, 14),
    (6, 16),
    (7, 18);
```

```
+-------------------------+
| number of rows inserted |
|-------------------------|
|                       7 |
+-------------------------+
```

### Calcul du classement des ventes

La colonne `Rank` est calculée à l'aide de la fonction `RANK` :

```sql
SELECT day, 
       sales_today, 
       RANK()
           OVER (ORDER BY sales_today DESC) AS Rank
    FROM store_sales_2
    ORDER BY day;
```

```
+-----+-------------+------+
| DAY | SALES_TODAY | RANK |
|-----+-------------+------|
|   1 |          10 |    5 |
|   2 |          14 |    3 |
|   3 |           6 |    6 |
|   4 |           6 |    6 |
|   5 |          14 |    3 |
|   6 |          16 |    2 |
|   7 |          18 |    1 |
+-----+-------------+------+
```

Bien qu'il y ait 7 jours dans la période, il n'y a que 5 classements différents (1, 2, 3, 5, 6). Il y a eu deux égalités (pour la 3ème place et la 6ème place), donc il n'y a pas de lignes avec les classements 4 ou 7.

### Calcul des ventes jusqu'à présent

La colonne `Sales So Far This Week` est calculée en utilisant `SUM` comme fonction de fenêtre avec un cadre de fenêtre :

```sql
SELECT day, 
       sales_today, 
       SUM(sales_today)
           OVER (ORDER BY day
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
               AS "SALES SO FAR THIS WEEK"
    FROM store_sales_2
    ORDER BY day;
```

```
+-----+-------------+------------------------+
| DAY | SALES_TODAY | SALES SO FAR THIS WEEK |
|-----+-------------+------------------------|
|   1 |          10 |                     10 |
|   2 |          14 |                     24 |
|   3 |           6 |                     30 |
|   4 |           6 |                     36 |
|   5 |          14 |                     50 |
|   6 |          16 |                     66 |
|   7 |          18 |                     84 |
+-----+-------------+------------------------+
```

Cette requête ordonne les lignes par date et ensuite, pour chaque date, calcule la somme des ventes depuis le début de la fenêtre jusqu'à la date actuelle (inclusive).

### Calcul du total des ventes cette semaine

La colonne `Total Sales This Week` est calculée en utilisant `SUM`.

```sql
SELECT day, 
       sales_today, 
       SUM(sales_today)
           OVER ()
               AS total_sales
    FROM store_sales_2
    ORDER BY day;
```

```
+-----+-------------+-------------+
| DAY | SALES_TODAY | TOTAL_SALES |
|-----+-------------+-------------|
|   1 |          10 |          84 |
|   2 |          14 |          84 |
|   3 |           6 |          84 |
|   4 |           6 |          84 |
|   5 |          14 |          84 |
|   6 |          16 |          84 |
|   7 |          18 |          84 |
+-----+-------------+-------------+
```

### Calcul d'une moyenne mobile sur trois jours

La colonne `3-Day Moving Average` est calculée en utilisant `AVG` comme fonction de fenêtre avec un cadre de fenêtre :

```sql
SELECT day, 
       sales_today, 
       AVG(sales_today)
           OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
               AS "3-DAY MOVING AVERAGE"
    FROM store_sales_2
    ORDER BY day;
```

```
+-----+-------------+----------------------+
| DAY | SALES_TODAY | 3-DAY MOVING AVERAGE |
|-----+-------------+----------------------|
|   1 |          10 |               10.000 |
|   2 |          14 |               12.000 |
|   3 |           6 |               10.000 |
|   4 |           6 |                8.666 |
|   5 |          14 |                8.666 |
|   6 |          16 |               12.000 |
|   7 |          18 |               16.000 |
+-----+-------------+----------------------+
```

La différence entre ce cadre de fenêtre et le cadre de fenêtre décrit précédemment est le point de départ : une limite fixe par rapport à un décalage explicite.

### Rassembler le tout

A vous de jouer! Quelle est la requête qui permet d'avoir ce tableau avec toutes les colonnes?

```
+-----+-------------+------+------------------------+-------------+----------------------+
| DAY | SALES_TODAY | RANK | SALES SO FAR THIS WEEK | TOTAL_SALES | 3-DAY MOVING AVERAGE |
|-----+-------------+------+------------------------+-------------+----------------------|
|   1 |          10 |    5 |                     10 |          84 |               10.000 |
|   2 |          14 |    3 |                     24 |          84 |               12.000 |
|   3 |           6 |    6 |                     30 |          84 |               10.000 |
|   4 |           6 |    6 |                     36 |          84 |                8.666 |
|   5 |          14 |    3 |                     50 |          84 |                8.666 |
|   6 |          16 |    2 |                     66 |          84 |               12.000 |
|   7 |          18 |    1 |                     84 |          84 |               16.000 |
+-----+-------------+------+------------------------+-------------+----------------------+
```

## Exemples supplémentaires

Cette section fournit plus d'exemples de fonctions de fenêtre et illustre comment les clauses `PARTITION BY` et `ORDER BY` fonctionnent ensemble.

Ces exemples utilisent la table et les données suivantes :

```sql
CREATE TABLE sales (sales_date DATE, quantity INTEGER);

INSERT INTO sales (sales_date, quantity) VALUES
    ('2018-01-01', 1),
    ('2018-01-02', 3),
    ('2018-01-03', 5),
    ('2018-02-01', 2)
    ;
```

### Fonction de fenêtre avec clause `ORDER BY`

La clause `ORDER BY` contrôle l'ordre des données dans chaque fenêtre (et chaque partition s'il y a plus d'une partition). Ceci est utile si vous voulez montrer une "somme courante" dans le temps à mesure que de nouvelles lignes sont ajoutées.

Une somme courante peut être calculée soit du début de la fenêtre à la ligne courante (inclusive), soit de la ligne courante à la fin de la fenêtre.

Une requête peut utiliser une fenêtre "glissante", qui est une fenêtre de largeur fixe qui traite un nombre spécifié de lignes relatif à la ligne courante (par exemple, les 10 lignes les plus récentes, y compris la ligne courante).

### Cadres de fenêtre avec limites fixes

Lorsque le cadre de fenêtre a une limite fixe, les valeurs peuvent être calculées depuis le début de la fenêtre jusqu'à la ligne courante (ou depuis la ligne courante jusqu'à la fin de la fenêtre) :

```sql
SELECT MONTH(sales_date) AS MONTH_NUM, 
       quantity, 
       SUM(quantity) OVER (ORDER BY MONTH(sales_date)
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
           AS CUMULATIVE_SUM_QUANTITY
    FROM sales
    ORDER BY sales_date;
```

Le résultat de la requête inclut des commentaires supplémentaires qui montrent comment la colonne `CUMULATIVE_SUM_QUANTITY` a été calculée :

```
+-----------+----------+-------------------------+
| MONTH_NUM | QUANTITY | CUMULATIVE_SUM_QUANTITY |
|-----------+----------+-------------------------|
|         1 |        1 |                       1 |  -- somme = 1
|         1 |        3 |                       4 |  -- somme = 1 + 3
|         1 |        5 |                       9 |  -- somme = 1 + 3 + 5
|         2 |        2 |                      11 |  -- somme = 1 + 3 + 5 + 2
+-----------+----------+-------------------------+
```

### Cadres de fenêtre avec décalages explicites

Dans le monde financier, les analystes étudient souvent les "moyennes mobiles".

Par exemple, vous pourriez avoir un graphique dans lequel l'axe X est le temps, et l'axe Y montre le prix moyen de l'action au cours des 13 dernières semaines (c'est-à-dire une moyenne mobile sur 13 semaines). Dans un graphique d'une moyenne mobile sur 13 semaines du prix d'une action, le prix affiché pour le 30 juin n'est pas le prix de l'action le 30 juin, mais le prix moyen de l'action pour les 13 semaines jusqu'au 30 juin inclus (du 1er avril au 30 juin). La valeur du 1er juillet est le prix moyen du 2 avril au 1er juillet ; la valeur du 2 juillet est le prix moyen du 3 avril au 2 juillet, et ainsi de suite. Chaque jour, la fenêtre ajoute effectivement la valeur du jour le plus récent à la moyenne mobile et supprime la valeur du jour le plus ancien. Cela lisse les fluctuations jour à jour et peut rendre les tendances plus faciles à reconnaître.

Les moyennes mobiles peuvent être calculées en utilisant un cadre de fenêtre glissant. Le cadre a une largeur spécifique en lignes. Dans l'exemple du prix des actions ci-dessus, 13 semaines représentent 91 jours, donc la fenêtre glissante serait de 91 lignes de "large".

Pour définir une fenêtre de 91 lignes de large :

```sql
SELECT AVG(price) OVER(ORDER BY timestamp1 ROWS BETWEEN 90 PRECEDING AND CURRENT ROW)
  FROM sales;
```

> **Note**
> Le cadre de fenêtre initial peut être inférieur à 91 jours de large. Par exemple, supposons que vous vouliez le prix moyen mobile sur 13 semaines d'une action. Si l'action a été créée pour la première fois le 1er avril, le 3 avril, seulement 3 jours d'information de prix existent, donc la fenêtre n'est large que de 3 lignes.

L'exemple suivant montre le résultat de la sommation sur un cadre de fenêtre glissant suffisamment large pour contenir deux échantillons :

```sql
SELECT MONTH(sales_date) AS MONTH_NUM,
       quantity,
       SUM(quantity) OVER (ORDER BY sales_date
                           ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) 
           AS SLIDING_SUM_QUANTITY
  FROM sales
  ORDER BY sales_date;
```

Le résultat de la requête inclut des commentaires supplémentaires qui montrent comment la colonne `SLIDING_SUM_QUANTITY` a été calculée :

```
+-----------+----------+----------------------+
| MONTH_NUM | QUANTITY | SLIDING_SUM_QUANTITY |
|-----------+----------+----------------------+
|         1 |        1 |                   1  |  -- somme = 1
|         1 |        3 |                   4  |  -- somme = 1 + 3
|         1 |        5 |                   8  |  -- somme = 3 + 5 (1 n'est plus dans la fenêtre)
|         2 |        2 |                   7  |  -- somme = 5 + 2 (3 n'est plus dans la fenêtre)
+-----------+----------+----------------------+
```

Notez que la fonctionnalité de "fenêtre glissante" nécessite la clause `ORDER BY` ; la fonction dépend de l'ordre des lignes qui entrent et sortent du cadre de fenêtre.

### Totaux courants avec clauses `PARTITION BY` et `ORDER BY`

Vous pouvez combiner les clauses `PARTITION BY` et `ORDER BY` pour obtenir des sommes courantes dans les partitions. Dans cet exemple, les partitions sont d'un mois, et parce que les sommes s'appliquent uniquement dans une partition, la somme est réinitialisée à 0 au début de chaque nouveau mois :

```sql
SELECT MONTH(sales_date) AS MONTH_NUM,
       SUM(quantity) OVER (PARTITION BY MONTH(sales_date) ORDER BY sales_date)
          AS MONTHLY_CUMULATIVE_SUM_QUANTITY
    FROM sales
    ORDER BY sales_date;
```

Le résultat de la requête inclut des commentaires supplémentaires montrant comment la colonne `MONTHLY_CUMULATIVE_SUM_QUANTITY` a été calculée :

```
+-----------+---------------------------------+
| MONTH_NUM | MONTHLY_CUMULATIVE_SUM_QUANTITY |
|-----------+---------------------------------+
|         1 |                               1 |  -- somme = 1
|         1 |                               4 |  -- somme = 1 + 3
|         1 |                               9 |  -- somme = 1 + 3 + 5
|         2 |                               2 |  -- somme = 0 + 2 (nouveau mois)
+-----------+---------------------------------+
```

Vous pouvez combiner des partitions et des cadres de fenêtre glissants. Dans l'exemple ci-dessous, la fenêtre glissante a généralement une largeur de deux lignes, mais chaque fois qu'une nouvelle partition (c'est-à-dire un nouveau mois) est atteinte, la fenêtre glissante commence avec seulement la première ligne de cette partition :

```sql
SELECT
       MONTH(sales_date) AS MONTH_NUM,
       quantity,
       SUM(quantity) OVER (PARTITION BY MONTH(sales_date) 
                           ORDER BY sales_date
                           ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) 
         AS MONTHLY_SLIDING_SUM_QUANTITY
    FROM sales
    ORDER BY sales_date;
```

Le résultat de la requête inclut des commentaires supplémentaires montrant comment la colonne `MONTHLY_SLIDING_SUM_QUANTITY` a été calculée :

```
+-----------+----------+------------------------------+
| MONTH_NUM | QUANTITY | MONTHLY_SLIDING_SUM_QUANTITY |
|-----------+----------+------------------------------+
|         1 |        1 |                           1  |  -- somme = 1
|         1 |        3 |                           4  |  -- somme = 1 + 3
|         1 |        5 |                           8  |  -- somme = 3 + 5
|         2 |        2 |                           2  |  -- somme = 0 + 2 (nouveau mois)
+-----------+----------+------------------------------+
```

### Calculer le ratio d'une valeur à une somme de valeurs

Vous pouvez utiliser la fonction `RATIO_TO_REPORT` pour calculer le ratio d'une valeur à la somme des valeurs dans une partition, puis renvoyer le ratio comme un pourcentage de cette somme. La fonction divise la valeur de la ligne courante par la somme des valeurs de toutes les lignes d'une partition.

```sql
SELECT branch_ID,
       city,
       100 * RATIO_TO_REPORT(net_profit) OVER (PARTITION BY city)
    FROM store_sales AS s1
    ORDER BY city, branch_ID;
```

```
+-----------+-----------+------------------------------------------------------------+
| BRANCH_ID | CITY      | 100 * RATIO_TO_REPORT(NET_PROFIT) OVER (PARTITION BY CITY) |
|-----------+-----------+------------------------------------------------------------|
|         3 | Montreal  |                                                52.63157900 |
|         4 | Montreal  |                                                47.36842100 |
|         1 | Vancouver |                                                40.00000000 |
|         2 | Vancouver |                                                60.00000000 |
+-----------+-----------+------------------------------------------------------------+
```

La clause `PARTITION BY` définit des partitions sur la colonne `city`. Si vous voulez voir le pourcentage de profit par rapport à l'ensemble de la chaîne, plutôt que seulement les magasins d'une ville spécifique, omettez la clause `PARTITION BY` :

```sql
SELECT branch_ID,
       100 * RATIO_TO_REPORT(net_profit) OVER ()
    FROM store_sales AS s1
    ORDER BY branch_ID;
```

```
+-----------+-------------------------------------------+
| BRANCH_ID | 100 * RATIO_TO_REPORT(NET_PROFIT) OVER () |
|-----------+-------------------------------------------|
|         1 |                               22.72727300 |
|         2 |                               34.09090900 |
|         3 |                               22.72727300 |
|         4 |                               20.45454500 |
+-----------+-------------------------------------------+
```

Corrigé:
Voici la version finale de la requête, montrant toutes les colonnes :

```sql
SELECT day, 
       sales_today, 
       RANK()
           OVER (ORDER BY sales_today DESC) AS Rank,
       SUM(sales_today)
           OVER (ORDER BY day
               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
               AS "SALES SO FAR THIS WEEK",
       SUM(sales_today)
           OVER ()
               AS total_sales,
       AVG(sales_today)
           OVER (ORDER BY day ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
               AS "3-DAY MOVING AVERAGE"
    FROM store_sales_2
    ORDER BY day;
```
