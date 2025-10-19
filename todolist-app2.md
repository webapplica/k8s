C'est un plaisir de vous aider à reformater votre application TodoList en PHP/MySQL pour un déploiement sur Kubernetes (K8s) en format Markdown.

Voici le plan d'action structuré, incluant le code de chaque fichier et les commandes de déploiement.

-----

## Guide de Déploiement TodoList PHP/MySQL sur K8s

Le chemin de base de l'application est supposé être : `/home/vagrant/app/code/`.

### 1\. Structure du Répertoire 📁

```
/home/vagrant/app/code/
├── index.php         # Application PHP
├── todolist.sql      # Schéma de la base de données
└── k8s/              # Manifestes Kubernetes
    ├── 01-config.yaml
    ├── 02-mysql-storage.yaml
    ├── 03-mysql.yaml
    └── 04-app.yaml
```

-----

### 2\. Code de l'Application (PHP & SQL) 💻

#### Fichier : `todolist.sql` (Schéma de la Base de Données)

```sql
-- todolist.sql
CREATE DATABASE IF NOT EXISTS tododb;
USE tododb;

CREATE TABLE IF NOT EXISTS todolist (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Fichier : `index.php` (Application PHP)

Ce script utilise les variables d'environnement injectées par K8s (ConfigMap et Secret) pour la connexion.

```php
<?php
// index.php

// Paramètres de connexion tirés des variables d'environnement K8s
$host = getenv('DB_HOST') ?: 'mysql-service'; // Nom du Service K8s
$db   = getenv('DB_NAME') ?: 'tododb';
$user = getenv('DB_USER') ?: 'appuser';
// Mot de passe injecté depuis le Secret
$pass = getenv('DB_PASSWORD'); 

$dsn = "mysql:host=$host;dbname=$db;charset=utf8mb4";
$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
];

try {
    $pdo = new PDO($dsn, $user, $pass, $options);
} catch (\PDOException $e) {
    // Affiche l'erreur de connexion si la base de données n'est pas prête
    die("Database connection failed: " . $e->getMessage());
}

// Gère l'ajout d'une nouvelle tâche
if ($_SERVER['REQUEST_METHOD'] === 'POST' && !empty($_POST['task'])) {
    $task = $_POST['task'];
    $stmt = $pdo->prepare("INSERT INTO todolist (task) VALUES (?)");
    $stmt->execute([$task]);
    header("Location: index.php"); 
    exit;
}

// Récupère toutes les tâches
$stmt = $pdo->query("SELECT * FROM todolist ORDER BY created_at DESC");
$tasks = $stmt->fetchAll();

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>K8s PHP ToDo List</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .task-list { margin-top: 20px; border: 1px solid #ccc; padding: 10px; }
        .task { border-bottom: 1px dashed #eee; padding: 5px 0; }
    </style>
</head>
<body>
    <h1>K8s PHP ToDo List</h1>

    <form method="POST">
        <input type="text" name="task" placeholder="Nouvelle tâche..." required>
        <button type="submit">Ajouter</button>
    </form>

    <div class="task-list">
        <h2>Tâches</h2>
        <?php if (count($tasks) > 0): ?>
            <?php foreach ($tasks as $task): ?>
                <div class="task"><?= htmlspecialchars($task['task']) ?> (<?= $task['created_at'] ?>)</div>
            <?php endforeach; ?>
        <?php else: ?>
            <p>Aucune tâche pour l'instant !</p>
        <?php endif; ?>
    </div>
</body>
</html>
```

-----

### 3\. Manifestes Kubernetes (K8s) 🚀

#### Fichier : `k8s/01-config.yaml` (ConfigMap et Secret)

> **NOTE :** Le champ `data` du Secret doit être encodé en **base64**.
>
> `echo -n "StrongDBPassword" | base64` (Utilisez votre propre mot de passe encodé).

```yaml
# k8s/01-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Paramètres de la base de données
  DB_NAME: "tododb"
  DB_USER: "appuser"
  DB_HOST: "mysql-service" # Nom du Service MySQL

---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  # Mot de passe ROOT encodé en base64
  MYSQL_ROOT_PASSWORD: "U3Ryb25nREJQYXNzd29yZA==" 
  # Mot de passe utilisateur APP encodé en base64
  DB_PASSWORD: "U3Ryb25nREJQYXNzd29yZA=="        
```

#### Fichier : `k8s/02-mysql-storage.yaml` (Storage - PV et PVC)

> **ATTENTION :** Le `hostPath` est utilisé pour la persistance locale sur un nœud. Assurez-vous que le chemin `/mnt/data/mysql` existe sur votre nœud K8s.

```yaml
# k8s/02-mysql-storage.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data/mysql" # Chemin sur le nœud K8s
  persistentVolumeReclaimPolicy: Retain

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

#### Fichier : `k8s/03-mysql.yaml` (Déploiement et Service MySQL)

```yaml
# k8s/03-mysql.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  labels:
    app: mysql
spec:
  replicas: 2 # 2 pods MySQL
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        envFrom:
          - configMapRef:
              name: app-config # Pour DB_NAME, DB_USER
        env:
          # Root password from the Secret
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: MYSQL_ROOT_PASSWORD
          # User password from the Secret
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: DB_PASSWORD
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d/ # Point d'initialisation du schéma
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: initdb
        hostPath:
          # Monte le fichier SQL pour initialiser la base
          path: /home/vagrant/app/code/todolist.sql 
          type: File

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  labels:
    app: mysql
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  clusterIP: None # Service Headless
```

#### Fichier : `k8s/04-app.yaml` (Déploiement et Service de l'Application PHP)

```yaml
# k8s/04-app.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app-deployment
  labels:
    app: php-app
spec:
  replicas: 2 # 2 pods d'application PHP
  selector:
    matchLabels:
      app: php-app
  template:
    metadata:
      labels:
        app: php-app
    spec:
      containers:
      - name: php-app
        image: php:8.0-apache # Image PHP + Serveur Web
        ports:
        - containerPort: 80
        volumeMounts:
        - name: app-volume
          mountPath: /var/www/html # Chemin racine du serveur Web
        envFrom:
          - configMapRef:
              name: app-config # Injecte DB_HOST, DB_NAME, DB_USER
        env:
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: DB_PASSWORD # Injecte DB_PASSWORD
      volumes:
      - name: app-volume
        hostPath:
          path: /home/vagrant/app/code # Montage du code source
          type: Directory

---
apiVersion: v1
kind: Service
metadata:
  name: php-app-service
  labels:
    app: php-app
spec:
  type: LoadBalancer # Exposition externe
  selector:
    app: php-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

-----

### 4\. Étapes de Déploiement 🛠️

Exécutez ces commandes dans l'ordre sur votre cluster K8s (après avoir créé les fichiers et le répertoire `/mnt/data/mysql` sur le nœud).

#### Commandes Préliminaires

```bash
# 1. Créer le répertoire de stockage sur le nœud
sudo mkdir -p /mnt/data/mysql
sudo chmod 777 /mnt/data/mysql 

# 2. Créer le sous-répertoire K8s
mkdir -p /home/vagrant/app/code/k8s
```

#### Déploiement K8s

```bash
# Appliquer les manifestes dans l'ordre
cd /home/vagrant/app/code/

# 1. Configuration (ConfigMap, Secret)
kubectl apply -f k8s/01-config.yaml

# 2. Stockage (PV, PVC)
kubectl apply -f k8s/02-mysql-storage.yaml

# 3. Base de Données (MySQL Deployment et Service)
kubectl apply -f k8s/03-mysql.yaml

# 4. Application (PHP Deployment et Service)
kubectl apply -f k8s/04-app.yaml
```

#### Vérification et Accès

```bash
# Vérifier l'état de tous les composants
kubectl get all

# Attendre que tous les pods soient en statut 'Running' (2/2 READY)

# Obtenir l'adresse d'accès à l'application
# Si vous utilisez Minikube :
# minikube service php-app-service

# Si vous utilisez un cluster :
kubectl get service php-app-service
# Accédez à l'application via : http://<EXTERNAL-IP> (ou http://<Node_IP>:<NodePort> si l'IP externe est pending)
```
