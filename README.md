# Spark Structured Streaming - Analyse de Commandes en Temps Réel

## Introduction

Ce projet démontre l'utilisation de **Apache Spark Structured Streaming** pour analyser des flux de données de commandes en temps réel. L'application lit des fichiers CSV depuis **HDFS (Hadoop Distributed File System)** et effectue diverses analyses en streaming, telles que :

- Agrégation des ventes totales
- Analyse des ventes par produit
- Analyse des ventes par client
- Statistiques par statut de commande
- Détection des commandes à haute valeur
- Classement des produits les plus vendus

Le projet supporte **deux schémas CSV différents** pour démontrer la flexibilité de l'application.

---

## Architecture du Projet

### Structure des Dossiers

```
spark-structured-streaming/
├── src/main/java/ma/enset/
│   └── Main.java                    # Application principale Spark
├── resources/
│   └── log4j2.properties            # Configuration des logs
├── data/
│   ├── orders1.csv                  # Données Schema V1
│   ├── orders2.csv                  # Données Schema V2
│   └── orders3.csv                  # Données Schema V2 (suite)
├── volumes/                         # Volumes Docker pour HDFS
├── config                           # Configuration Hadoop
├── docker-compose.yaml              # Orchestration des conteneurs
└── pom.xml                          # Configuration Maven
```

---

## Prérequis

- **Java 11** ou supérieur
- **Maven 3.6+**
- **Docker** et **Docker Compose**
- Au moins **4 GB de RAM** disponible pour Docker

---

## Explication du Code

### 1. Configuration du Projet (pom.xml)

Le fichier `pom.xml` configure les dépendances Maven nécessaires :

```xml
<dependencies>
    <!-- Spark SQL pour Structured Streaming -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.12</artifactId>
        <version>3.5.1</version>
        <scope>provided</scope>
    </dependency>
    
    <!-- Spark Streaming -->
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.12</artifactId>
        <version>3.5.1</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

**Points clés :**
- `spark-sql` : Fournit l'API Structured Streaming
- `spark-streaming` : Support pour le streaming de données
- `scope=provided` : Les dépendances sont fournies par le cluster Spark

### 2. Configuration des Logs (log4j2.properties)

```properties
rootLogger.level = WARN
logger.spark.name = org.apache.spark
logger.spark.level = WARN
logger.hadoop.name = org.apache.hadoop
logger.hadoop.level = WARN
```

**Objectif :** Réduire le bruit dans les logs en affichant uniquement les messages WARN et ERROR.

### 3. Code Principal (Main.java)

#### Initialisation de SparkSession

```java
SparkSession spark = SparkSession.builder()
    .appName("Spark Structured Streaming - Order Analytics")
    .getOrCreate();
```

**Rôle :** Crée le point d'entrée pour interagir avec Spark.

#### Définition du Schéma

**Schema V1 (orders1.csv)** :
```java
StructType schema = new StructType(new StructField[]{
    new StructField("order_id", DataTypes.LongType, true, Metadata.empty()),
    new StructField("client_id", DataTypes.StringType, true, Metadata.empty()),
    new StructField("client_name", DataTypes.StringType, true, Metadata.empty()),
    new StructField("product", DataTypes.StringType, true, Metadata.empty()),
    new StructField("quantity", DataTypes.IntegerType, true, Metadata.empty()),
    new StructField("price", DataTypes.DoubleType, true, Metadata.empty()),
    new StructField("order_date", DataTypes.StringType, true, Metadata.empty()),
    new StructField("status", DataTypes.StringType, true, Metadata.empty()),
    new StructField("total", DataTypes.DoubleType, true, Metadata.empty())
});
```

**Importance :** Définir le schéma explicitement améliore les performances et évite les erreurs de parsing.

#### Lecture du Stream

```java
Dataset<Row> ordersDF = spark.readStream()
    .schema(schema)
    .option("header", true)
    .csv("hdfs://namenode:8020/data/orders1/");
```

**Fonctionnement :** 
- Lit les fichiers CSV depuis HDFS
- Mode streaming : surveille le répertoire pour nouveaux fichiers
- Applique le schéma défini

#### Cas d'Analyse

**1. Affichage des Commandes Brutes**
```java
StreamingQuery rawOrdersQuery = ordersDF
    .writeStream()
    .format("console")
    .outputMode(OutputMode.Append())
    .queryName("raw_orders_v1")
    .start();
```
Affiche chaque nouvelle commande dès son arrivée.

**2. Agrégation des Ventes Totales**
```java
Dataset<Row> totalSalesDF = ordersDF
    .agg(
        sum("total").alias("total_sales"),
        count("order_id").alias("total_orders"),
        avg("total").alias("avg_order_value")
    );
```
Calcule les ventes totales, le nombre de commandes et la valeur moyenne.

**3. Ventes par Produit**
```java
Dataset<Row> salesByProductDF = ordersDF
    .groupBy("product")
    .agg(
        sum("total").alias("product_sales"),
        sum("quantity").alias("total_quantity"),
        count("order_id").alias("order_count")
    )
    .orderBy(desc("product_sales"));
```
Groupe les commandes par produit et calcule les ventes par produit.

**4. Ventes par Client**
```java
Dataset<Row> salesByClientDF = ordersDF
    .groupBy("client_id", "client_name")
    .agg(
        sum("total").alias("total_spent"),
        count("order_id").alias("order_count"),
        avg("total").alias("avg_order_value")
    )
    .orderBy(desc("total_spent"));
```
Analyse les dépenses de chaque client.

**5. Commandes par Statut**
```java
Dataset<Row> ordersByStatusDF = ordersDF
    .groupBy("status")
    .agg(
        count("order_id").alias("order_count"),
        sum("total").alias("total_value")
    );
```
Compte les commandes par statut (Completed, Pending, Cancelled).

**6. Commandes à Haute Valeur**
```java
Dataset<Row> highValueOrdersDF = ordersDF
    .filter(col("total").gt(100))
    .select("order_id", "client_name", "product", "total", "status");
```
Filtre les commandes supérieures à 100.

**7. Top Produits par Quantité**
```java
Dataset<Row> topProductsDF = ordersDF
    .groupBy("product")
    .agg(sum("quantity").alias("total_quantity_sold"))
    .orderBy(desc("total_quantity_sold"));
```
Classe les produits les plus vendus en quantité.

---

## Installation et Exécution

### Étape 1 : Démarrage de l'Environnement Docker

**Commande :**
```bash
docker compose up -d
```

**Explication :**
- Lance tous les conteneurs Docker en arrière-plan
- Démarre : Namenode, Datanode, ResourceManager, NodeManager, Spark Master, Spark Worker

**Résultat attendu :** Les conteneurs démarrent avec succès.

![Démarrage des conteneurs](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/docker_compose_up.png)

---

### Étape 2 : Vérification des Conteneurs

**Commande :**
```bash
docker ps
```

**Explication :**
- Liste tous les conteneurs Docker en cours d'exécution
- Vérifie que tous les services sont actifs

**Résultat attendu :** Affiche 6 conteneurs (namenode, datanode, resourcemanager, nodemanager, spark-master, spark-worker-1).

![Liste des conteneurs](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/docker_ps.png)

---

### Étape 3 : Vérification du Mode Safe HDFS

**Commande :**
```bash
docker exec namenode hdfs dfsadmin -safemode get
```

**Explication :**
- Vérifie si HDFS est en mode sécurisé (Safe Mode)
- En Safe Mode, HDFS est en lecture seule
- Doit afficher "Safe mode is OFF" pour pouvoir écrire

**Résultat attendu :** `Safe mode is OFF`

![Safe Mode Status](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/safemode.png)

---

### Étape 4 : Création des Répertoires HDFS

**Commandes :**
```bash
docker exec namenode hdfs dfs -mkdir -p /data/orders1
docker exec namenode hdfs dfs -mkdir -p /data/orders2
docker exec namenode hdfs dfs -ls /data
```

**Explication :**
- Crée les répertoires `/data/orders1` et `/data/orders2` dans HDFS
- `-p` : Crée les répertoires parents si nécessaire
- Liste le contenu du répertoire `/data` pour vérification

**Résultat attendu :** Affiche les deux répertoires créés.

![Création des répertoires HDFS](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/hdfs_directories.png)

---

### Étape 5 : Copie des Fichiers CSV vers le Conteneur

**Commandes :**
```bash
docker cp data/orders1.csv namenode:/tmp/orders1.csv
docker cp data/orders2.csv namenode:/tmp/orders2.csv
docker cp data/orders3.csv namenode:/tmp/orders3.csv
```

**Explication :**
- Copie les fichiers CSV depuis l'hôte vers le conteneur `namenode`
- Les fichiers sont placés dans `/tmp/` du conteneur
- Étape préparatoire avant le transfert vers HDFS

**Résultat attendu :** Aucune erreur, copies réussies.

![Copie des fichiers CSV](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/copy_csv_files_to_cont.png)

---

### Étape 6 : Vérification des Fichiers Copiés

**Commande :**
```bash
docker exec namenode sh -c "ls -la /tmp/*.csv"
```

**Explication :**
- Liste tous les fichiers `.csv` dans `/tmp/` du conteneur
- Vérifie que les 3 fichiers ont été copiés correctement
- Affiche la taille et les permissions des fichiers

**Résultat attendu :** Liste des 3 fichiers orders1.csv, orders2.csv, orders3.csv.

![Vérification des fichiers](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/verifying_files_in_cnts.png)

---

### Étape 7 : Compilation du Projet Maven

**Commande :**
```bash
mvn clean package -DskipTests
```

**Explication :**
- `clean` : Supprime les anciens fichiers compilés
- `package` : Compile le code et crée le fichier JAR
- `-DskipTests` : Ignore les tests unitaires pour accélérer la compilation


### Étape 8 : Vérification du JAR Créé

**Commande :**
```bash
ls target/*.jar
```

**Explication :**
- Liste tous les fichiers `.jar` dans le répertoire `target/`
- Vérifie que le fichier `spark-structured-streaming-1.0-SNAPSHOT.jar` a été créé

**Résultat attendu :** Affiche `target/spark-structured-streaming-1.0-SNAPSHOT.jar`.

![Vérification du JAR](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/jar_exists.png)

---

### Étape 9 : Copie du JAR vers Spark Master

**Commande :**
```bash
docker cp target/spark-structured-streaming-1.0-SNAPSHOT.jar spark-master:/opt/spark/
```

**Explication :**
- Copie le fichier JAR compilé vers le conteneur `spark-master`
- Destination : `/opt/spark/` (répertoire Spark)
- Prépare l'application pour l'exécution sur le cluster Spark

**Résultat attendu :** Copie réussie sans erreur.

![Copie du JAR](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/copy_jar.png)

---

### Étape 10 : Configuration des Logs Spark

**Commandes :**
```bash
docker exec spark-master mkdir -p /opt/spark/conf
docker exec spark-master sh -c 'cat > /opt/spark/conf/log4j2.properties << EOF
rootLogger.level = ERROR
rootLogger.appenderRef.stdout.ref = console
appender.console.type = Console
appender.console.name = console
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %d{HH:mm:ss} %-5level - %msg%n
logger.spark.name = org.apache.spark
logger.spark.level = ERROR
logger.hadoop.name = org.apache.hadoop
logger.hadoop.level = ERROR
logger.jetty.name = org.eclipse.jetty
logger.jetty.level = ERROR
logger.akka.name = akka
logger.akka.level = ERROR
EOF'
```

**Explication :**
- Crée le répertoire de configuration Spark
- Crée le fichier `log4j2.properties` pour configurer les logs
- Configure le niveau de log à ERROR pour réduire le bruit

**Résultat attendu :** Fichier de configuration créé sans erreur.

![Configuration des logs](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/spark_logging.png)

---

### Étape 11 : Exécution avec Schema V1 (orders1.csv)

**Commande :**
```bash
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --class ma.enset.Main \
  /opt/spark/spark-structured-streaming-1.0-SNAPSHOT.jar 1
```

**Explication :**
- `spark-submit` : Soumet l'application au cluster Spark
- `--master spark://spark-master:7077` : Adresse du master Spark
- `--class ma.enset.Main` : Classe principale à exécuter
- Argument `1` : Utilise le Schema V1 (orders1.csv)

**L'application attend maintenant des fichiers dans HDFS...**

**Résultat attendu :** L'application démarre et attend des données.

![Exécution Schema V1 - Démarrage](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/test_schema1.png)

---

### Étape 12 : Injection de Données Schema V1

**Commandes :**
```bash
docker exec namenode hdfs dfs -put /tmp/orders1.csv /data/orders1/
docker exec namenode hdfs dfs -ls /data/orders1/
```

**Explication :**
- Transfert de `orders1.csv` depuis `/tmp/` vers HDFS `/data/orders1/`
- Liste le contenu du répertoire pour vérification
- **Dès que le fichier est détecté, Spark traite les données automatiquement**

**Résultat attendu :** 
- Fichier transféré vers HDFS
- L'application Spark affiche les analyses en temps réel dans la console

![Injection et traitement Schema V1](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/upload_data_hdfs_sch1.png)

![Résultats des analyses Schema V1](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/schema1_res2.png)
![Résultats des analyses Schema V1](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/schema1_res3.png)

---

### Étape 13 : Exécution avec Schema V2 (orders2/orders3.csv)

**Commande :**
```bash
docker exec -it spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --class ma.enset.Main \
  /opt/spark/spark-structured-streaming-1.0-SNAPSHOT.jar 2
```

**Explication :**
- Lance l'application avec le Schema V2
- Argument `2` : Utilise le Schema V2 (orders2/orders3.csv)
- L'application lit depuis `/data/orders2/`

**L'application attend maintenant des fichiers dans HDFS...**

**Résultat attendu :** L'application démarre avec le Schema V2.

![Exécution Schema V2 - Démarrage](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/test_schema2.png)

---

### Étape 14 : Injection de Données Schema V2

**Commandes :**
```bash
docker exec namenode hdfs dfs -put /tmp/orders2.csv /data/orders2/
docker exec namenode hdfs dfs -put /tmp/orders3.csv /data/orders2/
docker exec namenode hdfs dfs -ls /data/orders2/
```

**Explication :**
- Transfert de `orders2.csv` et `orders3.csv` vers HDFS
- Injection de **deux fichiers simultanément** pour simuler un flux continu
- Spark détecte et traite les deux fichiers automatiquement

**Résultat attendu :**
- Les deux fichiers sont transférés vers HDFS
- L'application traite les données et affiche les analyses

![Injection Schema V2](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/upload_data_hdfs_sch2.png)

![Résultats des analyses Schema V2](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/schema2_res1.png)
![Résultats des analyses Schema V2](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/schema2_res2.png)
![Résultats des analyses Schema V2](https://github.com/FatihaELHABTI/tp5_Spark-structured-Streaming/blob/main/screenshots/schema2_res3.png)

---

## Analyses Produites

L'application génère **7 analyses différentes** en temps réel :

### 1. Commandes Brutes (`raw_orders`)
Affiche toutes les commandes entrantes avec tous les champs.

### 2. Agrégation des Ventes Totales (`total_sales`)
- **total_sales** : Somme de toutes les ventes
- **total_orders** : Nombre total de commandes
- **avg_order_value** : Valeur moyenne d'une commande

### 3. Ventes par Produit (`sales_by_product`)
- **product_sales** : Ventes totales par produit
- **total_quantity** : Quantité totale vendue
- **order_count** : Nombre de commandes par produit

### 4. Ventes par Client (`sales_by_client`)
- **total_spent** : Montant total dépensé par client
- **order_count** : Nombre de commandes par client
- **avg_order_value** : Valeur moyenne des commandes

### 5. Commandes par Statut (`orders_by_status`)
- **Completed** : Commandes terminées
- **Pending** : Commandes en attente
- **Cancelled** : Commandes annulées

### 6. Commandes à Haute Valeur (`high_value_orders`)
Filtre les commandes **> 100** pour détecter les transactions importantes.

### 7. Top Produits (`top_products`)
Classement des produits les plus vendus par **quantité totale**.

---

## Points Clés du Projet

### Avantages de Structured Streaming

1. **API Unifiée** : Même code pour batch et streaming
2. **Fault Tolerance** : Récupération automatique en cas d'erreur
3. **Exactly-Once Semantics** : Garantit que chaque enregistrement est traité exactement une fois
4. **Support Multiple Sources** : HDFS, Kafka, Socket, etc.
5. **Output Modes Flexibles** : Append, Complete, Update

### Modes de Sortie (Output Modes)

- **Append** : Ajoute seulement les nouvelles lignes (pour les commandes brutes)
- **Complete** : Réécrit la table entière (pour les agrégations)
- **Update** : Met à jour uniquement les lignes modifiées

### Architecture Hadoop + Spark

```
┌─────────────────────────────────────────────┐
│           Spark Master (Port 8080)          │
│  Gère l'orchestration des jobs Spark       │
└─────────────────┬───────────────────────────┘
                  │
          ┌───────┴────────┐
          │                │
┌─────────▼────────┐  ┌───▼──────────────┐
│  Spark Worker 1  │  │  Spark Worker N  │
│  Exécute les     │  │  (scalable)      │
│  tâches Spark    │  │                  │
└──────────────────┘  └──────────────────┘
                  │
          ┌───────┴────────┐
          │                │
┌─────────▼────────┐  ┌───▼──────────────┐
│  HDFS Namenode   │  │  HDFS Datanode   │
│  (Port 9870)     │  │  Stockage        │
│  Métadonnées     │  │  distribué       │
└──────────────────┘  └──────────────────┘
```

---

## Dépannage

### Problème : Safe Mode activé

**Symptôme :** `Safe mode is ON`

**Solution :**
```bash
docker exec namenode hdfs dfsadmin -safemode leave
```

### Problème : Conteneurs non démarrés

**Solution :**
```bash
docker compose down
docker compose up -d
```

### Problème : Port déjà utilisé

**Solution :** Modifier les ports dans `docker-compose.yaml` ou arrêter les services conflictuels.

### Problème : Mémoire insuffisante

**Solution :** Augmenter la mémoire Docker dans les paramètres Docker Desktop (minimum 4 GB).

---

## Accès aux Interfaces Web

- **Spark Master UI** : http://localhost:8080
- **HDFS Namenode UI** : http://localhost:9870
- **YARN ResourceManager** : http://localhost:8088

---

## Concepts Clés Appris

### 1. Spark Structured Streaming
- Traitement de flux de données en quasi temps réel
- API déclarative basée sur DataFrame/Dataset
- Tolérance aux pannes intégrée

### 2. HDFS (Hadoop Distributed File System)
- Stockage distribué de fichiers volumineux
- Réplication des données pour la haute disponibilité
- Architecture Master-Worker (Namenode-Datanode)

### 3. Intégration Spark + Hadoop
- Spark lit directement depuis HDFS
- YARN gère les ressources du cluster
- Exécution distribuée des tâches

### 4. Analyses en Temps Réel
- Agrégations continues (sum, count, avg)
- Fenêtrage temporel
- Détection de patterns

---



## Conclusion

Ce projet démontre une **architecture complète de traitement de données en streaming** utilisant les technologies Big Data modernes. L'application peut être facilement étendue pour :

- Intégrer Apache Kafka pour du streaming en temps réel
- Sauvegarder les résultats dans une base de données (Cassandra, HBase)
- Ajouter des visualisations avec des dashboards (Grafana, Kibana)
- Intégrer du Machine Learning avec Spark MLlib
- Implémenter du traitement par fenêtres temporelles (windowing)

**Félicitations !** Vous avez maintenant une application de streaming fonctionnelle prête pour le traitement de données massives en temps réel.

---
