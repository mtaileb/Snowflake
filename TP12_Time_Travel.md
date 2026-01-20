## Suppression et restauration d'une table plusieurs fois

Dans l'exemple suivant, le schéma `mytestdb.public` contient deux tables : `loaddata1` et `proddata1`. La table `loaddata1` est supprimée et recréée deux fois, créant ainsi **trois versions** de la table :

1.  Version **courante**
2.  Version **supprimée la plus récente** (deuxième)
3.  Version **supprimée la plus ancienne** (première)

L'exemple illustre ensuite comment restaurer les deux versions supprimées de la table :

1.  D'abord, la table courante portant le même nom est **renommée** en `loaddata3`. Cela permet de restaurer la version la plus récente de la table supprimée, en fonction de l'horodatage.
2.  Ensuite, la version la plus récente de la table supprimée est **restaurée**.
3.  La table restaurée est **renommée** en `loaddata2` pour permettre la restauration de la première version de la table supprimée.
4.  Enfin, la **première version** de la table supprimée est restaurée.

```sql
-- 1. Afficher les tables actuelles
SHOW TABLES HISTORY;

-- 2. Supprimer la table loaddata1
DROP TABLE loaddata1;

-- 3. Afficher l'historique après suppression
SHOW TABLES HISTORY;

-- 4. Recréer loaddata1 (nouvelle version) et insérer des données
CREATE TABLE loaddata1 (c1 number);
INSERT INTO loaddata1 VALUES (1111), (2222), (3333), (4444);

-- 5. Supprimer à nouveau loaddata1
DROP TABLE loaddata1;

-- 6. Créer une autre nouvelle version de loaddata1 avec un schéma différent
CREATE TABLE loaddata1 (c1 varchar);

-- 7. Afficher l'historique complet (3 versions de loaddata1)
SHOW TABLES HISTORY;

-- 8. Renommer la version actuelle pour libérer le nom 'loaddata1'
ALTER TABLE loaddata1 RENAME TO loaddata3;

-- 9. Restaurer la version supprimée la plus récente de loaddata1
UNDROP TABLE loaddata1;

-- 10. Afficher l'historique après restauration
SHOW TABLES HISTORY;

-- 11. Renommer la version restaurée pour libérer à nouveau 'loaddata1'
ALTER TABLE loaddata1 RENAME TO loaddata2;

-- 12. Restaurer la première version supprimée de loaddata1
UNDROP TABLE loaddata1;

-- 13. Afficher l'historique final avec les 3 versions restaurées
SHOW TABLES HISTORY;
```

**Résumé des versions finales :**
*   `LOADDATA1` : Première version originale restaurée (48 lignes)
*   `LOADDATA2` : Deuxième version restaurée (4 lignes de nombres)
*   `LOADDATA3` : Version courante (0 ligne, colonne varchar)
*   `PRODDATA1` : Table indépendante non modifiée
