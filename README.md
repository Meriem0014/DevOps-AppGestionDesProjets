
DevOps – Automatisation du Continuous Deployment avec Jenkins et Docker
1. Présentation du projet

Ce projet consiste à automatiser le déploiement d'une application backend Spring Boot à l'aide d'un pipeline Continuous Deployment (CD) orchestré par Jenkins. Le pipeline récupère le code source depuis GitHub, compile et package l'application avec Maven, construit une image Docker, publie cette image sur Docker Hub, puis déploie les conteneurs de l'application et de sa base de données MySQL.

L'objectif est de rendre le déploiement reproductible et de vérifier automatiquement que les services sont lancés.
2. Technologies utilisées

    Git / GitHub : versionnement et hébergement du code source.
    Jenkins : orchestration du pipeline CI/CD.
    Maven : compilation et packaging de l'application Java.
    Docker : création et exécution des conteneurs.
    Docker Hub : stockage et distribution de l'image Docker du backend (repository privé).
    Spring Boot : application backend et API REST.
    MySQL 8.0 : base de données de l'application.

3. Structure du dépôt
DevOps-AppGestionDesProjets/
├── README.md
├── Jenkinsfile
└── backend/
    ├── Dockerfile
    ├── pom.xml
    └── src/
        ├── main/
        └── test/

Le Jenkinsfile est placé à la racine du dépôt. Le projet Maven et le Dockerfile se trouvent dans le dossier backend/.
4. Architecture et déroulement du déploiement
GitHub (branche main)
        |
        v
      Jenkins
        |
        +--> Checkout du code source
        |
        +--> Build Maven / package
        |
        +--> Construction de l'image Docker
        |
        +--> Publication sur Docker Hub (repository privé)
        |
        +--> Déploiement du conteneur MySQL
        |
        +--> Déploiement du backend Spring Boot
        |
        +--> Vérification des conteneurs et des logs

Jenkins exécute les étapes dans l'ordre défini dans le Jenkinsfile. Si une étape échoue, le pipeline ne peut pas être considéré comme un déploiement réussi.
5. Pipeline Jenkins (CD)

Le pipeline est défini dans le fichier Jenkinsfile et comporte les étapes suivantes :
5.1 Checkout

Jenkins récupère le dépôt GitHub sur la branche main :

    Repository : https://github.com/Meriem0014/DevOps-AppGestionDesProjets.git
    Branch : main

Cette étape met le code source à disposition de Jenkins pour la suite du pipeline.
5.2 Build Maven

Dans le répertoire backend, Maven nettoie le précédent build et génère le package de l'application :
mvn clean package -DskipTests

L'option -DskipTests permet de ne pas exécuter les tests automatisés pendant ce pipeline. Le package est ensuite utilisé pour construire l'image Docker.
5.3 Docker Build

Docker construit l'image à partir du Dockerfile présent dans le dossier backend. Le pipeline utilise le nom et le tag suivants :
backend-app:latest
meriem0014/backend-app:latest

Le tag latest désigne la version publiée sous ce tag. L'image est préparée pour être déployée dans le conteneur backend.
5.4 Docker Push

Jenkins s'authentifie auprès de Docker Hub avec un credential Jenkins configuré (le token n'est pas stocké dans le dépôt), puis publie l'image :
meriem0014/backend-app:latest

Le repository Docker Hub utilisé est privé. Les identifiants sont gérés dans Jenkins Credentials et ne doivent pas être écrits dans le Jenkinsfile, le README ou les captures d'écran.
5.5 Deploy MySQL

Le pipeline crée ou réutilise le réseau Docker backend-network et déploie le conteneur MySQL avec l'image mysql:8.0. La base utilisée par l'application est test_db.

La configuration de déploiement prévoit un volume Docker nommé mysql_data afin de conserver les données de la base lors du remplacement ou du redémarrage du conteneur, ainsi qu'un contrôle de santé (health check) pour vérifier que MySQL est prêt.
5.6 Deploy Backend

Le pipeline récupère l'image backend publiée, remplace le conteneur backend précédent si nécessaire et lance le nouveau conteneur sur le réseau backend-network.

Le backend est exposé sur le port :
8080 (hôte) -> 8080 (conteneur)

La connexion à MySQL utilise le nom du service/conteneur mysql sur le réseau Docker, plutôt que localhost depuis le conteneur backend :
jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true
5.7 Verify Deployment

La dernière étape affiche l'état des conteneurs et les logs du backend afin de vérifier que le déploiement a démarré. Le résultat attendu est un conteneur MySQL healthy et un backend en cours d'exécution.
6. Commandes Docker utilisées

Les commandes ci-dessous sont les commandes principales utilisées ou utiles pour reproduire et vérifier les étapes du pipeline.
Construire l'image backend

Depuis le répertoire du projet :
cd backend
docker build -t backend-app:latest .
Taguer l'image pour Docker Hub
docker tag backend-app:latest meriem0014/backend-app:latest
Publier l'image sur Docker Hub

Après authentification Docker Hub (Jenkins utilise un credential sécurisé pour cette opération) :
docker push meriem0014/backend-app:latest
Télécharger l'image depuis Docker Hub
docker pull meriem0014/backend-app:latest
Afficher les conteneurs en cours d'exécution
docker ps
Afficher les images locales
docker images
Afficher les logs du backend
docker logs --tail 50 backend-app
Tester l'API REST
curl -i http://localhost:8080/entreprise/all
7. Vérification du déploiement

Le déploiement a été vérifié à l'aide des éléments suivants :

    Pipeline Jenkins terminé avec des stages au vert dans la vue Stage View.
    Images backend présentes localement : backend-app:latest et meriem0014/backend-app:latest.
    Image MySQL utilisée : mysql:8.0.
    Logs du backend indiquant le démarrage de Spring Boot et de Tomcat.
    Appel HTTP à l'endpoint GET /entreprise/all avec le code de réponse HTTP 200.

La réponse JSON [] signifie que l'endpoint fonctionne mais qu'aucune entreprise n'est encore enregistrée dans la base. Ce n'est pas une erreur HTTP.
8. Captures d'écran / preuves

Les captures fournies sont incluses dans le dossier screenshots/.
8.1 Construction de l'image Docker et extrait du Jenkinsfile

Figure 1 – Sortie de la construction de l'image Docker et extrait du Jenkinsfile.
8.2 Pipeline Jenkins, image Docker, Docker Hub et logs de démarrage

Figure 2 – Pipeline Jenkins avec stages réussis, images Docker locales, repository Docker Hub et extrait des logs de démarrage Spring Boot.
8.3 Test de l'API REST

Figure 3 – Test de l'endpoint GET /entreprise/all : réponse HTTP 200.
Capture à ajouter pour docker ps

Le document de captures fourni ne contient pas une capture clairement dédiée à la commande docker ps montrant simultanément les deux conteneurs (mysql avec l'état healthy et backend-app en cours d'exécution). Ajouter cette capture ici avant la remise finale :

    Figure 4 – Conteneurs MySQL et backend en cours d'exécution (docker ps).

Commande à exécuter :
docker ps

Prendre une capture où l'on voit les noms mysql et backend-app, leurs images, leurs statuts et le mapping du port du backend.
9. Conclusion

Ce travail met en place un pipeline CD Jenkins qui automatise la récupération du code, le packaging Maven, la construction et la publication de l'image Docker, puis le déploiement des conteneurs MySQL et backend. La vérification finale confirme le démarrage de l'application et l'accessibilité de l'API REST via une réponse HTTP 200.

Le déploiement est ainsi automatisé et les étapes ainsi que les preuves d'exécution sont documentées dans ce repository.
