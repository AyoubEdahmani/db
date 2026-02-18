# **Guide Complet sur les Bases de Données Distribuées (Oracle)**
## **Définitions, Concepts, Fragmentation, Réplication et Administration**
### **Avec explication détaillée des mots-clés SQL et conseils pour l’examen**

---

## **Introduction : Problématique et Motivation**

### **Pourquoi une base de données centralisée pose-t-elle problème ?**

| **Avantages** | **Inconvénients** |
|---------------|-------------------|
| Un seul administrateur | Panne du site central ⇒ tout est indisponible |
| Un seul SGBD | Problème réseau = indisponibilité |
| Maintenance centralisée | Coût de communication et de transfert |
| Mises à jour faciles | Charge élevée sur le serveur central |

### **Pourquoi une base complètement répliquée sur chaque site n’est pas idéale ?**

| **Avantages** | **Inconvénients** |
|---------------|-------------------|
| Données locales donc accès rapide | Problème de mise à jour et d’incohérence |
| Traitement local | Plusieurs administrateurs |
| Réduction des coûts de communication | Redondance des données |
| | Charge sur le serveur central |

**Objectif :** Construire une base qui répond aux besoins des utilisateurs distants tout en rendant les données disponibles partout, de manière cohérente et efficace.

---

## **1. Qu’est-ce qu’une Base de Données Distribuée ?**

**Définition :**  
Une **base de données distribuée** est un ensemble de bases de données *logiquement liées*, réparties sur plusieurs nœuds interconnectés par un réseau. Chaque nœud est géré de manière autonome par un SGBD local.

- **Base locale** : celle à laquelle l’utilisateur est directement connecté.
- **Base distante** : accessible via le réseau à partir de la base locale.

**Objectif :** Donner à l’utilisateur l’illusion d’une base unique et cohérente, malgré la distribution physique.

### **Rôle d’un SGBD Réparti (ex: Oracle)**

Un SGBD distribué assure la transparence de la répartition grâce à :
- **Dictionnaire des données réparties**
- **Traitement des requêtes réparties**
- **Gestion de transactions réparties**
- **Gestion de la cohérence et de la sécurité**

---

## **2. Avantages des Bases de Données Distribuées**

1. **Placement des données** : stockage près du lieu d’utilisation → réduction des échanges réseau.
2. **Traitement local** : amélioration des performances et autonomie des sites.
3. **Fiabilité accrue** : le système continue à fonctionner même si un site tombe en panne.
4. **Sécurité et performance** : contrôle d’accès plus fin, traitements parallèles.
5. **Couplage de données entre institutions** : partage de données tout en conservant l’autonomie.
6. **Partage des responsabilités** : administration décentralisée.
7. **Construction progressive et disponibilité** : ajout facile de nouveaux sites, réplication pour haute disponibilité.

### **Cas d’utilisation typiques**
- Banques (agences réparties)
- Entreprises industrielles (usines multiples)
- Systèmes de réservation (compagnies aériennes)

---

## **3. Lien avec l’architecture Multitenant d’Oracle**

Oracle **Multitenant** permet d’avoir une **CDB** (Container Database) contenant plusieurs **PDB** (Pluggable Databases).

- Chaque PDB peut participer à une base distribuée.
- Les **DB links** peuvent être créés :
  - entre deux PDB de la même CDB,
  - entre PDB de CDB différentes,
  - entre une PDB et une base non-multitenant.

**Important :** La distribution se fait au niveau des PDB, pas directement au niveau CDB.

### Exemple d’architecture

```
Serveur 1
  CDB1
    PDB Scolarité
    PDB Finance

Serveur 2
  CDB2
    PDB Personnel
    PDB Bibliothèque
```

Chaque PDB gère un domaine fonctionnel et communique via des DB links.

---

## **4. Conception d’une Base de Données Répartie**

### **4.1 Décomposition d’un schéma**

- **Schéma global** : vision unifiée de toute la base distribuée, perçue comme une seule base logique.
- **Schémas locaux** : définissent la structure des données stockées sur chaque site physique.
- **Bases de données physiques** (BD1, BD2, …) : instances réelles hébergées sur des serveurs distincts.

### **4.2 Étapes de conception**

1. **Normalisation**
2. **Fragmentation** (horizontale, verticale, mixte)
3. **Duplication (réplication)**

---

## **5. Fragmentation**

La fragmentation consiste à découper une relation (table) en fragments répartis sur différents sites.

---

### **5.1 Fragmentation horizontale**

**Définition :** Division d’une relation en sous-ensembles de lignes à l’aide d’une sélection (`WHERE`).  
Chaque fragment contient un sous-ensemble de lignes répondant à une condition.

**Exemple :** Table `Client`

| nclient | nom      | ville |
|---------|----------|-------|
| C1      | alamioudi| Fès   |
| C2      | saoudi   | Rabat |
| C3      | saber    | Casa  |
| C4      | anouari  | Fès   |

#### **Requête de création d’un fragment :**
```sql
CREATE TABLE Client1 AS
SELECT * FROM Client WHERE ville = 'Fès';
```

**Explication des mots-clés :**
- **CREATE TABLE** : crée une nouvelle table en base de données.
- **Client1** : nom de la table créée (le fragment).
- **AS** : introduit la requête qui va fournir les données à insérer dans la nouvelle table.
- **SELECT * FROM Client** : sélectionne toutes les colonnes de la table `Client`.
- **WHERE ville = 'Fès'** : filtre les lignes dont la colonne `ville` vaut `'Fès'`. C’est la condition de fragmentation horizontale.

> **Important pour l’examen :** La fragmentation horizontale correspond à une opération de **sélection** (restriction) en algèbre relationnelle. Elle permet de répartir les lignes d’une table sur plusieurs sites selon un critère (ex: région, agence).

#### **Reconstruction avec une vue :**
```sql
CREATE VIEW Client AS
SELECT * FROM Client1
UNION
SELECT * FROM Client2;
```

**Explication :**
- **CREATE VIEW** : crée une vue (table virtuelle) nommée `Client`. La vue n’existe pas physiquement, elle donne une vision unifiée des fragments.
- **SELECT * FROM Client1** : récupère toutes les lignes du fragment `Client1`.
- **UNION** : fusionne les résultats de deux requêtes en éliminant les doublons (si on veut tous les garder, on utiliserait `UNION ALL`).
- **SELECT * FROM Client2** : récupère toutes les lignes du fragment `Client2`.

> **Important :** `UNION` exige que les deux requêtes aient le même nombre de colonnes et des types compatibles. La vue permet aux utilisateurs d’interroger `Client` comme si la table était entière, sans se soucier de la fragmentation.

#### **Fragmentation par semi-jointure :**
```sql
CREATE TABLE Cde1 AS
SELECT cde.* FROM Cde, Client1
WHERE Cde.nclient = Client1.nclient;
```

**Explication :**
- **SELECT cde.*** : sélectionne toutes les colonnes de la table `Cde` (alias `cde`).
- **FROM Cde, Client1** : effectue un produit cartésien entre `Cde` et `Client1` (jointure implicite).
- **WHERE Cde.nclient = Client1.nclient** : condition de jointure. On ne garde que les commandes dont le client est présent dans le fragment `Client1`.

> **Important :** Cette technique permet de créer un fragment de `Cde` associé aux clients d’un site particulier. On parle de **fragmentation horizontale dérivée**.

---

### **5.2 Fragmentation verticale**

**Définition :** Division d’une relation en sous-ensembles de colonnes (attributs) à l’aide d’une projection.  
Chaque fragment contient une partie des colonnes, avec répétition de la clé pour permettre la reconstruction.

**Exemple :** Table `Cde`

| ncde | nclient | produit | qté |
|------|---------|---------|-----|
| D1   | C1      | P1      | 10  |
| D2   | C1      | P2      | 20  |
| D3   | C2      | P3      | 5   |
| D4   | C4      | P4      | 10  |

#### **Création des fragments verticaux :**
```sql
-- Fragment 1 : ncde, nclient
CREATE TABLE Cde1 AS
SELECT ncde, nclient FROM Cde;

-- Fragment 2 : ncde, produit, qté
CREATE TABLE Cde2 AS
SELECT ncde, produit, qté FROM Cde;
```

**Explication :**
- **SELECT ncde, nclient** : ne sélectionne que les colonnes spécifiées. C’est une **projection**.
- **FROM Cde** : source des données.

> **Important pour l’examen :** La fragmentation verticale correspond à des **projections** en algèbre relationnelle. La clé primaire (ou un identifiant unique) doit apparaître dans tous les fragments pour pouvoir reconstruire la table originale par jointure.

#### **Reconstruction par jointure :**
```sql
SELECT Cde1.ncde, Cde1.nclient, Cde2.produit, Cde2.qté
FROM Cde1, Cde2
WHERE Cde1.ncde = Cde2.ncde;
```

**Explication :**
- **SELECT** avec colonnes des deux fragments.
- **FROM Cde1, Cde2** : produit cartésien.
- **WHERE Cde1.ncde = Cde2.ncde** : jointure sur la clé commune.

> **Important :** La reconstruction se fait par **jointure naturelle** (ou équijoin) sur la clé.

---

### **5.3 Fragmentation mixte**

Combinaison des fragmentations horizontale et verticale.

**Exemple :** On veut séparer les employés jeunes et matures, et en plus stocker leur salaire à part.

```sql
-- Fragment horizontal jeunes + projection
CREATE TABLE Emp_jeune AS
SELECT NumEmp, Nom, Prenom, Age, Service
FROM Employe
WHERE Age < 35;

-- Fragment horizontal matures + projection
CREATE TABLE Emp_mature AS
SELECT NumEmp, Nom, Prenom, Age, Service
FROM Employe
WHERE Age >= 35;

-- Fragment vertical (salaire, ville) pour tous
CREATE TABLE Emp_salaire AS
SELECT NumEmp, Salaire, Ville FROM Employe;
```

**Explication :**
- Les deux premières requêtes sont des fragmentations horizontales avec projection (on ne prend que certaines colonnes).
- La troisième est une fragmentation verticale.

#### **Recomposition :**
```sql
SELECT *
FROM (SELECT * FROM Emp_jeune UNION SELECT * FROM Emp_mature) E,
     Emp_salaire S
WHERE E.NumEmp = S.NumEmp;
```

**Explication :**
- La sous-requête `(SELECT * FROM Emp_jeune UNION SELECT * FROM Emp_mature)` reconstitue la table horizontale complète par union.
- Ensuite, on joint ce résultat avec le fragment `Emp_salaire` sur `NumEmp`.

> **Important :** La fragmentation mixte permet une distribution très fine des données, mais complexifie la reconstruction.

---

## **6. Réplication**

**Définition :** La réplication consiste à dupliquer les données (partiellement ou totalement) sur plusieurs sites.  
Les copies sont appelées **vues matérialisées** (materialized views) ou **snapshots** (ancienne terminologie).

### **Avantages de la réplication**
- Disponibilité accrue
- Amélioration des performances (accès local)
- Réduction de la charge réseau
- Décharge des sites maîtres
- Sécurité (accès alternatifs, sous-ensembles)

### **Mécanisme**
- **Tables maîtres** : sources originales.
- **Vues matérialisées** : copies (totales ou partielles) stockées physiquement.
- Le SGBD assure :
  - **Convergence** des copies vers un état cohérent.
  - **Transparence** pour les utilisateurs.
  - **Diffusion des mises à jour** et **rafraîchissement** des vues.

---

## **7. Liens de base de données (Database Links)**

Un **Database Link** est un canal de communication logique entre deux bases de données Oracle.

### **Création d’un lien**
```sql
CREATE [PUBLIC] DATABASE LINK nom_du_lien
CONNECT TO utilisateur_distant IDENTIFIED BY mot_de_passe_distant
USING 'nom_du_service_BD_distante';
```

**Explication des mots-clés :**
- **CREATE DATABASE LINK** : commande de création d’un lien de base de données.
- **PUBLIC** (optionnel) : si présent, le lien est accessible à tous les utilisateurs de la base locale ; sinon, il est privé (créé dans le schéma de l’utilisateur courant).
- **nom_du_lien** : identifiant du lien (ex: `lien_rabat`).
- **CONNECT TO** : spécifie l’utilisateur distant auquel se connecter.
- **IDENTIFIED BY** : mot de passe de cet utilisateur distant.
- **USING** : chaîne de connexion (alias TNS ou adresse complète) pointant vers la base distante.

**Exemple :**
```sql
CREATE DATABASE LINK lien_rabat
CONNECT TO userrabat IDENTIFIED BY userrabat
USING 'rabat';
```

> **Important pour l’examen :** Un DB link est un objet du dictionnaire. Il permet d’accéder aux objets distants en suffixant `@nom_du_lien`. Par exemple : `SELECT * FROM table@lien_rabat;`.

### **Utilisation**
```sql
SELECT * FROM table_distante@lien_rabat;
```
- `@lien_rabat` : indique que la table se trouve sur la base distante accessible via ce lien.

### **Suppression**
```sql
DROP [PUBLIC] DATABASE LINK lien_rabat;
```

### **Vues pour consulter les liens**
- `USER_DB_LINKS` : liens créés par l’utilisateur courant.
- `ALL_DB_LINKS` : liens accessibles à l’utilisateur courant.
- `DBA_DB_LINKS` : tous les liens (privilège DBA requis).

---

## **8. Vues Matérialisées et Snapshots**

Une **vue matérialisée** (ou **snapshot**) est une copie physique de données distantes, stockée localement. Elle peut être rafraîchie périodiquement.

### **Syntaxe de création**
```sql
CREATE MATERIALIZED VIEW nom_vue
REFRESH [FAST | COMPLETE | FORCE]
[NEXT date_prochaine_rafraichissement]
AS requête_SQL;
```

**Explication des mots-clés :**
- **CREATE MATERIALIZED VIEW** : crée un objet physique contenant le résultat de la requête.
- **REFRESH** : définit la méthode de rafraîchissement.
  - **FAST** : rafraîchissement incrémental (seules les modifications depuis le dernier rafraîchissement sont appliquées). Nécessite des **logs de vue matérialisée** sur la table maître.
  - **COMPLETE** : recréation intégrale de la vue (lourde).
  - **FORCE** : essaie FAST, sinon COMPLETE.
- **NEXT** : planification automatique du prochain rafraîchissement (utilise une date, souvent `SYSDATE + intervalle`).
- **AS requête_SQL** : la requête qui définit les données à copier. Peut contenir des jointures, des sélections, etc.

**Exemple 1 :** Copie de la table `Employee` depuis la base distante `Casa`, rafraîchie rapidement tous les 7 jours.
```sql
CREATE MATERIALIZED VIEW EmpCasa
REFRESH FAST NEXT SYSDATE + 7
AS SELECT * FROM Employee@Casa;
```

**Explication supplémentaire :**
- `SYSDATE` : fonction retournant la date et l’heure courantes du serveur.
- `+ 7` : ajoute 7 jours. Donc prochain rafraîchissement dans 7 jours.

**Exemple 2 (snapshot) :** Jointure entre deux tables distantes, rafraîchie toutes les 3 minutes.
```sql
CREATE SNAPSHOT PersDepts
REFRESH NEXT SYSDATE + 1/480   -- 1/480 jour = 3 minutes
AS SELECT Nom, Prenom, NomDept
FROM Personnes@oracle8i P, Departements@oracle8i D
WHERE P.NoDept = D.NoDept
AND NomDept != 'AAA';
```

**Explication :**
- `1/480` de jour : car 1 jour = 1440 minutes, 1440/480 = 3 minutes.
- La requête fait une jointure entre deux tables distantes (`Personnes` et `Departements` situées sur une base `oracle8i`) et filtre les départements nommés 'AAA'.
- `SNAPSHOT` est un synonyme historique de `MATERIALIZED VIEW` (Oracle 8i, 9i). Aujourd’hui on utilise `MATERIALIZED VIEW` mais le mot `SNAPSHOT` fonctionne encore.

> **Important pour l’examen :** Une vue matérialisée améliore les performances des requêtes distantes en stockant localement les données. Le choix du mode de rafraîchissement dépend de la tolérance à la fraîcheur des données et de la charge réseau.

---

## **9. Exemple de Schéma Global pour une Base Distribuée**

**Schéma global :**
- `Produit (NoProd, DesProd, Prix, NoFour)`
- `Fournisseur (NoFour, NomFour, VilleFour)`
- `Client (NoCli, NomCli, VilleCli)`
- `Représentants (NoRep, NomRep, VilleRep)`
- `Commande (NoProd, NoCli, Date, Qte, NoRep)`

**Schéma de placement (fragmentation) :**
- `Client` = `Client1 @ Site1` ∪ `Client2 @ Site2`
- `Cde` = `Cde @ Site3`

Cela signifie que les clients sont répartis sur deux sites selon un critère (ex: région), et les commandes sont centralisées sur un site spécifique.

---

## **Résumé des Points Clés**

- Une **base de données distribuée** est un ensemble de bases locales reliées par un réseau, apparaissant comme une base unique.
- La **fragmentation** (horizontale, verticale, mixte) permet de découper les données pour les répartir.
  - Horizontale : `SELECT ... WHERE condition`
  - Verticale : `SELECT colonnes ...` (projection)
  - Mixte : combinaison des deux.
- La **réplication** (via vues matérialisées) améliore la disponibilité et les performances.
  - `CREATE MATERIALIZED VIEW ... REFRESH ... AS ...`
- Les **Database Links** (`CREATE DATABASE LINK`) sont essentiels pour relier les bases entre elles.
- L’architecture **Multitenant** d’Oracle s’intègre parfaitement avec la distribution (chaque PDB peut être un nœud).

---

**Note pour l’examen :** 
- Maîtrisez les définitions des différents types de fragmentation.
- Sachez écrire les requêtes de création de fragments et de reconstruction (vues, unions, jointures).
- Comprenez le rôle des DB links et des vues matérialisées.
- Connaissez les mots-clés : `CREATE TABLE AS`, `UNION`, `CREATE VIEW`, `CREATE DATABASE LINK`, `CREATE MATERIALIZED VIEW`, `REFRESH FAST`, `NEXT`, `SYSDATE`, etc.
- Sachez expliquer les avantages et inconvénients des bases distribuées.

Ce guide reprend l’essentiel du cours avec des explications détaillées de chaque mot-clé SQL pour vous préparer efficacement à l’examen.
