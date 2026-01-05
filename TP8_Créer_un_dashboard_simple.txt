Dashboards
New Dashboard -> New tile -> SQL worksheet
Sélectionner la base de données SNOWFLAKE_SAMPLE_DATA
Exécuter: SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS;
En analysant les résultats (nota: la colonne o_orderstatus a les modalités Open, Filled, et Pending), que ferait la commande: SELECT o_orderstatus, COUNT(1) FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.ORDERS GROUP BY o_orderstatus;
Une fois les résultats affichés, analysez-les, puis cliquez sur 'Chart'
Ensuite modifiez le type de graphique pour trouver le plus adéquat
Une fois un graphique adéquat trouvé, cliquez sur 'Return to <dashboard>' en haut à gauche
Vous pouvez ajouter des nouvelles tiles au dashboard
A faire:
1) Créez un nouveau tile en refaisant le même exercice pour LINEITEM et l_linestatus
2) Partagez le dashboard avec vos camarades
