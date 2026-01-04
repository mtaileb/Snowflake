Ce tutoriel utilise une **marque fictive de camion-restaurant nommée Tasty Bytes** pour vous montrer comment **charger et interroger des données dans Snowflake en utilisant Snowpark Python**. Vous utilisez une **feuille de travail Python préchargée** dans Snowsight pour accomplir ces tâches.

L'illustration suivante fournit une **vue d'ensemble de Tasty Bytes**.

<img width="1140" height="559" alt="image" src="https://github.com/user-attachments/assets/3727954b-1ab4-4fc9-97b4-460061e4c5e6" />

### **Ce que vous allez apprendre**

Dans ce tutoriel, vous apprendrez à accomplir les tâches suivantes en utilisant l'**API Snowpark Python** :

1.  **Créer une base de données et un schéma**.
2.  **Créer un *stage*** qui contient des données dans un **bucket Amazon S3**.
3.  **Créer un DataFrame** pour spécifier le *stage* qui est la **source des données**.
4.  **Créer une table** qui contient des données provenant de **fichiers situés sur un *stage***.
5.  **Configurer un DataFrame** pour **interroger la nouvelle table** et **filtrer les données**.

### **Prérequis**

Ce tutoriel suppose que :

*   Vous disposez d'un **navigur pris en charge**.
*   Vous avez un **compte d'essai**. Si vous n'avez pas encore de compte d'essai, vous pouvez vous inscrire pour un **essai gratuit**. Vous pouvez choisir n'importe quelle **région cloud Snowflake**.
*   Votre utilisateur est l'**administrateur du compte** et s'est vu attribuer le **rôle système `ACCOUNTADMIN`**. Pour plus d'informations, consultez **Utilisation du rôle ACCOUNTADMIN**.

### **Étape 1 : Se connecter en utilisant Snowsight**

Pour accéder à Snowsight via l'Internet public, procédez comme suit :

1.  Dans un navigateur web pris en charge, rendez-vous sur **https://app.snowflake.com**.
2.  Saisissez votre **identifiant de compte** ou votre **URL de compte**. Si vous vous êtes déjà connecté à Snowsight auparavant, vous verrez peut-être un nom de compte que vous pouvez sélectionner.
3.  Connectez-vous en utilisant vos **identifiants de compte Snowflake**.

### **Étape 2 : Ouvrir la feuille de travail Python**

Vous pouvez utiliser des **feuilles de travail Python** pour écrire et exécuter du code Python. Votre compte d'essai a accès à une **feuille de travail Python préchargée** pour ce tutoriel. Cette feuille de travail contient le **code Python** que vous allez exécuter pour créer une base de données, y charger des données et interroger ces données. Pour plus d'informations sur les feuilles de travail Python, consultez **Écriture de code Snowpark dans les feuilles de travail Python**.

**Pour ouvrir la feuille de travail Python préchargée du tutoriel :**

1.  Dans le **menu de navigation**, sélectionnez **Projets » Feuilles de travail**.
2.  Ouvrez la feuille de travail intitulée **[Tutoriel] Utilisation de Python pour charger et interroger des exemples de données**.
3.  Votre feuille de travail devrait ressembler à l'image suivante :

<img width="1870" height="802" alt="image" src="https://github.com/user-attachments/assets/9511b6a4-514b-4d93-9ca1-256389c05461" />

Cette feuille de travail Python préchargée utilise **automatiquement le rôle système `ACCOUNTADMIN`**, vous permettant ainsi de visualiser et gérer les objets de votre compte. Pour plus d'informations, consultez **Utilisation du rôle ACCOUNTADMIN**.

La feuille de travail utilise également l'**entrepôt virtuel `COMPUTE_WH`**. Un entrepôt fournit les **ressources nécessaires** pour créer et gérer des objets et exécuter des commandes SQL. Ces ressources incluent le **CPU, la mémoire et le stockage temporaire**. Pour plus d'informations, consultez **Entrepôts virtuels**.

### **Étape 3 : Apprendre à utiliser les feuilles de travail Python**

Les **feuilles de travail Python** vous permettent d'utiliser **Snowpark Python** dans Snowsight pour exécuter des instructions SQL. Cette étape du tutoriel décrit le code de chaque section de la feuille de travail Python.

**Important :** Lorsque vous utilisez une feuille de travail Python, vous **ne pouvez pas exécuter des blocs de code individuellement**. Vous devez exécuter **la feuille de travail entière**. Avant de sélectionner **Exécuter**, passez en revue les étapes suivantes pour mieux comprendre le code Python.

#### **1. Importations**
```python
import snowflake.snowpark as snowpark
from snowflake.snowpark.functions import col
from snowflake.snowpark.types import StructField, StructType, IntegerType, StringType, VariantType
```
Ce tutoriel importe le package **`snowpark`** ainsi que des classes et fonctions sélectionnées pour les rendre disponibles à votre code.

#### **2. Fonction principale (Handler)**
```python
def main(session: snowpark.Session):
```
Cette ligne définit la fonction handler **`main`** par défaut. Cette fonction contient le code que vous allez exécuter. Elle reçoit un objet **`Session`** que vous pouvez utiliser pour exécuter des instructions SQL dans Snowflake.

#### **3. Création de la base de données**
```python
session.sql('CREATE OR REPLACE DATABASE tasty_bytes_sample_data;').collect()
```
*   Crée une base de données nommée **`tasty_bytes_sample_data`**.
*   Utilise la méthode **`sql`** pour créer un DataFrame représentant les résultats de l'instruction SQL.
*   La méthode **`collect`** exécute l'instruction SQL représentée par le DataFrame.

#### **4. Création du schéma**
```python
session.sql('CREATE OR REPLACE SCHEMA tasty_bytes_sample_data.raw_pos;').collect()
```
Crée un schéma nommé **`raw_pos`** dans la base de données `tasty_bytes_sample_data`.

#### **5. Création du *stage* (emplacement de données)**
```python
session.sql('CREATE OR REPLACE STAGE tasty_bytes_sample_data.public.blob_stage url = "s3://sfquickstarts/tastybytes/" file_format = (type = csv);').collect()
```
*   Crée un *stage* nommé **`blob_stage`**.
*   Ce *stage* charge des données depuis un **bucket Amazon S3** existant contenant un fichier CSV.
*   Le format de fichier est défini comme CSV.

#### **6. Définition du schéma des données (Structure)**
```python
menu_schema = StructType([StructField("menu_id",IntegerType()),\
                       StructField("menu_type_id",IntegerType()),\
                       StructField("menu_type",StringType()),\
                       StructField("truck_brand_name",StringType()),\
                       StructField("menu_item_id",IntegerType()),\
                       StructField("menu_item_name",StringType()),\
                       StructField("item_category",StringType()),\
                       StructField("item_subcategory",StringType()),\
                       StructField("cost_of_goods_usd",IntegerType()),\
                       StructField("sale_price_usd",IntegerType()),\
                       StructField("menu_item_health_metrics_obj",VariantType())])
```
Crée un objet **`StructType`** nommé **`menu_schema`** qui décrit les champs (colonnes) du fichier CSV dans le *stage*.

#### **7. Création d'un DataFrame à partir du fichier dans le *stage***
```python
df_blob_stage_read = session.read.schema(menu_schema).csv('@tasty_bytes_sample_data.public.blob_stage/raw_pos/menu/')
```
Crée le DataFrame **`df_blob_stage_read`**, configuré pour lire les données du fichier CSV situé dans le *stage*, en utilisant le schéma `menu_schema`.

#### **8. Sauvegarde du DataFrame en tant que table**
```python
df_blob_stage_read.write.mode("overwrite").save_as_table("tasty_bytes_sample_data.raw_pos.menu")
```
Utilise la méthode **`save_as_table`** pour créer la table **`menu`** et charger les données du *stage* dedans.

#### **9. Création d'un DataFrame filtré à partir de la table**
```python
df_menu_freezing_point = session.table("tasty_bytes_sample_data.raw_pos.menu").filter(col("truck_brand_name") == 'Freezing Point')
```
*   Crée le DataFrame **`df_menu_freezing_point`** configuré pour interroger la table `menu`.
*   La méthode **`filter`** prépare le SQL avec une expression conditionnelle (similaire à une clause `WHERE`) pour ne renvoyer que les lignes où **`truck_brand_name`** est égal à **'Freezing Point'**.

#### **10. Retour du DataFrame (évaluation différée)**
```python
return df_menu_freezing_point
```
Retourne le DataFrame `df_menu_freezing_point`. Les DataFrames Snowpark utilisent une **évaluation différée (*lazy evaluation*)** : cette ligne ne déclenche pas l'exécution de la requête sur le serveur. L'exécution aura lieu lors de l'appel de la fonction `main` (quand vous cliquerez sur **Exécuter**).

---

**Lorsque vous êtes prêt, sélectionnez `Exécuter` pour lancer le code et voir le résultat.** L'exécution génère et exécute les instructions SQL, et les résultats de la requête du DataFrame retourné s'affichent dans la feuille de travail.

Votre sortie devrait ressembler à l'image suivante.

<img width="1542" height="656" alt="image" src="https://github.com/user-attachments/assets/ff848df4-1b6e-41ae-a738-471959ae48ba" />

### **Étape 4 : Nettoyage, résumé et ressources supplémentaires**

**Félicitations !** Vous avez terminé avec succès ce tutoriel pour les comptes d'essai.

Prenez quelques minutes pour revoir un **bref résumé** et les **points clés** abordés. Envisagez de **nettoyer** en supprimant les objets que vous avez créés. Pour en savoir plus, consultez d'autres rubriques de la **Documentation Snowflake**.

#### **Nettoyer les objets du tutoriel (optionnel)**
Si les objets que vous avez créés ne sont **plus nécessaires**, vous pouvez les supprimer du système avec des commandes **`DROP <object>`**.

Pour supprimer la base de données créée, exécutez la commande suivante :

```sql
DROP DATABASE IF EXISTS tasty_bytes_sample_data;
```

#### **Résumé et points clés**
En résumé, vous avez utilisé une **feuille de travail Python préchargée** dans Snowsight pour accomplir les étapes suivantes en code Python :

1.  **Importer les modules Snowpark** pour une application Python.
2.  **Créer une fonction Python**.
3.  **Créer une base de données et un schéma**.
4.  **Créer un *stage*** contenant des données dans un **bucket Amazon S3**.
5.  **Créer un DataFrame** pour spécifier la **source des données** dans un *stage*.
6.  **Créer une table** contenant des données provenant de **fichiers sur le *stage***.
7.  **Configurer un DataFrame** pour **interroger la nouvelle table** et **filtrer les données**.

**Voici quelques points clés** à retenir sur le chargement et l'interrogation des données :

*   Vous avez utilisé **Snowpark** pour exécuter des **instructions SQL dans du code Python**.
*   Vous avez créé un **stage** pour charger des données depuis un **fichier CSV**.
*   Vous avez créé une **base de données** pour stocker les données et un **schéma** pour regrouper logiquement les objets de la base de données.
*   Vous avez utilisé un **DataFrame** pour spécifier la **source de données** et **filtrer les données** pour une requête.
