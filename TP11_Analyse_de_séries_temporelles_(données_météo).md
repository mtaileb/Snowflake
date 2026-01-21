# Introduction aux données de séries temporelles

Une série temporelle (time series) consiste en des observations séquentielles qui capturent l'évolution des systèmes, processus et comportements sur une période donnée. Les données de séries temporelles sont collectées à partir d'une large gamme d'appareils et d'industries. Parmi les exemples courants, on trouve les données de trading boursier collectées pour les applications financières, les observations météorologiques, les relevés de température collectés par des capteurs dans des usines intelligentes et les journaux de clics utilisateurs dans la publicité numérique.

Un enregistrement unique dans une série temporelle comporte généralement les composants suivants :

*   Une date, une heure ou un horodatage ayant un niveau de granularité cohérent (millisecondes, secondes, minutes, heures, etc.).
*   Une ou plusieurs mesures ou métriques de quelque nature, généralement numériques (des faits pouvant révéler des tendances ou des anomalies dans les données).
*   Des dimensions d'intérêt associées à la mesure, telles qu'un emplacement pour un relevé de température ou un symbole boursier pour une transaction donnée.

Par exemple, l'observation météorologique suivante a des horodatages de début et de fin, une mesure de précipitations (0,32) et des informations de localisation :

```
EVENTID | TYPE   | SEVERITY  | START_TIME              | END_TIME                | PRECIP | TIME_ZONE   | CITY       | COUNTY    | STATE | ZIP
W100    | Rain   | Moderate  | 2020-12-20 16:35:00.000 | 2020-12-20 17:15:00.000 |   0.32 | US/Eastern  | Southport  | Brunswick | NC    | 28461
```

Les données suivantes collectées depuis un appareil d'usine ont un espace de nom (`IOT`), un ID de balise ou de capteur (`3000`), un horodatage pour la lecture de température sur l'appareil, la lecture de température elle-même (21,1673), et un "horodatage du broker", qui est le moment où les données sont par la suite arrivées au broker de données. Par exemple, le broker de données pourrait être un serveur Kafka qui ingère les données dans une table Snowflake.

```
DEVICE | LINE | DEVICE_TIMESTAMP        | TEMP     | BROKER_TIMESTAMP
IOT    | 3000 | 2023-01-01 00:01:00.000 | 21.1673  | 2023-01-01 00:01:32.000
```

Une série temporelle peut révéler des pics lorsque les lectures changent brutalement pour une raison quelconque. Par exemple, l'image suivante montre une séquence de relevés de température pris à des intervalles de 15 secondes, avec des valeurs culminant au-dessus de 40°C après être restées régulièrement dans la plage des 35°C le jour précédent.

![Graphique linéaire montrant des relevés de température de capteurs augmentant considérablement pendant un certain temps.](https://github.com/mtaileb/Snowflake/blob/main/images/time-series-data-spike.png?raw=true)

Les sections suivantes montrent comment analyser et visualiser de grands volumes de ce type de données avec des fonctions et des jointures SQL qui fournissent des résultats rapides et précis.

## Comment stocker les données de séries temporelles

Les types de données datetime suivants sont pris en charge :

*   `DATE`
*   `TIME`
*   `TIMESTAMP` (et ses variantes, y compris `TIMESTAMP_TZ`)

Pour des informations sur le chargement, la gestion et l'interrogation des données utilisant ces types de données, voir [Working with date and time values](https://docs.snowflake.com/en/sql-reference/data-types-datetime.html).

Un certain nombre de fonctions SQL couramment utilisées sont disponibles pour aider au stockage et à l'interrogation des données de séries temporelles. Par exemple, vous pouvez utiliser `CONVERT_TIMEZONE` pour convertir les horodatages d'un fuseau horaire à un autre, et vous pouvez utiliser des fonctions comme `EXTRACT` et `TIMEADD` pour manipuler les données temporelles selon les besoins.

> **Note**
> Pour les données `TIMESTAMP_TZ`, Snowflake stocke le décalage d'un fuseau horaire donné, et non le fuseau horaire réel, au moment de la création d'une valeur donnée.

Pour optimiser les performances des requêtes, les tables utilisées pour l'analyse des séries temporelles sont souvent partitionnées par le temps (et parfois aussi par l'ID du capteur ou une dimension similaire). 

![Clustering Keys & Clustered Tables](https://github.com/mtaileb/Snowflake/blob/main/images/tables-clustered2.png?raw=true)

## Agrégation des données de séries temporelles

La gestion des données de séries temporelles peut nécessiter l'agrégation de grands volumes d'enregistrements granulaires en une forme plus résumée (un processus parfois appelé "ré-échantillonnage vers le bas" ou "downsampling"). Étant donné un grand ensemble d'enregistrements avec une granularité temporelle spécifique (millisecondes, secondes, minutes, etc.), vous pouvez regrouper ces enregistrements à une granularité plus grossière, produisant effectivement un échantillon plus petit.

Le ré-échantillonnage vers le bas est précieux car il réduit la taille d'un ensemble de données et ses exigences de stockage. Un niveau de granularité plus grossier réduit également les besoins en ressources de calcul lors de l'exécution des requêtes. Une autre raison clé pour le ré-échantillonnage vers le bas est qu'un grand nombre d'enregistrements dans une série temporelle peuvent être redondants du point de vue d'un analyste. Par exemple, si un capteur émet une nouvelle valeur toutes les secondes, mais que cette mesure change rarement dans chaque intervalle de 60 secondes, les données peuvent être regroupées au niveau de la minute pour l'analyse.

Un autre cas de ré-échantillonnage vers le bas se produit lorsque deux ensembles de données différents doivent être analysés comme un seul, mais qu'ils ont des granularités temporelles différentes. Par exemple, le Capteur A dans une usine collecte des données toutes les 15 secondes, mais le Capteur B collecte des données connexes toutes les 30 secondes. Dans ce cas, agréger les enregistrements en "buckets" d'une minute peut être une bonne solution. Les IDs et dimensions dans chaque ensemble de données sont conservés tels quels, mais les mesures numériques sont sommées ou moyennées par un intervalle de temps commun.

### Exemples de ré-échantillonnage vers le bas

Vous pouvez ré-échantillonner vers le bas un ensemble de données stocké dans une table en utilisant la fonction `TIME_SLICE`. Cette fonction calcule les heures de début et de fin de "buckets" de largeur fixe pour que les enregistrements individuels puissent être groupés et résumés, en utilisant des fonctions d'agrégation standard comme `SUM` et `AVG`.

De même, la fonction `DATE_TRUNC` tronque une partie d'une série de valeurs de date ou d'horodatage, réduisant leur granularité. Les sections suivantes montrent des exemples de chaque fonction.

#### Ré-échantillonnage avec `TIME_SLICE`

L'exemple suivant ré-échantillonne vers le bas une table nommée `sensor_data_ts`, qui contient des lectures de deux capteurs d'usine et contient 5,3 millions de lignes. Ces lectures étaient ingérées par seconde, donc 5,3 millions de lignes représentent seulement un mois de données, avec un peu plus de 2,5 millions de lignes par capteur. Vous pouvez utiliser la fonction `TIME_SLICE` pour agréger jusqu'à une seule ligne par minute, par heure ou par jour, par exemple.

Pour exécuter cet exemple, créez et chargez d'abord la table `sensor_data_ts`.

## Création de la table `sensor_data_ts`

Si vous souhaitez tester les exemples de cette section qui interrogent la table `sensor_data_ts`, vous pouvez créer et charger une copie de cette table en exécutant le script SQL suivant. Le script génère un mois de données synthétiques de lectures de capteurs en appelant les fonctions `UNIFORM`, `RANDOM`, et `GENERATOR` ; par conséquent, votre copie de la table ne renverra pas des résultats identiques. Les lectures seront dans la même plage mais ne seront pas les mêmes.

```sql
CREATE OR REPLACE TABLE sensor_data_device1 (
   device_id VARCHAR(10),
   timestamp TIMESTAMP,
   temperature DECIMAL(6,4),
   vibration DECIMAL(6,4),
   motor_rpm INT
);

INSERT INTO sensor_data_device1 (device_id, timestamp, temperature, vibration, motor_rpm)
   SELECT 'DEVICE1', timestamp,
     UNIFORM(25.1111, 40.2222, RANDOM()), -- Plage de température en °C
     UNIFORM(0.2985, 0.3412, RANDOM()), -- Plage de vibration en mm/s
     UNIFORM(1400, 1495, RANDOM()) -- Plage de RPM moteur
   FROM (
     SELECT DATEADD(SECOND, SEQ4(), '2024-03-01') AS timestamp
       FROM TABLE(GENERATOR(ROWCOUNT => 2678400)) -- secondes dans 31 jours
);

CREATE OR REPLACE TABLE sensor_data_device2 (
   device_id VARCHAR(10),
   timestamp TIMESTAMP,
   temperature DECIMAL(6,4),
   vibration DECIMAL(6,4),
   motor_rpm INT
);

INSERT INTO sensor_data_device2 (device_id, timestamp, temperature, vibration, motor_rpm)
   SELECT 'DEVICE2', timestamp,
     UNIFORM(24.6642, 36.3107, RANDOM()), -- Plage de température en °C
     UNIFORM(0.2876, 0.3333, RANDOM()), -- Plage de vibration en mm/s
     UNIFORM(1425, 1505, RANDOM()) -- Plage de RPM moteur
   FROM (
     SELECT DATEADD(SECOND, SEQ4(), '2024-03-01') AS timestamp
       FROM TABLE(GENERATOR(ROWCOUNT => 2678400)) -- secondes dans 31 jours
);

INSERT INTO sensor_data_device1 SELECT * FROM sensor_data_device2;

DROP TABLE IF EXISTS sensor_data_ts;

ALTER TABLE sensor_data_device1 RENAME TO sensor_data_ts;

DROP TABLE sensor_data_device2;

SELECT COUNT(*) FROM sensor_data_ts; -- vérifier le nombre de lignes = 5356800
```

## Création de la table `heavy_weather`

Le script suivant crée et charge la table `heavy_weather`, qui est utilisée dans les exemples pour les fonctions `MAX_BY`. La table contient 55 lignes d'enregistrements de précipitations de neige pour des villes californiennes pendant la dernière semaine de 2021.

```sql
CREATE OR REPLACE TABLE heavy_weather
   (start_time TIMESTAMP, precip NUMBER(3,2), city VARCHAR(20), county VARCHAR(20));

INSERT INTO heavy_weather VALUES
  ('2021-12-23 06:56:00.000',0.08,'Mount Shasta','Siskiyou'),
  ('2021-12-23 07:51:00.000',0.09,'Mount Shasta','Siskiyou'),
  ('2021-12-23 16:23:00.000',0.56,'South Lake Tahoe','El Dorado'),
  ('2021-12-23 17:24:00.000',0.38,'South Lake Tahoe','El Dorado'),
  ('2021-12-23 18:30:00.000',0.28,'South Lake Tahoe','El Dorado'),
  ('2021-12-23 19:35:00.000',0.37,'Mammoth Lakes','Mono'),
  ('2021-12-23 19:36:00.000',0.80,'South Lake Tahoe','El Dorado'),
  ('2021-12-24 04:43:00.000',0.25,'Alta','Placer'),
  ('2021-12-24 05:26:00.000',0.34,'Alta','Placer'),
  ('2021-12-24 05:35:00.000',0.42,'Big Bear City','San Bernardino'),
  ('2021-12-24 06:49:00.000',0.17,'South Lake Tahoe','El Dorado'),
  ('2021-12-24 07:40:00.000',0.07,'Alta','Placer'),
  ('2021-12-24 08:36:00.000',0.07,'Alta','Placer'),
  ('2021-12-24 11:52:00.000',0.08,'Alta','Placer'),
  ('2021-12-24 12:52:00.000',0.38,'Alta','Placer'),
  ('2021-12-24 15:44:00.000',0.13,'Alta','Placer'),
  ('2021-12-24 15:53:00.000',0.07,'South Lake Tahoe','El Dorado'),
  ('2021-12-24 16:55:00.000',0.09,'Big Bear City','San Bernardino'),
  ('2021-12-24 21:53:00.000',0.07,'Montague','Siskiyou'),
  ('2021-12-25 02:52:00.000',0.07,'Alta','Placer'),
  ('2021-12-25 07:52:00.000',0.07,'Alta','Placer'),
  ('2021-12-25 08:52:00.000',0.08,'Alta','Placer'),
  ('2021-12-25 09:48:00.000',0.18,'Alta','Placer'),
  ('2021-12-25 12:52:00.000',0.10,'Alta','Placer'),
  ('2021-12-25 17:21:00.000',0.23,'Alturas','Modoc'),
  ('2021-12-25 17:52:00.000',1.54,'Alta','Placer'),
  ('2021-12-26 01:52:00.000',0.61,'Alta','Placer'),
  ('2021-12-26 05:43:00.000',0.16,'South Lake Tahoe','El Dorado'),
  ('2021-12-26 05:56:00.000',0.08,'Bishop','Inyo'),
  ('2021-12-26 06:52:00.000',0.75,'Bishop','Inyo'),
  ('2021-12-26 06:53:00.000',0.08,'Lebec','Los Angeles'),
  ('2021-12-26 07:52:00.000',0.65,'Alta','Placer'),
  ('2021-12-26 09:52:00.000',2.78,'Alta','Placer'),
  ('2021-12-26 09:55:00.000',0.07,'Big Bear City','San Bernardino'),
  ('2021-12-26 14:22:00.000',0.32,'Alta','Placer'),
  ('2021-12-26 14:52:00.000',0.34,'Alta','Placer'),
  ('2021-12-26 15:43:00.000',0.35,'Alta','Placer'),
  ('2021-12-26 17:31:00.000',5.24,'Alta','Placer'),
  ('2021-12-26 22:52:00.000',0.07,'Alta','Placer'),
  ('2021-12-26 23:15:00.000',0.52,'Alta','Placer'),
  ('2021-12-27 02:52:00.000',0.08,'Alta','Placer'),
  ('2021-12-27 03:52:00.000',0.14,'Alta','Placer'),
  ('2021-12-27 04:52:00.000',1.52,'Alta','Placer'),
  ('2021-12-27 14:37:00.000',0.89,'Alta','Placer'),
  ('2021-12-27 14:53:00.000',0.07,'South Lake Tahoe','El Dorado'),
  ('2021-12-27 17:53:00.000',0.07,'South Lake Tahoe','El Dorado'),
  ('2021-12-30 11:23:00.000',0.12,'Lebec','Los Angeles'),
  ('2021-12-30 11:43:00.000',0.98,'Lebec','Los Angeles'),
  ('2021-12-30 13:53:00.000',0.23,'Lebec','Los Angeles'),
  ('2021-12-30 14:53:00.000',0.13,'Lebec','Los Angeles'),
  ('2021-12-30 15:15:00.000',0.29,'Lebec','Los Angeles'),
  ('2021-12-30 17:53:00.000',0.10,'Lebec','Los Angeles'),
  ('2021-12-30 18:53:00.000',0.09,'Lebec','Los Angeles'),
  ('2021-12-30 19:53:00.000',0.07,'Lebec','Los Angeles'),
  ('2021-12-30 20:53:00.000',0.07,'Lebec','Los Angeles')
  ;
```

Aperçu de la table:

```
+-----------+-------------------------+-------------+-----------+-----------+
| DEVICE_ID | TIMESTAMP               | TEMPERATURE | VIBRATION | MOTOR_RPM |
|-----------+-------------------------+-------------+-----------+-----------|
| DEVICE1   | 2024-03-01 00:00:00.000 |     32.6908 |    0.3158 |      1492 |
| DEVICE2   | 2024-03-01 00:00:00.000 |     35.2086 |    0.3232 |      1461 |
| DEVICE1   | 2024-03-01 00:00:01.000 |     35.9578 |    0.3302 |      1452 |
| DEVICE2   | 2024-03-01 00:00:01.000 |     26.2468 |    0.3029 |      1455 |
+-----------+-------------------------+-------------+-----------+-----------+
```

La table contient 60 lectures comme celles-ci par minute pour chaque appareil, comme le montre cette requête :

```sql
SELECT device_id, count(*) FROM sensor_data_ts
  WHERE TIMESTAMP >= ('2024-03-01 00:01:00')
    AND TIMESTAMP < ('2024-03-01 00:02:00')
  GROUP BY device_id;
```

```
+-----------+----------+
| DEVICE_ID | COUNT(*) |
|-----------+----------|
| DEVICE2   |       60 |
| DEVICE1   |       60 |
+-----------+----------+
```

Dans cette requête de ré-échantillonnage, la fonction `TIME_SLICE` définit des buckets d'une minute et renvoie l'heure de début de chaque bucket. La fonction `AVG` calcule la température moyenne pour chaque bucket par appareil. La fonction `COUNT(*)` est incluse pour référence, simplement pour montrer combien de lignes atterrissent dans chaque bucket temporel.

Les colonnes `vibration` et `motor_rpm` ne sont pas incluses, mais elles pourraient être agrégées de la même manière que la colonne `temperature` ou en utilisant différentes fonctions d'agrégation.

> **Important**
> Si vous exécutez cet exemple vous-même, votre sortie ne correspondra pas exactement car la table `sensor_data_ts` est chargée avec des valeurs générées aléatoirement.

```sql
SELECT
    TIME_SLICE(TO_TIMESTAMP_NTZ(timestamp), 1, 'MINUTE') minute_slice,
    device_id,
    COUNT(*),
    AVG(temperature) avg_temp
  FROM sensor_data_ts
  WHERE TIMESTAMP >= ('2024-03-01 00:01:00')
    AND TIMESTAMP < ('2024-03-01 00:02:00')
  GROUP BY 1,2
  ORDER BY 1,2;
```

```
+-------------------------+-----------+----------+---------------+
| MINUTE_SLICE            | DEVICE_ID | COUNT(*) |      AVG_TEMP |
|-------------------------+-----------+----------+---------------|
| 2024-03-01 00:01:00.000 | DEVICE1   |       60 | 32.4315466667 |
| 2024-03-01 00:01:00.000 | DEVICE2   |       60 | 30.4967783333 |
+-------------------------+-----------+----------+---------------+
```

Explication de la ligne de code om est utilisée `TIME_SLICE`:

1. timestamp (colonne d'entrée)

    C'est la colonne source contenant les horodatages bruts

    Dans l'exemple, ce sont des lectures de capteurs prises chaque seconde

2. TO_TIMESTAMP_NTZ(timestamp) (conversion de type)

    TO_TIMESTAMP_NTZ() = Convertit en TIMESTAMP No Time Zone (sans fuseau horaire)

    Assure que le format est correct pour TIME_SLICE()

    NTZ = "No Time Zone" - utile quand les données n'ont pas de fuseau horaire spécifique

3. TIME_SLICE(..., 1, 'MINUTE') (la fonction principale)

    TIME_SLICE() : Fonction qui divise le temps en intervalles fixes

    1 : Largeur de l'intervalle = 1 unité

    'MINUTE' : Unité de l'intervalle = minutes

    Traduction : "Regroupe les données par intervalles de 1 minute"

4. minute_slice (alias de colonne)

    Nom donné au résultat dans le SELECT

    Contiendra l'heure de début de chaque intervalle d'une minute

En utilisant la fonction `TIME_SLICE`, vous pouvez créer des tables agrégées plus petites pour l'analyse, et vous pouvez appliquer le processus de ré-échantillonnage à différents niveaux (heure, jour, semaine, etc.).

#### Ré-échantillonnage avec `DATE_TRUNC`

L'exemple suivant sélectionne des données d'une table nommée `order_header` dans le schéma `raw.pos` de l'exemple de base de données Tasty Bytes. Cette table contient 248M de lignes (pour charger cette base de données voir ici: https://docs.snowflake.com/en/user-guide/tutorials/tasty-bytes-sql-load#step-4-use-a-database-schema-and-table).

La table `order_header` a une colonne `TIMESTAMP` nommée `order_ts`. La requête crée une série temporelle agrégée en utilisant cette colonne comme second argument de la fonction `DATE_TRUNC`. Le premier argument spécifie un intervalle de jour. Cela signifie que les enregistrements individuels, qui ont une granularité heures/minutes/secondes, sont regroupés par jour.

La requête groupe les enregistrements par deux dimensions : `truck_id` et `location_id`. La colonne `avg_amount` renvoie le prix moyen par commande, par camion de restauration, par emplacement pour chaque jour ouvrable enregistré.

La requête montrée ici limite les résultats aux 25 premières lignes du 1er janvier 2022. Si vous supprimez ce filtre de date et la clause `LIMIT`, la requête ré-échantillonne vers le bas les 248M lignes originales à environ 500 000 lignes.

```sql
SELECT DATE_TRUNC('day', order_ts)::date sliced_ts, truck_id, location_id, AVG(order_amount)::NUMBER(4,2) as avg_amount
  FROM order_header
  WHERE EXTRACT(YEAR FROM order_ts)='2022'
  GROUP BY date_trunc('day', order_ts), truck_id, location_id
  ORDER BY 1, 2, 3 LIMIT 25; -- Trie par : 1=date, 2=camion, 3=lieu
```

```
+------------+----------+-------------+------------+
| SLICED_TS  | TRUCK_ID | LOCATION_ID | AVG_AMOUNT |
|------------+----------+-------------+------------|
| 2022-01-01 |        1 |        3223 |      19.23 |
| 2022-01-01 |        1 |        3869 |      20.15 |
| 2022-01-01 |        2 |        2401 |      39.29 |
| 2022-01-01 |        2 |        4199 |      34.29 |
| 2022-01-01 |        3 |        2883 |      35.01 |
| 2022-01-01 |        3 |        2961 |      39.15 |
| 2022-01-01 |        4 |        2614 |      35.95 |
| 2022-01-01 |        4 |        2899 |      40.29 |
| 2022-01-01 |        6 |        1946 |      26.58 |
| 2022-01-01 |        6 |       14960 |      18.59 |
| 2022-01-01 |        7 |        1427 |      26.91 |
| 2022-01-01 |        7 |        3224 |      28.88 |
| 2022-01-01 |        9 |        1557 |      35.52 |
| 2022-01-01 |        9 |        2612 |      43.80 |
| 2022-01-01 |       10 |        2217 |      32.35 |
| 2022-01-01 |       10 |        2694 |      32.23 |
| 2022-01-01 |       11 |        2656 |      44.23 |
| 2022-01-01 |       11 |        3327 |      52.00 |
| 2022-01-01 |       12 |        3181 |      52.84 |
| 2022-01-01 |       12 |        3622 |      49.59 |
| 2022-01-01 |       13 |        2516 |      31.13 |
| 2022-01-01 |       13 |        3876 |      28.13 |
| 2022-01-01 |       14 |        1359 |      72.04 |
| 2022-01-01 |       14 |        2505 |      68.75 |
| 2022-01-01 |       15 |        2901 |      41.90 |
+------------+----------+-------------+------------+
```

## Utilisation d'agrégations fenêtrées pour des calculs glissants

En utilisant des fonctions d'agrégation fenêtrées pour observer comment une métrique évolue dans le temps, vous pouvez analyser une série temporelle pour détecter des tendances. Les agrégations fenêtrées sont utiles pour analyser les données au sein de sous-ensembles définis ("fenêtres") d'un plus grand ensemble de données. Vous pouvez calculer des calculs glissants (comme des moyennes mobiles et des sommes) pour chaque ligne d'un ensemble de données, en tenant compte d'un groupe de lignes avant, après ou autour de la ligne courante. Ce type d'analyse contraste avec les agrégations régulières, qui résument l'ensemble des données.

En utilisant des cadres de fenêtre basés sur une plage (`RANGE`) avec des décalages explicites, vous pouvez appliquer une approche très flexible pour calculer ces agrégations glissantes. Le cadre de fenêtre `RANGE BETWEEN`, ordonné soit par des horodatages, soit par des nombres, n'est pas perturbé par les écarts qui peuvent survenir dans les données de séries temporelles. Par exemple, dans l'illustration suivante, le fait que les données du Jour 4 manquent dans la série d'enregistrements n'affecte pas le calcul des fonctions d'agrégation sur une fenêtre mobile de trois jours. En particulier, les cadres 3, 4 et 5 sont calculés correctement, en tenant compte du fait que les données du Jour 4 sont inconnues.
![Graphique montrant un cadre de fenêtre mobile pour sept jours avec un enregistrement manquant pour le Jour 4.](https://docs.snowflake.com/en/_images/window-functions-sliding-frame-gap.png)

L'exemple suivant calcule une somme mobile sur des données météorologiques qui enregistrent des relevés de précipitations horaires dans différentes villes et comtés. Vous pouvez exécuter ce type de requête pour évaluer les tendances dans divers ensembles de données de séries temporelles, tels que les capteurs et autres appareils IoT, en particulier lorsque ces ensembles de données sont connus ou supposés avoir des écarts.

La fonction de fenêtre inclut dans son cadre la lecture de précipitations courante et toutes les lectures qui se situent dans l'intervalle de temps spécifié avant la lecture courante. Le calcul glissant est basé sur cette plage logique et flexible de lignes plutôt que sur un nombre exact de lignes. La première ligne pour chaque ville a des valeurs `precip` et `moving_sum_precip` correspondantes. Ensuite, la somme est recalculée pour chaque ligne suivante dans le cadre. Les valeurs brutes fluctuent considérablement, mais les sommes mobiles ont un fort effet de lissage.

Pour exécuter cet exemple, suivez d'abord ces instructions : [Créer et charger la table heavy_weather](#création-de-la-table-heavy_weather). Cette toute petite table contient des observations météorologiques horaires sporadiques, avec beaucoup d'écarts, y compris un jour manquant. La requête renvoie la somme mobile des valeurs de précipitations ordonnées par la colonne `start_time`. Le cadre de fenêtre définit une plage entre 12 heures avant la ligne courante et la ligne courante. Par conséquent, le cadre se compose de la ligne courante plus seulement les lignes qui ont des horodatages jusqu'à 12 heures plus tôt que l'horodatage `ORDER BY` de la ligne courante.

```sql
SELECT city, start_time, precip,
    SUM(precip) OVER(
      PARTITION BY city
      ORDER BY start_time
      RANGE BETWEEN INTERVAL '12 hours' PRECEDING AND CURRENT ROW) moving_sum_precip
  FROM heavy_weather
  WHERE city IN('South Lake Tahoe','Big Bear City')
  GROUP BY city, precip, start_time
  ORDER BY city;
```

```
+------------------+-------------------------+--------+-------------------+
| CITY             | START_TIME              | PRECIP | MOVING_SUM_PRECIP |
|------------------+-------------------------+--------+-------------------|
| Big Bear City    | 2021-12-24 05:35:00.000 |   0.42 |              0.42 |
| Big Bear City    | 2021-12-24 16:55:00.000 |   0.09 |              0.51 |
| Big Bear City    | 2021-12-26 09:55:00.000 |   0.07 |              0.07 |
| South Lake Tahoe | 2021-12-23 16:23:00.000 |   0.56 |              0.56 |
| South Lake Tahoe | 2021-12-23 17:24:00.000 |   0.38 |              0.94 |
| South Lake Tahoe | 2021-12-23 18:30:00.000 |   0.28 |              1.22 |
| South Lake Tahoe | 2021-12-23 19:36:00.000 |   0.80 |              2.02 |
| South Lake Tahoe | 2021-12-24 06:49:00.000 |   0.17 |              0.97 |
| South Lake Tahoe | 2021-12-24 15:53:00.000 |   0.07 |              0.24 |
| South Lake Tahoe | 2021-12-26 05:43:00.000 |   0.16 |              0.16 |
| South Lake Tahoe | 2021-12-27 14:53:00.000 |   0.07 |              0.07 |
| South Lake Tahoe | 2021-12-27 17:53:00.000 |   0.07 |              0.14 |
+------------------+-------------------------+--------+-------------------+
```

Les trois valeurs `moving_sum_precip` pour Big Bear City sont calculées comme suit :
*   0,42 = 0,42 (pas de lignes précédentes)
*   0,42 + 0,09 = 0,51 (les deux premières lignes sont dans la fenêtre de 12 heures)
*   0,07 = 0,07 (aucune ligne précédente n'est dans la fenêtre de 12 heures)

Les lignes de South Lake Tahoe incluent ces calculs, par exemple :
*   0,56 + 0,38 + 0,28 + 0,80 = 2,02 (les quatre lignes pour le 2024-12-23 sont à moins de 12 heures les unes des autres)
*   0,80 + 0,17 = 0,97 (une ligne précédente est dans la fenêtre de 12 heures)

D'autres fonctions de fenêtre, telles que les fonctions de classement `LEAD` et `LAG`, sont également couramment utilisées dans l'analyse des séries temporelles. Utilisez la fonction de fenêtre `LEAD` pour trouver le prochain point de données dans la série temporelle, par rapport au point de données courant, et la fonction `LAG` pour trouver le point de données précédent.

## Visualisation des résultats de requête dans Snowsight

Vous pouvez utiliser Snowsight pour visualiser les résultats des requêtes d'agrégation et mieux comprendre l'effet de lissage des calculs avec des cadres de fenêtre glissants. Dans la feuille de requête, cliquez sur le bouton **Chart** à côté de **Results**.

Par exemple, la ligne jaune dans le graphique à barres suivant montre une tendance beaucoup plus lisse pour la température moyenne par rapport à la ligne bleue pour la température brute. La requête elle-même ressemble à ceci :

```sql
SELECT device_id, timestamp, temperature, AVG(temperature)
  OVER (PARTITION BY device_id ORDER BY timestamp
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_temp
FROM sensor_data_ts
WHERE timestamp BETWEEN '2024-03-15 00:00:59.000' AND '2024-03-15 00:01:10.000'
ORDER BY 1, 2;
```

![Graphique linéaire montrant une ligne plus irrégulière dans le temps pour la température et une ligne plus lisse pour la température moyenne.](https://docs.snowflake.com/en/_images/window-functions-sliding-frame-chart.png)

## Utilisation des fonctions d'agrégation `MIN_BY` et `MAX_BY`

La capacité à sélectionner une colonne en fonction de la valeur minimale ou maximale d'une autre colonne dans la même ligne est une exigence courante pour les développeurs SQL travaillant avec des données de séries temporelles. `MIN_BY` et `MAX_BY` sont des fonctions pratiques qui renvoient les valeurs de début et de fin (ou les plus hautes et les plus basses, ou les premières et dernières) dans une table lorsque les données sont triées par une autre colonne, comme un horodatage.

Le premier exemple trouve simplement la dernière valeur `precip` (la plus récente) dans toute la table. La fonction `MAX_BY` trie toutes les lignes par leur valeur `start_time`, puis renvoie la valeur `precip` pour l'heure de début "max".

Pour créer et charger la table utilisée dans les exemples suivants, voir [Création de la table heavy_weather](#création-de-la-table-heavy_weather).

```sql
SELECT MAX_BY(precip, start_time) most_recent_precip
  FROM heavy_weather;
```

```
+--------------------+
| MOST_RECENT_PRECIP |
|--------------------|
|               0.07 |
+--------------------+
```

Vous pouvez vérifier ce résultat (et obtenir plus d'informations à son sujet) en exécutant cette requête :

```sql
SELECT * FROM heavy_weather WHERE start_time=
  (SELECT MAX(start_time) FROM heavy_weather);
```

```
+-------------------------+--------+-------+-------------+
| START_TIME              | PRECIP | CITY  | COUNTY      |
|-------------------------+--------+-------+-------------|
| 2021-12-30 20:53:00.000 |   0.07 | Lebec | Los Angeles |
+-------------------------+--------+-------+-------------+
```

Vous pouvez ajouter une clause `GROUP BY` pour poser des questions plus intéressantes sur ces données. Par exemple, la requête suivante trouve la dernière valeur de précipitation observée pour chaque ville en Californie, triées par valeurs `precip` (du plus élevé au plus bas). Les résultats sont groupés par ville pour renvoyer la dernière valeur `precip` pour chaque ville différente.

```sql
SELECT city, MAX_BY(precip, start_time) most_recent_precip
  FROM heavy_weather
  GROUP BY city
  ORDER BY 2 DESC;
```

```
+------------------+--------------------+
| CITY             | MOST_RECENT_PRECIP |
|------------------+--------------------|
| Alta             |               0.89 |
| Bishop           |               0.75 |
| Mammoth Lakes    |               0.37 |
| Alturas          |               0.23 |
| Mount Shasta     |               0.09 |
| South Lake Tahoe |               0.07 |
| Big Bear City    |               0.07 |
| Montague         |               0.07 |
| Lebec            |               0.07 |
+------------------+--------------------+
```

La dernière fois qu'une observation a été prise pour la ville d'Alta, la valeur `precip` était de 0,89, et la dernière fois qu'une observation a été prise pour les villes de South Lake Tahoe, Big Bear City, Montague et Lebec, la valeur `precip` était de 0,07 pour les quatre emplacements. (Notez que la requête ne vous dit pas quand ces observations ont été prises.)

Vous pouvez renvoyer l'ensemble de résultats "opposé" (enregistrement `precip` le plus ancien par rapport au plus récent) en utilisant la fonction `MIN_BY`.

```sql
SELECT city, MIN_BY(precip, start_time) oldest_precip
  FROM heavy_weather
  GROUP BY city
  ORDER BY 2 DESC;
```

```
+------------------+---------------+
| CITY             | OLDEST_PRECIP |
|------------------+---------------|
| South Lake Tahoe |          0.56 |
| Big Bear City    |          0.42 |
| Mammoth Lakes    |          0.37 |
| Alta             |          0.25 |
| Alturas          |          0.23 |
| Bishop           |          0.08 |
| Lebec            |          0.08 |
| Mount Shasta     |          0.08 |
| Montague         |          0.07 |
+------------------+---------------+
```

## Jointure des données de séries temporelles

Vous pouvez utiliser la construction `ASOF JOIN` pour joindre des tables contenant des données de séries temporelles. Bien que les requêtes `ASOF JOIN` puissent être émulées par l'utilisation de SQL complexe, d'autres types de jointures et de fonctions de fenêtre, ces requêtes sont plus faciles à écrire (et sont optimisées) si vous utilisez la syntaxe `ASOF JOIN`.

Une utilisation courante des jointures ASOF est l'analyse des données de trading financier. L'analyse des coûts de transaction, par exemple, nécessite des calculs de "slippage", qui mesurent la différence entre le prix coté au moment d'une décision d'achat d'actions et le prix réellement payé lorsque la transaction a été exécutée et enregistrée. L'`ASOF JOIN` peut accélérer ce type d'analyse. Étant donné que la capacité clé de cette méthode de jointure est l'analyse d'une série temporelle par rapport à une autre, l'`ASOF JOIN` peut être utile pour analyser tout ensemble de données de nature historique. Dans beaucoup de ces cas d'utilisation, l'`ASOF JOIN` peut être utilisé pour associer des données lorsque les lectures de différents appareils ont des horodatages qui ne sont pas exactement les mêmes.

L'hypothèse est que les données de séries temporelles que vous devez analyser existent dans deux tables, et il y a un horodatage pour chaque ligne dans chaque table. Cet horodatage représente la date et l'heure précises "à partir de" pour un événement enregistré. Pour chaque ligne de la première (ou gauche) table, la jointure utilise une "condition de correspondance" avec un opérateur de comparaison que vous spécifiez pour trouver une seule ligne dans la seconde (ou droite) table où la valeur d'horodatage est l'une des suivantes :

*   Inférieure ou égale à la valeur d'horodatage dans la table gauche.
*   Supérieure ou égale à la valeur d'horodatage dans la table gauche.
*   Inférieure à la valeur d'horodatage dans la table gauche.
*   Supérieure à la valeur d'horodatage dans la table gauche.

La ligne qualifiée du côté droit est la correspondance la plus proche, qui pourrait être égale dans le temps, plus tôt dans le temps ou plus tard dans le temps, selon l'opérateur de comparaison spécifié.

La cardinalité du résultat de l'`ASOF JOIN` est toujours égale à la cardinalité de la table gauche. Si la table gauche contient 40 millions de lignes, l'`ASOF JOIN` renvoie 40 millions de lignes. Par conséquent, la table gauche peut être considérée comme la table "préservante", et la table droite comme la table "référencée".

### Joindre deux tables sur la correspondance la plus proche (alignement)

Par exemple, dans une application financière, vous pourriez avoir une table nommée `quotes` et une table nommée `trades`. Une table enregistre l'historique des offres d'achat d'actions, et l'autre enregistre l'historique des transactions réelles. Une offre d'achat d'actions se produit avant la transaction (ou peut-être au "même" moment, selon la granularité de l'heure enregistrée). Les deux tables ont des horodatages, et les deux ont d'autres colonnes d'intérêt que vous pourriez vouloir comparer. Une requête `ASOF JOIN` simple renverra la cotation la plus proche (dans le temps) avant chaque transaction. En d'autres termes, la requête demande : Quel était le prix d'une action donnée au moment où j'ai effectué une transaction ?

Supposons que la table `trades` contienne trois lignes et la table `quotes` sept lignes. La couleur de fond des cellules montre quelles trois lignes de `quotes` seront qualifiées pour l'`ASOF JOIN` lorsque les lignes sont jointes sur des symboles boursiers correspondants et que leurs colonnes d'horodatage sont comparées.

**Table TRADES (Table Gauche ou "Préservante")**
```
| STOCK_SYMBOL | TRADE_TIME              | QUANTITY |
|--------------|-------------------------|----------|
| SNOW         | 2023-10-01 09:00:05.000 |     1000 |
| AAPL         | 2023-10-01 09:00:05.000 |     2000 |
| SNOW         | 2023-10-01 09:00:10.000 |     1500 |
```
*Données de la table trades, constituées de trois lignes, qui sont jointes avec trois lignes de la table quotes.*

**Table QUOTES (Table Droite ou "Référencée")**
```
| STOCK_SYMBOL | QUOTE_TIME              | PRICE |
|--------------|-------------------------|-------|
| SNOW         | 2023-10-01 09:00:01.000 | 166.00|
| SNOW         | 2023-10-01 09:00:02.000 | 163.00|
| SNOW         | 2023-10-01 09:00:07.000 | 166.00|
| SNOW         | 2023-10-01 09:00:08.000 | 165.00|
| AAPL         | 2023-10-01 09:00:03.000 | 139.00|
| AAPL         | 2023-10-01 09:00:07.000 | 142.00|
| AAPL         | 2023-10-01 09:00:11.000 | 142.00|
```
*Données de la table quotes, constituées de sept lignes, identifiant les trois lignes spécifiques qui se qualifient pour la jointure avec la table trades.*

Cet exemple conceptuel est facile à transformer en requête `ASOF JOIN` spécifique :

```sql
SELECT t.stock_symbol, t.trade_time, t.quantity, q.quote_time, q.price
  FROM trades t ASOF JOIN quotes q
    MATCH_CONDITION(t.trade_time >= quote_time)
    ON t.stock_symbol=q.stock_symbol
  ORDER BY t.stock_symbol;
```

```
+--------------+-------------------------+----------+-------------------------+--------------+
| STOCK_SYMBOL | TRADE_TIME              | QUANTITY | QUOTE_TIME              |        PRICE |
|--------------+-------------------------+----------+-------------------------+--------------|
| AAPL         | 2023-10-01 09:00:05.000 |     2000 | 2023-10-01 09:00:03.000 | 139.00000000 |
| SNOW         | 2023-10-01 09:00:05.000 |     1000 | 2023-10-01 09:00:02.000 | 163.00000000 |
| SNOW         | 2023-10-01 09:00:10.
