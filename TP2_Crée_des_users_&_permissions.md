### **Ce que vous allez apprendre**

Dans ce tutoriel, vous apprendrez à :

1.  **Utiliser un rôle** qui dispose des **privilèges nécessaires** pour créer et utiliser les **objets Snowflake** requis par ce tutoriel.
2.  **Créer un utilisateur**.
3.  **Accorder un rôle à l'utilisateur** et lui **accorder l'accès à un entrepôt virtuel** (*warehouse*).
4.  **Explorer les utilisateurs et les rôles** présents dans votre compte.
5.  **Supprimer l'utilisateur** que vous avez créé.

---

**Étape 1 : Se connecter en utilisant Snowsight**

Pour accéder à Snowsight via l'Internet public, procédez comme suit :

1.  Dans un navigateur web pris en charge, rendez-vous sur **https://app.snowflake.com**.
2.  Saisissez votre **identifiant de compte** ou votre **URL de compte**. Si vous vous êtes déjà connecté à Snowsight auparavant, vous verrez peut-être un nom de compte que vous pouvez sélectionner.
3.  Connectez-vous en utilisant vos **identifiants de compte Snowflake**.

**Étape 2 : Ouvrir la feuille de travail [Modèle]**

Vous pouvez utiliser des **feuilles de travail** pour écrire et exécuter des commandes SQL sur votre base de données. Votre compte d'essai a accès à une **feuille de travail modèle préchargée** pour ce tutoriel. Cette feuille de travail contient les **commandes SQL** que vous allez exécuter pour définir le contexte de rôle, créer un utilisateur et accorder des privilèges de rôle. Comme il s'agit d'un modèle, vous serez invité à saisir vos propres valeurs pour certains paramètres SQL.

Pour plus d'informations sur les feuilles de travail, consultez la rubrique **Prise en main des feuilles de travail**.

**Pour ouvrir la feuille de travail :**

1.  Dans le **menu de navigation**, sélectionnez **Projets » Feuilles de travail**.
2.  Ouvrez la feuille de travail intitulée **[Modèle] Ajout d'un utilisateur et attribution de rôles**.
3.  Votre navigateur devrait ressembler à l'image suivante.

<img width="3052" height="1412" alt="image" src="https://github.com/user-attachments/assets/7c93b198-bfb3-48a4-b6d6-d8ac09ed7eda" />

**Étape 3 : Définir le rôle à utiliser**

Le rôle que vous utilisez détermine les privilèges dont vous disposez. Dans ce tutoriel, utilisez le rôle système **`USERADMIN`** afin de pouvoir créer et gérer les utilisateurs et rôles dans votre compte. Pour plus d'informations, consultez **Vue d'ensemble du contrôle d'accès**.

**Pour définir le rôle à utiliser, procédez comme suit :**

1.  Dans la feuille de travail ouverte, placez votre curseur sur la ligne **`USE ROLE`**.

    ```sql
    USE ROLE USERADMIN;
    ```

2.  Dans le coin supérieur droit de la feuille de travail, sélectionnez **Exécuter**.

    **Note :** Dans ce tutoriel, exécutez les instructions SQL **une par une**. **Ne sélectionnez pas Exécuter tout.**

**Étape 4 : Créer un utilisateur**

Un **utilisateur Snowflake** possède des identifiants de connexion. Lorsqu'un utilisateur se voit attribuer un rôle, il peut effectuer toutes les opérations que ce rôle permet, via les privilèges accordés au rôle. Pour plus d'informations, consultez **Gestion des utilisateurs**.

Dans cette étape du tutoriel, vous allez créer un utilisateur avec un nom, un mot de passe et quelques autres propriétés.

Dans la feuille de travail ouverte :

1.  Placez votre curseur sur la ligne **`CREATE USER`**.
2.  Insérez un **nom d'utilisateur** et d'autres **valeurs de paramètres** de votre choix (un exemple est donné ci-dessous).
3.  Sélectionnez **Exécuter**.

Pour **`MUST_CHANGE_PASSWORD`**, définissez la valeur sur **`true`**, ce qui garantit qu'une réinitialisation du mot de passe sera demandée lors de la première connexion. Pour **`DEFAULT_WAREHOUSE`**, utilisez **`COMPUTE_WH`**.

```sql
CREATE OR REPLACE USER snowman
PASSWORD = 'sn0wf@ll'
LOGIN_NAME = 'snowstorm'
FIRST_NAME = 'Snow'
LAST_NAME = 'Storm'
EMAIL = 'snow.storm@snowflake.com'
MUST_CHANGE_PASSWORD = true
DEFAULT_WAREHOUSE = COMPUTE_WH;
```

Cette commande renvoie le résultat suivant :

```
User SNOWMAN successfully created.
```

Si vous créiez un **utilisateur réel** dans un **compte Snowflake réel**, vous devriez maintenant envoyer les informations suivantes de manière sécurisée à la personne qui aura besoin d'accéder à ce nouveau compte :

*   **URL du compte Snowflake** : le lien du compte Snowflake où l'utilisateur se connectera. Vous pouvez trouver ce lien en haut de votre navigateur (par exemple : `https://app.snowflake.com/myorg/myaccount/`, où `myorg` est l'ID d'organisation Snowflake et `myaccount` est l'ID du compte).
*   **`LOGIN_NAME`**, tel que spécifié dans la commande `CREATE USER`.
*   **`PASSWORD`**, tel que spécifié dans la commande `CREATE USER`.

**Étape 5 : Accorder un rôle système et l'accès à un entrepôt à l'utilisateur**

Maintenant que vous avez créé un utilisateur, vous pouvez utiliser le rôle **`SECURITYADMIN`** pour accorder le rôle **`SYSADMIN`** à cet utilisateur, ainsi que l'autorisation **`USAGE`** sur l'entrepôt **`COMPUTE_WH`**.

Accorder un rôle à un autre rôle crée une **relation parent-enfant** entre les rôles (également appelée **hiérarchie de rôles**). Accorder un rôle à un utilisateur permet à cet utilisateur d'effectuer toutes les opérations autorisées par ce rôle (via les privilèges d'accès accordés au rôle).

Le rôle **`SYSADMIN`** possède les privilèges nécessaires pour créer des entrepôts, bases de données et objets de base de données dans un compte, et pour accorder ces privilèges à d'autres rôles. N'accorde ce rôle qu'aux utilisateurs qui devraient avoir ces privilèges. Pour plus d'informations sur les autres rôles prédéfinis, consultez **Vue d'ensemble du contrôle d'accès**.

**Pour accorder à l'utilisateur l'accès à un rôle et à un entrepôt, procédez comme suit :**

1.  Dans la feuille de travail ouverte, placez votre curseur sur la ligne **`USE ROLE`**, puis sélectionnez **Exécuter**.
    ```sql
    USE ROLE SECURITYADMIN;
    ```

2.  Placez votre curseur sur la ligne **`GRANT ROLE`**, saisissez le **nom de l'utilisateur** que vous avez créé, puis sélectionnez **Exécuter**.
    ```sql
    GRANT ROLE SYSADMIN TO USER snowman;
    ```

3.  Placez votre curseur sur la ligne **`GRANT USAGE`**, puis sélectionnez **Exécuter**.
    ```sql
    GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE SYSADMIN;
    ```

**Étape 6 : Explorer les utilisateurs et rôles dans votre compte**

Vous pouvez maintenant explorer tous les utilisateurs et rôles de votre compte en utilisant le rôle **`ACCOUNTADMIN`**.

**Pour explorer les utilisateurs et rôles, procédez comme suit :**

1.  Dans la feuille de travail ouverte, placez votre curseur sur la ligne **`USE ROLE`**, puis sélectionnez **Exécuter**.
    ```sql
    USE ROLE ACCOUNTADMIN;
    ```

2.  Placez votre curseur sur la ligne **`SHOW USERS`**, puis sélectionnez **Exécuter**.
    ```sql
    SHOW USERS;
    ```
    Votre sortie devrait ressembler à l'image suivante :

    <img width="1740" height="300" alt="image" src="https://github.com/user-attachments/assets/070aaffb-96bf-4005-bfb6-c44954899a07" />


4.  Placez votre curseur sur la ligne **`SHOW ROLES`**, puis sélectionnez **Exécuter**.
    ```sql
    SHOW ROLES;
    ```
    Votre sortie devrait ressembler à l'image suivante :
    
    <img width="1736" height="502" alt="image" src="https://github.com/user-attachments/assets/76088d5c-06a1-40a9-8b55-e12e53e5666c" />

**Étape 7 : Supprimer l'utilisateur et revoir les points clés**

**Félicitations !** Vous avez terminé avec succès ce tutoriel pour les comptes d'essai. Prenez quelques minutes pour revoir les **points clés** abordés. Pour en savoir plus, consultez d'autres rubriques de la **Documentation Snowflake**.

#### **Supprimer l'utilisateur**
En supposant qu'il ne soit plus nécessaire, vous pouvez maintenant **supprimer** l'utilisateur que vous avez créé.

Dans la feuille de travail ouverte :

1.  Placez votre curseur sur la ligne **`DROP USER`**.
2.  Saisissez le **nom de l'utilisateur** que vous avez créé.
3.  Sélectionnez **Exécuter**.

```sql
DROP USER snowman;
```

#### **Revoir les points clés**
En résumé, vous avez utilisé une **feuille de travail préchargée** dans Snowsight pour effectuer les étapes suivantes :

1.  **Définir le rôle** à utiliser.
2.  **Créer un nouvel utilisateur**.
3.  **Accorder à l'utilisateur** des privilèges de rôle et l'**accès à un entrepôt**.
4.  **Explorer les utilisateurs et rôles** du compte.
5.  **Supprimer** l'utilisateur que vous avez créé.

**Voici quelques points clés** à retenir concernant les utilisateurs et les rôles :

*   Vous avez besoin des **permissions requises** pour créer et gérer des objets dans votre compte. Dans ce tutoriel, vous avez utilisé les **rôles système** **`USERADMIN`**, **`SECURITYADMIN`**, **`SYSADMIN`** et **`ACCOUNTADMIN`** à des fins différentes.
*   Le rôle **`ACCOUNTADMIN`** n'est normalement **pas utilisé pour créer des objets**. Nous vous recommandons plutôt de créer une **hiérarchie de rôles** alignée sur les **fonctions métier** de votre organisation. Pour plus d'informations, consultez **Utilisation du rôle ACCOUNTADMIN**.
*   Un **entrepôt** fournit les **ressources de calcul** dont vous avez besoin pour exécuter des **opérations DML**, **charger des données** et **exécuter des requêtes**. Ce tutoriel utilise l'entrepôt **`compute_wh`** inclus avec votre compte d'essai.

   
