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
Pour installer SnowSQL, consultez le guide **[Installing SnowSQL](https://docs.snowflake.com/en/user-guide/snowsql-install-config)** (Installation de SnowSQL).

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

**Vocabulaire clé :**
*   **Stage data files** → Mettre en *stage* (stocker) les fichiers de données
*   **Snowflake stage** → Stage Snowflake
*   **Location in cloud storage** → Emplacement dans le stockage cloud
*   **Load and unload data** → Charger et décharger des données
*   **Internal stages** → Stages internes
*   **Store data files internally within Snowflake** → Stocker des fichiers de données en interne dans Snowflake
*   **External stages** → Stages externes
*   **Store data files externally** → Stocker des fichiers de données en externe
*   **Upload the sample data files** → Télécharger/Téléverser les fichiers de données d'exemple
*   **Internal stage for the emp_basic table** → Stage interne pour la table emp_basic
*   **PUT command** → Commande `PUT`
*   **Upload local data files** → Téléverser des fichiers de données locaux
*   **Table stage** → Stage de table
*   **Full directory path** → Chemin de répertoire complet
*   **File system wildcards** → Caractères génériques du système de fichiers
*   **Staged files** → Fichiers mis en *stage*
*   **Compresses files by default** → Compresse les fichiers par défaut
*   **LIST command** → Commande `LIST`
*   **List the staged files** → Lister les fichiers mis en *stage*
