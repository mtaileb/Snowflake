Ce tutoriel vous montre comment **charger des données depuis le stockage cloud** vers Snowflake en utilisant **SQL**. Vous utilisez une **feuille de travail modèle** dans Snowsight pour accomplir ces tâches.

**Vous pouvez choisir le fournisseur cloud** que vous souhaitez utiliser :
*   **Amazon S3**
*   **Microsoft Azure**
*   **Google Cloud Storage (GCS)**

La feuille de travail contient des **commandes SQL personnalisées** pour assurer la compatibilité avec chaque type de stockage.

### **Ce que vous allez apprendre**

Dans ce tutoriel, vous apprendrez à :

1.  **Utiliser un rôle** qui dispose des **privilèges nécessaires** pour créer et utiliser les **objets Snowflake** requis.
2.  **Utiliser un entrepôt virtuel** pour accéder aux ressources de calcul.
3.  **Créer une base de données** et un **schéma**.
4.  **Créer une table**.
5.  **Créer une intégration de stockage** pour votre plateforme cloud.
6.  **Créer un *stage*** pour votre intégration de stockage.
7.  **Charger des données** dans la table depuis le *stage*.
8.  **Interroger les données** dans la table.

### **Prérequis**

Ce tutoriel suppose que :

*   Vous disposez d'un **navigateur pris en charge**.
*   Vous avez un **compte d'essai**. Si vous n'avez pas encore de compte d'essai, vous pouvez vous inscrire pour un **essai gratuit**. Vous pouvez choisir n'importe quelle **région cloud Snowflake**.
*   Vous avez un **compte que vous pouvez utiliser** pour le chargement en masse de données depuis l'un des **fournisseurs cloud** suivants :
    *   **AWS S3** – Consultez *Chargement en masse depuis Amazon S3*.
    *   **Microsoft Azure** – Consultez *Chargement en masse depuis Microsoft Azure*.
    *   **Google Cloud Storage** – Consultez *Chargement en masse depuis Google Cloud Storage*.
 
### **Étape 1 : Se connecter en utilisant Snowsight**

Pour accéder à Snowsight via l'Internet public, procédez comme suit :

1.  Dans un navigateur web pris en charge, rendez-vous sur **https://app.snowflake.com**.
2.  Saisissez votre **identifiant de compte** ou votre **URL de compte**. Si vous vous êtes déjà connecté à Snowsight auparavant, vous verrez peut-être un nom de compte que vous pouvez sélectionner.
3.  Connectez-vous en utilisant vos **identifiants de compte Snowflake**.

### **Étape 2 : Ouvrir la feuille de travail "Charger des données depuis le stockage cloud"**

Vous pouvez utiliser des **feuilles de travail** pour écrire et exécuter des commandes SQL sur votre base de données. Votre compte d'essai a accès à une **feuille de travail modèle** pour ce tutoriel. Cette feuille de travail contient les **commandes SQL** que vous allez exécuter pour créer des objets de base de données, charger des données et les interroger. Comme il s'agit d'un **modèle**, vous serez invité à saisir vos propres valeurs pour certains paramètres SQL. Pour plus d'informations sur les feuilles de travail, consultez **Prise en main des feuilles de travail**.

**La feuille de travail pour ce tutoriel n'est pas préchargée dans le compte d'essai.** Pour l'ouvrir, suivez ces étapes :

1.  **Si vous vous connectez à votre compte d'essai Snowsight pour la première fois**, sélectionnez **Commencer** sous l'option **Charger des données dans Snowflake** sur l'écran **Par où voulez-vous commencer ?**.
2.  **Si vous avez quitté cet écran d'accueil**, allez dans l'onglet **Feuilles de travail** et sélectionnez **Continuer** dans la bannière.
3.  **Cliquez n'importe où dans le panneau central** intitulé **Charger des données depuis le stockage cloud**.
4.  La feuille de travail **[Modèle] Charger des données depuis le stockage cloud** s'ouvre, et votre navigateur ressemble à l'image suivante.

<img width="2462" height="1176" alt="image" src="https://github.com/user-attachments/assets/a8c4c490-6a10-48c1-8fce-2822b600a42a" />

### **Étape 3 : Définir le rôle et l'entrepôt à utiliser**

*   Le **rôle** que vous utilisez détermine les **privilèges** dont vous disposez. Dans ce tutoriel, utilisez le rôle système **`ACCOUNTADMIN`** afin de pouvoir visualiser et gérer les objets de votre compte. Pour plus d'informations, consultez **Utilisation du rôle ACCOUNTADMIN**.
*   Un **entrepôt** fournit les **ressources de calcul** dont vous avez besoin pour exécuter des opérations DML, charger des données et exécuter des requêtes. Ces ressources incluent le **CPU, la mémoire et le stockage temporaire**. Votre compte d'essai dispose d'un **entrepôt virtuel (`compute_wh`)** que vous pouvez utiliser pour ce tutoriel. Pour plus d'informations, consultez **Entrepôts virtuels**.

**Pour définir le rôle et l'entrepôt à utiliser, procédez comme suit :**

1.  Dans la feuille de travail ouverte, placez votre curseur sur la ligne **`USE ROLE`**.
    ```sql
    USE ROLE accountadmin;
    ```

2.  Dans le coin supérieur droit de la feuille de travail, sélectionnez **Exécuter**.
    **Note :** Dans ce tutoriel, exécutez les instructions SQL **une par une**. **Ne sélectionnez pas Exécuter tout.**

3.  Placez votre curseur sur la ligne **`USE WAREHOUSE`**, puis sélectionnez **Exécuter**.
    ```sql
    USE WAREHOUSE compute_wh;
    ```

### **Étape 4 : Configurer une table à charger**

*   Une **base de données** est un référentiel pour vos données. Les données sont stockées dans des **tables** que vous pouvez gérer et interroger.
*   Un **schéma** est un regroupement logique d'objets de base de données, tels que des tables et des vues. Par exemple, un schéma peut contenir les objets nécessaires à une application spécifique. Pour plus d'informations, consultez **Vue d'ensemble des bases de données, tables et vues**.

**Pour créer une base de données, un schéma et une table à charger, procédez comme suit :**

1.  **Créer la base de données**  
    Dans la feuille de travail, placez votre curseur sur la ligne **`CREATE OR REPLACE DATABASE`**, saisissez un **nom pour votre base de données** et un **commentaire facultatif**, puis sélectionnez **Exécuter**.  
    **Exemple :**
    ```sql
    CREATE OR REPLACE DATABASE cloud_data_db
      COMMENT = 'Base de données pour le chargement de données cloud';
    ```

2.  **Créer le schéma**  
    Placez votre curseur sur la ligne **`CREATE OR REPLACE SCHEMA`**, saisissez un **nom pour votre schéma** et un **commentaire facultatif**, puis sélectionnez **Exécuter**.  
    **Exemple :**
    ```sql
    CREATE OR REPLACE SCHEMA cloud_data_db.azure_data
      COMMENT = 'Schéma pour les tables chargées depuis Azure';
    ```

3.  **Créer la table**  
    Placez votre curseur sur les lignes **`CREATE OR REPLACE TABLE`**, **complétez la définition de la table**, ajoutez un **commentaire facultatif**, puis sélectionnez **Exécuter**.  
    **Exemple** (table avec six colonnes) :
    ```sql
    CREATE OR REPLACE TABLE cloud_data_db.azure_data.calendar
      (
      full_date DATE
      ,day_name VARCHAR(10)
      ,month_name VARCHAR(10)
      ,day_number VARCHAR(2)
      ,full_year VARCHAR(4)
      ,holiday BOOLEAN
      )
      COMMENT = 'Table à charger depuis un fichier de données calendrier Azure';
    ```

4.  **Vérifier la création de la table**  
    Pour confirmer que la table a été créée avec succès, placez votre curseur sur la ligne **`SELECT`**, puis sélectionnez **Exécuter**.
    ```sql
    SELECT * FROM cloud_data_db.azure_data.calendar;
    ```
    La sortie affiche les **colonnes de la table que vous avez créée**. Actuellement, la table **ne contient aucune ligne**.


### **Étape 5 : Créer une intégration de stockage**

Avant de pouvoir charger des données depuis le stockage cloud, vous devez configurer une **intégration de stockage** spécifique à votre fournisseur cloud. L'exemple suivant concerne spécifiquement le stockage **Microsoft Azure**.

Les **intégrations de stockage** sont des **objets Snowflake de premier niveau nommés** qui évitent de devoir transmettre des identifiants explicites du fournisseur cloud (comme des clés secrètes ou des jetons d'accès). Ces objets stockent une identité Azure IAM (appelée **inscription d'application** ou *app registration*).

**Pour créer une intégration de stockage pour Azure, procédez comme suit :**

1.  **Configurer un conteneur Azure**  
    Utilisez le **portail Azure** pour configurer un conteneur Azure pour le chargement de données. Pour les détails, consultez *Configuration d'un conteneur Azure pour le chargement de données*.

2.  **Créer l'intégration de stockage dans Snowflake**  
    Dans la feuille de travail, placez votre curseur sur les lignes **`CREATE OR REPLACE STORAGE INTEGRATION`**, définissez les paramètres requis, puis sélectionnez **Exécuter**.  
    **Exemple :**
    ```sql
    CREATE OR REPLACE STORAGE INTEGRATION azure_data_integration
      TYPE = EXTERNAL_STAGE
      STORAGE_PROVIDER = 'AZURE'
      AZURE_TENANT_ID = '075f576e-6f9b-4955-8e99-4086736225d9'
      ENABLED = TRUE
      STORAGE_ALLOWED_LOCATIONS = ('azure://tutorial99.blob.core.windows.net/snow-tutorial-container/');
    ```
    *   **`AZURE_TENANT_ID`** : Définissez-le sur l'**ID client Office 365** du compte de stockage contenant les emplacements autorisés. Vous pouvez trouver cet ID dans le portail Azure sous **Microsoft Entra ID > Propriétés > ID client** (Microsoft Entra ID est le nouveau nom d'Azure Active Directory).
    *   **`STORAGE_ALLOWED_LOCATIONS`** : Définissez-le sur le **chemin du conteneur Azure** où votre fichier source est stocké. Utilisez le format montré dans l'exemple, où `tutorial99` est le nom du compte de stockage et `snow-tutorial-container` est le nom du conteneur.

3.  **Décrire l'intégration pour obtenir des informations cruciales**  
    Placez votre curseur sur la ligne **`DESCRIBE INTEGRATION`**, spécifiez le nom de votre intégration, puis sélectionnez **Exécuter**.
    ```sql
    DESCRIBE INTEGRATION azure_data_integration;
    ```
    Cette commande récupère l'**`AZURE_CONSENT_URL`** et l'**`AZURE_MULTI_TENANT_APP_NAME`** pour l'**application cliente** créée automatiquement pour votre compte Snowflake. Vous utiliserez ces valeurs pour configurer les permissions de Snowflake dans le portail Azure.

    La sortie ressemble à ceci :

    <img width="1378" height="618" alt="image" src="https://github.com/user-attachments/assets/6eed5395-5d79-4c0a-a6db-92692d0ce3b5" />

5.  **Vérifier l'intégration**  
    Placez votre curseur sur la ligne **`SHOW INTEGRATIONS`** et sélectionnez **Exécuter**. Cette commande renvoie des informations sur l'intégration créée.
    ```sql
    SHOW INTEGRATIONS;
    ```
    La sortie ressemble à ceci :

    <img width="2118" height="166" alt="image" src="https://github.com/user-attachments/assets/ab7890e4-fbbb-4883-9cff-76123dbec9ca" />

7.  **Configurer les permissions dans Azure**  
    Utilisez le **portail Azure** pour configurer les **permissions de l'application cliente** (créée automatiquement pour votre compte d'essai) afin qu'elle puisse accéder aux conteneurs de stockage. Suivez l'**Étape 2 : Accorder à Snowflake l'accès aux emplacements de stockage** dans le guide *Configuration d'un conteneur Azure pour le chargement de données*.

### **Étape 6 : Créer un *stage***

Un ***stage*** est un **emplacement** qui contient les fichiers de données à charger dans une base de données Snowflake. Ce tutoriel crée un *stage* capable de charger des données depuis un type spécifique de stockage cloud, comme un **conteneur Azure**.

**Pour créer un *stage*, procédez comme suit :**

1.  **Créer le *stage***  
    Dans la feuille de travail, placez votre curseur sur les lignes **`CREATE OR REPLACE STAGE`**, spécifiez :
    *   Un **nom** pour le *stage*.
    *   L'**intégration de stockage** que vous avez créée.
    *   L'**URL du bucket/conteneur**.
    *   Le **format de fichier** correct (CSV ici).  
    Puis sélectionnez **Exécuter**.

    **Exemple :**
    ```sql
    CREATE OR REPLACE STAGE cloud_data_db.azure_data.azuredata_stage
      STORAGE_INTEGRATION = azure_data_integration
      URL = 'azure://tutorial99.blob.core.windows.net/snow-tutorial-container/'
      FILE_FORMAT = (TYPE = CSV);
    ```

2.  **Afficher les informations sur le *stage***  
    Pour visualiser les informations sur le *stage* que vous avez créé, exécutez :
    ```sql
    SHOW STAGES;
    ```
    La sortie de cette commande ressemble à ceci :

    <img width="2370" height="132" alt="image" src="https://github.com/user-attachments/assets/cd883b85-5232-404b-9422-b2ee41efc98e" />

### **Étape 7 : Charger les données depuis le *stage***

Chargez la table depuis le *stage* que vous avez créé en utilisant la commande **`COPY INTO <table>`**. Pour plus d'informations sur le chargement depuis des conteneurs Azure, consultez *Copie de données depuis un stage Azure*.

**Pour charger les données dans la table :**

1.  Placez votre curseur sur les lignes **`COPY INTO`**.
2.  Spécifiez :
    *   Le **nom de la table** cible.
    *   Le ***stage*** que vous avez créé.
    *   Le **nom du fichier** (ou des fichiers) à charger.
3.  Sélectionnez **Exécuter**.

**Exemple :**
```sql
COPY INTO cloud_data_db.azure_data.calendar
  FROM @cloud_data_db.azure_data.azuredata_stage
    FILES = ('calendar.txt');
```

Votre sortie ressemble à l'image suivante.

<img width="1848" height="138" alt="image" src="https://github.com/user-attachments/assets/0baa36de-6dc1-4ea4-9cb5-91055855eea1" />

### **Étape 8 : Interroger la table**

Maintenant que les données sont chargées, vous pouvez exécuter des requêtes sur la table **`calendar`**.

**Pour exécuter une requête dans la feuille de travail ouverte :**

1.  Sélectionnez la ou les lignes de la commande **`SELECT`**.
2.  Sélectionnez **Exécuter**.

**Exemple :** Exécutez la requête suivante :
```sql
SELECT * FROM cloud_data_db.azure_data.calendar;
```

Votre sortie ressemble à l'image suivante.

<img width="911" height="210" alt="image" src="https://github.com/user-attachments/assets/bba3eec8-4a18-4c4d-aa7d-f7b943cef07e" />

### **Étape 9 : Nettoyage, résumé et ressources supplémentaires**

**Félicitations !** Vous avez terminé avec succès ce tutoriel pour les comptes d'essai.

Prenez quelques minutes pour revoir un **bref résumé** et les **points clés** abordés. Vous pouvez également envisager de **nettoyer** en supprimant les objets que vous avez créés. Par exemple, vous pouvez souhaiter supprimer la table que vous avez créée et chargée :

```sql
DROP TABLE calendar;
```

S'ils ne sont **plus nécessaires**, vous pouvez aussi supprimer les **autres objets** créés, comme l'**intégration de stockage**, le ***stage***, la **base de données** et le **schéma**. Pour plus de détails, consultez les **commandes de langage de définition de données (DDL)**.

---

#### **Résumé et points clés**
En résumé, vous avez utilisé une **feuille de travail modèle préchargée** dans Snowsight pour accomplir les étapes suivantes :

1.  **Définir le rôle et l'entrepôt** à utiliser.
2.  **Créer une base de données, un schéma et une table**.
3.  **Créer une intégration de stockage** et **configurer les permissions** sur le stockage cloud.
4.  **Créer un *stage*** et **charger les données** depuis le *stage* dans la table.
5.  **Interroger les données**.

**Voici quelques points clés** à retenir sur le chargement et l'interrogation des données :

*   Vous avez besoin des **permissions requises** pour créer et gérer des objets dans votre compte. Dans ce tutoriel, vous avez utilisé le rôle système **`ACCOUNTADMIN`** pour ces privilèges.
*   Ce rôle n'est normalement **pas utilisé pour créer des objets**. Nous vous recommandons plutôt de créer une **hiérarchie de rôles** alignée sur les **fonctions métier** de votre organisation. Pour plus d'informations, consultez **Utilisation du rôle ACCOUNTADMIN**.
*   Vous avez besoin d'un **entrepôt** pour les **ressources nécessaires** à la création et à la gestion des objets, ainsi qu'à l'exécution des commandes SQL. Ce tutoriel utilise l'entrepôt **`compute_wh`** inclus avec votre compte d'essai.
*   Vous avez créé une **base de données** pour stocker les données et un **schéma** pour regrouper logiquement les objets de la base de données.
*   Vous avez créé une **intégration de stockage** et un ***stage*** pour charger des données depuis un **fichier CSV** stocké dans un **conteneur Azure**.
*   Après le chargement des données dans votre base de données, vous les avez **interrogées** à l'aide d'une instruction **`SELECT`**.





