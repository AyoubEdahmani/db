# **Guide Complet d'Architecture et d'Administration d'Oracle Database (Versions 11g - 23ai) - Avec Définitions et Explications Complètes**

## **Introduction : Qu'est-ce qu'Oracle Database ?**

**Définition :** Oracle Database est le système de gestion de base de données relationnelle (SGBDR) le plus utilisé au monde. C'est un système intégré pour gérer, stocker et traiter des données avec une haute fiabilité et sécurité. Il fonctionne sur le principe client-serveur, où les utilisateurs se connectent via un réseau pour accéder aux données stockées sur le serveur.

**Composants de base :**
1. **Instance :** Les programmes, processus et mémoire active en mémoire vive
2. **Base de données :** Les fichiers physiques sur le disque dur
3. **Schémas :** Les utilisateurs et les objets qu'ils possèdent

---

## **1. Composants de Base d'un Serveur Oracle Database**

### **A. L'Instance**

**Définition :** L'instance est un ensemble de structures mémoire et de processus en arrière-plan qui gèrent les fichiers de base de données. C'est la partie exécutive qui s'exécute en mémoire lorsque la base de données est démarrée.

**Caractéristiques principales :**
- Ouvre une seule base de données
- Composée de deux zones mémoire principales : SGA et PGA
- Configurée via le fichier de paramètres (Parameter File)
- Contient des processus en arrière-plan (Background Processes)

**Exemples pratiques :**
```sql
-- Afficher les informations de l'instance
SELECT instance_name, host_name, version, status 
FROM v$instance;

-- Afficher les paramètres de l'instance
SHOW PARAMETER instance_name;
```

**À retenir pour l'examen :**
- L'instance existe en mémoire et disparaît à l'arrêt de la base de données
- Plusieurs instances peuvent fonctionner sur le même serveur (dans le cas de RAC)
- Chaque instance a un identifiant unique appelé SID

### **B. La Base de Données**

**Définition :** La base de données est l'ensemble physique des fichiers stockés sur le disque dur qui contiennent les données et les métadonnées.

**Types de fichiers principaux :**
1. **Fichiers de données (Data Files) :** Contiennent les données réelles des tables et index
2. **Fichiers de contrôle (Control Files) :** Contiennent les informations structurelles sur la base de données
3. **Fichiers de journaux de réapplication (Redo Log Files) :** Enregistrent tous les changements survenant dans la base de données

**Exemples pratiques :**
```sql
-- Vérifier l'état de la base de données
SELECT name, dbid, created, log_mode, database_role 
FROM v$database;

-- Afficher les fichiers de contrôle
SELECT name FROM v$controlfile;
```

---

## **2. Zones Mémoire dans Oracle**

### **A. Zone Globale Système (SGA - System Global Area)**

**Définition :** La SGA est une zone mémoire partagée entre tous les processus Oracle. Elle contient des données et des informations de contrôle pour l'instance. Elle est allouée au démarrage de l'instance et libérée à son arrêt.

#### **1. Cache de Tampons de Base de Données (Database Buffer Cache)**

**Définition :** Zone de cache qui stocke les blocs de données lus depuis les fichiers de données. Réduit les lectures disque.

**Comment ça fonctionne :**
- Quand un utilisateur demande des données, Oracle cherche d'abord dans le cache
- Si les données sont trouvées (Hit), elles sont retournées directement
- Si non trouvées (Miss), lecture depuis le disque et stockage dans le cache

```sql
-- Calculer le taux de réussite du cache
SELECT (1 - (physical_reads / (db_block_gets + consistent_gets))) * 100 "Hit Ratio"
FROM v$buffer_pool_statistics;
```

#### **2. Tampon de Journaux de Réapplication (Redo Log Buffer)**

**Définition :** Zone mémoire circulaire qui stocke toutes les opérations de changement (INSERT, UPDATE, DELETE) avant leur écriture dans les fichiers Redo Log.

**Rôle dans la protection :**
- Garantit la récupération des données en cas de panne
- Est vidé sur disque lors d'un COMMIT
- Sa taille est déterminée par LOG_BUFFER dans le fichier de paramètres

#### **3. Pool Partagé (Shared Pool)**

**Définition :** Zone mémoire contenant des informations partagées entre tous les utilisateurs.

**Ses composants principaux :**
- **Cache de bibliothèque (Library Cache) :** Stocke les plans d'exécution SQL et PL/SQL
- **Cache du dictionnaire (Data Dictionary Cache) :** Stocke les métadonnées (structure des tables, privilèges, utilisateurs)

```sql
-- Afficher la taille du pool partagé
SELECT pool, name, bytes 
FROM v$sgastat 
WHERE pool = 'shared pool';
```

### **B. Zone Globale Programme (PGA - Program Global Area)**

**Définition :** Zone mémoire privée à chaque processus utilisateur. Allouée à la connexion de l'utilisateur et libérée à la fin de la session.

**Composants de la PGA :**
- **Zone de tri (Sort Area) :** Utilisée pour les opérations de tri (ORDER BY, GROUP BY)
- **Zone de hachage (Hash Area) :** Utilisée pour les jointures par hachage
- **Zone de session (Session Area) :** Stocke les informations de session

**À retenir pour l'examen :**
- SGA est partagée entre tous les utilisateurs
- PGA est privée à chaque utilisateur
- Taille SGA déterminée par MEMORY_TARGET ou SGA_TARGET
- Taille PGA déterminée par PGA_AGGREGATE_TARGET

---

## **3. Processus Oracle (Processes)**

### **Types de processus :**

#### **1. Processus Utilisateur (User Processes)**
- S'exécutent sur la machine cliente (comme SQL*Plus)
- Génèrent des requêtes SQL
- N'interagissent pas directement avec le serveur de base de données

#### **2. Processus d'Écoute (Oracle Net Listener)**
- Écoute les demandes de connexion des clients
- Dirige les connexions vers les processus appropriés
- S'exécute sur un port spécifique (par défaut 1521)

#### **3. Processus Serveur (Server Processes)**
- Exécutent les requêtes SQL pour le compte du client
- Peuvent être dédiés (Dedicated) ou partagés (Shared)

#### **4. Processus d'Arrière-Plan (Background Processes) - Les plus importants**

**A. Écrivain de Base de Données (DBWn - Database Writer)**
**Rôle :** Écrit les blocs de données modifiés de la mémoire vers les fichiers de données sur disque.
**S'active quand :**
- Le cache mémoire est plein
- Aux points de contrôle (Checkpoints)
- Toutes les 3 secondes (par défaut)

**B. Écrivain de Journaux (LGWR - Log Writer)**
**Rôle :** Écrit du Redo Log Buffer vers les fichiers Redo Log sur disque.
**S'active quand :**
- Lors d'un COMMIT
- Quand le Redo Log Buffer est plein au ⅓
- Toutes les 3 secondes

**C. Point de Contrôle (CKPT - Checkpoint)**
**Rôle :** Met à jour les points de contrôle dans la base de données pour réduire le temps de récupération.
**Ce qu'il fait :**
- Met à jour les en-têtes des fichiers de données et de contrôle
- Informe DBWn d'écrire sur disque

**D. Archiveur (ARCn - Archiver)**
**Rôle :** Archive les fichiers Redo Log lorsqu'ils sont pleins.
**Important :** Nécessaire seulement en mode ARCHIVELOG

**E. Moniteur Système (SMON - System Monitor)**
**Rôle :** Effectue la récupération de la base de données après un crash système.

**F. Moniteur Processus (PMON - Process Monitor)**
**Rôle :** Surveille et nettoie les processus défaillants.

**Exemple de flux de transaction :**
```sql
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 20;
COMMIT;
```
**Ce qui se passe :**
1. Modification dans le Database Buffer Cache
2. Enregistrement dans le Redo Log Buffer
3. Au COMMIT : LGWR écrit dans les Redo Log Files
4. Plus tard : DBWn écrit dans les Data Files

---

## **4. Fichiers de la Base de Données**

### **A. Fichiers de Données (Data Files)**
**Définition :** Les fichiers physiques qui contiennent les données. Groupés en tablespaces.

### **B. Fichiers de Contrôle (Control Files)**
**Définition :** Fichiers binaires contenant des informations structurelles sur la base de données.
**Informations qu'ils contiennent :**
- Noms et emplacements de tous les fichiers de données et Redo Log
- Numéro SCN courant (System Change Number)
- Informations d'archivage

```sql
-- Afficher les emplacements des fichiers de contrôle
SHOW PARAMETER control_files;
```

**Important :** Vous devez avoir plusieurs copies des fichiers de contrôle sur des disques différents.

### **C. Fichiers de Journaux de Réapplication (Redo Log Files)**
**Définition :** Enregistrent tous les changements dans la base de données pour assurer la récupération.
**Caractéristiques :**
- Fonctionnent de manière circulaire
- Vous devez avoir au moins deux groupes (Groups)
- Chaque groupe contient au moins un membre (Member)

---

## **5. Structure Logique de Stockage**

### **Hiérarchie :**
```
Base de données (Database) → Tablespaces → Segments → Extents → Blocs de données (Data Blocks)
```

### **A. Tablespaces**
**Définition :** Conteneur logique regroupant des fichiers de données liés.

**Tablespaces de base :**
1. **SYSTEM :** Contient le dictionnaire de données
2. **SYSAUX :** Contient des informations système auxiliaires
3. **TEMP :** Pour les fichiers temporaires et opérations nécessitant un tri
4. **UNDO :** Pour la gestion des annulations (Rollbacks)
5. **USERS :** Pour stocker les données des utilisateurs

```sql
-- Créer un nouveau tablespace
CREATE TABLESPACE app_data
DATAFILE '/u01/oradata/app01.dbf' SIZE 100M
AUTOEXTEND ON NEXT 10M MAXSIZE UNLIMITED;
```

### **B. Segments, Extents et Blocs de Données**

**1. Bloc de Données (Data Block) :**
- Plus petite unité de stockage dans Oracle
- Taille par défaut : 8KB
- Déterminée par DB_BLOCK_SIZE

**2. Extent :**
- Ensemble de blocs contigus
- Unité d'allocation de stockage

**3. Segment :**
- Ensemble d'extents
- Représente un objet de base de données (table, index, etc.)

---

## **6. Fichiers de Paramètres (Parameter Files)**

### **A. Types de fichiers de paramètres :**

#### **1. PFILE (Parameter File)**
- Fichier texte modifiable manuellement
- Format : init<SID>.ora
- Les modifications ne persistent qu'après redémarrage

#### **2. SPFILE (Server Parameter File)**
- Fichier binaire géré par Oracle
- Format : spfile<SID>.ora
- Les modifications persistent immédiatement

### **B. Comparaison :**

| Caractéristique | PFILE | SPFILE |
|-----------------|-------|--------|
| Type | Texte | Binaire |
| Modification | Manuelle | Par commande SQL |
| Persistance | Non automatique | Automatique |
| Utilisation | Création BD, dépannage | Gestion quotidienne |

### **C. Paramètres importants :**

```sql
-- Afficher les paramètres importants
SHOW PARAMETER db_name;
SHOW PARAMETER memory_target;
SHOW PARAMETER processes;
SHOW PARAMETER control_files;
```

**À retenir pour l'examen :**
- PFILE peut être converti en SPFILE et vice versa
- Au démarrage, Oracle cherche d'abord SPFILE puis PFILE
- PFILE peut être créé à partir de SPFILE et vice versa

---

## **7. Dictionnaire de Données (Data Dictionary)**

### **Définition :** Le dictionnaire de données est un ensemble de tables et de vues qui stockent des informations sur la base de données. Ce sont les métadonnées qui décrivent la structure de la base de données.

### **Types de vues du dictionnaire :**

#### **1. Vues statiques (Static Views)**
- Commencent par USER_, ALL_, DBA_
- Contiennent des informations sur les objets de la base de données

```sql
-- Vues utilisateur : affichent les objets de l'utilisateur courant
SELECT * FROM USER_TABLES;

-- Vues ALL : affichent les objets accessibles à l'utilisateur courant
SELECT * FROM ALL_TABLES;

-- Vues DBA : affichent tous les objets (nécessite privilège DBA)
SELECT * FROM DBA_TABLES;
```

#### **2. Vues dynamiques (Dynamic Views)**
- Commencent par V$ ou GV$
- Affichent les informations de performance et d'activité courante

```sql
-- Informations sur l'instance
SELECT * FROM V$INSTANCE;

-- Informations sur la base de données
SELECT * FROM V$DATABASE;

-- Informations sur la mémoire
SELECT * FROM V$SGA;
```

### **Rôle du dictionnaire de données :**

1. **À la connexion :** Vérifie la validité du nom d'utilisateur et du mot de passe
2. **À l'exécution d'une requête :** Vérifie l'existence des tables et des privilèges
3. **À l'exécution de DDL :** Met à jour automatiquement le dictionnaire

**Important :** Le dictionnaire de données se trouve dans le tablespace SYSTEM et ne peut être modifié directement par les utilisateurs.

---

## **7. Création et Gestion des Utilisateurs**

### **A. Étapes de Création d'un Nouvel Utilisateur**

#### **1. Choix du Nom d'Utilisateur**
```sql
CREATE USER ali ...;
```

#### **2. Choix de la Méthode d'Authentification**
```sql
-- Authentification par mot de passe
CREATE USER ali IDENTIFIED BY password123;

-- Authentification externe (par système d'exploitation)
CREATE USER ops$ali IDENTIFIED EXTERNALLY;
```

#### **3. Définition des Tablespaces et Quotas**
```sql
CREATE USER ali
IDENTIFIED BY password123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 500M ON users
QUOTA UNLIMITED ON data_app;
```

#### **4. Paramètres Supplémentaires du Compte**
```sql
PASSWORD EXPIRE      -- Changement du mot de passe à la première connexion
ACCOUNT UNLOCK       -- Compte non verrouillé
ACCOUNT LOCK         -- Verrouillage du compte
```

### **B. Exemple Complet de Création d'Utilisateur**
```sql
CREATE USER ali
IDENTIFIED BY monMotDePasse123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 500M ON users
QUOTA UNLIMITED ON data_app
PASSWORD EXPIRE
ACCOUNT UNLOCK;
```

### **C. Modification d'un Utilisateur Existant**
```sql
-- Changement du mot de passe
ALTER USER ali IDENTIFIED BY newPassword;

-- Changement des Tablespaces
ALTER USER scott 
DEFAULT TABLESPACE tablespace2
TEMPORARY TABLESPACE tmp_tablespace2;

-- Changement des quotas
ALTER USER ali
QUOTA 15M ON tablespace1
QUOTA 5M ON tablespace2;

-- Verrouillage/Déverrouillage du compte
ALTER USER ali ACCOUNT LOCK;
ALTER USER ali ACCOUNT UNLOCK;
```

### **D. Suppression d'Utilisateur**
```sql
-- Suppression d'un utilisateur avec schéma vide
DROP USER test;

-- Suppression d'un utilisateur avec tous ses objets
DROP USER test CASCADE;
```

### **E. Requêtes d'Information sur les Utilisateurs**
```sql
-- Afficher les informations des utilisateurs
SELECT * FROM DBA_USERS;
SELECT * FROM DBA_TS_QUOTAS;

-- Afficher les informations d'un utilisateur spécifique
SELECT * FROM DBA_USERS WHERE username = 'ALI';
SELECT tablespace_name, bytes, max_bytes 
FROM DBA_TS_QUOTAS 
WHERE username = 'ALI';
```

---

## **8. Gestion des Privilèges (Privileges)**

### **A. Types de Privilèges**

#### **1. Privilèges Système (System Privileges)**
- Permettent d'exécuter des opérations au niveau système
- Oracle contient environ 127 privilèges système

**Exemples de privilèges système :**
```sql
-- Privilèges de session
GRANT CREATE SESSION TO ali;      -- Permettre la connexion
GRANT ALTER SESSION TO ali;       -- Permettre la modification de session

-- Privilèges sur les tables
GRANT CREATE TABLE TO ali;        -- Créer des tables dans son schéma
GRANT CREATE ANY TABLE TO ali;    -- Créer des tables dans n'importe quel schéma
GRANT ALTER ANY TABLE TO ali;     -- Modifier n'importe quelle table
GRANT DROP ANY TABLE TO ali;      -- Supprimer n'importe quelle table
GRANT SELECT ANY TABLE TO ali;    -- Interroger n'importe quelle table

-- Privilèges sur les Tablespaces
GRANT CREATE TABLESPACE TO ali;
GRANT ALTER TABLESPACE TO ali;
GRANT DROP TABLESPACE TO ali;
GRANT UNLIMITED TABLESPACE TO ali;
```

#### **2. Privilèges Objet (Object Privileges)**
- Permettent d'accéder à des objets spécifiques

**Privilèges disponibles sur les tables :**
- ALTER : Modifier la définition de la table
- DELETE : Supprimer des lignes
- INSERT : Insérer des lignes
- REFERENCES : Créer des contraintes d'intégrité
- SELECT : Interroger
- UPDATE : Mettre à jour

### **B. Attribution des Privilèges**

#### **1. Attribution des Privilèges Système**
```sql
-- Syntaxe générale
GRANT {system_priv | role} TO {user | role | PUBLIC} 
[WITH ADMIN OPTION];

-- Exemples
GRANT CREATE SESSION TO ali;
GRANT CREATE TABLE TO ali;
GRANT CREATE VIEW TO ali;

-- Avec option ADMIN
GRANT CREATE TABLE TO ali WITH ADMIN OPTION;
```

#### **2. Attribution des Privilèges Objet**
```sql
-- Attribuer plusieurs privilèges sur une table
GRANT SELECT, INSERT, UPDATE, DELETE ON SYS.client TO ali;

-- Attribuer le privilège UPDATE sur des colonnes spécifiques
GRANT UPDATE (nom, prenom) ON SYS.client TO ali;
```

### **C. Révocation des Privilèges**

#### **1. Révocation des Privilèges Système**
```sql
REVOKE CREATE SESSION FROM scott;
REVOKE ALTER ANY TABLE FROM PUBLIC;
```

#### **2. Révocation des Privilèges Objet**
```sql
REVOKE DELETE ON Bonus FROM scott;
REVOKE UPDATE ON emp FROM public;
REVOKE ALL ON bonus FROM PUBLIC;
```

### **D. Requêtes sur les Privilèges**
```sql
-- Afficher les privilèges système attribués
SELECT * FROM DBA_SYS_PRIVS ORDER BY grantee, privilege;

-- Afficher les privilèges objet attribués
SELECT * FROM DBA_TAB_PRIVS WHERE grantee = 'ALI';

-- Afficher les privilèges dans la session courante
SELECT * FROM SESSION_PRIVS;
```

---

## **9. Gestion des Rôles (Roles)**

### **A. Concept des Rôles**
- Regroupement de privilèges pour faciliter la gestion
- Un rôle peut contenir d'autres rôles

### **B. Création de Rôles**

#### **1. Création d'un rôle sans mot de passe**
```sql
CREATE ROLE rl_etudiant;
```

#### **2. Création d'un rôle avec mot de passe**
```sql
CREATE ROLE rl_admin_secu IDENTIFIED BY secu_pass;
```

#### **3. Création d'un rôle externe (External Role)**
```sql
CREATE ROLE app_role IDENTIFIED EXTERNALLY;
```

### **C. Attribution de Privilèges aux Rôles**
```sql
-- Création d'un rôle et ajout de privilèges
CREATE ROLE compta;
GRANT SELECT, INSERT, UPDATE, DELETE ON client TO compta;
GRANT SELECT, INSERT, UPDATE ON fournisseur TO compta;
```

### **D. Attribution de Rôles aux Utilisateurs**
```sql
GRANT compta TO ali;
```

### **E. Rôles Standards**

#### **1. Rôle CONNECT**
```sql
GRANT CONNECT TO ali;
```
**Privilèges inclus dans CONNECT :**
- CREATE SESSION

#### **2. Rôle RESOURCE**
```sql
GRANT RESOURCE TO ali;
```
**Privilèges inclus dans RESOURCE :**
- CREATE TABLE, CREATE SEQUENCE, CREATE PROCEDURE, etc.

#### **3. Rôle DBA**
```sql
GRANT DBA TO ali;
```
**Accorde tous les privilèges système avec ADMIN OPTION**

### **F. Requêtes sur les Rôles**
```sql
-- Afficher les rôles attribués à un utilisateur
SELECT * FROM DBA_ROLE_PRIVS WHERE grantee = 'ALI';

-- Afficher le contenu d'un rôle spécifique
SELECT * FROM DBA_SYS_PRIVS WHERE grantee = 'CONNECT';
SELECT * FROM DBA_SYS_PRIVS WHERE grantee = 'RESOURCE';
SELECT * FROM DBA_SYS_PRIVS WHERE grantee = 'DBA' ORDER BY PRIVILEGE;

-- Afficher les rôles dans la session courante
SELECT * FROM SESSION_ROLES;
```

### **G. Suppression de Rôles**
```sql
DROP ROLE rl_admin_secu;
```

---

## **10. Les Profils (Profiles)**


---

# 🔹 Qu’est-ce qu’un **Profile** ?

Un **Profile** dans **Oracle Database** est un **ensemble de restrictions appliquées aux comptes utilisateurs**, par exemple :

* Durée de vie du mot de passe
* Nombre de tentatives de connexion échouées
* Ressources autorisées (CPU, sessions, etc.)

**Objectif :** sécuriser les comptes et contrôler les ressources utilisées par chaque utilisateur.

---

# 🔹 Création d’un Profile

Exemple pratique :

```sql
CREATE PROFILE etudiants_profile
LIMIT 
  FAILED_LOGIN_ATTEMPTS 5        -- nombre de tentatives échouées avant verrouillage
  PASSWORD_LIFE_TIME 90          -- durée de validité du mot de passe (en jours)
  PASSWORD_REUSE_TIME 180        -- délai minimum avant de réutiliser un ancien mot de passe
  PASSWORD_REUSE_MAX 5           -- nombre maximum de réutilisations autorisées
  PASSWORD_LOCK_TIME 1;           -- durée du verrouillage du compte (en jours)
```

---

# 🔹 Explication des paramètres

* `FAILED_LOGIN_ATTEMPTS` → verrouille le compte si l’utilisateur échoue trop de fois
* `PASSWORD_LIFE_TIME` → durée avant que le mot de passe doive être changé
* `PASSWORD_REUSE_TIME` → interdit de réutiliser un ancien mot de passe avant ce délai
* `PASSWORD_REUSE_MAX` → limite le nombre de réutilisations d’un mot de passe
* `PASSWORD_LOCK_TIME` → durée du verrouillage après trop de tentatives échouées

---

# 🔹 Associer un Profile à un utilisateur

Après création, il faut l’appliquer à un utilisateur :

```sql
ALTER USER ali PROFILE etudiants_profile;
```

➡️ Maintenant, l’utilisateur `ali` sera soumis aux règles définies dans le profile.

---

✅ **Résumé simple à retenir :**

* **Profile = ensemble de règles pour un compte**
* Sert à **protéger les comptes** et **gérer les ressources**
* Création : `CREATE PROFILE` → application : `ALTER USER … PROFILE`

---

#### **3. Exemple Complet**
```sql
CREATE PROFILE pf_agent LIMIT
SESSIONS_PER_USER 2
CPU_PER_SESSION UNLIMITED
CPU_PER_CALL 1000
COMPOSITE_LIMIT 20000
PRIVATE_SGA 32K
FAILED_LOGIN_ATTEMPTS 3
PASSWORD_LIFE_TIME 90
PASSWORD_REUSE_TIME 30
PASSWORD_REUSE_MAX 5;
```

### **D. Attribution de Profil aux Utilisateurs**
```sql
-- À la création
CREATE USER rackham
IDENTIFIED BY Ierouge
PROFILE pf_secretaire;

-- À la modification
ALTER USER rackham PROFILE pf_agent;
```

### **E. Modification et Suppression de Profils**
```sql
-- Modification d'un profil (requiert ALTER PROFILE)
ALTER PROFILE pf_secretaire LIMIT
SESSIONS_PER_USER 3
IDLE_TIME 60;

-- Suppression d'un profil
DROP PROFILE pf_secretaire CASCADE;
```

### **F. Requêtes sur les Profils**
```sql
-- Afficher tous les profils
SELECT profile, resource_name, limit 
FROM dba_profiles
ORDER BY profile, resource_name;

-- Afficher un profil spécifique
SELECT resource_name, limit 
FROM dba_profiles
WHERE profile = 'PF_SECRETAIRE';
```

---

## **12. Création d'une Base de Données**

### **Méthodes de création de base de données :**

#### **1. Utilisation de DBCA (outil graphique)**
- Database Configuration Assistant
- Interface graphique facile

#### **2. Manuellement avec SQL*Plus**

### **Étapes manuelles :**

1. **Configurer les variables d'environnement :**
```bash
export ORACLE_SID=orcl
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
```

2. **Créer le fichier init.ora :**
```ini
db_name='orcl'
memory_target=1G
control_files=('/u01/oradata/orcl/control01.ctl',
               '/u02/oradata/orcl/control02.ctl')
```

3. **Démarrer l'instance en mode NOMOUNT :**
```sql
STARTUP NOMOUNT PFILE='/u01/app/oracle/admin/orcl/pfile/init.ora';
```

4. **Exécuter CREATE DATABASE :**
```sql
CREATE DATABASE orcl
LOGFILE GROUP 1 ('/u01/oradata/orcl/redo01.log') SIZE 100M,
        GROUP 2 ('/u01/oradata/orcl/redo02.log') SIZE 100M
DATAFILE '/u01/oradata/orcl/system01.dbf' SIZE 500M
SYSAUX DATAFILE '/u01/oradata/orcl/sysaux01.dbf' SIZE 250M
UNDO TABLESPACE undotbs1 DATAFILE '/u01/oradata/orcl/undotbs01.dbf' SIZE 200M
DEFAULT TABLESPACE users DATAFILE '/u01/oradata/orcl/users01.dbf' SIZE 100M
DEFAULT TEMPORARY TABLESPACE temp TEMPFILE '/u01/oradata/orcl/temp01.dbf' SIZE 100M
CHARACTER SET AL32UTF8;
```

5. **Exécuter les scripts de création du dictionnaire :**
```sql
@?/rdbms/admin/catalog.sql
@?/rdbms/admin/catproc.sql
```

---

## **13. Démarrage et Arrêt de la Base de Données**

### **Phases de démarrage :**

#### **1. NOMOUNT :**
```sql
STARTUP NOMOUNT;
```
- Démarre seulement l'instance
- Lit le fichier de paramètres
- Alloue la SGA et démarre les processus en arrière-plan

#### **2. MOUNT :**
```sql
STARTUP MOUNT;
-- Ou
ALTER DATABASE MOUNT;
```
- Lit les fichiers de contrôle
- Connaît la structure de la base de données

#### **3. OPEN :**
```sql
STARTUP OPEN;
-- Ou
ALTER DATABASE OPEN;
```
- Ouvre les fichiers de données et Redo Logs
- Base de données prête à l'utilisation

### **Modes d'arrêt :**

#### **1. NORMAL (par défaut) :**
```sql
SHUTDOWN NORMAL;
```
- N'autorise pas de nouvelles connexions
- Attend que tous les utilisateurs se déconnectent
- Ne nécessite pas de récupération au démarrage suivant

#### **2. TRANSACTIONAL :**
```sql
SHUTDOWN TRANSACTIONAL;
```
- Attend la fin des transactions en cours

#### **3. IMMEDIATE :**
```sql
SHUTDOWN IMMEDIATE;
```
- Annule les sessions en cours
- Effectue un rollback des transactions incomplètes
- Ne nécessite pas de récupération

#### **4. ABORT :**
```sql
SHUTDOWN ABORT;
```
- Immédiat, n'attend rien
- Nécessite une récupération au démarrage suivant
- Utilisé seulement en cas d'urgence

---

## **14. Oracle 23ai et Versions Récentes**

### **Nouvelles fonctionnalités :**
- **Bases de données enfichables (Pluggable Databases) :** Permettent d'exécuter plusieurs bases de données dans une seule instance
- **Gestion améliorée de la mémoire :** Allocation dynamique plus efficace
- **Fonctionnalités de sécurité avancées :** Chiffrement, audit, meilleure protection

### **Limitations de la version gratuite :**
- SID fixe : FREE
- Impossible de changer le SID
- Impossible de créer plusieurs CDBs
- Ressources limitées

---

## **15. Bonnes Pratiques**

### **A. Gestion de la mémoire :**
1. Ajustez SGA_SIZE et PGA_AGGREGATE_TARGET selon la mémoire du serveur
2. Surveillez Buffer Cache Hit Ratio (doit être > 90%)
3. Ajustez Shared Pool pour réduire les re-parses des requêtes

### **B. Gestion du stockage :**
1. Séparez les fichiers de données, Redo Logs et Archive Logs sur des disques différents
2. Utilisez ASM pour la gestion automatique du stockage
3. Implémentez des sauvegardes régulières avec RMAN

### **C. Sécurité :**
1. **Principe du moindre privilège :** Accordez seulement les privilèges nécessaires
2. Utilisez les rôles au lieu d'accorder les privilèges directement
3. Implémentez des profils pour contrôler les ressources et mots de passe
4. Auditez les activités sensibles

### **D. Performance :**
1. Ajustez les paramètres mémoire selon la charge de travail
2. Utilisez AWR pour l'analyse des performances
3. Utilisez SQL Tuning Advisor pour optimiser les requêtes
4. Surveillez les événements d'attente (Wait Events)

---

## **16. Termes Importants pour l'Examen**

1. **Instance :** Composants en mémoire (SGA + Background Processes)
2. **Base de données :** Fichiers sur disque
3. **SGA :** Zone mémoire partagée
4. **PGA :** Zone mémoire privée
5. **Tablespace :** Conteneur logique pour un groupe de fichiers de données
6. **Segment :** Objet de base de données (table, index)
7. **Extent :** Ensemble de blocs contigus
8. **Data Block :** Plus petite unité de stockage
9. **Redo Log :** Enregistre les changements pour la récupération
10. **Control File :** Contient les informations structurelles de la base de données
11. **Data Dictionary :** Métadonnées de la base de données
12. **Privilège :** Droit d'exécuter une action
13. **Rôle :** Groupe de privilèges
14. **Profil :** Limites sur les ressources et mots de passe
15. **Commit :** Rend les changements permanents
16. **Rollback :** Annule les changements
17. **Checkpoint :** Point de synchronisation entre mémoire et disque

---

## **17. Questions Fréquentes à l'Examen**

1. **Quelle est la différence entre Instance et Database ?**
   - Instance : En mémoire, temporaire
   - Database : Sur disque, permanente

2. **Quels sont les composants de la SGA ?**
   - Database Buffer Cache
   - Redo Log Buffer  
   - Shared Pool (Library Cache + Data Dictionary Cache)
   - Java Pool, Large Pool, Streams Pool

3. **Quel est le rôle de LGWR et DBWn ?**
   - LGWR : Écrit Redo Log Buffer vers Redo Log Files
   - DBWn : Écrit Database Buffer Cache vers Data Files

4. **Quelles sont les phases de démarrage d'une base de données ?**
   - NOMOUNT → MOUNT → OPEN

5. **Quelle est la différence entre System Privileges et Object Privileges ?**
   - System : Opérations au niveau système
   - Object : Accès à des objets spécifiques

6. **Qu'est-ce que le Data Dictionary ?**
   - Ensemble des tables et vues contenant les métadonnées de la base de données

---

Ce guide couvre tous les concepts fondamentaux dont vous avez besoin pour comprendre et administrer Oracle Database, en mettant l'accent sur les points importants pour les examens. Chaque concept est expliqué avec des exemples pratiques et des définitions claires.
