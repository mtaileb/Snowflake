Dans ce tutoriel, vous apprendrez à :

*   **Créer des objets Snowflake — Vous créerez une base de données et une table pour stocker des données.**

*   **Installer SnowSQL — Vous installerez et utiliserez SnowSQL, l'outil de requête en ligne de commande de Snowflake. Les utilisateurs de Visual Studio Code peuvent envisager d'utiliser l'extension Snowflake pour Visual Studio Code au lieu de SnowSQL.**

*   **Charger des fichiers de données CSV — Vous utiliserez différents mécanismes pour charger des données dans des tables à partir de fichiers CSV.**

*   **Écrire et exécuter des requêtes d'exemple — Vous écrirez et exécuterez différents types de requêtes sur les données nouvellement chargées.**



### **Prérequis**

Ce tutoriel nécessite une **base de données**, une **table** et un **entrepôt virtuel** pour charger et interroger des données. La création de ces objets Snowflake requiert un **utilisateur Snowflake disposant d'un rôle** ayant les **privilèges de contrôle d'accès** nécessaires. De plus, **SnowSQL est requis** pour exécuter les instructions SQL du tutoriel. Enfin, le tutoriel nécessite des **fichiers CSV contenant des données d'exemple** à charger.

Vous pouvez réaliser ce tutoriel en utilisant un **entrepôt, une base de données et une table Snowflake existants**, ainsi que vos propres **fichiers de données locaux**, mais nous vous recommandons d'utiliser les **objets Snowflake** et l'**ensemble de données fourni**.

Pour configurer Snowflake pour ce tutoriel, effectuez les étapes suivantes **avant de continuer** :

---

#### **1. Créer un utilisateur**
Pour créer la base de données, la table et l'entrepôt virtuel, vous devez être **connecté en tant qu'utilisateur Snowflake** avec un rôle qui vous **accorde les privilèges** pour créer ces objets.

*   **Si vous utilisez un compte d'essai de 30 jours,** vous pouvez vous connecter avec **l'utilisateur créé pour le compte**. Cet utilisateur dispose du rôle avec les privilèges nécessaires pour créer les objets.
*   **Si vous n'avez pas d'utilisateur Snowflake, vous ne pouvez pas réaliser ce tutoriel.** Si vous n'avez pas de rôle vous permettant de créer un utilisateur, **demandez à une personne qui en a la capacité** d'effectuer cette étape pour vous. Les utilisateurs ayant le rôle **`ACCOUNTADMIN`** ou **`SECURITYADMIN`** peuvent créer des utilisateurs.

---

#### **2. Installer SnowSQL**
Pour installer SnowSQL, consultez le guide **[Installing SnowSQL](https://www.snowflake.com/en/developers/downloads/snowsql/)** (Installation de SnowSQL).

---

#### **3. Télécharger les fichiers de données d'exemple**
Pour ce tutoriel, vous téléchargez des **fichiers de données d'exemple d'employés au format CSV** fournis par Snowflake.

**Pour télécharger et dézipper les fichiers de données d'exemple :**

1.  **Téléchargez l'ensemble de fichiers de données d'exemple.** Faites un clic droit sur **https://docs.snowflake.com/en/_downloads/34f4a66f56d00340f8f7a92acaccd977/getting-started.zip** et enregistrez le lien/le fichier sur votre **système de fichiers local**.
2.  **Dézippez les fichiers d'exemple.** Le tutoriel suppose que vous avez extrait les fichiers dans l'un des répertoires suivants :
    *   **Linux/macOS :** `/tmp`
    *   **Windows :** `C:\temp`

**Contenu des fichiers :**
*   Chaque fichier contient **cinq enregistrements de données**.
*   Les données utilisent le caractère **virgule (`,`)** comme **séparateur de champs**.
*   Voici un exemple d'enregistrement :
    ```
    Althea,Featherstone,afeatherstona@sf_tuts.com,"8172 Browning Street, Apt B",Calatrava,7/12/2017
    ```

**Important :** Il **n'y a aucun espace** avant ou après les virgules séparant les champs dans chaque enregistrement. C'est le **format par défaut** que Snowflake attend lors du chargement de données CSV.

**Nota:** dans **Linux**, installer le RPM avec le gestionnaire d'installation. Ensuite aller dans le répertoire d'installation usr/<utilisateur>/bin et ouvrir le bash, et lancer:
**./snowsql -a <account_identifier> -u <user_name>**
Puis taper votre mot-de-passe.

---

### **Créer des objets Snowflake**

Lors de cette étape, vous allez créer les **objets Snowflake suivants** :

1.  **Une base de données** (`sf_tuts`) et **une table** (`emp_basic`). Vous chargerez des données d'exemple dans cette table.
2.  **Un entrepôt virtuel** (`sf_tuts_wh`). Cet entrepôt fournit les **ressources de calcul** nécessaires pour charger des données dans la table et l'interroger. Pour ce tutoriel, vous créerez un entrepôt de taille **X-Small**.

À la fin de ce tutoriel, vous **supprimerez ces objets**.

---

#### **Créer une base de données**
Créez la base de données `sf_tuts` à l'aide de la commande **`CREATE DATABASE`** :

```sql
CREATE OR REPLACE DATABASE sf_tuts;
```

Dans ce tutoriel, vous utiliserez le **schéma par défaut (`public`)** disponible pour chaque base de données, plutôt que d'en créer un nouveau.

**Note :** La base de données et le schéma que vous venez de créer sont maintenant **actifs pour votre session courante**, comme indiqué dans l'invite de commande SnowSQL. Vous pouvez également utiliser les **fonctions de contexte** pour obtenir cette information.

```sql
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA();
```

Voici un exemple de résultat :

```
+--------------------+------------------+
| CURRENT_DATABASE() | CURRENT_SCHEMA() |
|--------------------+------------------|
| SF_TUTS            | PUBLIC           |
+--------------------+------------------+
```

---

#### **Créer une table**
Créez une table nommée `emp_basic` dans `sf_tuts.public` à l'aide de la commande **`CREATE TABLE`** :

```sql
CREATE OR REPLACE TABLE emp_basic (
   first_name STRING,
   last_name STRING,
   email STRING,
   streetaddress STRING,
   city STRING,
   start_date DATE
);
```

**Note :** Le **nombre de colonnes** dans la table, leur **position** et leurs **types de données** correspondent aux **champs dans les fichiers de données CSV d'exemple** que vous mettrez en *stage* à l'étape suivante de ce tutoriel.

---

#### **Créer un entrepôt virtuel**
Créez un entrepôt **X-Small** nommé `sf_tuts_wh` à l'aide de la commande **`CREATE WAREHOUSE`** :

```sql
CREATE OR REPLACE WAREHOUSE sf_tuts_wh WITH
   WAREHOUSE_SIZE='X-SMALL'
   AUTO_SUSPEND = 180
   AUTO_RESUME = TRUE
   INITIALLY_SUSPENDED=TRUE;
```

L'entrepôt `sf_tuts_wh` est initialement **suspendu**, mais l'instruction DML définit également `AUTO_RESUME = TRUE`. Le paramètre **`AUTO_RESUME`** permet à un entrepôt de **démarrer automatiquement** lorsque des instructions SQL nécessitant des ressources de calcul sont exécutées.

Après avoir créé l'entrepôt, celui-ci est maintenant **actif pour votre session courante**. Cette information s'affiche dans votre invite de commande SnowSQL. Vous pouvez également récupérer le nom de l'entrepôt en utilisant la fonction de contexte suivante :

```sql
SELECT CURRENT_WAREHOUSE();
```

Voici un exemple de résultat :

```
+---------------------+
| CURRENT_WAREHOUSE() |
|---------------------|
| SF_TUTS_WH          |
+---------------------+
```

### **Mettre en *stage* (stocker) les fichiers de données**

Un **stage Snowflake** est un **emplacement dans le stockage cloud** que vous utilisez pour **charger et décharger des données** depuis ou vers une table. Snowflake prend en charge les **types de stages suivants** :

*   **Stages internes** — Utilisés pour stocker des fichiers de données **en interne dans Snowflake**. Chaque utilisateur et chaque table dans Snowflake obtient par défaut un stage interne pour y mettre des fichiers de données.
*   **Stages externes** — Utilisés pour stocker des fichiers de données **en externe** dans **Amazon S3, Google Cloud Storage ou Microsoft Azure**. Si vos données sont déjà stockées dans ces services de stockage cloud, vous pouvez utiliser un stage externe pour les charger dans des tables Snowflake.

Dans ce tutoriel, nous **téléchargeons les fichiers de données d'exemple** (téléchargés dans les *Prérequis*) vers le **stage interne** de la table `emp_basic` que vous avez créée précédemment. Vous utilisez la commande **`PUT`** pour téléverser les fichiers de données d'exemple vers ce *stage*.

---

#### **Mise en *stage* des fichiers de données d'exemple**

Exécutez la commande **`PUT`** dans SnowSQL pour **téléverser les fichiers de données locaux** vers le *stage* de table fourni pour la table `emp_basic` que vous avez créée.

```sql
PUT file://<chemin-du-fichier>[/\]employees0*.csv @sf_tuts.public.%emp_basic;
```

**Exemples :**

*   **Linux ou macOS**
    ```sql
    PUT file:///tmp/employees0*.csv @sf_tuts.public.%emp_basic;
    ```
*   **Windows**
    ```sql
    PUT file://C:\temp\employees0*.csv @sf_tuts.public.%emp_basic;
    ```

**Examinons la commande de plus près :**

*   **`file://<chemin-du-fichier>[/]employees0*.csv`** spécifie le **chemin de répertoire complet** et les **noms des fichiers** sur votre machine locale à mettre en *stage*. Notez que les **caractères génériques** (*wildcards*) du système de fichiers sont autorisés, et si plusieurs fichiers correspondent au motif, ils sont tous affichés.
*   **`@<namespace>.%<nom_table>`** indique d'utiliser le *stage* pour la **table spécifiée**, dans ce cas la table `emp_basic`.

La commande renvoie le **résultat suivant**, montrant les fichiers mis en *stage* :

```
+-----------------+--------------------+-------------+-------------+--------------------+--------------------+----------+---------+
| source          | target             | source_size | target_size | source_compression | target_compression | status   | message |
|-----------------+--------------------+-------------+-------------+--------------------+--------------------+----------+---------|
| employees01.csv | employees01.csv.gz |         360 |         287 | NONE               | GZIP               | UPLOADED |         |
| employees02.csv | employees02.csv.gz |         355 |         274 | NONE               | GZIP               | UPLOADED |         |
| employees03.csv | employees03.csv.gz |         397 |         295 | NONE               | GZIP               | UPLOADED |         |
| employees04.csv | employees04.csv.gz |         366 |         288 | NONE               | GZIP               | UPLOADED |         |
| employees05.csv | employees05.csv.gz |         394 |         299 | NONE               | GZIP               | UPLOADED |         |
+-----------------+--------------------+-------------+-------------+--------------------+--------------------+----------+---------+
```

La commande **`PUT`** **compresse les fichiers par défaut** en utilisant **gzip**, comme indiqué dans la colonne **`TARGET_COMPRESSION`**.

---

#### **Lister les fichiers mis en *stage* (Optionnel)**

Vous pouvez **lister les fichiers mis en *stage*** en utilisant la commande **`LIST`**.

```sql
LIST @sf_tuts.public.%emp_basic;
```

Voici un **exemple de résultat** :

```
+--------------------+------+----------------------------------+------------------------------+
| name               | size | md5                              | last_modified                |
|--------------------+------+----------------------------------+------------------------------|
| employees01.csv.gz |  288 | a851f2cc56138b0cd16cb603a97e74b1 | Tue, 9 Jan 2018 15:31:44 GMT |
| employees02.csv.gz |  288 | 125f5645ea500b0fde0cdd5f54029db9 | Tue, 9 Jan 2018 15:31:44 GMT |
| employees03.csv.gz |  304 | eafee33d3e62f079a054260503ddb921 | Tue, 9 Jan 2018 15:31:45 GMT |
| employees04.csv.gz |  304 | 9984ab077684fbcec93ae37479fa2f4d | Tue, 9 Jan 2018 15:31:44 GMT |
| employees05.csv.gz |  304 | 8ad4dc63a095332e158786cb6e8532d0 | Tue, 9 Jan 2018 15:31:44 GMT |
+--------------------+------+----------------------------------+------------------------------+
```

---

### **Copier les données dans les tables cibles**

Pour **charger vos données mises en *stage* dans la table cible**, exécutez **`COPY INTO <table>`**.

La commande **`COPY INTO <table>`** utilise l'**entrepôt virtuel** que vous avez créé dans *Créer des objets Snowflake* pour copier les fichiers.

```sql
COPY INTO emp_basic
  FROM @%emp_basic
  FILE_FORMAT = (type = csv field_optionally_enclosed_by='"')
  PATTERN = '.*employees0[1-5].csv.gz'
  ON_ERROR = 'skip_file';
```

Où :

*   La clause **`FROM`** spécifie l'**emplacement contenant les fichiers de données** (le *stage* interne pour la table).
*   La clause **`FILE_FORMAT`** spécifie le **type de fichier comme CSV**, et indique que le **caractère double-guillemet (`"`)** est utilisé pour **encadrer les chaînes de caractères**. Snowflake prend en charge **divers types de fichiers et options**. Ceux-ci sont décrits dans **`CREATE FILE FORMAT`**.
*   La clause **`PATTERN`** spécifie que la commande doit **charger les données depuis les noms de fichiers correspondant** à cette **expression régulière** (`.*employees0[1-5].csv.gz`).
*   La clause **`ON_ERROR`** spécifie **l'action à effectuer** lorsque la commande `COPY` rencontre des erreurs dans les fichiers. **Par défaut**, la commande **arrête le chargement** des données dès la première erreur rencontrée. Cet exemple **ignore tout fichier contenant une erreur** et passe au chargement du fichier suivant. (Aucun des fichiers de ce tutoriel ne contient d'erreur ; cela est inclus à titre d'illustration.)

La commande `COPY` propose également une **option pour valider les fichiers avant leur chargement**. Pour plus d'informations sur les **vérifications d'erreur supplémentaires** et les instructions de validation, consultez la rubrique **`COPY INTO <table>`** et les autres tutoriels sur le chargement de données.

La commande `COPY` renvoie un **résultat** montrant la **liste des fichiers copiés** et les informations associées :

```
+--------------------+--------+-------------+-------------+-------------+-------------+-------------+------------------+-----------------------+-------------------------+
| file               | status | rows_parsed | rows_loaded | error_limit | errors_seen | first_error | first_error_line | first_error_character | first_error_column_name |
|--------------------+--------+-------------+-------------+-------------+-------------+-------------+------------------+-----------------------+-------------------------|
| employees02.csv.gz | LOADED |           5 |           5 |           1 |           0 | NULL        |             NULL |                  NULL | NULL                    |
| employees04.csv.gz | LOADED |           5 |           5 |           1 |           0 | NULL        |             NULL |                  NULL | NULL                    |
| employees05.csv.gz | LOADED |           5 |           5 |           1 |           0 | NULL        |             NULL |                  NULL | NULL                    |
| employees03.csv.gz | LOADED |           5 |           5 |           1 |           0 | NULL        |             NULL |                  NULL | NULL                    |
| employees01.csv.gz | LOADED |           5 |           5 |           1 |           0 | NULL        |             NULL |                  NULL | NULL                    |
+--------------------+--------+-------------+-------------+-------------+-------------+-------------+------------------+-----------------------+-------------------------+
```

---

### **Interroger les données chargées**

Vous pouvez **interroger les données chargées** dans la table `emp_basic` en utilisant le **SQL standard** et **toutes les fonctions et opérateurs pris en charge**.

Vous pouvez également **manipuler les données**, comme mettre à jour les données chargées ou insérer plus de données, en utilisant des **commandes DML standard**.

---

#### **Récupérer toutes les données**
Retourne **toutes les lignes et colonnes** de la table :

```sql
SELECT * FROM emp_basic;
```

Voici un **résultat partiel** :

```
+------------+--------------+---------------------------+-----------------------------+--------------------+------------+
| FIRST_NAME | LAST_NAME    | EMAIL                     | STREETADDRESS               | CITY               | START_DATE |
|------------+--------------+---------------------------+-----------------------------+--------------------+------------|
| Arlene     | Davidovits   | adavidovitsk@sf_tuts.com  | 7571 New Castle Circle      | Meniko             | 2017-05-03 |
| Violette   | Shermore     | vshermorel@sf_tuts.com    | 899 Merchant Center         | Troitsk            | 2017-01-19 |
| Ron        | Mattys       | rmattysm@sf_tuts.com      | 423 Lien Pass               | Bayaguana          | 2017-11-15 |
 ...
 ...
 ...
| Carson     | Bedder       | cbedderh@sf_tuts.co.au    | 71 Clyde Gallagher Place    | Leninskoye         | 2017-03-29 |
| Dana       | Avory        | davoryi@sf_tuts.com       | 2 Holy Cross Pass           | Wenlin             | 2017-05-11 |
| Ronny      | Talmadge     | rtalmadgej@sf_tuts.co.uk  | 588 Chinook Street          | Yawata             | 2017-06-02 |
+------------+--------------+---------------------------+-----------------------------+--------------------+------------+
```

---

#### **Insérer des lignes de données supplémentaires**
En plus de charger des données à partir de fichiers mis en *stage* dans une table, vous pouvez **insérer des lignes directement** dans une table en utilisant la **commande DML `INSERT`**.

Par exemple, pour insérer **deux lignes supplémentaires** dans la table :

```sql
INSERT INTO emp_basic VALUES
   ('Clementine','Adamou','cadamou@sf_tuts.com','10510 Sachs Road','Klenak','2017-9-22'),
   ('Marlowe','De Anesy','madamouc@sf_tuts.co.uk','36768 Northfield Plaza','Fangshan','2017-1-26');
```

---

#### **Interroger les lignes en fonction de l'adresse e-mail**
Retourne une **liste d'adresses e-mail** avec des **domaines de premier niveau du Royaume-Uni** en utilisant la fonction **`[ NOT ] LIKE`** :

```sql
SELECT email FROM emp_basic WHERE email LIKE '%.uk';
```

Voici un **exemple de résultat** :

```
+--------------------------+
| EMAIL                    |
|--------------------------|
| gbassfordo@sf_tuts.co.uk |
| rtalmadgej@sf_tuts.co.uk |
| madamouc@sf_tuts.co.uk   |
+--------------------------+
```

---

#### **Interroger les lignes en fonction de la date d'embauche**
Par exemple, pour **calculer quand certains avantages sociaux des employés pourraient commencer**, ajoutez **90 jours** aux dates d'embauche des employés en utilisant la fonction **`DATEADD`**. Filtrez la liste par les employés dont la date d'embauche est **antérieure au 1ᵉʳ janvier 2017** :

```sql
SELECT first_name, last_name, DATEADD('day', 90, start_date)
FROM emp_basic
WHERE start_date <= '2017-01-01';
```

Voici un **exemple de résultat** :

```
+------------+-----------+------------------------------+
| FIRST_NAME | LAST_NAME | DATEADD('DAY',90,START_DATE) |
|------------+-----------+------------------------------|
| Granger    | Bassford  | 2017-03-30                   |
| Catherin   | Devereu   | 2017-03-17                   |
| Cesar      | Hovie     | 2017-03-21                   |
| Wallis     | Sizey     | 2017-03-30                   |
+------------+-----------+------------------------------+
```

---

### **Récapitulatif, nettoyage et ressources supplémentaires**

**Félicitations !** Vous avez **terminé avec succès ce tutoriel d'introduction**.

Prenez quelques minutes pour revoir un **bref résumé** et les **points clés** abordés dans ce tutoriel. Vous pouvez également envisager de **nettoyer** en supprimant les objets que vous avez créés. Pour en savoir plus, consultez d'autres rubriques de la **Documentation Snowflake**.

---

#### **Résumé et points clés**
En résumé, le **chargement de données** s'effectue en **deux étapes** :

1.  **Mettre en *stage* les fichiers de données** à charger. Les fichiers peuvent être mis en *stage* en **interne** (dans Snowflake) ou dans un **emplacement externe**. Dans ce tutoriel, vous avez mis les fichiers en *stage* en interne.
2.  **Copier les données** des fichiers mis en *stage* vers une **table cible existante**. Un **entrepôt en cours d'exécution** est requis pour cette étape.

**Rappelez-vous ces points clés** concernant le chargement de fichiers **CSV** :

*   Un fichier CSV est constitué d'**un ou plusieurs enregistrements**, contenant **un ou plusieurs champs** chacun, et comporte parfois une **ligne d'en-tête**.
*   Les **enregistrements** et les **champs** dans chaque fichier sont séparés par des **délimiteurs**. Les délimiteurs **par défaut** sont :
    *   **Pour les enregistrements** : caractères de **nouvelle ligne**.
    *   **Pour les champs** : **virgules**.
*   En d'autres termes, Snowflake s'attend à ce que **chaque enregistrement** dans un fichier CSV soit **séparé par un saut de ligne** et que les **champs** (c'est-à-dire les valeurs individuelles) dans chaque enregistrement soient **séparés par des virgules**. Si d'autres caractères sont utilisés comme délimiteurs, vous devez le **spécifier explicitement** dans le format de fichier lors du chargement.
*   Il existe une **corrélation directe** entre les **champs dans les fichiers** et les **colonnes dans la table** que vous allez charger, en termes de :
    *   **Nombre** de champs (dans le fichier) et de colonnes (dans la table cible).
    *   **Position** des champs et des colonnes dans leur fichier/table respectif.
    *   **Types de données** (chaîne, nombre, date, etc.) des champs et colonnes.
*   **Les enregistrements ne seront PAS chargés** si les nombres, positions et types de données **ne correspondent pas**.
*   **Note :** Snowflake prend en charge le chargement de fichiers où les champs ne correspondent pas exactement aux colonnes de la table cible ; cependant, il s'agit d'un sujet de chargement de données plus avancé (traité dans *Transforming data during a load*).

---

#### **Nettoyage du tutoriel (Optionnel)**
Si les objets que vous avez créés dans ce tutoriel ne sont **plus nécessaires**, vous pouvez les **supprimer** du système avec des instructions **`DROP <object>`**.

```sql
DROP DATABASE IF EXISTS sf_tuts;

DROP WAREHOUSE IF EXISTS sf_tuts_wh;
```

---

#### **Quitter la connexion**
Pour **quitter une connexion**, utilisez la commande **`!exit`** pour SnowSQL (ou son alias **`!disconnect`**).

`!exit` ferme la **connexion courante** et ** SnowSQL** s'il s'agit de la dernière connexion.

---
