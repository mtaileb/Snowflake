📊 Guide : Création de Dashboard dans Snowsight
📋 Liste des Actions à Suivre
Étape 1 : Accéder aux Dashboards

    Naviguez vers Projects → Dashboards

Étape 2 : Créer un Nouveau Dashboard

    Cliquez sur New Dashboard → New tile

Étape 3 : Créer une Feuille SQL

    Dans le nouveau tile, créez une SQL worksheet

Étape 4 : Sélectionner la Base de Données

    Sélectionnez la base de données : SNOWFLAKE_SAMPLE_DATA

Étape 5 : Exécuter la Première Requête
sql

SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;

📝 Note d'analyse :
La colonne o_orderstatus contient trois modalités :

    Open (O) - Commande ouverte

    Filled (F) - Commande remplie/terminée

    Pending (P) - Commande en cours de traitement

Étape 6 : Exécuter la Requête d'Aggrégation
sql

SELECT 
    o_orderstatus, 
    COUNT(1) as nombre_commandes
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS 
GROUP BY o_orderstatus;

Étape 7 : Visualiser les Résultats

    Analysez les résultats affichés

    Cliquez sur le bouton 'Chart' pour créer une visualisation

Étape 8 : Configurer le Graphique

    Modifiez le type de graphique pour trouver le format le plus adéquat

    Suggestions :

        Diagramme en barres pour comparer les counts

        Diagramme circulaire (pie chart) pour les proportions

        Diagramme en anneau (donut chart) pour une vue alternative

Étape 9 : Sauvegarder et Retourner

    Une fois le graphique optimal trouvé, cliquez sur 'Return to <dashboard>' en haut à gauche

Étape 10 : Ajouter de Nouvelles Tiles

    Vous pouvez maintenant ajouter de nouvelles tiles au dashboard

Voici une explication détaillée de toutes les colonnes de la table SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS :

📋 Vue d'ensemble de la table

La table ORDERS contient les commandes clients dans le schéma TPC-H (benchmark de base de données transactionnelle).
sql

-- Voir la structure
DESCRIBE TABLE SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;

📊 Description détaillée des colonnes
1. O_ORDERKEY (NUMBER)

    Clé primaire unique identifiant chaque commande

    Type : Numérique (entier)

    Exemple : 1, 2, 3, ...

sql

SELECT MIN(o_orderkey), MAX(o_orderkey) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;

2. O_CUSTKEY (NUMBER)

    Clé étrangère vers la table CUSTOMER

    Identifie le client qui a passé la commande

    Relation : ORDERS.O_CUSTKEY → CUSTOMER.C_CUSTKEY

sql

-- Nombre de commandes par client
SELECT o_custkey, COUNT(*) as nb_commandes 
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_custkey
ORDER BY nb_commandes DESC
LIMIT 10;

3. O_ORDERSTATUS (VARCHAR(1))

    Statut de la commande (déjà expliqué)

    Valeurs : 'O' (Open/ouvert), 'F' (Filled/terminé), 'P' (Pending/en cours)

sql

SELECT 
    o_orderstatus,
    COUNT(*) as total,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM ORDERS), 2) as pourcentage
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_orderstatus;

4. O_TOTALPRICE (NUMBER(12,2))

    Prix total de la commande (incluant taxes, frais de port, etc.)

    Type : Nombre décimal avec 2 décimales

    Exemple : 173665.47

sql

-- Statistiques sur les prix
SELECT 
    MIN(o_totalprice) as prix_min,
    MAX(o_totalprice) as prix_max,
    AVG(o_totalprice) as prix_moyen,
    STDDEV(o_totalprice) as ecart_type
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;

5. O_ORDERDATE (DATE)

    Date de passage de la commande

    Format : YYYY-MM-DD

sql

-- Commandes par année
SELECT 
    YEAR(o_orderdate) as annee,
    COUNT(*) as nb_commandes,
    SUM(o_totalprice) as chiffre_affaires
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY YEAR(o_orderdate)
ORDER BY annee;

6. O_ORDERPRIORITY (VARCHAR(15))

    Priorité de la commande

    5 niveaux de priorité :

        1-URGENT

        2-HIGH

        3-MEDIUM

        4-NOT SPECIFIED

        5-LOW

sql

-- Distribution des priorités
SELECT 
    o_orderpriority,
    COUNT(*) as nb_commandes,
    AVG(o_totalprice) as prix_moyen
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_orderpriority
ORDER BY o_orderpriority;

7. O_CLERK (VARCHAR(15))

    Identifiant du commis/vendeur qui a traité la commande

    Format : Clerk#000000001

sql

-- Top 10 vendeurs
SELECT 
    o_clerk,
    COUNT(*) as nb_commandes,
    SUM(o_totalprice) as ca_total
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_clerk
ORDER BY ca_total DESC
LIMIT 10;

8. O_SHIPPRIORITY (NUMBER)

    Priorité d'expédition

    0 = priorité normale

        0 = priorité élevée

sql

SELECT 
    o_shippriority,
    COUNT(*) as nb_commandes,
    AVG(o_totalprice) as prix_moyen
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_shippriority
ORDER BY o_shippriority;

9. O_COMMENT (VARCHAR(79))

    Commentaires sur la commande

    Champ texte libre

    Peut contenir des remarques, instructions spéciales

sql

-- Commandes avec commentaires
SELECT 
    o_orderkey,
    o_comment,
    LENGTH(o_comment) as longueur_commentaire
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
WHERE o_comment IS NOT NULL
LIMIT 5;

🔗 Relations avec autres tables
Schéma des relations :
text

CUSTOMER (C_CUSTKEY) 
    ↓
ORDERS (O_CUSTKEY) → LINEITEM (L_ORDERKEY)
    ↓
PARTSUPP (PS_PARTKEY, PS_SUPPKEY)

Requêtes de jointure courantes :
sql

-- Commandes avec infos clients
SELECT 
    o.o_orderkey,
    o.o_orderdate,
    o.o_totalprice,
    c.c_name as nom_client,
    c.c_phone as telephone
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS o
JOIN SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.CUSTOMER c 
    ON o.o_custkey = c.c_custkey
LIMIT 10;

-- Commandes avec détails des lignes
SELECT 
    o.o_orderkey,
    o.o_orderdate,
    COUNT(l.l_linenumber) as nb_lignes,
    SUM(l.l_quantity) as quantite_totale
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS o
JOIN SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM l 
    ON o.o_orderkey = l.l_orderkey
GROUP BY o.o_orderkey, o.o_orderdate
ORDER BY o.o_orderdate
LIMIT 10;

📈 Métriques business importantes
sql

-- KPI principaux
SELECT
    -- Volume
    COUNT(DISTINCT o_orderkey) as total_commandes,
    COUNT(DISTINCT o_custkey) as clients_uniques,
    
    -- Monétaire
    SUM(o_totalprice) as chiffre_affaires_total,
    AVG(o_totalprice) as panier_moyen,
    
    -- Temporel
    MIN(o_orderdate) as premiere_commande,
    MAX(o_orderdate) as derniere_commande,
    DATEDIFF(day, MIN(o_orderdate), MAX(o_orderdate)) as periode_jours
    
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;

🎯 Cas d'usage pratiques
1. Analyse de la valeur client
sql

WITH client_stats AS (
    SELECT 
        o_custkey,
        COUNT(*) as nb_commandes,
        SUM(o_totalprice) as ca_total,
        MIN(o_orderdate) as premiere_commande,
        MAX(o_orderdate) as derniere_commande
    FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
    GROUP BY o_custkey
)
SELECT 
    CASE 
        WHEN nb_commandes >= 10 THEN 'Client fidèle'
        WHEN nb_commandes >= 5 THEN 'Client régulier'
        ELSE 'Client occasionnel'
    END as segment,
    COUNT(*) as nb_clients,
    AVG(ca_total) as ca_moyen
FROM client_stats
GROUP BY segment
ORDER BY ca_moyen DESC;

2. Analyse temporelle
sql

SELECT
    DATE_TRUNC('month', o_orderdate) as mois,
    o_orderstatus,
    COUNT(*) as nb_commandes,
    SUM(o_totalprice) as ca_mensuel,
    AVG(o_totalprice) as panier_moyen
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY mois, o_orderstatus
ORDER BY mois, o_orderstatus;

3. Performance des vendeurs
sql

SELECT
    o_clerk as vendeur,
    COUNT(DISTINCT o_custkey) as clients_uniques,
    COUNT(*) as nb_commandes,
    SUM(o_totalprice) as ca_total,
    AVG(o_totalprice) as panier_moyen,
    -- Taux de conversion (commandes/commandes ouvertes)
    SUM(CASE WHEN o_orderstatus = 'F' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as taux_remplissage
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
GROUP BY o_clerk
HAVING nb_commandes >= 10
ORDER BY ca_total DESC
LIMIT 20;

💡 Bonnes pratiques de requêtage
sql

-- Toujours limiter les résultats en exploration
SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS LIMIT 100;

-- Utiliser WHERE pour filtrer tôt
SELECT o_orderkey, o_orderdate, o_totalprice
FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS
WHERE o_orderdate >= '1995-01-01'
  AND o_orderdate < '1995-02-01'
ORDER BY o_totalprice DESC;

-- Indexer sur les colonnes fréquemment filtrées
-- (Dans Snowflake, c'est automatique via le clustering)

📊 Résumé visuel des colonnes :
text

┌─────────────────────────────────────────────────────────────┐
│                    TABLE: ORDERS                            │
├─────────────┬──────────────┬────────────────┬───────────────┤
│ Colonne     │ Type         │ Description    │ Exemple       │
├─────────────┼──────────────┼────────────────┼───────────────┤
│ O_ORDERKEY  │ NUMBER(38)   │ ID Commande    │ 1             │
│ O_CUSTKEY   │ NUMBER(38)   │ ID Client      │ 36901         │
│ O_ORDERSTATUS│ VARCHAR(1)  │ Statut         │ F             │
│ O_TOTALPRICE│ NUMBER(12,2) │ Prix total     │ 173665.47     │
│ O_ORDERDATE │ DATE         │ Date commande  │ 1996-01-02    │
│ O_ORDERPRIORITY│ VARCHAR(15) │ Priorité      │ 1-URGENT      │
│ O_CLERK     │ VARCHAR(15)  │ Vendeur        │ Clerk#0000001 │
│ O_SHIPPRIORITY│ NUMBER(38)  │ Priorité exp.  │ 0             │
│ O_COMMENT   │ VARCHAR(79)  │ Commentaires   │ "Urgent!"     │
└─────────────┴──────────────┴────────────────┴───────────────┘

Cette table est centrale dans le schéma TPC-H et permet d'analyser les performances commerciales, le comportement des clients, et l'efficacité des opérations.
