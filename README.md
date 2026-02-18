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

## **8. Gestion des Utilisateurs**

### **Définition :** Un utilisateur dans Oracle est un compte qui peut se connecter à la base de données et posséder des objets.

### **Étapes de création d'un nouvel utilisateur :**

```sql
CREATE USER ali
IDENTIFIED BY password123
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 100M ON users
PASSWORD EXPIRE
ACCOUNT UNLOCK;
```

### **Composants de la commande CREATE USER :**

1. **IDENTIFIED BY :** Méthode d'authentification
2. **DEFAULT TABLESPACE :** Emplacement de stockage des objets de l'utilisateur
3. **TEMPORARY TABLESPACE :** Pour les fichiers temporaires
4. **QUOTA :** Espace autorisé
5. **PASSWORD EXPIRE :** Le mot de passe expire à la première connexion
6. **ACCOUNT UNLOCK :** Compte non verrouillé

### **Modification et suppression des utilisateurs :**

```sql
-- Changer le mot de passe
ALTER USER ali IDENTIFIED BY newpassword;

-- Changer le quota
ALTER USER ali QUOTA 200M ON users;

-- Verrouiller le compte
ALTER USER ali ACCOUNT LOCK;

-- Supprimer l'utilisateur
DROP USER ali CASCADE; -- avec tous ses objets
```

---

## **9. Gestion des Privilèges**

### **Définition :** Un privilège est le droit d'exécuter une action spécifique dans la base de données.

### **Types de privilèges :**

#### **1. Privilèges système (System Privileges)**
- Permettent d'exécuter des opérations au niveau système
- Environ 127 privilèges système dans Oracle

**Exemples :**
```sql
GRANT CREATE SESSION TO ali;      -- Permettre la connexion
GRANT CREATE TABLE TO ali;        -- Permettre de créer des tables
GRANT CREATE ANY TABLE TO ali;    -- Permettre de créer des tables dans n'importe quel schéma
```

#### **2. Privilèges objet (Object Privileges)**
- Permettent d'accéder à des objets spécifiques

**Privilèges sur les tables :**
- SELECT, INSERT, UPDATE, DELETE, ALTER, INDEX, REFERENCES

```sql
-- Accorder des privilèges sur une table
GRANT SELECT, INSERT ON employees TO ali;

-- Accorder UPDATE sur des colonnes spécifiques
GRANT UPDATE (salary, commission_pct) ON employees TO ali;
```

### **Révocation des privilèges :**

```sql
REVOKE CREATE TABLE FROM ali;
REVOKE SELECT ON employees FROM ali;
```

**À retenir pour l'examen :**
- WITH ADMIN OPTION : L'utilisateur peut accorder le privilège à d'autres
- WITH GRANT OPTION : Même chose mais pour les privilèges objet
- Les privilèges accordés à PUBLIC sont donnés à tous les utilisateurs

---

## **10. Gestion des Rôles**

### **Définition :** Un rôle est un ensemble de privilèges accordés comme une seule unité. Facilite la gestion des privilèges.

### **Création d'un rôle :**

```sql
CREATE ROLE manager_role;
GRANT CREATE TABLE, CREATE VIEW, CREATE PROCEDURE TO manager_role;
GRANT manager_role TO ali;
```

### **Rôles standards :**

1. **CONNECT :** Contient seulement CREATE SESSION
2. **RESOURCE :** Contient des privilèges de création d'objets
3. **DBA :** Contient tous les privilèges système avec ADMIN OPTION

### **Afficher les rôles :**

```sql
-- Rôles accordés à un utilisateur
SELECT * FROM DBA_ROLE_PRIVS WHERE grantee = 'ALI';

-- Contenu d'un rôle
SELECT * FROM DBA_SYS_PRIVS WHERE grantee = 'RESOURCE';
```

---

## **11. Gestion des Profils**

### **Définition :** Un profil est un ensemble de limites sur les ressources et les mots de passe appliquées aux utilisateurs.

### **Création d'un profil :**

```sql
CREATE PROFILE clerk LIMIT
SESSIONS_PER_USER 2
CPU_PER_SESSION 100000
LOGICAL_READS_PER_SESSION 10000
CONNECT_TIME 480
IDLE_TIME 30
FAILED_LOGIN_ATTEMPTS 3
PASSWORD_LOCK_TIME 1/24  -- Verrouillé pendant 1 heure
PASSWORD_LIFE_TIME 90;
```

### **Application du profil :**

```sql
CREATE USER clerk_user IDENTIFIED BY password PROFILE clerk;
-- Ou
ALTER USER existing_user PROFILE clerk;
```

### **Activer les limites de ressources :**

```sql
ALTER SYSTEM SET resource_limit = TRUE;
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
