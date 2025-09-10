---
lab:
  topic: Azure API Management
  title: Importer et configurer une API avec Azure API Management
  description: 'Découvrez comment importer, publier et tester une API conforme à la spécification OpenAPI.'
---

# Importer et configurer une API avec Azure API Management

Dans cet exercice, vous allez créer une instance Azure API Management, importer une API back-end à partir d’une spécification OpenAPI, configurer les paramètres de l’API, notamment l’URL du service web et les conditions d’abonnement, puis tester les opérations de l’API afin de vérifier leur bon fonctionnement.

Tâches effectuées dans cet exercice :

* Créer une instance Gestion des API (APIM) Azure
* Importer une API
* Configurer les paramètres back-end
* Tester l’API

Cet exercice dure environ **20** minutes.

## Création d'une instance Gestion des API

Dans cette partie de l’exercice, vous allez créer un groupe de ressources et un compte de stockage Azure. Vous enregistrez également le point de terminaison et la clé d’accès pour le compte.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus2** par une région proche de vous si nécessaire. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser.

    ```azurecli
    az group create --location eastus2 --name myResourceGroup
    ```

1. Créez quelques variables pour les commandes CLI afin de réduire la quantité de saisie. Remplacez **myLocation** par la valeur que vous avez choisie précédemment. Le nom de l’APIM doit être un nom globalement unique, et le script suivant génère une chaîne aléatoire. Remplacez **myEmail** par une adresse e-mail à laquelle vous avez accès.

    ```bash
    myApiName=import-apim-$RANDOM
    myLocation=myLocation
    myEmail=myEmail
    ```

1. Créez une instance APIM. La commande **az apim create** permet de créer l’instance. Remplacez **myResourceGroup** par la valeur que vous avez choisie précédemment.

    ```bash
    az apim create -n $myApiName \
        --location $myLocation \
        --publisher-email $myEmail  \
        --resource-group myResourceGroup \
        --publisher-name Import-API-Exercise \
        --sku-name Consumption 
    ```
    > **Remarque :** L’opération doit prendre environ cinq minutes. 

## Importer une API back-end

Cette section montre comment importer et publier une API de serveur principal à la spécification OpenAPI.

1. Sur le portail Azure, recherchez et sélectionnez **Services Gestion des API**.

1. Dans l’écran **Services de gestion des API**, sélectionnez l’instance de gestion des API que vous avez créée.

1. Dans le volet de navigation **Service de gestion des API**, sélectionnez **> API**, puis sélectionnez **API**.

    ![Capture d’écran de la section API du volet de navigation.](./media/select-apis-navigation-pane.png)


1. Sélectionnez **OpenAPI** dans la section **Créer à partir d’une définition**, puis définissez le bouton bascule **Basique/Complet** sur **Complet** dans la fenêtre contextuelle qui s’affiche.

    ![Capture d’écran de la boîte de dialogue OpenAPI. Les champs sont détaillés dans le tableau suivant.](./media/create-api.png)

    Utilisez les valeurs du tableau suivant pour remplir le formulaire. Vous pouvez laisser les champs non mentionnés à leur valeur par défaut.

    | Paramètre | Valeur | Description |
    |--|--|--|
    | **Spécification OpenAPI** | `https://bigconference.azurewebsites.net/` | Fait référence au service qui implémente l’API, les demandes sont transférées à cette adresse. La plupart des informations nécessaires dans le formulaire sont automatiquement renseignées lorsque vous entrez cette valeur. |
    | **Modèle d’URL** | Sélectionnez **HTTPS**. | Définit le niveau de sécurité du protocole HTTP accepté par l’API. |

1. Sélectionnez **Créer**.

## Configurer les paramètres de l’API

La valeur *Big Conference API (API de grande conférence)* est créée. Il est maintenant temps de configurer les paramètres de l’API. 

1. Sélectionnez **Paramètres** dans le menu.

1. Entrez `https://bigconference.azurewebsites.net/` dans le champ **URL du service web**.

1. Décochez la case **Abonnement obligatoire**.

1. Cliquez sur **Enregistrer**.

## Tester l’API

Maintenant que l’API a été importée et configurée, il est temps de tester l’API.

1. Sélectionnez **Test** dans la barre de menus. Toutes les opérations disponibles dans l’API s’affichent alors.

1. Recherchez et sélectionnez l’opération **Speakers_Get**. 

1. Sélectionnez **Envoyer**. Vous aurez peut-être besoin de faire défiler la page vers le bas pour consulter la réponse HTTP.

    Le serveur principal répond avec **200 OK** et certaines données.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
