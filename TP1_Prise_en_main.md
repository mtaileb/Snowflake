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

1.  **Téléchargez l'ensemble de fichiers de données d'exemple.** Faites un clic droit sur le nom du fichier archive, **`getting-started.zip`**, et enregistrez le lien/le fichier sur votre **système de fichiers local**.
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

