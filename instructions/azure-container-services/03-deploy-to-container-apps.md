---
lab:
  topic: Azure container services
  title: Déployer un conteneur dans Azure Container Apps à l’aide d’Azure CLI
  description: Découvrez comment utiliser les commandes Azure CLI pour créer un environnement Azure Container Apps sécurisé et déployer un conteneur.
---

# Déployer un conteneur dans Azure Container Apps à l’aide d’Azure CLI

Dans cet exercice, vous allez déployer une application conteneurisée vers Azure Container Apps à l’aide d’Azure CLI. Vous apprendrez à créer un environnement d'application conteneurisée, à déployer votre conteneur et à vérifier que votre application s'exécute dans Azure.

Tâches effectuées dans cet exercice :

* Créer des ressources dans Azure
* Créer un environnement Azure Container Apps
* Déployez une application conteneur dans l’environnement

Cet exercice dure environ **15** minutes.

## Créez un groupe de ressources et préparez l’environnement Azure

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser.

    ```azurecli
    az group create --location eastus --name myResourceGroup
    ```

1. Exécutez la commande suivante pour vous assurer que la dernière version de l’extension Azure Container Apps pour l’interface de ligne de commande est installée.

    ```azurecli
    az extension add --name containerapp --upgrade
    ```

### Enregistrer les espaces de noms

Deux espaces de noms doivent être enregistrés pour Azure Container Apps. Vous devez vous assurer qu’ils le sont dans les étapes suivantes. Chaque enregistrement peut prendre quelques minutes si ces espaces de noms ne sont pas déjà configurés dans votre abonnement. 

1. Enregistrez l’espace de noms **Microsoft.App**. 

    ```bash
    az provider register --namespace Microsoft.App
    ```

1. Enregistrez le fournisseur **Microsoft.OperationalInsights** pour l’espace de travail Azure Monitor Log Analytics si vous ne l’avez jamais utilisé auparavant.

    ```bash
    az provider register --namespace Microsoft.OperationalInsights
    ```

## Créer un environnement Azure Container Apps

Un environnement dans Azure Container Apps crée une limite sécurisée autour d’un groupe d’applications de conteneur. Les applications de conteneur déployées dans le même environnement sont déployées dans le même réseau virtuel et écrivent les journaux dans le même espace de travail Log Analytics.

1. Créez un environnement à l’aide de la commande **az containerapp env create**. Remplacez **myResourceGroup** et **myLocation** par les valeurs que vous avez utilisées précédemment. L’exécution de cette opération prend quelques minutes.

    ```bash
    az containerapp env create \
        --name my-container-env \
        --resource-group myResourceGroup \
        --location myLocation
    ```

## Déployez une application conteneur dans l’environnement

Une fois le déploiement de l’environnement d’application conteneur terminé, vous pouvez déployer une image conteneur dans votre environnement.

1. Déployez une image conteneur d’application exemple à l’aide de la commande **containerapp create**. Remplacez **myResourceGroup** par la valeur que vous avez utilisée précédemment.

    ```bash
    az containerapp create \
        --name my-container-app \
        --resource-group myResourceGroup \
        --environment my-container-env \
        --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
        --target-port 80 \
        --ingress 'external' \
        --query properties.configuration.ingress.fqdn
    ```

    En définissant **--ingress** sur **external**, vous rendez l’application conteneur accessible aux requêtes publiques. La commande retourne un lien pour accéder à votre application.

    ```
    Container app created. Access your app at <url>
    ```

Pour vérifier le déploiement, sélectionnez l’URL renvoyée par la commande **az containerapp create** afin de vérifier que l’application conteneur est en cours d’exécution.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
