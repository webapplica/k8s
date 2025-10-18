```markdown
# Créer une image Docker pour votre code PHP ToDoList

Voici comment créer une [translate:image docker] pour votre code PHP ToDoList fourni.

---

## 1. Créez un fichier `Dockerfile`

Placez ce fichier dans le même dossier que votre code PHP (ex. `index.php`).

```
# Choisir une image de base officielle PHP avec Apache et support PDO MySQL
FROM php:8.0-apache

# Installer les extensions nécessaires à PDO MySQL
RUN docker-php-ext-install pdo pdo_mysql

# Copier le code PHP dans le dossier racine web d'Apache
COPY ./ /var/www/html/

# Exposer le port 80
EXPOSE 80

# Démarrer le serveur Apache en mode foreground (comportement par défaut de l'image)
CMD ["apache2-foreground"]
```

---

## 2. Construisez l’image Docker

Dans le terminal, à la racine du dossier contenant `Dockerfile` et `index.php`, exécutez :

```
docker build -t todolist-php-image .
```

---

## 3. Testez l’image localement

Pour tester votre image localement et injecter les variables d’environnement nécessaires à MySQL :

```
docker run -p 8080:80 -e DB_HOST=mysql-service -e DB_NAME=tododb -e DB_USER=todo_user -e DB_PASS=monmotdepasse todolist-php-image
```

Accédez ensuite à l’application via [translate:http://localhost:8080].

---

## Notes

- L’image installe les extensions PHP nécessaires au support MySQL.
- Le code PHP est copié dans le dossier web d’Apache.
- Les variables d’environnement pour la connexion à la base sont dynamiques et définies au runtime.
- Ce conteneur est prêt à être déployé en Kubernetes, où ConfigMap et Secret peuvent fournir ces variables.

---

Si vous le souhaitez, je peux vous aider à rédiger le manifeste Kubernetes qui déploie cette image dans un cluster.

Ce processus est la méthode standard pour packager une application PHP connectée à MySQL dans Docker et Kubernetes.
```

[1](https://github.com/kauffinger/html2mkdwn/)
[2](https://github.com/kobucom/docker-pandoc)
[3](https://hub.docker.com/r/spawnia/md-to-pdf)
[4](https://edu.chainguard.dev/chainguard/migration/dockerfile-conversion/)
[5](https://dillinger.io)
[6](https://stackoverflow.com/questions/76139495/how-can-i-export-a-markdown-file-as-a-pdf-using-the-exact-same-style-as-github)
[7](https://manios.org/2020/01/08/convert-markdown-to-pdf-using-docker-pandoc-asciidoctor)
[8](https://blueprints.mozilla.ai/all-blueprints/convert-documents-to-markdown-format)
