In this tutorial you will learn the basics of using JSON with Snowflake.
What you will learn

In this tutorial, you learn how to do the following:

    Upload sample JSON data from a public S3 bucket into a column of the variant type in a Snowflake table.

    Test simple queries for JSON data in the table.

    Explore the FLATTEN function to flatten JSON data into a relational representation and save it in another table.

    Explore ways to ensure uniqueness as you insert rows in the flattened version of the data.

### **Prérequis**

Ce tutoriel suppose que :

*   Vous disposez d'un **compte Snowflake** configuré pour utiliser **Amazon AWS** et d'un **utilisateur** ayant un **rôle** accordant les **privilèges nécessaires** pour créer une base de données, des tables et des objets d'entrepôt virtuel.
*   Vous avez **SnowSQL (client CLI) installé**.

Le tutoriel **Snowflake en 20 minutes** fournit les instructions étape par étape pour remplir ces conditions.

Snowflake fournit des **fichiers de données d'exemple** dans un **bucket S3 public** pour ce tutoriel. Mais avant de commencer, vous devez créer une **base de données, des tables, un entrepôt virtuel et un *stage* externe**. Ce sont les **objets Snowflake de base** nécessaires pour la plupart des activités Snowflake.

---

### **À propos du fichier de données d'exemple**
Pour ce tutoriel, vous utilisez des **données d'événements d'application JSON** fournies dans un bucket S3 public.

**Exemple de structure JSON :**
```json
{
"device_type": "server",
"events": [
  {
    "f": 83,
    "rv": "15219.64,783.63,48674.48,84679.52,27499.78,2178.83,0.42,74900.19",
    "t": 1437560931139,
    "v": {
      "ACHZ": 42869,
      "ACV": 709489,
      "DCA": 232,
      "DCV": 62287,
      "ENJR": 2599,
      "ERRS": 205,
      "MXEC": 487,
      "TMPI": 9
    },
    "vd": 54,
    "z": 1437644222811
  },
  ...
],
"version": 2.6
}
```

**Explication des données :**
*   Elles représentent des **événements** que des applications envoient vers S3.
*   Les applications peuvent **grouper des événements** en **lots** (*batches*). Un lot contient des **en-têtes communs** (`device_type`, `version`) à tous ses événements.
*   Amazon S3 permet d'utiliser le concept de **dossiers** pour organiser un bucket. Les applications peuvent utiliser cette fonctionnalité pour **partitionner les données d'événements**. Les schémas de partitionnement identifient généralement des détails comme l'application, la localisation et la date de l'événement.

**Exemples de chemins partitionnés :**
```
s3://nom_du_bucket/application_a/2016/07/01/11/
s3://nom_du_bucket/application_b/location_c/2016/07/01/14/
```

**Note :** S3 transmet une **liste de répertoires** avec chaque instruction `COPY` utilisée par Snowflake. **Réduire le nombre de fichiers dans chaque répertoire améliore les performances** des commandes `COPY`. Vous pouvez même envisager de créer des dossiers par incréments de 10-15 minutes dans chaque heure.

---

### **Création de la base de données, de la table, de l'entrepôt et du *stage* externe**
Exécutez les instructions suivantes pour créer les objets nécessaires au tutoriel. Vous pourrez les supprimer après avoir terminé.

```sql
-- Créer la base de données
CREATE OR REPLACE DATABASE mydatabase;

-- Utiliser le schéma par défaut 'public'
USE SCHEMA mydatabase.public;

-- Créer la table cible pour les données JSON
CREATE OR REPLACE TABLE raw_source (
  SRC VARIANT
);

-- Créer un entrepôt virtuel X-Small
CREATE OR REPLACE WAREHOUSE mywarehouse WITH
  WAREHOUSE_SIZE='X-SMALL'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED=TRUE;

-- Utiliser l'entrepôt créé
USE WAREHOUSE mywarehouse;

-- Créer un *stage* externe pointant vers le bucket S3 des fichiers d'exemple
CREATE OR REPLACE STAGE my_stage
  URL = 's3://snowflake-docs/tutorials/json';
```

**Notes importantes :**
*   `CREATE DATABASE` crée une base de données incluant automatiquement un schéma nommé `'public'`.
*   `USE SCHEMA` définit la base de données et le schéma actifs pour la session utilisateur.
*   `CREATE TABLE` crée une table cible avec une colonne de type `VARIANT` pour stocker les données JSON brutes.
*   `CREATE WAREHOUSE` crée un entrepôt **suspendu initialement**, configuré pour **démarrer automatiquement** (`AUTO_RESUME = TRUE`) lors de l'exécution de commandes SQL nécessitant des ressources de calcul.
*   `CREATE STAGE` crée un ***stage* externe** pointant vers le **bucket S3** contenant les fichiers d'exemple du tutoriel.

### **Copier les données dans la table cible**

Exécutez la commande **`COPY INTO <table>`** pour charger vos données mises en *stage* dans la table cible **`RAW_SOURCE`**.

```sql
COPY INTO raw_source
  FROM @my_stage/server/2.6/2016/07/15/15
  FILE_FORMAT = (TYPE = JSON);
```

La commande copie **toutes les nouvelles données** du **chemin spécifié** sur le *stage* externe vers la table cible `RAW_SOURCE`. Dans cet exemple, le chemin cible les données écrites durant la **15ᵉ heure (15h00) du 15 juillet 2016**. Snowflake vérifie la valeur **ETag S3** de chaque fichier pour s'assurer qu'il n'est copié qu'une seule fois.

---

Exécutez une requête **`SELECT`** pour vérifier que les données ont été copiées avec succès.

```sql
SELECT * FROM raw_source;
```

La requête renvoie le résultat suivant (données JSON brutes) :

```
+-----------------------------------------------------------------------------------+
| SRC                                                                               |
|-----------------------------------------------------------------------------------|
| {                                                                                 |
|   "device_type": "server",                                                        |
|   "events": [                                                                     |
|     {                                                                             |
|       "f": 83,                                                                    |
|       "rv": "15219.64,783.63,48674.48,84679.52,27499.78,2178.83,0.42,74900.19",   |
|       "t": 1437560931139,                                                         |
|       "v": {                                                                      |
|         "ACHZ": 42869,                                                            |
|         "ACV": 709489,                                                            |
|         "DCA": 232,                                                               |
|         "DCV": 62287,                                                             |
|         "ENJR": 2599,                                                             |
|         "ERRS": 205,                                                              |
|         "MXEC": 487,                                                              |
|         "TMPI": 9                                                                 |
|       },                                                                          |
|       "vd": 54,                                                                   |
|       "z": 1437644222811                                                          |
|     },                                                                            |
|     {                                                                             |
|       "f": 1000083,                                                               |
|       "rv": "8070.52,54470.71,85331.27,9.10,70825.85,65191.82,46564.53,29422.22", |
|       "t": 1437036965027,                                                         |
|       "v": {                                                                      |
|         "ACHZ": 6953,                                                             |
|         "ACV": 346795,                                                            |
|         "DCA": 250,                                                               |
|         "DCV": 46066,                                                             |
|         "ENJR": 9033,                                                             |
|         "ERRS": 615,                                                              |
|         "MXEC": 0,                                                                |
|         "TMPI": 112                                                               |
|       },                                                                          |
|       "vd": 626,                                                                  |
|       "z": 1437660796958                                                          |
|     }                                                                             |
|   ],                                                                              |
|   "version": 2.6                                                                  |
| }                                                                                 |
+-----------------------------------------------------------------------------------+
```

Dans cet exemple de données JSON, il y a **deux événements**. Les paires clé-valeur **`device_type`** et **`version`** identifient une **source de données** et une **version** pour les événements provenant d'un appareil spécifique.

### **Interroger les données**

Dans cette section, vous explorez des instructions **`SELECT`** pour interroger les données JSON.

#### **1. Récupérer `device_type`**
```sql
SELECT src:device_type
  FROM raw_source;
```
**Résultat :**
```
+-----------------+
| SRC:DEVICE_TYPE |
|-----------------|
| "server"        |
+-----------------+
```
*   La notation **`src:device_type`** spécifie le nom de la colonne et l'élément JSON à extraire.
*   Snowflake permet de spécifier une **sous-colonne** dans une colonne parente, déduite dynamiquement du schéma intégré aux données JSON.
*   **Note :** Le nom de la colonne est **insensible à la casse**, mais les noms d'éléments JSON sont **sensibles à la casse**.

#### **2. Récupérer la valeur `device_type` sans guillemets**
Pour retirer les guillemets, castez les données vers un type spécifique (ici `string`). Un alias peut être utilisé pour nommer la colonne.
```sql
SELECT src:device_type::string AS device_type
  FROM raw_source;
```
**Résultat :**
```
+-------------+
| DEVICE_TYPE |
|-------------|
| server      |
+-------------+
```

#### **3. Récupérer les clés répétitives `f` imbriquées dans le tableau `events`**
Les données JSON incluent un tableau `events`. Chaque objet `event` dans le tableau contient un champ `f`.  
Pour extraire ces clés imbriquées, utilisez la fonction **`FLATTEN`** qui aplatit les `events` en lignes séparées.
```sql
SELECT
  value:f::number
  FROM
    raw_source
  , LATERAL FLATTEN( INPUT => SRC:events );
```
**Résultat :**
```
+-----------------+
| VALUE:F::NUMBER |
|-----------------|
|              83 |
|         1000083 |
+-----------------+
```
*   **`value`** est l'une des colonnes renvoyées par la fonction `FLATTEN`.
*   L'étape suivante fournira plus de détails sur l'utilisation de `FLATTEN`.

### **Aplatir les données**

**`FLATTEN`** est une **fonction de table** qui produit une **vue latérale** d'une colonne de type **`VARIANT`**, **`OBJECT`** ou **`ARRAY`**. Dans cette étape, vous utilisez cette fonction pour explorer différents niveaux d'aplatissement.

---

#### **1. Aplatir les objets du tableau `events` dans une colonne `VARIANT`**
Vous pouvez aplatir les objets `event` du tableau `events` en lignes séparées avec `FLATTEN`. La sortie inclut une colonne **`VALUE`** qui stocke ces événements individuels.

Utilisez le modificateur **`LATERAL`** pour joindre la sortie de `FLATTEN` avec des informations extérieures à l'objet (ici `device_type` et `version`).

**a) Interroger les données pour chaque événement :**
```sql
SELECT src:device_type::string,
    src:version::String,
    VALUE
FROM
    raw_source,
    LATERAL FLATTEN( INPUT => SRC:events );
```
*Résultat :* Affiche deux lignes, chacune contenant `device_type`, `version` et l'objet `event` complet (colonne `VALUE`) sous forme JSON.

**b) Stocker le résultat dans une table avec `CREATE TABLE AS SELECT` :**
```sql
CREATE OR REPLACE TABLE flattened_source AS
  SELECT
    src:device_type::string AS device_type,
    src:version::string     AS version,
    VALUE                   AS src
  FROM
    raw_source,
    LATERAL FLATTEN( INPUT => SRC:events );
```

**c) Interroger la nouvelle table :**
```sql
SELECT * FROM flattened_source;
```
*Résultat :* Deux lignes avec `DEVICE_TYPE`, `VERSION` et `SRC` (l'objet `event` JSON complet).

---

#### **2. Aplatir les clés des objets dans des colonnes séparées**
Dans l'exemple précédent, la table `flattened_source` conserve la structure des événements dans une colonne `VARIANT`. L'avantage est la **flexibilité en cas de changement de format**.

Cependant, vous pouvez aussi copier les **clés individuelles** de l'objet `event` dans des **colonnes typées séparées**, ce qui facilite l'analyse.

**Exemple de `CREATE TABLE AS SELECT` pour créer une table `events` avec colonnes dédiées :**
```sql
CREATE OR REPLACE TABLE events AS
  SELECT
    src:device_type::string                             AS device_type
  , src:version::string                                 AS version
  , value:f::number                                     AS f
  , value:rv::variant                                   AS rv
  , value:t::number                                     AS t
  , value:v.ACHZ::number                                AS achz
  , value:v.ACV::number                                 AS acv
  , value:v.DCA::number                                 AS dca
  , value:v.DCV::number                                 AS dcv
  , value:v.ENJR::number                                AS enjr
  , value:v.ERRS::number                                AS errs
  , value:v.MXEC::number                                AS mxec
  , value:v.TMPI::number                                AS tmpi
  , value:vd::number                                    AS vd
  , value:z::number                                     AS z
  FROM
    raw_source
  , LATERAL FLATTEN ( INPUT => SRC:events );
```

*   L'instruction **aplatit les données imbriquées** dans `SRC:events`.
*   Chaque valeur est **convertie** (`::`) vers un type de données approprié (`number`, `string`, `variant`).
*   Si le *casting* est omis, la colonne prend le type `VARIANT` par défaut.

**Interroger la table `events` :**
```sql
SELECT * FROM events;
```

**Résultat (deux premières lignes) :**
```
+-------------+---------+---------+----------------------------------------------------------------------+---------------+-------+--------+-----+-------+------+------+------+------+-----+---------------+
| DEVICE_TYPE | VERSION |       F | RV                                                                   |             T |  ACHZ |    ACV | DCA |   DCV | ENJR | ERRS | MXEC | TMPI |  VD |             Z |
|-------------+---------+---------+----------------------------------------------------------------------+---------------+-------+--------+-----+-------+------+------+------+------+-----+---------------|
| server      | 2.6     |      83 | "15219.64,783.63,48674.48,84679.52,27499.78,2178.83,0.42,74900.19"   | 1437560931139 | 42869 | 709489 | 232 | 62287 | 2599 |  205 |  487 |    9 |  54 | 1437644222811 |
| server      | 2.6     | 1000083 | "8070.52,54470.71,85331.27,9.10,70825.85,65191.82,46564.53,29422.22" | 1437036965027 |  6953 | 346795 | 250 | 46066 | 9033 |  615 |    0 |  112 | 626 | 1437660796958 |
+-------------+---------+---------+----------------------------------------------------------------------+---------------+-------+--------+-----+-------+------+------+------+------+-----+---------------+
```

La table `events` est maintenant **structurée et typée**, idéale pour des requêtes analytiques performantes.

### **Mettre à jour les données**

Jusqu'à présent dans ce tutoriel, vous avez :

1.  **Copié** des données d'événements JSON d'un bucket S3 dans la table `RAW_SOURCE` et exploré des requêtes simples.
2.  **Exploré la fonction `FLATTEN`** pour aplatir les données JSON et obtenir une **représentation relationnelle** des données (table `EVENTS` avec colonnes séparées).

Dans un **scénario réel**, de nouveaux événements sont ajoutés en continu au bucket S3. Un script peut copier ces nouvelles données dans `RAW_SOURCE`, mais **comment insérer uniquement les nouveaux événements** dans la table `EVENTS` ?

Il existe plusieurs méthodes pour maintenir la cohérence des données. Cette section en explique deux.

---

#### **Option 1 : Utiliser des colonnes de clé primaire pour la comparaison**

**Principe :** Ajouter une **contrainte de clé primaire** (métadonnée) à `EVENTS` pour garantir l'unicité, puis insérer uniquement les lignes absentes.

**Étapes :**

1.  **Identifier des candidats pour la clé primaire** dans les données JSON (ex: la combinaison `device_type` + `rv`).
    *Note : Snowflake **n'applique pas** la contrainte, elle sert de métadonnée dans le schéma d'information.*

2.  **Ajouter la contrainte** à la table `EVENTS` :
    ```sql
    ALTER TABLE events ADD CONSTRAINT pk_DeviceType PRIMARY KEY (device_type, rv);
    ```

3.  **Insérer un nouvel enregistrement JSON** dans `RAW_SOURCE` (simulation d'un nouvel événement) :
    ```sql
    INSERT INTO raw_source
      SELECT
      PARSE_JSON ('{
        "device_type": "cell_phone",
        "events": [
          {
            "f": 79,
            "rv": "786954.67,492.68,3577.48,40.11,343.00,345.8,0.22,8765.22",
            "t": 5769784730576,
            "v": { ... },
            "vd": 54,
            "z": 1437644222811
          }
        ],
        "version": 3.2
      }');
    ```

4.  **Insérer le nouvel enregistrement dans `EVENTS`** en comparant les valeurs de la clé primaire (évite les doublons) :
    ```sql
    INSERT INTO events
    SELECT
          src:device_type::string
        , src:version::string
        , value:f::number
        , value:rv::variant
        , value:t::number
        , value:v.ACHZ::number
        , value:v.ACV::number
        , value:v.DCA::number
        , value:v.DCV::number
        , value:v.ENJR::number
        , value:v.ERRS::number
        , value:v.MXEC::number
        , value:v.TMPI::number
        , value:vd::number
        , value:z::number
    FROM
          raw_source
        , LATERAL FLATTEN( INPUT => src:events )
    WHERE NOT EXISTS
        (SELECT 'x'
          FROM events
          WHERE events.device_type = src:device_type
            AND events.rv = value:rv);
    ```
    La clause **`WHERE NOT EXISTS`** empêche l'insertion si une ligne avec la même combinaison `device_type`/`rv` existe déjà.

5.  **Vérifier** le contenu de `EVENTS` :
    ```sql
    SELECT * FROM EVENTS;
    ```
    → La nouvelle ligne `cell_phone` apparaît.

---

#### **Option 2 : Utiliser toutes les colonnes pour la comparaison**

**Principe :** Si les données JSON **n'ont pas de champ(s) naturellement unique**, comparer **toutes les colonnes** répétitives entre `RAW_SOURCE` et `EVENTS`.

**Étapes :**

1.  **Insérer un autre nouvel enregistrement** dans `RAW_SOURCE` :
    ```sql
    INSERT INTO raw_source
      SELECT
      PARSE_JSON ('{
        "device_type": "web_browser",
        "events": [
          {
            "f": 79,
            "rv": "122375.99,744.89,386.99,12.45,78.08,43.7,9.22,8765.43",
            "t": 5769784730576,
            "v": { ... },
            "vd": 55,
            "z": 8745598047355
          }
        ],
        "version": 8.7
      }');
    ```

2.  **Insérer dans `EVENTS`** en comparant **toutes les colonnes correspondantes** :
    ```sql
    INSERT INTO events
    SELECT
          src:device_type::string
        , src:version::string
        , value:f::number
        , value:rv::variant
        , value:t::number
        , value:v.ACHZ::number
        , value:v.ACV::number
        , value:v.DCA::number
        , value:v.DCV::number
        , value:v.ENJR::number
        , value:v.ERRS::number
        , value:v.MXEC::number
        , value:v.TMPI::number
        , value:vd::number
        , value:z::number
    FROM
          raw_source
        , LATERAL FLATTEN( INPUT => src:events )
    WHERE NOT EXISTS
        (SELECT 'x'
          FROM events
          WHERE events.device_type = src:device_type
            AND events.version = src:version
            AND events.f = value:f
            AND events.rv = value:rv
            AND events.t = value:t
            AND events.achz = value:v.ACHZ
            AND events.acv = value:v.ACV
            AND events.dca = value:v.DCA
            AND events.dcv = value:v.DCV
            AND events.enjr = value:v.ENJR
            AND events.errs = value:v.ERRS
            AND events.mxec = value:v.MXEC
            AND events.tmpi = value:v.TMPI
            AND events.vd = value:vd
            AND events.z = value:z);
    ```
    La condition `WHERE NOT EXISTS` est plus longue car elle compare **tous les champs** pour détecter les doublons exacts.

3.  **Vérifier** le contenu de `EVENTS` :
    ```sql
    SELECT * FROM EVENTS;
    ```
    → La nouvelle ligne `web_browser` apparaît en plus des trois précédentes.

**Conclusion :**  
La **première méthode** (clé primaire) est **plus efficace** si vos données ont des identifiants naturels uniques.  
La **seconde méthode** (comparaison totale) est plus **robuste** pour éviter tout doublon, mais peut être **moins performante** sur de gros volumes.

### **Félicitations, vous avez terminé ce tutoriel avec succès !**

---

#### **Points clés du tutoriel**

1.  **Partitionnement des données dans S3** : Organiser les données d'événements dans votre bucket S3 avec des **chemins logiques et granulaires** (ex: par date/heure) vous permet de copier **un sous-ensemble précis** des données dans Snowflake avec une **seule commande** `COPY`.

2.  **Notation `colonne:clé` pour les données semi-structurées** : La notation **`src:device_type`** (similaire à `table.colonne` SQL) permet d'interroger efficacement une **sous-colonne** dans une colonne `VARIANT`, dérivée dynamiquement du schéma embarqué dans les données JSON.

3.  **Fonction `FLATTEN`** : Cette fonction est essentielle pour **parser les données JSON** et les transformer en **colonnes relationnelles séparées**, facilitant ainsi l'analyse.

4.  **Mises à jour incrémentielles** : Plusieurs options existent pour **mettre à jour les tables** en comparant avec les nouveaux fichiers stagés, notamment l'utilisation de **`WHERE NOT EXISTS`** avec des clés primaires ou la comparaison de toutes les colonnes.

---

#### **Nettoyage du tutoriel (optionnel)**

Pour **restaurer votre système** à son état avant le tutoriel, exécutez les commandes suivantes :

```sql
-- Supprime la base de données et tous ses objets enfants (tables, schémas)
DROP DATABASE IF EXISTS mydatabase;

-- Supprime l'entrepôt virtuel créé
DROP WAREHOUSE IF EXISTS mywarehouse;
```

**Note :** La suppression de la base de données **supprime automatiquement tous les objets enfants** (tables, schémas, etc.) qu'elle contient.

