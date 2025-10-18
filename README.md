# Application TodoList PHP avec Déploiement Kubernetes

## Structure du Projet

### Fichiers de l'Application

#### 1. `index.php`
```php
<?php
// Configuration de la base de données
$db_host = getenv('DB_HOST') ?: 'mysql-service';
$db_name = getenv('DB_NAME') ?: 'tododb';
$db_user = getenv('DB_USER') ?: 'todo_user';
$db_pass = getenv('DB_PASS') ?: '';

// Connexion à la base de données
try {
    $pdo = new PDO("mysql:host=$db_host;dbname=$db_name", $db_user, $db_pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    die("Erreur de connexion: " . $e->getMessage());
}

// Création de la table si elle n'existe pas
$createTable = "CREATE TABLE IF NOT EXISTS todolist (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task VARCHAR(255) NOT NULL,
    completed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)";
$pdo->exec($createTable);

// Traitement du formulaire
if ($_POST['action'] === 'add' && !empty($_POST['task'])) {
    $stmt = $pdo->prepare("INSERT INTO todolist (task) VALUES (?)");
    $stmt->execute([$_POST['task']]);
    header("Location: ".$_SERVER['PHP_SELF']);
    exit;
}

if ($_POST['action'] === 'delete') {
    $stmt = $pdo->prepare("DELETE FROM todolist WHERE id = ?");
    $stmt->execute([$_POST['id']]);
    header("Location: ".$_SERVER['PHP_SELF']);
    exit;
}

if ($_POST['action'] === 'toggle') {
    $stmt = $pdo->prepare("UPDATE todolist SET completed = NOT completed WHERE id = ?");
    $stmt->execute([$_POST['id']]);
    header("Location: ".$_SERVER['PHP_SELF']);
    exit;
}

// Récupération des tâches
$stmt = $pdo->query("SELECT * FROM todolist ORDER BY created_at DESC");
$tasks = $stmt->fetchAll(PDO::FETCH_ASSOC);
?>

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TodoList App</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .task { padding: 10px; margin: 5px 0; border: 1px solid #ddd; border-radius: 4px; }
        .completed { text-decoration: line-through; color: #888; }
        form { margin-bottom: 20px; }
        input[type="text"] { padding: 8px; width: 300px; }
        button { padding: 8px 15px; margin-left: 5px; }
    </style>
</head>
<body>
    <h1>Ma TodoList</h1>
    
    <form method="POST">
        <input type="text" name="task" placeholder="Nouvelle tâche..." required>
        <input type="hidden" name="action" value="add">
        <button type="submit">Ajouter</button>
    </form>

    <div class="tasks">
        <?php foreach ($tasks as $task): ?>
            <div class="task <?php echo $task['completed'] ? 'completed' : ''; ?>">
                <span><?php echo htmlspecialchars($task['task']); ?></span>
                <form method="POST" style="display: inline;">
                    <input type="hidden" name="id" value="<?php echo $task['id']; ?>">
                    <input type="hidden" name="action" value="toggle">
                    <button type="submit"><?php echo $task['completed'] ? 'Rétablir' : 'Terminer'; ?></button>
                </form>
                <form method="POST" style="display: inline;">
                    <input type="hidden" name="id" value="<?php echo $task['id']; ?>">
                    <input type="hidden" name="action" value="delete">
                    <button type="submit" style="color: red;">Supprimer</button>
                </form>
            </div>
        <?php endforeach; ?>
    </div>
</body>
</html>
```

#### 2. `Dockerfile`
```dockerfile
FROM php:8.1-apache

# Installer les extensions PHP nécessaires
RUN docker-php-ext-install pdo pdo_mysql

# Copier l'application
COPY index.php /var/www/html/

# Configurer les permissions
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html

EXPOSE 80
```

## Fichiers de Configuration Kubernetes

### Configuration MySQL

#### 3. `mysql-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  database.name: "tododb"
  database.user: "todo_user"
```

#### 4. `mysql-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  database.password: "dG9kb19wYXNzd29yZA=="  # todo_password en base64
```

#### 5. `mysql-storage.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data/mysql"

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
      storage: 1Gi
  storageClassName: manual
```

#### 6. `mysql-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 2
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
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: database.password
        - name: MYSQL_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mysql-config
              key: database.name
        - name: MYSQL_USER
          valueFrom:
            configMapKeyRef:
              name: mysql-config
              key: database.user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: database.password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None
```

### Configuration de l'Application

#### 7. `app-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "mysql-service"
  DB_NAME: "tododb"
  DB_USER: "todo_user"
```

#### 8. `app-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASS: "dG9kb19wYXNzd29yZA=="  # todo_password en base64
```

#### 9. `app-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todoapp-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todoapp
  template:
    metadata:
      labels:
        app: todoapp
    spec:
      containers:
      - name: todoapp
        image: todoapp:latest
        ports:
        - containerPort: 80
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_USER
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASS

---
apiVersion: v1
kind: Service
metadata:
  name: todoapp-service
spec:
  type: LoadBalancer
  selector:
    app: todoapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

## Scripts de Déploiement

#### 10. `deploy.sh`
```bash
#!/bin/bash

echo "Déploiement de l'application TodoList..."

# Construire l'image Docker
echo "Construction de l'image Docker..."
docker build -t todoapp:latest /home/vagrant/app/code

# Créer les ressources Kubernetes
echo "Création des ConfigMaps et Secrets..."
kubectl apply -f mysql-configmap.yaml
kubectl apply -f mysql-secret.yaml
kubectl apply -f app-configmap.yaml
kubectl apply -f app-secret.yaml

# Créer le stockage
echo "Configuration du stockage..."
mkdir -p /mnt/data/mysql
kubectl apply -f mysql-storage.yaml

# Déployer MySQL
echo "Déploiement de MySQL..."
kubectl apply -f mysql-deployment.yaml

# Attendre que MySQL soit prêt
echo "Attente du démarrage de MySQL..."
kubectl wait --for=condition=ready pod -l app=mysql --timeout=300s

# Déployer l'application
echo "Déploiement de l'application..."
kubectl apply -f app-deployment.yaml

# Attendre que l'application soit prête
echo "Attente du démarrage de l'application..."
kubectl wait --for=condition=ready pod -l app=todoapp --timeout=300s

# Afficher les informations de déploiement
echo "Déploiement terminé!"
echo "Pods déployés:"
kubectl get pods

echo "Services:"
kubectl get services

echo "Pour accéder à l'application, utilisez l'adresse du nœud avec le port 30007"
```

#### 11. `check-status.sh`
```bash
#!/bin/bash

echo "=== Statut des Pods ==="
kubectl get pods

echo -e "\n=== Statut des Services ==="
kubectl get services

echo -e "\n=== Logs de l'application ==="
kubectl logs -l app=todoapp --tail=10

echo -e "\n=== Logs de MySQL ==="
kubectl logs -l app=mysql --tail=10
```

## Ordre d'Exécution

### Étapes de Déploiement

1. **Placer tous les fichiers dans `/home/vagrant/app/code/`**

2. **Rendre les scripts exécutables:**
   ```bash
   chmod +x /home/vagrant/app/code/deploy.sh
   chmod +x /home/vagrant/app/code/check-status.sh
   ```

3. **Exécuter le déploiement:**
   ```bash
   cd /home/vagrant/app/code
   ./deploy.sh
   ```

4. **Vérifier le statut:**
   ```bash
   ./check-status.sh
   ```

## Accès à l'Application

L'application sera accessible via l'adresse suivante :
```
http://<adresse-ip-du-node>:30007
```

## Architecture du Déploiement

- **2 pods** pour l'application PHP
- **2 pods** pour MySQL
- **ConfigMap** pour les paramètres de configuration
- **Secret** pour les mots de passe
- **Stockage persistant** pour la base de données
- **Service LoadBalancer** pour l'accès externe

## Notes Importantes

- Mot de passe MySQL : `todo_password` (encodé en base64)
- Les données sont persistées dans `/mnt/data/mysql` sur le nœud
- Le port d'accès de l'application est `30007`
- La base de données MySQL utilise le service DNS `mysql-service`
