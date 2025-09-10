---
lab:
  topic: Azure container services
  title: Déployer un conteneur sur Azure Container Instances à l’aide des commandes Azure CLI
  description: Découvrez comment utiliser les commandes Azure CLI pour déployer un conteneur dans Azure Container Instances.
---

# Déployer un conteneur sur Azure Container Instances à l’aide des commandes Azure CLI

Dans cet exercice, vous allez déployer et exécuter un conteneur dans Azure Container Instances (ACI) à l’aide d’Azure CLI. Vous apprendrez à créer un groupe de conteneurs, à définir les paramètres des conteneurs et à vérifier que votre application conteneurisée s'exécute dans le cloud.

Tâches effectuées dans cet exercice :

* Créez des ressources Azure Container Instance dans Azure
* Créez et déployez un conteneur
* Vérifier que le conteneur est en cours d’exécution

Cet exercice dure environ **15** minutes.

## Créer un groupe de ressources

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser.

    ```
    az group create --location eastus --name myResourceGroup
    ```

## Créez et déployez un conteneur

Vous créez un conteneur en spécifiant un nom, une image Docker et un groupe de ressources Azure dans la commande **az container create**. Vous exposez le conteneur sur Internet en spécifiant une étiquette de nom DNS.

1. Exécutez la commande suivante pour créer un nom DNS utilisé pour exposer votre conteneur à Internet. Votre nom DNS doit être unique, exécutez cette commande à partir de Cloud Shell pour créer une variable qui contient un nom unique.

    ```bash
    DNS_NAME_LABEL=aci-example-$RANDOM
    ```

1. Exécutez la commande suivante pour créer une instance de conteneur. Remplacez **myResourceGroup** et **myLocation** par les valeurs que vous avez utilisées précédemment. L’exécution de cette opération prend quelques minutes.

    ```bash
    az container create --resource-group myResourceGroup \
        --name mycontainer \
        --image mcr.microsoft.com/azuredocs/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL --location myLocation \
        --os-type Linux \
        --cpu 1 \
        --memory 1.5 
    ```

    Dans la commande précédente, **$DNS_NAME_LABEL** spécifie votre nom DNS. Le nom de l’image, **mcr.microsoft.com/azuredocs/aci-helloworld**, fait référence à une image Docker qui exécute une application web Node.js de base.

Passez à la section suivante une fois la commande **az container create** terminée.

## Vérifier que le conteneur est en cours d’exécution

Vous pouvez vérifier l’état de génération des conteneurs à l’aide de la commande **az container show**. 

1. Exécutez la commande suivante pour vérifier l’état d’approvisionnement du conteneur que vous avez créé. Remplacez **myResourceGroup** par la valeur que vous avez utilisée précédemment.

    ```bash
    az container show --resource-group myResourceGroup \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table 
    ```

    Vous voyez le nom de domaine complet du conteneur et son état de provisionnement. Voici un exemple.

    ```
    FQDN                                    ProvisioningState
    --------------------------------------  -------------------
    aci-wt.eastus.azurecontainer.io         Succeeded
    ```

    > **Remarque :** Si votre conteneur est dans l’état **Création**, attendez quelques instants et réexécutez la commande jusqu’à son état soit **Réussite**.

1. Dans un navigateur, accédez au nom de domaine complet de votre conteneur pour le voir en cours d’exécution. Il est possible qu’un avertissement indiquant que le site n’est pas sécurisé s’affiche.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
