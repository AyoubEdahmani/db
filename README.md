# **Guide Complet de l'Architecture et de l'Administration d'Oracle Database (Versions 11g - 23ai)**

## **1. Composants de Base d'un Serveur Oracle Database**

### **A. Structure Fondamentale**
Un serveur Oracle est composé de trois éléments principaux :
- **Instance (Instance)** : Ensemble de processus et de zones mémoire
- **Database (Base de données)** : Ensemble de fichiers sur disque
- **Schemas (Schémas)** : Équivalents aux utilisateurs dans le système

### **B. L'Instance**
```sql
-- Afficher les paramètres de l'instance
SHOW PARAMETER INSTANCE;

-- Afficher les détails de l'instance
DESC V$INSTANCE;
SELECT * FROM V$INSTANCE;
```
**Caractéristiques de l'instance :**
- Ouvre une seule base de données
- Contient deux zones mémoire principales : SGA et PGA
- Configurée via le fichier de paramètres (Parameter File)

---

## **2. Zones Mémoire dans Oracle**

### **A. System Global Area (SGA)**
**Caractéristiques :**
- Zone mémoire partagée entre tous les processus de l'instance
- Allouée au démarrage de l'instance et libérée à l'arrêt
- Sa taille est déterminée par le fichier de paramètres

**Composants principaux de la SGA :**

#### **1. Database Buffer Cache**
```sql
-- Exemple : fonctionnement du Database Buffer Cache
SELECT nom, salaire FROM EMPLOYES WHERE id = 100;
```
**Processus d'exécution :**
1. Recherche des données dans le cache mémoire d'abord
2. Si absentes, lecture depuis le disque et stockage dans le cache
3. Utilisation du cache pour les requêtes suivantes

#### **2. Redo Log Buffer**
```sql
-- Exemple : fonctionnement du Redo Log Buffer
UPDATE employees SET salaire = 2000 WHERE id = 1;
COMMIT;
```
**Processus d'enregistrement :**
1. Écriture des modifications dans le Redo Log Buffer
2. Au COMMIT : écriture du contenu dans les Redo Log Files
3. Garantie de la durabilité (Durability) en cas de panne

#### **3. Shared Pool**
**Ses composants :**
- **Library Cache** : Stocke les plans d'exécution SQL/PLSQL
- **Data Dictionary Cache** : Stocke les métadonnées (tables, colonnes, privilèges)

```sql
-- Afficher la taille du Shared Pool
SELECT component, current_size, min_size, user_specified_size 
FROM v$sga_dynamic_components 
WHERE component = 'shared pool';

-- Afficher les infos du Data Dictionary Cache
SELECT * FROM v$sgastat WHERE name LIKE 'row cache%';
```

#### **4. Autres zones mémoire dans la SGA**
- **Java Pool** : Pour l'utilisation par la machine virtuelle Java
- **Large Pool** : Pour les opérations volumineuses (sauvegarde, restauration)
- **Streams Pool** : Pour la fonctionnalité Streams

```sql
-- Afficher les tailles des zones mémoire
SELECT component, current_size, min_size, max_size 
FROM v$sga_dynamic_components 
WHERE component IN ('java pool', 'streams pool');
```

### **B. Program Global Area (PGA)**
**Caractéristiques :**
- Zone mémoire privée à chaque processus utilisateur
- Allouée lors de la connexion de l'utilisateur et libérée à la fin de la session

```sql
-- Afficher les paramètres de la PGA
SELECT name, value FROM v$parameter 
WHERE name IN ('sort_area_size', 'hash_area_size', 
               'bitmap_merge_area_size', 'create_bitmap_area_size');

-- Afficher les statistiques de tri
SELECT name, value FROM v$sysstat 
WHERE name in ('sorts (memory)', 'sorts (disk)');
```

**Zone de tri (Sort Area) :**
- Pour les opérations nécessitant un tri (ORDER BY, GROUP BY, JOIN)
- Taille par défaut : 65000 bytes
- En cas de saturation : utilisation d'un segment temporaire sur disque

---

## **3. Processus Oracle (Processes)**

### **A. Types de Processus**

#### **1. Processus Utilisateur (User Processes)**
- Exécutés côté client (SQL*Plus, SQL Developer)
- Génèrent des requêtes SQL
- N'interagissent pas directement avec le serveur

#### **2. Processus d'Écoute (Oracle Net Listener)**
- Point d'entrée des connexions
- Reçoit les demandes de connexion (login + password)
- Dirige les demandes vers les processus serveur appropriés

#### **3. Processus Serveur (Server Processes)**
- Exécutent les requêtes pour le compte du client
- Fonctionnent en mode dédié (Dedicated) ou partagé (Shared)
- Chaque processus serveur a sa propre zone PGA

#### **4. Processus d'Arrière-Plan (Background Processes)**

##### **DBWn (Database Writer)**
- Écrit les données modifiées de la mémoire vers les fichiers de données
- S'exécute quand : cache plein, points de contrôle, intervalles définis

##### **LGWR (Log Writer)**
- Écrit le contenu du Redo Log Buffer vers les Redo Log Files
- S'exécute lors des COMMIT pour garantir la durabilité des transactions

##### **CKPT (Checkpoint)**
- Marque les points de contrôle dans la base de données
- Met à jour les fichiers de contrôle et de données
- Réduit le temps de récupération après un crash

##### **ARCn (Archiver)**
- Archive les fichiers Redo Log lorsqu'ils sont pleins
- Essentiel en mode ARCHIVELOG

##### **SMON (System Monitor)**
- Récupère l'instance après un crash
- Nettoie les segments temporaires inutilisés
- Fusionne les espaces libres

##### **PMON (Process Monitor)**
- Surveille et nettoie les processus utilisateur défectueux
- Libère les verrous

### **B. Exemple : Flux d'une Transaction Type**
```sql
UPDATE compte SET solde = solde - 100 WHERE id = 1;
COMMIT;
```
**Ce qui se passe en arrière-plan :**
1. Modification des données en mémoire (SGA)
2. Enregistrement des modifications dans le Redo Log Buffer
3. Au COMMIT : LGWR écrit le Redo Log Buffer dans les fichiers
4. Plus tard : DBWn écrit les données modifiées vers les fichiers de données
5. CKPT met à jour les en-têtes

---

## **4. La Base de Données (Database)**

### **A. Fichiers de la Base de Données**

#### **1. Fichiers de Données (Data Files)**
- Contiennent les données réelles (tables, index)
- Organisés en Tablespaces

```sql
-- Afficher les informations de la base de données
DESC V$DATABASE;
SELECT NAME, DBID, CREATED, LOG_MODE, DATABASE_ROLE, FLASHBACK_ON 
FROM V$DATABASE;
```

#### **2. Fichiers de Contrôle (Control Files)**
- Contiennent les informations structurelles sur la base de données
- Peuvent être multiplexés (Multiplexing) pour la sécurité

```sql
-- Afficher les emplacements des fichiers de contrôle
SHOW PARAMETER CONTROL_FILES;

-- Afficher les informations des fichiers de contrôle
SELECT * FROM V$CONTROLFILE;
SELECT * FROM V$CONTROLFILE_RECORD_SECTION;
```

#### **3. Fichiers de Journaux de Réapplication (Redo Log Files)**
- Enregistrent toutes les modifications de la base de données
- Organisés en groupes circulaires (au moins deux groupes)
- Essentiels pour la récupération après un crash

### **B. Structure Logique de Stockage**

#### **1. Hiérarchie de Stockage**
```
Database → Tablespaces → Segments → Extents → Data Blocks
```

#### **2. Tablespaces (Espaces de Tables)**
**Types principaux :**
- **SYSTEM** : Contient le dictionnaire de données
- **SYSAUX** : Tables système auxiliaires
- **Temporary** : Pour les opérations temporaires
- **Undo** : Pour la gestion de l'annulation des transactions
- **User Tablespaces** : Pour le stockage des données applicatives

#### **3. Segments, Extents et Data Blocks**
- **Segment** : Objet (table, index)
- **Extent** : Ensemble contigu de blocs de données
- **Data Block** : Plus petite unité de stockage (généralement 8KB)

---

## **5. Fichiers de Paramètres (Parameter Files)**

### **A. Types de Fichiers de Paramètres**

#### **1. PFILE (Parameter File)**
- Fichier texte modifiable manuellement
- Extension : `init<SID>.ora`
- Exemple :
```ini
db_name='FREE'
memory_target=1G
processes=150
db_block_size=8192
control_files=(ora_control1, ora_control2)
```

#### **2. SPFILE (Server Parameter File)**
- Fichier binaire géré par Oracle
- Extension : `spfile<SID>.ora`
- Modifié par commandes SQL

### **B. Comparaison PFILE vs SPFILE**
| Caractéristique | PFILE | SPFILE |
|-----------------|-------|--------|
| Type | Texte | Binaire |
| Modification | Manuelle | Par commande SQL |
| Persistance | Non automatique | Automatique |
| Utilisation | Création BD, dépannage | Gestion courante |

### **C. Paramètres Importants dans le Fichier d'Initialisation**
```ini
# Exemples de paramètres importants
db_name='ORCL'
memory_target=2G
processes=300
db_block_size=8192
control_files=('/u01/oradata/control01.ctl', '/u02/oradata/control02.ctl')
db_recovery_file_dest='/u03/flash_recovery_area'
db_recovery_file_dest_size=10G
undo_tablespace='UNDOTBS1'
```

---

## **6. Dictionnaire de Données (Data Dictionary)**

### **A. Concept du Dictionnaire de Données**
- Référence centrale des métadonnées dans Oracle
- Stocké dans le Tablespace SYSTEM
- Propriétaire : utilisateur SYS
- En lecture seule pour les utilisateurs

### **B. Types de Vues du Dictionnaire**

#### **1. Vues Statiques (Static Views)**
```sql
-- Par portée
SELECT * FROM USER_TABLES;     -- Objets de l'utilisateur courant
SELECT * FROM ALL_TABLES;      -- Objets accessibles
SELECT * FROM DBA_TABLES;      -- Tous les objets de la base

-- Exemples de vues courantes
DESC USER_TABLES;
DESC USER_TAB_COLUMNS;
DESC USER_INDEXES;
DESC USER_CONSTRAINTS;
DESC DBA_USERS;
DESC DBA_OBJECTS;
```

#### **2. Vues Dynamiques (Dynamic Views)**
- Commencent par `V$`
- Affichent les informations de performance et d'activité courante

```sql
-- Exemples de vues dynamiques
SELECT * FROM V$DATABASE;
SELECT * FROM V$INSTANCE;
SELECT * FROM V$SGA;
SELECT * FROM V$PARAMETER;
SELECT * FROM V$SQL;
```

### **C. Utilisations du Dictionnaire de Données**

#### **1. Recherche d'Informations sur les Utilisateurs**
```sql
-- Lors de la connexion : CONNECT ali/1234
-- Oracle recherche dans le dictionnaire :
-- 1. L'existence de l'utilisateur ali
-- 2. La validité du mot de passe
-- 3. L'état du compte (verrouillé/déverrouillé)
-- 4. Les privilèges et rôles

SELECT * FROM DBA_USERS WHERE username = 'ALI';
SELECT * FROM DBA_ROLE_PRIVS WHERE grantee = 'ALI';
```

#### **2. Recherche d'Informations sur les Objets**
```sql
-- Lors de l'exécution : SELECT * FROM emp;
-- Oracle recherche dans le dictionnaire :
-- 1. L'existence de la table EMP
-- 2. Le schéma auquel elle appartient
-- 3. Les colonnes et leurs types
-- 4. Les index et contraintes

SELECT * FROM USER_TABLES WHERE table_name = 'EMP';
SELECT * FROM USER_TAB_COLUMNS WHERE table_name = 'EMP';
```

#### **3. Mise à Jour Automatique du Dictionnaire**
```sql
-- Lors de l'exécution d'une commande DDL comme :
CREATE TABLE etudiant (id NUMBER, nom VARCHAR2(30));

-- Oracle effectue automatiquement :
-- 1. L'ajout d'un enregistrement dans le dictionnaire
-- 2. L'enregistrement des colonnes
-- 3. L'enregistrement du propriétaire
-- 4. L'enregistrement du Tablespace

-- L'utilisateur ne modifie jamais directement le dictionnaire
```

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

### **A. Concept des Profils**
- Pour contrôler la consommation des ressources et les mots de passe
- Profil par défaut : DEFAULT
- Limites de DEFAULT : UNLIMITED

### **B. Activation du Contrôle des Ressources**
```sql
-- Dans le fichier initSID.ora
RESOURCE_LIMIT = TRUE

-- Dynamiquement
ALTER SYSTEM SET resource_limit = true;
```

### **C. Création d'un Profil**

#### **1. Définition des Limites de Ressources**
```sql
CREATE PROFILE pf_secretaire LIMIT
SESSIONS_PER_USER 2
CPU_PER_SESSION UNLIMITED
CPU_PER_CALL 1000
LOGICAL_READS_PER_SESSION UNLIMITED
LOGICAL_READS_PER_CALL 100
IDLE_TIME 30
CONNECT_TIME 480;
```

#### **2. Définition des Limites de Mots de Passe**
```sql
CREATE PROFILE pf_admin LIMIT
PASSWORD_LIFE_TIME 200
PASSWORD_REUSE_MAX DEFAULT
PASSWORD_REUSE_TIME UNLIMITED
FAILED_LOGIN_ATTEMPTS 5
PASSWORD_LOCK_TIME 1
PASSWORD_GRACE_TIME 7
CPU_PER_SESSION UNLIMITED;
```

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

## **11. Création d'une Base de Données**

### **A. Méthodes de Création de Base de Données**

#### **1. Utilisation de DBCA (Outil Graphique)**
- Database Configuration Assistant
- Interface graphique facile à utiliser

#### **2. Manuellement avec SQL*Plus**

### **B. Variables d'Environnement Requises**
```bash
ORACLE_SID=nom_de_instance
ORACLE_HOME=home_oracle
ORACLE_BASE=home_des_bases_Oracle
```

### **C. Étapes de Création Manuelle**

#### **1. Préparation de l'Arborescence**
```
ora9data/
├── dbtest/
│   ├── admin/
│   ├── tssys/
│   ├── tsuusers/
│   ├── tstemp/
│   └── tsrbs/
```

#### **2. Création du Fichier d'Initialisation initDBTEST.ora**
```ini
db_name='DBTEST'
memory_target=1G
processes=150
db_block_size=8192
control_files=('C:\ora9data\dbtest\control01.ctl',
               'C:\ora9data\dbtest\control02.ctl')
db_recovery_file_dest='C:\ora9data\dbtest\flash_recovery_area'
db_recovery_file_dest_size=2G
undo_tablespace='UNDOTBS1'
```

#### **3. Création du Service Windows (si nécessaire)**
```cmd
C:\> oradim –new –sid dbtest –intpwd manager –startmode auto –pfile c:\ora9data\dbtest\admin\initDBTEST.ora
```

#### **4. Lancement de SQL*Plus et Création de la Base**
```sql
-- Connexion en tant que SYSDBA
CONNECT sys AS SYSDBA
-- Mot de passe : manager

-- Démarrage de l'instance en mode NOMOUNT
STARTUP NOMOUNT PFILE='c:\ora9data\dbtest\admin\initDBTEST.ora';

-- Exécution du script de création de base de données
@crDBTEST.sql
```

### **D. Exemple de Commande CREATE DATABASE**
```sql
CREATE DATABASE mabdd2
USER SYS IDENTIFIED BY "VotreMotDePasseSYS"
USER SYSTEM IDENTIFIED BY "VotreMotDePasseSYS"
MAXINSTANCES 8
MAXLOGFILES 32
MAXLOGMEMBERS 5
MAXDATAFILES 1024
CHARACTER SET AL32UTF8
NATIONAL CHARACTER SET AL16UTF16
EXTENT MANAGEMENT LOCAL
LOGFILE
  GROUP 1 ('D:\oradata\mabdd2\redo01.log') SIZE 200M,
  GROUP 2 ('D:\oradata\mabdd2\redo02.log') SIZE 200M,
  GROUP 3 ('D:\oradata\mabdd2\redo03.log') SIZE 200M
UNDO TABLESPACE undotbs1
DATAFILE SIZE 500M AUTOEXTEND ON MAXSIZE UNLIMITED
DEFAULT TEMPORARY TABLESPACE temp
TEMPFILE SIZE 200M AUTOEXTEND ON MAXSIZE UNLIMITED
DEFAULT TABLESPACE users
DATAFILE SIZE 100M AUTOEXTEND ON MAXSIZE UNLIMITED
ENABLE PLUGGABLE DATABASE
SEED
SYSTEM DATAFILES SIZE 500M AUTOEXTEND ON MAXSIZE UNLIMITED
SYSAUX DATAFILES SIZE 600M AUTOEXTEND ON MAXSIZE UNLIMITED;
```

### **E. Installation du Dictionnaire de Données**
```sql
-- Exécution des scripts système
@catalog.sql    -- Création des vues du dictionnaire
@catproc.sql    -- Installation de PL/SQL
@catdbsyn.sql   -- Création de synonymes supplémentaires
```

---

## **12. Démarrage et Arrêt de la Base de Données**

### **A. Phases de Démarrage de la Base de Données**

#### **1. NOMOUNT (Niveau 1)**
```sql
STARTUP NOMOUNT;
```
**Ce qui se passe :**
- Démarrage de l'instance en mémoire
- Lecture du fichier de paramètres
- Allocation de la SGA et des processus d'arrière-plan
- **Accessible :** V$INSTANCE, V$PARAMETER, V$SGA

#### **2. MOUNT (Niveau 2)**
```sql
STARTUP MOUNT;
-- ou
ALTER DATABASE MOUNT;
```
**Ce qui se passe :**
- Lecture des fichiers de contrôle
- Connaissance de la structure de la base de données
- **Accessible :** V$DATABASE (seulement pour SYSDBA et SYSOPER)

#### **3. OPEN (Niveau 3)**
```sql
STARTUP OPEN;
-- ou
ALTER DATABASE OPEN;
```
**Ce qui se passe :**
- Ouverture des fichiers de données et Redo Logs
- Base de données prête à l'utilisation
- **Accessible :** Toutes les vues et objets

### **B. Modes de Démarrage Spéciaux**

#### **1. Mode Restreint (RESTRICT)**
```sql
STARTUP RESTRICT;
```
**But :** Maintenance, tuning, import/export

**Pour désactiver les restrictions :**
```sql
ALTER SYSTEM DISABLE RESTRICTED SESSION;
```

#### **2. Mode Forcé (FORCE)**
```sql
STARTUP FORCE;
```
**But :** En cas d'échec du démarrage normal (effectue un SHUTDOWN ABORT puis redémarre)

#### **3. Ouverture en Lecture Seule**
```sql
STARTUP OPEN READ ONLY;
-- ou
ALTER DATABASE OPEN READ ONLY;

-- Pour revenir en mode lecture/écriture
ALTER DATABASE OPEN READ WRITE;
```

### **C. Utilisation d'un PFILE Spécifique**
```sql
STARTUP PFILE='C:\app\oracle\admin\orcl\pfile\init.ora';
```

### **D. Modes d'Arrêt de la Base de Données**

#### **1. SHUTDOWN NORMAL (Par Défaut)**
```sql
SHUTDOWN NORMAL;
-- ou
SHUTDOWN;
```
**Caractéristiques :**
- N'autorise pas de nouvelles connexions
- Attend la déconnexion de tous les utilisateurs
- Ne nécessite pas de récupération au démarrage suivant

#### **2. SHUTDOWN TRANSACTIONAL**
```sql
SHUTDOWN TRANSACTIONAL;
```
**Caractéristiques :**
- N'autorise pas de nouvelles connexions
- Attend la fin des transactions en cours
- Ne nécessite pas de récupération

#### **3. SHUTDOWN IMMEDIATE**
```sql
SHUTDOWN IMMEDIATE;
```
**Caractéristiques :**
- N'autorise pas de nouvelles connexions
- Annule les sessions en cours
- Effectue un rollback des transactions non terminées
- Ne nécessite pas de récupération

#### **4. SHUTDOWN ABORT**
```sql
SHUTDOWN ABORT;
```
**Caractéristiques :**
- Arrêt immédiat
- Ne termine pas les transactions en cours
- Nécessite une récupération au démarrage suivant
- Utilisé seulement en cas d'urgence

---

## **13. Oracle Database 23ai et Versions Récentes**

### **A. Nouvelles Fonctionnalités**
- **Bases de données enfichables (Pluggable Databases)**
- **Gestion de mémoire améliorée**
- **Fonctionnalités de sécurité avancées**

### **B. Limitations de la Version Free**
```sql
-- Oracle Database 23ai Free Edition
-- SID fixe : FREE
-- Impossible : changer le SID, créer plusieurs CDBs, créer plusieurs instances FREE
```

### **C. Compatibilité avec les Versions Antérieures**
```sql
-- Paramètre de compatibilité dans le fichier d'initialisation
compatible='23.0.0'  -- ou '19.0.0', '12.2.0', etc.
```

---

## **14. Meilleures Pratiques et Recommandations**

### **A. Gestion de la Mémoire**
1. **Ajuster SGA_SIZE et PGA_AGGREGATE_TARGET** selon la mémoire du serveur
2. **Surveiller le taux de succès du cache (Buffer Cache Hit Ratio)**
3. **Ajuster le Shared Pool** pour réduire les re-parses des requêtes

### **B. Gestion du Stockage**
1. **Séparer les fichiers de données, Redo Logs et Archive Logs** sur des disques différents
2. **Utiliser Automatic Storage Management (ASM)** pour la gestion automatique
3. **Implémenter RMAN** pour les sauvegardes et restaurations

### **C. Sécurité**
1. **Principe du moindre privilège (Least Privilege)** : Accorder seulement les privilèges nécessaires
2. **Utiliser les rôles** plutôt que d'accorder les privilèges directement
3. **Implémenter des profils** pour contrôler les ressources et mots de passe
4. **Auditer les activités** pour les opérations sensibles

### **D. Performance**
1. **Ajuster les paramètres de mémoire** selon la charge de travail
2. **Utiliser Automatic Workload Repository (AWR)** pour l'analyse des performances
3. **Implémenter SQL Tuning Advisor** pour optimiser les requêtes
4. **Surveiller les attentes de la base de données (Wait Events)**

---

## **15. Commandes de Référence Rapide**

### **A. Commandes d'Affichage (SHOW)**
```sql
SHOW PARAMETER instance;
SHOW PARAMETER control_files;
SHOW PARAMETER sga;
SHOW PARAMETER pga;
SHOW PARAMETER db_block_size;
```

### **B. Commandes de Description (DESC)**
```sql
DESC V$INSTANCE;
DESC V$DATABASE;
DESC V$SGA;
DESC DBA_USERS;
DESC DBA_TABLES;
```

### **C. Commandes de Sélection (SELECT) Courantes**
```sql
-- Informations sur l'instance
SELECT * FROM V$INSTANCE;

-- Informations sur la base de données
SELECT NAME, DBID, CREATED, LOG_MODE, DATABASE_ROLE FROM V$DATABASE;

-- Informations sur la SGA
SELECT component, current_size, min_size, max_size 
FROM V$SGA_DYNAMIC_COMPONENTS;

-- Informations sur les utilisateurs
SELECT username, account_status, default_tablespace, temporary_tablespace 
FROM DBA_USERS;

-- Privilèges accordés
SELECT grantee, privilege, admin_option 
FROM DBA_SYS_PRIVS 
WHERE grantee = 'ALI';

-- Rôles accordés
SELECT grantee, granted_role, admin_option 
FROM DBA_ROLE_PRIVS 
WHERE grantee = 'ALI';
```

---

Ce guide complet rassemble **toutes les informations détaillées** du document, avec :
- **Des exemples pratiques** pour chaque concept
- **Des commandes SQL complètes** exécutables
- **Des explications détaillées** des processus internes
- **Les meilleures pratiques** et recommandations
- **Une référence rapide** des commandes courantes

Ce guide peut être utilisé comme référence complète pour comprendre et gérer Oracle Database efficacement.
