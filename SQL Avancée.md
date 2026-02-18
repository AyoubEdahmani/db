# **Guide Complet sur les Vues, Index, Séquences, Synonymes et Transactions Oracle**
## **Définitions, Concepts, Syntaxes SQL et Explications détaillées pour l’examen**

---

## **Introduction**

Ce guide reprend l’intégralité du contenu du fichier **BDA-134-191.pdf** en l’enrichissant d’explications détaillées sur chaque mot-clé SQL, de conseils pour l’examen et d’exemples concrets. Vous y trouverez :

- Les **vues** (simples et matérialisées)
- Les **index** (B-tree et Bitmap)
- Les **séquences**
- Les **synonymes**
- Les **transactions** et leurs propriétés ACID

---

## **1. Les Vues (Views)**

### **1.1 Définition**

Une **vue** est une **table virtuelle** dont le contenu est défini par une requête SQL. Elle ne stocke pas physiquement les données mais donne accès à un sous-ensemble ou une transformation des données des tables de base.

**Caractéristiques :**
- Relation virtuelle calculable par une requête.
- Fenêtre dynamique : toute modification dans les tables sous-jacentes se reflète immédiatement dans la vue.
- Aucun fichier physique associé.

### **1.2 Rôle des vues**

- **Définir des schémas externes** : masquer la complexité de la structure réelle.
- **Simplifier les requêtes** : encapsuler des jointures complexes.
- **Améliorer la sécurité** : restreindre l’accès à certaines colonnes ou lignes.
- **Optimiser les performances** (dans certains cas).

> **Important pour l’examen :** Les vues font partie du **schéma externe** (niveau 3 de l’architecture ANSI/SPARC).

### **1.3 Syntaxe de création**

```sql
CREATE [OR REPLACE] VIEW nom_vue [(liste_attributs)]
AS requête_sql_select
[WITH CHECK OPTION];
```

**Explication des mots-clés :**
- **CREATE VIEW** : crée une nouvelle vue.
- **OR REPLACE** : si la vue existe déjà, elle est remplacée (évite une erreur).
- **nom_vue** : identifiant de la vue.
- **liste_attributs** : facultative, permet de renommer les colonnes de la vue.
- **AS** : introduit la requête qui définit la vue.
- **WITH CHECK OPTION** : garantit que les modifications via la vue respectent la condition de la vue (voir plus loin).

### **1.4 Exemples simples**

**Exemple 1 :** Vue des clients parisiens
```sql
CREATE VIEW V_Clients_Paris AS
SELECT id_client, nom, prenom, telephone
FROM Clients
WHERE ville = 'Paris';
```

**Exemple 2 :** Vue avec alias de colonnes
```sql
CREATE VIEW V_Employes_Recents (emp_id, nom_complet, salaire) AS
SELECT employee_id, first_name || ' ' || last_name, salary
FROM employees
WHERE hire_date > SYSDATE - 365;
```

**Exemple 3 :** Vue avec agrégation
```sql
CREATE VIEW V_Somme_Salaires (job_titre, total_salaire) AS
SELECT j.job_title, SUM(e.salary)
FROM employees e, jobs j
WHERE e.job_id = j.job_id
GROUP BY j.job_title;
```

**Explication des mots-clés supplémentaires :**
- `||` : opérateur de concaténation en Oracle.
- `SYSDATE` : fonction retournant la date courante.
- `SUM` : fonction d'agrégation.
- `GROUP BY` : regroupe les lignes par titre de poste.

### **1.5 Nommage explicite des colonnes**

Il est **obligatoire** de spécifier les noms des colonnes dans la vue lorsque la requête contient :
- des **fonctions d’agrégation** (COUNT, SUM, AVG…),
- des **expressions calculées** (colonne * 1.1),
- des **jointures** avec des noms de colonnes ambigus.

**Exemple avec agrégation :**
```sql
-- Sans alias : la colonne s'appellera COUNT(*)
CREATE VIEW Vue1 AS SELECT COUNT(*), AVG(salaire) FROM employees;

-- Avec alias dans la requête
CREATE VIEW Vue2 AS SELECT COUNT(*) AS Nb_Employes, AVG(salaire) AS Salaire_Moyen FROM employees;

-- Avec liste d'attributs (recommandé)
CREATE VIEW Vue3 (Nb_Employes, Salaire_Moyen) AS
SELECT COUNT(*), AVG(salaire) FROM employees;
```

**Exemple avec expression calculée :**
```sql
CREATE VIEW Commandes_Avec_Remise (commande_id, montant_initial, montant_remise) AS
SELECT commande_id, montant_total, montant_total * 0.9
FROM commandes
WHERE statut = 'VALIDEE';
```

**Exemple avec jointure et colonnes dupliquées :**
```sql
CREATE VIEW Contacts_Mixte (
  client_id, client_nom, client_ville,
  fournisseur_id, fournisseur_nom, fournisseur_ville
) AS
SELECT c.client_id, c.nom, c.ville,
       f.fournisseur_id, f.nom, f.ville
FROM clients c
LEFT JOIN fournisseurs f ON c.ville = f.ville;
```

> **Astuce examen :** Toujours nommer les colonnes explicitement dans la vue pour éviter des noms par défaut peu explicites ou des erreurs.

### **1.6 Vues modifiables (Updatable Views)**

Une vue est dite **modifiable** (on peut y faire des INSERT, UPDATE, DELETE) si elle respecte certaines conditions. Oracle impose la notion de **"key-preserved table"** : la vue doit être basée sur une table dont la clé primaire est présente et unique dans chaque ligne de la vue.

**Conditions pour qu’une vue soit modifiable :**
1. Chaque colonne de la vue correspond directement à une colonne d’une table de base (pas d’expression).
2. La vue ne contient **pas** :
   - d’opérateur d’ensemble (UNION, INTERSECT, MINUS),
   - de DISTINCT,
   - de fonctions de groupe (SUM, AVG…),
   - d’expressions calculées,
   - de GROUP BY ou ORDER BY,
   - de jointures (sauf si une table est "key-preserved").

**Exemple de vue modifiable :**
```sql
CREATE VIEW v_employees AS
SELECT id, nom, salaire FROM employes;
-- On peut faire UPDATE v_employees SET salaire = 5000 WHERE id = 1;
```

**Exemple de vue non modifiable :**
```sql
CREATE VIEW Vue_Employes_Dept AS
SELECT e.id_emp, e.nom, d.nom_dept
FROM employes e JOIN departements d ON e.dept_id = d.dept_id;
-- Tentative de mise à jour sur nom_dept échoue car Oracle ne sait pas quelle table modifier.
```

**Tableau récapitulatif (d’après le cours) :**

| Requête de la vue | Modifiable ? |
|-------------------|---------------|
| `SELECT * FROM Clients` | OUI |
| `SELECT nom, prenom FROM Clients` | NON (clé absente) |
| `SELECT * FROM Clients WHERE ville = 'Paris'` | OUI |
| `SELECT client_id, nom FROM Clients WHERE age > 30` | OUI |
| `SELECT nom, prenom, salaire*12 FROM Employes` | NON (expression) |
| `SELECT client_id, UPPER(nom) FROM Clients` | NON |
| `SELECT client_id, nom FROM Clients WHERE ville LIKE '%ris'` | OUI |
| `SELECT c.client_id, c.nom, v.ville FROM Clients c JOIN Villes v ON c.ville_id = v.id` | OUI (si clé préservée) |

> **Important examen :** Une vue avec jointure peut être modifiable si la table dont la clé est préservée est la table cible de la modification.

### **1.7 L’option WITH CHECK OPTION**

Cette option empêche les modifications qui feraient que la ligne ne corresponde plus à la définition de la vue.

**Exemple sans WITH CHECK OPTION :**
```sql
CREATE VIEW Nom_Emp AS
SELECT * FROM EMPLOYEES WHERE last_name LIKE 'JOHN%';
-- On peut insérer un employé avec un nom différent
INSERT INTO Nom_Emp VALUES (6666, 'TATA', 'TINTIN', ...);
-- Cette ligne apparaît dans la vue alors qu'elle ne devrait pas.
```

**Avec WITH CHECK OPTION :**
```sql
CREATE VIEW Nom_Emp AS
SELECT * FROM EMPLOYEES WHERE last_name LIKE 'JOHN%'
WITH CHECK OPTION;
-- L'insertion d'un nom différent génère une erreur ORA-01402.
```

**Explication :** `WITH CHECK OPTION` force le SGBD à vérifier que toute modification (INSERT, UPDATE) via la vue respecte la condition `WHERE` de la vue.

### **1.8 Renommer et supprimer une vue**

```sql
RENAME ancien_nom TO nouveau_nom;
DROP VIEW nom_de_la_vue;
```

---

## **2. Les Vues Matérialisées (Materialized Views)**

### **2.1 Définition**

Une **vue matérialisée** est une copie physique des données résultant d’une requête. Contrairement à une vue simple, elle stocke les données sur disque, ce qui améliore les performances pour les requêtes complexes ou distantes.

### **2.2 Syntaxe**

```sql
CREATE MATERIALIZED VIEW nom_vue
REFRESH [FAST | COMPLETE | FORCE]
ON [COMMIT | DEMAND]
AS requête_SQL;
```

**Mots-clés expliqués :**
- **REFRESH** : mode de rafraîchissement.
  - **FAST** : rafraîchissement incrémental (nécessite un **Materialized View Log** sur la table source).
  - **COMPLETE** : recalcul intégral.
  - **FORCE** : Oracle choisit FAST si possible, sinon COMPLETE.
- **ON COMMIT** : la vue est rafraîchie automatiquement après chaque COMMIT sur la table source.
- **ON DEMAND** : le rafraîchissement est déclenché manuellement (par défaut).

### **2.3 Exemple**

```sql
CREATE MATERIALIZED VIEW mv_salaire
REFRESH FAST
ON DEMAND
AS SELECT job_id, SUM(salary) AS total_sal
   FROM employees
   GROUP BY job_id;
```

Pour utiliser **REFRESH FAST**, il faut créer un log sur la table source :
```sql
CREATE MATERIALIZED VIEW LOG ON employees;
```

### **2.4 Rafraîchissement manuel**

```sql
EXEC DBMS_MVIEW.REFRESH('mv_salaire', 'C');  -- 'C' pour COMPLETE
```

### **2.5 Qu’est-ce qu’un Materialized View Log ?**

C’est une table spéciale qui enregistre les modifications (INSERT, UPDATE, DELETE) sur la table source. Lors d’un rafraîchissement FAST, seules ces modifications sont appliquées à la vue, ce qui évite de tout recalculer.

**Exemple de contenu de log :**
```
id=5 | OLD v=Marseille | NEW v=Paris | time=2025-02-20
```

> **Important examen :** Les vues matérialisées sont utilisées dans les bases de données distribuées pour répliquer les données et améliorer les performances.

---

## **3. Les Index**

### **3.1 Définition**

Un **index** est un objet de base de données qui accélère l’accès aux lignes d’une table. C’est comme l’index d’un livre : il permet de trouver rapidement une information sans parcourir tout l’ouvrage.

### **3.2 Types d’index dans Oracle**

- **B-tree (arbre équilibré)** : efficace pour les colonnes avec beaucoup de valeurs distinctes (clé primaire, colonnes très sélectives).
- **Bitmap** : efficace pour les colonnes avec peu de valeurs distinctes (faible cardinalité), comme `sexe`, `couleur`, `statut`.

### **3.3 Quand créer un index ?**

**À créer :**
- Colonnes utilisées dans les **jointures**.
- Colonnes souvent utilisées dans les **conditions WHERE**.
- Tables volumineuses avec des sélections portant sur moins de 15% des lignes.

**À éviter :**
- Colonnes souvent modifiées (l’index doit être mis à jour à chaque modification).
- Tables de petite taille.
- Requêtes avec `IS NULL` (les NULL ne sont pas indexés dans un index B-tree).

**Règles de choix :**
- **Bitmap** : attribut avec peu de valeurs distinctes.
- **B-tree** : attribut avec beaucoup de valeurs distinctes.

### **3.4 Syntaxe de création**

```sql
CREATE [UNIQUE] [BITMAP] INDEX nom_index
ON nom_table (colonne1 [ASC|DESC], colonne2, ...)
[TABLESPACE nom_tablespace];
```

**Mots-clés :**
- **UNIQUE** : garantit que toutes les valeurs de la colonne (ou combinaison) sont uniques.
- **BITMAP** : crée un index bitmap.
- **ASC|DESC** : ordre de tri (par défaut ASC).
- **TABLESPACE** : emplacement physique de l’index (optionnel).

**Exemples :**
```sql
CREATE INDEX idx_clients_nom ON Clients(nom);
CREATE INDEX idx_emp_salaire ON employees(salary DESC);
CREATE UNIQUE INDEX idx_emp_email ON employees(email);
CREATE BITMAP INDEX idx_emp_new ON employees(new_emp);
```

### **3.5 Fonctionnement d’un index B-tree**

Un index B-tree est une structure arborescente :
- **Niveau racine (Root)** : contient les plages de valeurs.
- **Niveaux intermédiaires (Branch)** : orientent vers les feuilles.
- **Feuilles (Leaf)** : contiennent les valeurs et les **ROWID** (adresses physiques des lignes).

**ROWID** : identifiant unique d’une ligne en Oracle (ex : `AAAEACAABAAAAGiAAA`).

**Exemple de parcours :** Pour trouver la ligne avec `id = 50`, l’index parcourt la racine, puis la branche, puis la feuille, et obtient le ROWID pour accéder directement à la ligne.

### **3.6 Fonctionnement d’un index Bitmap**

Pour une colonne avec peu de valeurs distinctes, chaque valeur se voit attribuer un **vecteur de bits** (bitmap) indiquant les lignes où elle apparaît.

**Exemple avec la table Employé :**

| EmpNo | EmpName | Job         | New_Emp | Salary |
|-------|---------|-------------|---------|--------|
| 1     | Alice   | Analyst     | Yes     | 15000  |
| 2     | Joe     | Salesperson | No      | 10000  |
| 3     | Katy    | Clerk       | No      | 12000  |
| 4     | Annie   | Manager     | Yes     | 25000  |

**Bitmap pour New_Emp :**
- Yes → 1 0 0 1  (lignes 1 et 4)
- No  → 0 1 1 0  (lignes 2 et 3)

**Bitmap pour Job :**
- Analyst     → 1 0 0 0
- Salesperson → 0 1 0 0
- Clerk       → 0 0 1 0
- Manager     → 0 0 0 1

**Requête :** `SELECT * FROM Employé WHERE New_Emp = 'No' AND Job = 'Salesperson';`
Le SGBD effectue un **ET logique** entre les deux bitmaps :
- No :     0 1 1 0
- Sales :  0 1 0 0
- Résultat:0 1 0 0 → seule la ligne 2 est retournée.

**Création d’un index bitmap :**
```sql
CREATE BITMAP INDEX idx_emp_new ON Employé(New_Emp);
```

> **Important examen :** Les index bitmap sont très efficaces pour les requêtes avec plusieurs conditions sur des colonnes de faible cardinalité (entrepôts de données).

### **3.7 Suppression d’un index**

```sql
DROP INDEX nom_index;
```

---

## **4. Les Séquences (Sequences)**

### **4.1 Définition**

Une **séquence** est un objet de base de données qui génère automatiquement des nombres entiers uniques et ordonnés. Elle est souvent utilisée pour créer des clés primaires.

### **4.2 Pseudo-colonnes**

- **NEXTVAL** : incrémente la séquence et retourne la nouvelle valeur.
- **CURRVAL** : retourne la valeur courante de la séquence (nécessite un NEXTVAL préalable dans la session).

**Exemple :**
```sql
SELECT ma_sequence.NEXTVAL FROM dual;  -- retourne 1
SELECT ma_sequence.CURRVAL FROM dual;  -- retourne 1
SELECT ma_sequence.NEXTVAL FROM dual;  -- retourne 2
```

### **4.3 Syntaxe de création**

```sql
CREATE SEQUENCE nom_sequence
[START WITH n]
[INCREMENT BY n]
[MINVALUE n | NOMINVALUE]
[MAXVALUE n | NOMAXVALUE]
[CYCLE | NOCYCLE]
[CACHE n | NOCACHE];
```

**Explication des options :**
- **START WITH** : valeur initiale (par défaut 1).
- **INCREMENT BY** : pas d’incrémentation (peut être négatif pour décroître).
- **MINVALUE** : valeur minimale (par défaut NOMINVALUE = 1 pour asc, -10^26 pour desc).
- **MAXVALUE** : valeur maximale (par défaut NOMAXVALUE = 10^27).
- **CYCLE** : après avoir atteint la limite, la séquence recommence à MINVALUE (pour asc) ou MAXVALUE (pour desc).
- **NOCYCLE** (par défaut) : arrêt avec erreur après dépassement.
- **CACHE** : nombre de valeurs pré-allouées en mémoire pour améliorer les performances (par défaut 20).
- **NOCACHE** : pas de cache.

**Exemples :**
```sql
-- Séquence simple
CREATE SEQUENCE seq_emp;

-- Séquence commençant à 100, incrément de 2
CREATE SEQUENCE seq_dept START WITH 100 INCREMENT BY 2;

-- Séquence descendante
CREATE SEQUENCE seq_desc INCREMENT BY -10 START WITH 100;

-- Séquence avec bornes et cycle
CREATE SEQUENCE seq_cycle MINVALUE 1 MAXVALUE 10 CYCLE;
```

**Comportement de CYCLE :**
- Ascendante : 1,2,...,10, puis 1,2,...
- Descendante : si `INCREMENT BY -1 START WITH 5 MINVALUE 1 MAXVALUE 5 CYCLE` : 5,4,3,2,1,5,4,...

### **4.4 Suppression**

```sql
DROP SEQUENCE nom_sequence;
```

---

## **5. Les Synonymes (Synonyms)**

### **5.1 Définition**

Un **synonyme** est un **alias** pour un objet de base de données (table, vue, séquence, procédure…). Il permet de simplifier l’accès en masquant le nom complet (propriétaire + objet).

### **5.2 Types de synonymes**

- **Privé** : accessible uniquement par son propriétaire.
- **Public** : accessible par tous les utilisateurs.

### **5.3 Syntaxe**

```sql
-- Synonyme privé
CREATE SYNONYM nom_synonyme FOR schema.objet;

-- Synonyme public
CREATE PUBLIC SYNONYM nom_synonyme FOR schema.objet;
```

**Exemples :**
```sql
-- Pour éviter de préfixer par hr.
CREATE SYNONYM emp FOR hr.employees;
SELECT * FROM emp;

-- Synonyme public pour une séquence
CREATE PUBLIC SYNONYM seq_globale FOR system.global_sequence;
```

### **5.4 Utilisation**

Une fois créé, on utilise le synonyme comme le nom de l’objet lui-même.

### **5.5 Suppression**

```sql
DROP SYNONYM nom_synonyme;
DROP PUBLIC SYNONYM nom_synonyme;
```

---

## **6. Les Transactions**

### **6.1 Définition**

Une **transaction** est une suite d’instructions SQL (INSERT, UPDATE, DELETE) qui doivent être exécutées de manière **indivisible** : soit toutes réussissent, soit toutes échouent (tout ou rien). Elle fait passer la base d’un état cohérent à un autre.

### **6.2 Propriétés ACID**

- **Atomicité** : la transaction est une unité atomique (pas de succès partiel).
- **Cohérence** : après la transaction, la base est dans un état cohérent (respect des contraintes).
- **Isolation** : les transactions s’exécutent comme si elles étaient seules, sans interférence.
- **Durabilité** : les modifications validées persistent même en cas de panne.

### **6.3 Commandes de contrôle de transaction**

- **COMMIT** : valide définitivement toutes les modifications de la transaction en cours.
- **ROLLBACK** : annule toutes les modifications depuis le dernier COMMIT (ou depuis un point de sauvegarde).
- **SAVEPOINT** : crée un point de sauvegarde intermédiaire.
- **ROLLBACK TO SAVEPOINT** : annule les modifications jusqu’au point de sauvegarde, mais conserve les modifications antérieures.

### **6.4 Exemple avec SAVEPOINT**

```sql
INSERT INTO T1 VALUES (...);  -- instruction 1
COMMIT;                       -- validation de la transaction 1

UPDATE T2 SET ... ;            -- instruction 2
DELETE FROM T1 WHERE ... ;     -- instruction 3
SAVEPOINT SV1;                 -- point de sauvegarde
UPDATE T1 SET ... ;            -- instruction 4
DELETE FROM T2 WHERE ... ;     -- instruction 5

IF (condition) THEN
    ROLLBACK TO SAVEPOINT SV1; -- annule instructions 4 et 5
END IF;
COMMIT;                        -- valide instructions 2,3 et éventuellement 4,5 si condition fausse
```

**Analyse des transactions :**
- **Transaction 1** : instruction 1 + COMMIT.
- **Transaction 2** : tout ce qui suit jusqu’au prochain COMMIT (instructions 2 à 5 ou 2-3 selon le rollback).


> **Conseil examen :** Bien identifier les bornes des transactions : chaque COMMIT termine la transaction en cours. Un ROLLBACK TO SAVEPOINT n’est pas un COMMIT, il annule seulement une partie.

---

## **Résumé des points clés pour l’examen**

- **Vues** : virtuelles, peuvent être modifiables si elles respectent la règle de la clé préservée. `WITH CHECK OPTION` empêche les insertions hors condition.
- **Vues matérialisées** : copies physiques, rafraîchissables (FAST nécessite un log).
- **Index** : B-tree (valeurs nombreuses) vs Bitmap (peu de valeurs). `CREATE INDEX`, `CREATE BITMAP INDEX`.
- **Séquences** : génèrent des entiers uniques. `NEXTVAL` et `CURRVAL`. Options `CYCLE`, `START WITH`, etc.
- **Synonymes** : alias pour objets. `PUBLIC` pour tous.
- **Transactions** : ACID, `COMMIT`, `ROLLBACK`, `SAVEPOINT`. Savoir compter le nombre de transactions dans un script.

---

Ce guide complet reprend toutes les notions du cours avec des explications détaillées des mots-clés SQL et des conseils pour réussir l’examen. Bonne révision !
