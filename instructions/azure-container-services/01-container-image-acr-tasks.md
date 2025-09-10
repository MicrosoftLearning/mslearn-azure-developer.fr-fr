---
lab:
  topic: Azure container services
  title: Générer et exécuter une image conteneur avec Azure Container Registry Tasks
  description: Découvrez comment utiliser les commandes Azure CLI pour générer et exécuter des images conteneur avec Azure Container Registry Tasks.
---

# Générer et exécuter une image conteneur avec Azure Container Registry Tasks

Dans cet exercice, vous allez générer une image conteneur à partir du code de votre application et l’envoyer vers Azure Container Registry à l’aide d’Azure CLI. Vous apprendrez comment préparer votre application pour la conteneurisation, créer une instance ACR et stocker votre image de conteneur dans Azure.

Tâches effectuées dans cet exercice :

* Créez une ressource Azure Container Registry
* Générez et envoyez (push) une image à partir d’un fichier Dockerfile
* Vérifier les résultats
* Exécutez l’image dans Azure Container Registry

Cet exercice dure environ **20** minutes.

## Créez une ressource Azure Container Registry

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. Exécutez la commande suivante pour créer un registre de conteneurs de base. Le nom du registre doit être unique dans Azure et contenir entre 5 et 50 caractères alphanumériques. Remplacez **myResourceGroup** par le nom que vous avez utilisé précédemment, et **myContainerRegistry** par une valeur unique.

    ```bash
    az acr create --resource-group myResourceGroup \
        --name myContainerRegistry --sku Basic
    ```

    > **Remarque :** La commande crée un registre *simple*, une option à coût optimisé pour les développeurs qui découvrent Azure Container Registry.

## Générez et envoyez (push) une image à partir d’un fichier Dockerfile

Ensuite, vous générez et envoyez (push) une image basée sur un fichier Dockerfile.

1. Exécutez la commande suivante pour créer le fichier Dockerfile. Le fichier Dockerfile contient une seule ligne qui fait référence à l’image *hello-world* hébergée dans le registre de conteneurs Microsoft.

    ```bash
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

1. Exécutez la commande **az acr build** suivante, qui génère l’image et, une fois celle-ci générée, l’envoie vers votre registre. Remplacez **myContainerRegistry** par le nom que vous avez créé précédemment.

    ```bash
    az acr build --image sample/hello-world:v1  \
        --registry myContainerRegistry \
        --file Dockerfile .
    ```

    Voici un court échantillon de la sortie de la commande précédente montrant les dernières lignes avec les résultats finaux. Vous pouvez voir dans le champ *référentiel* que l’image *sample/hello-word* est répertoriée.

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

## Vérifier les résultats

1. Exécutez la commande suivante pour répertorier les référentiels dans votre registre. Remplacez **myContainerRegistry** par le nom que vous avez créé précédemment.

    ```bash
    az acr repository list --name myContainerRegistry --output table
    ```

    Sortie :

    ```
    Result
    ----------------
    sample/hello-world
    ```

1. Exécutez la commande suivante pour répertorier les balises du référentiel **sample/hello-world**. Remplacez **myContainerRegistry** par le nom que vous avez utilisé précédemment.

    ```bash
    az acr repository show-tags --name myContainerRegistry \
        --repository sample/hello-world --output table
    ```

    Sortie :

    ```
    Result
    --------
    v1
    ```

## Exécuter l’image dans ACR

1. Exécutez l’image conteneur *sample/hello-world:v1* à partir de votre registre de conteneurs à l’aide de la commande **az acr run**. L’exemple suivant utilise **$Registry** pour spécifier le registre dans lequel vous exécutez la commande. Remplacez **myContainerRegistry** par le nom que vous avez utilisé précédemment.

    ```bash
    az acr run --registry myContainerRegistry \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    Le paramètre **cmd** dans cet exemple exécute le conteneur dans sa configuration par défaut, mais **cmd** prend en charge d’autres paramètres **docker run** ou même d’autres commandes **docker**. 

    L’exemple de sortie suivant est raccourci :

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
