
# Installer un registre Docker local

Pour installer un registre Docker local ([translate:local Docker registry]) qui permet de stocker et distribuer vos images Docker en local, suivez ces étapes simples :

---

## Lancer un registre Docker local

1. Exécutez la commande suivante pour lancer un conteneur de registre officiel sur le port 5000 :

```
docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
```

Cette commande télécharge et démarre le registre Docker local dans un conteneur écoutant sur le port 5000.

2. Vérifiez que le conteneur tourne bien :

```
docker ps
```

Vous devez voir un conteneur nommé `local-registry`.

---

## Utiliser ce registre local

- Pour taguer une image Docker locale afin de la pousser dans ce registre :

```
docker tag todolist-php-image localhost:5000/todolist-php-image
```

- Puis poussez l'image dans le registre local :

```
docker push localhost:5000/todolist-php-image
```

---

## Utilisation avec Kubernetes

Dans vos manifests Kubernetes, spécifiez l'image comme suit :

```
image: localhost:5000/todolist-php-image
```

Si votre cluster est distant, assurez-vous d’ouvrir le port 5000 pour l’accès réseau.

---

## Options avancées

- Pour la persistance de votre registre, montez un volume de stockage sur le chemin `/var/lib/registry` :

```
docker run -d -p 5000:5000 --restart=always --name local-registry \
  -v /path/to/registry/data:/var/lib/registry \
  registry:2
```

- Pour sécuriser le registre (TLS, authentification), configurez un proxy inverse comme NGINX avec TLS.

---

Cette méthode simple est idéale pour gérer des images Docker en local, notamment dans un environnement de développement avant un déploiement en Kubernetes.

N’hésitez pas à demander si vous souhaitez un exemple avec Docker Compose ou une configuration plus avancée.

[web:55][web:60][web:62]
```

[1](https://www.it-connect.fr/convertir-un-document-au-format-markdown-avec-markitdown/)
[2](https://md2doc.com)
[3](https://stackoverflow.com/questions/56820220/convert-a-markdown-text-file-into-a-google-document)
[4](https://www.w3docs.com/nx/marked)
[5](https://products.aspose.app/cells/fr/conversion/text-to-markdown)
[6](https://monkt.com/word-to-markdown/)
[7](https://pleasetools.com/tools/html-vers-markdown)
[8](https://support.google.com/docs/answer/12014036?hl=en)
[9](https://apyhub.com/utility/converter-markdown-json)
