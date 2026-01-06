**Cortex Analyst** transforme des **questions en langage naturel** sur vos données en résultats, en **générant et exécutant automatiquement des requêtes SQL**. Ce tutoriel décrit comment configurer Cortex Analyst pour répondre à des questions sur un jeu de données de **revenus chronologiques**.

### **Ce que vous allez apprendre**
1.  **Établir un modèle sémantique** pour le jeu de données.
2.  **Créer une application Streamlit** qui interroge Cortex Analyst.

### **Prérequis**
Les conditions suivantes sont nécessaires pour réaliser ce tutoriel :

*   Vous disposez d'un **compte Snowflake** et d'un **utilisateur** ayant un **rôle** accordant les **privilèges nécessaires** pour créer une base de données, un schéma, des tables, un *stage* et des objets d'entrepôt virtuel.
*   Vous avez **Streamlit installé et configuré** sur votre système local: https://docs.streamlit.io/get-started/installation/command-line
Ci-dessous les commandes pour installer Streamlit sur un système Ubuntu:
$ sudo apt install python3.10-venv
$ python3 -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt (le fichier requirements se trouve ici: https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst/raw/refs/heads/main/requirements.txt)


### **Étape 1 : Configuration**

#### **1. Obtention des données d'exemple**
Vous utiliserez un jeu de données d'exemple téléchargé depuis GitHub https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst/tree/main/data. Téléchargez les **fichiers suivants** sur votre système :

*   `daily_revenue.csv`
*   `product.csv`
*   `region.csv`

Téléchargez également le **modèle sémantique YAML** depuis GitHub https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-analyst/blob/main/revenue_timeseries.yaml

Il est conseillé d'examiner ce **modèle sémantique** avant de continuer. Il complète le schéma SQL de chaque table avec des **informations supplémentaires** qui aident Cortex Analyst à comprendre les questions sur les données. Pour plus d'informations, consultez la *Spécification du modèle sémantique Cortex Analyst*.

**Note :** Dans un contexte réel (hors tutoriel), vous utiliseriez **vos propres données** (déjà dans une table Snowflake) et développeriez **votre propre modèle sémantique**.

---

#### **2. Création des objets Snowflake**
Utilisez **Snowsight** (l'interface Snowflake) pour créer les objets nécessaires. Vous pourrez les supprimer après le tutoriel.

**Note :** Utilisez un rôle pouvant créer des bases de données, schémas, entrepôts, *stages* et tables.

**Procédure :**

1.  **Connectez-vous** à Snowsight.
2.  Dans le menu de navigation, sélectionnez **Projets » Feuilles de travail**, puis cliquez sur **+**. Une nouvelle **feuille de travail SQL** s'ouvre.
3.  **Collez le code SQL ci-dessous** dans la feuille, puis sélectionnez **Exécuter tout** dans le menu déroulant en haut à droite.

**⚠️ Important :** Remplacez **`<votre_utilisateur>`** par votre nom d'utilisateur Snowflake réel.

```sql
/*--
• Création de la base de données, du schéma, de l'entrepôt et du stage
--*/

USE ROLE SECURITYADMIN;

CREATE ROLE IF NOT EXISTS cortex_user_role;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE cortex_user_role;

GRANT ROLE cortex_user_role TO USER <votre_utilisateur>; -- ⚠️ REMPLACER <votre_utilisateur>

USE ROLE sysadmin;

-- Créer la base de données de démo
CREATE OR REPLACE DATABASE cortex_analyst_demo;

-- Créer le schéma
CREATE OR REPLACE SCHEMA cortex_analyst_demo.revenue_timeseries;

-- Créer l'entrepôt virtuel
CREATE OR REPLACE WAREHOUSE cortex_analyst_wh
    WAREHOUSE_SIZE = 'large'
    WAREHOUSE_TYPE = 'standard'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
COMMENT = 'Entrepôt pour la démo Cortex Analyst';

GRANT USAGE ON WAREHOUSE cortex_analyst_wh TO ROLE cortex_user_role;
GRANT OPERATE ON WAREHOUSE cortex_analyst_wh TO ROLE cortex_user_role;

GRANT OWNERSHIP ON SCHEMA cortex_analyst_demo.revenue_timeseries TO ROLE cortex_user_role;
GRANT OWNERSHIP ON DATABASE cortex_analyst_demo TO ROLE cortex_user_role;

USE ROLE cortex_user_role;

-- Utiliser l'entrepôt créé
USE WAREHOUSE cortex_analyst_wh;

USE DATABASE cortex_analyst_demo;
USE SCHEMA cortex_analyst_demo.revenue_timeseries;

-- Créer un stage pour les données brutes
CREATE OR REPLACE STAGE raw_data DIRECTORY = (ENABLE = TRUE);

/*--
• Création des tables de fait et de dimension
--*/

-- Table de fait : daily_revenue
CREATE OR REPLACE TABLE cortex_analyst_demo.revenue_timeseries.daily_revenue (
    date DATE,
    revenue FLOAT,
    cogs FLOAT,
    forecasted_revenue FLOAT,
    product_id INT,
    region_id INT
);

-- Table de dimension : product_dim
CREATE OR REPLACE TABLE cortex_analyst_demo.revenue_timeseries.product_dim (
    product_id INT,
    product_line VARCHAR(16777216)
);

-- Table de dimension : region_dim
CREATE OR REPLACE TABLE cortex_analyst_demo.revenue_timeseries.region_dim (
    region_id INT,
    sales_region VARCHAR(16777216),
    state VARCHAR(16777216)
);
```

---

**Ce que crée le SQL ci-dessus :**

*   Une **base de données** nommée `cortex_analyst_demo`
*   Un **schéma** dans cette base de données appelé `revenue_timeseries`
*   **Trois tables** dans ce schéma :
    *   `daily_revenue` (table de **fait** principale)
    *   `product_dim` (table de **dimension** produits)
    *   `region_dim` (table de **dimension** régions)
*   Un ***stage*** nommé `raw_data` qui contiendra les données brutes à charger.
*   Un **entrepôt virtuel** nommé `cortex_analyst_wh`

**Note :** L'entrepôt virtuel est initialement **suspendu**. Il démarre **automatiquement** lors de l'exécution d'une requête.

### **Étape 2 : Charger les données dans Snowflake**

Pour importer les données des fichiers CSV vers Snowflake, vous allez :
1.  **Téléverser** les fichiers vers le *stage*.
2.  **Charger** les données du *stage* vers les tables.

Vous allez également téléverser le fichier YAML du modèle sémantique pour une étape ultérieure.

**Fichiers à téléverser :**
*   `daily_revenue.csv`
*   `product.csv`
*   `region.csv`
*   `revenue_timeseries.yaml` (modèle sémantique)

---

#### **1. Téléversement des fichiers dans Snowsight**

1.  Connectez-vous à **Snowsight**.
2.  Dans le menu de navigation, sélectionnez **Ingestion » Ajouter des données**, puis **Charger des fichiers vers un stage**.
3.  **Glissez-déposez les quatre fichiers** téléchargés dans la fenêtre Snowsight.
4.  Choisissez la base de données **`cortex_analyst_demo`** et le stage **`raw_data`**, puis cliquez sur le bouton **Téléverser**.

---

#### **2. Chargement des données depuis les CSV vers les tables**

Exécutez les commandes SQL suivantes dans une **feuille de travail Snowsight**.

```sql
-- Utiliser l'entrepôt
USE WAREHOUSE cortex_analyst_wh;

-- Charger daily_revenue.csv dans la table daily_revenue
COPY INTO cortex_analyst_demo.revenue_timeseries.daily_revenue
FROM @raw_data
FILES = ('daily_revenue.csv')
FILE_FORMAT = (
    TYPE=CSV,
    SKIP_HEADER=1,
    FIELD_DELIMITER=',',
    TRIM_SPACE=FALSE,
    FIELD_OPTIONALLY_ENCLOSED_BY=NONE,
    REPLACE_INVALID_CHARACTERS=TRUE,
    DATE_FORMAT=AUTO,
    TIME_FORMAT=AUTO,
    TIMESTAMP_FORMAT=AUTO,
    EMPTY_FIELD_AS_NULL = FALSE,
    ERROR_ON_COLUMN_COUNT_MISMATCH=FALSE
)
ON_ERROR=CONTINUE
FORCE = TRUE;

-- Charger product.csv dans la table product_dim
COPY INTO cortex_analyst_demo.revenue_timeseries.product_dim
FROM @raw_data
FILES = ('product.csv')
FILE_FORMAT = (
    TYPE=CSV,
    SKIP_HEADER=1,
    FIELD_DELIMITER=',',
    TRIM_SPACE=FALSE,
    FIELD_OPTIONALLY_ENCLOSED_BY=NONE,
    REPLACE_INVALID_CHARACTERS=TRUE,
    DATE_FORMAT=AUTO,
    TIME_FORMAT=AUTO,
    TIMESTAMP_FORMAT=AUTO,
    EMPTY_FIELD_AS_NULL = FALSE,
    ERROR_ON_COLUMN_COUNT_MISMATCH=FALSE
)
ON_ERROR=CONTINUE
FORCE = TRUE;

-- Charger region.csv dans la table region_dim
COPY INTO cortex_analyst_demo.revenue_timeseries.region_dim
FROM @raw_data
FILES = ('region.csv')
FILE_FORMAT = (
    TYPE=CSV,
    SKIP_HEADER=1,
    FIELD_DELIMITER=',',
    TRIM_SPACE=FALSE,
    FIELD_OPTIONALLY_ENCLOSED_BY=NONE,
    REPLACE_INVALID_CHARACTERS=TRUE,
    DATE_FORMAT=AUTO,
    TIME_FORMAT=AUTO,
    TIMESTAMP_FORMAT=AUTO,
    EMPTY_FIELD_AS_NULL = FALSE,
    ERROR_ON_COLUMN_COUNT_MISMATCH=FALSE
)
ON_ERROR=CONTINUE
FORCE = TRUE;
```

**Note :** Seul le résultat de la **dernière commande** s'affiche dans le panneau de sortie. Vous pouvez exécuter les commandes **ligne par ligne** pour voir le résultat de chacune.

### **Étape 3 : Créer une application Streamlit pour interroger vos données via Cortex Analyst**

Pour créer une application Streamlit utilisant Cortex Analyst :

1.  **Créez un fichier Python local** nommé `analyst_demo.py`.
2.  **Copiez le code ci-dessous** dans le fichier.
3.  **Remplacez les valeurs fictives** par vos informations de compte Snowflake.
4.  **Exécutez l'application Streamlit** avec `streamlit run analyst_demo.py`.

---

**Code de l'application (`analyst_demo.py`) :**

```python
from typing import Any, Dict, List, Optional

import pandas as pd
import requests
import snowflake.connector
import streamlit as st

# -- CONFIGURATION SNOWFLAKE (À MODIFIER !) --
DATABASE = "CORTEX_ANALYST_DEMO"
SCHEMA = "REVENUE_TIMESERIES"
STAGE = "RAW_DATA"
SEMANTIC_MODEL_FILE = "revenue_timeseries.yaml"
WAREHOUSE = "cortex_analyst_wh"

# ⚠️ REMPLACEZ LES VALEURS CI-DESSOUS AVEC VOS INFORMATIONS DE CONNEXION
HOST = "<host>"         # Ex: "myaccount.snowflakecomputing.com"
ACCOUNT = "<account>"   # Votre identifiant de compte
USER = "<user>"         # Votre nom d'utilisateur
PASSWORD = "<password>" # Votre mot de passe
ROLE = "<role>"         # Rôle avec accès (ex: "cortex_user_role")

# -- INITIALISATION DE LA CONNEXION SNOWFLAKE --
if 'CONN' not in st.session_state or st.session_state.CONN is None:
    st.session_state.CONN = snowflake.connector.connect(
        user=USER,
        password=PASSWORD,
        account=ACCOUNT,
        host=HOST,
        port=443,
        warehouse=WAREHOUSE,
        role=ROLE,
    )

# -- FONCTION POUR INTERROGER CORTEX ANALYST (API REST) --
def send_message(prompt: str) -> Dict[str, Any]:
    """Appelle l'API REST Cortex Analyst et retourne la réponse."""
    request_body = {
        "messages": [{"role": "user", "content": [{"type": "text", "text": prompt}]}],
        "semantic_model_file": f"@{DATABASE}.{SCHEMA}.{STAGE}/{SEMANTIC_MODEL_FILE}",
    }
    resp = requests.post(
        url=f"https://{HOST}/api/v2/cortex/analyst/message",
        json=request_body,
        headers={
            "Authorization": f'Snowflake Token="{st.session_state.CONN.rest.token}"',
            "Content-Type": "application/json",
        },
    )
    request_id = resp.headers.get("X-Snowflake-Request-Id")
    if resp.status_code < 400:
        return {**resp.json(), "request_id": request_id}
    else:
        raise Exception(
            f"Échec de la requête (id: {request_id}) - Statut {resp.status_code}: {resp.text}"
        )

# -- TRAITEMENT D'UNE QUESTION DE L'UTILISATEUR --
def process_message(prompt: str) -> None:
    """Traite un message et ajoute la réponse au chat."""
    # Ajouter la question à l'historique
    st.session_state.messages.append(
        {"role": "user", "content": [{"type": "text", "text": prompt}]}
    )
    # Afficher la question
    with st.chat_message("user"):
        st.markdown(prompt)
    
    # Générer et afficher la réponse
    with st.chat_message("assistant"):
        with st.spinner("Génération de la réponse..."):
            response = send_message(prompt=prompt)
            request_id = response["request_id"]
            content = response["message"]["content"]
            display_content(content=content, request_id=request_id)
    
    # Ajouter la réponse à l'historique
    st.session_state.messages.append(
        {"role": "assistant", "content": content, "request_id": request_id}
    )

# -- AFFICHAGE DU CONTENU DE LA RÉPONSE (texte, suggestions, SQL, résultats) --
def display_content(
    content: List[Dict[str, str]],
    request_id: Optional[str] = None,
    message_index: Optional[int] = None,
) -> None:
    """Affiche les différents éléments d'une réponse."""
    message_index = message_index or len(st.session_state.messages)
    
    # Afficher l'ID de requête (optionnel)
    if request_id:
        with st.expander("ID de requête", expanded=False):
            st.markdown(request_id)
    
    # Parcourir les éléments de la réponse
    for item in content:
        if item["type"] == "text":
            st.markdown(item["text"])  # Texte explicatif
        elif item["type"] == "suggestions":
            # Boutons de suggestions (questions suivantes possibles)
            with st.expander("Suggestions", expanded=True):
                for suggestion_index, suggestion in enumerate(item["suggestions"]):
                    if st.button(suggestion, key=f"{message_index}_{suggestion_index}"):
                        st.session_state.active_suggestion = suggestion
        elif item["type"] == "sql":
            # Afficher la requête SQL générée
            with st.expander("Requête SQL", expanded=False):
                st.code(item["statement"], language="sql")
            # Exécuter la requête et afficher les résultats
            with st.expander("Résultats", expanded=True):
                with st.spinner("Exécution de la requête SQL..."):
                    df = pd.read_sql(item["statement"], st.session_state.CONN)
                    if len(df.index) > 1:
                        # Onglets pour visualiser les données
                        data_tab, line_tab, bar_tab = st.tabs(
                            ["Données", "Graphique linéaire", "Graphique à barres"]
                        )
                        data_tab.dataframe(df)
                        if len(df.columns) > 1:
                            df = df.set_index(df.columns[0])
                        with line_tab:
                            st.line_chart(df)
                        with bar_tab:
                            st.bar_chart(df)
                    else:
                        st.dataframe(df)

# -- INTERFACE STREAMLIT --
st.title("Cortex Analyst")
st.markdown(f"Modèle sémantique : `{SEMANTIC_MODEL_FILE}`")

# Initialisation de l'état de la session
if "messages" not in st.session_state:
    st.session_state.messages = []
    st.session_state.suggestions = []
    st.session_state.active_suggestion = None

# Affichage de l'historique des messages
for message_index, message in enumerate(st.session_state.messages):
    with st.chat_message(message["role"]):
        display_content(
            content=message["content"],
            request_id=message.get("request_id"),
            message_index=message_index,
        )

# Champ de saisie pour une nouvelle question
if user_input := st.chat_input("Quelle est votre question ?"):
    process_message(prompt=user_input)

# Gestion des suggestions cliquées
if st.session_state.active_suggestion:
    process_message(prompt=st.session_state.active_suggestion)
    st.session_state.active_suggestion = None
```

---

#### **Instructions d'exécution :**

1.  **Remplacez les valeurs entre `<>`** dans la section configuration par vos propres informations Snowflake :
    *   `HOST` : Votre host Snowflake (ex: `xyz123.snowflakecomputing.com`)
    *   `ACCOUNT` : Votre identifiant de compte
    *   `USER` : Votre nom d'utilisateur
    *   `PASSWORD` : Votre mot de passe
    *   `ROLE` : Le rôle que vous avez créé (`cortex_user_role`)

2.  **Lancez l'application** depuis votre terminal :
    ```bash
    streamlit run analyst_demo.py
    ```

3.  **Posez une question** dans l'interface qui s'ouvre dans votre navigateur. Commencez par **"What questions can I ask?"** (Quelles questions puis-je poser ?) et essayez les suggestions proposées. L'application affichera :
    *   Les **réponses textuelles** de Cortex Analyst
    *   Les **requêtes SQL générées**
    *   Les **résultats sous forme de tableau**
    *   Des **visualisations graphiques** automatiques
    *   Des **suggestions de questions** connexes

<img width="3574" height="1692" alt="image" src="https://github.com/user-attachments/assets/ac1ef213-ea95-4e1a-b3e9-16c55c5b2af5" />

  ### **Étape 4 : Nettoyage**

#### **Nettoyage (optionnel)**

Pour **restaurer votre système** à son état d'origine avant ce tutoriel, exécutez les commandes suivantes dans une feuille de travail Snowsight :

```sql
-- Supprime la base de données et tous ses objets enfants (tables, schémas, stages)
DROP DATABASE IF EXISTS cortex_analyst_demo;

-- Supprime l'entrepôt virtuel créé pour la démonstration
DROP WAREHOUSE IF EXISTS cortex_analyst_wh;
```

**Note importante :** La suppression de la base de données **entraîne automatiquement la suppression de tous ses objets enfants**, y compris :
*   Les tables (`daily_revenue`, `product_dim`, `region_dim`)
*   Le schéma (`revenue_timeseries`)
*   Le stage (`raw_data`)
*   Tous les autres objets éventuellement créés dans cette base de données

**Conseil :** Effectuez cette étape uniquement si vous n'avez plus besoin des données et objets créés pendant le tutoriel.
