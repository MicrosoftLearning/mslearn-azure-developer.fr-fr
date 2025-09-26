---
lab:
  topic: Secure solutions in Azure
  title: Récupérer les paramètres de configuration à partir d’Azure App Configuration
  description: 'Découvrez comment créer une ressource Azure App Configuration et définir des informations de configuration à l’aide d’Azure CLI. Ensuite, utilisez **ConfigurationBuilder** pour récupérer les paramètres de votre application.'
---

# Récupérer les paramètres de configuration à partir d’Azure App Configuration

Dans cet exercice, vous allez créer une ressource Azure App Configuration, stocker les paramètres de configuration à l’aide de l’interface Azure CLI et créer une application console .NET qui utilise **ConfigurationBuilder** pour récupérer les valeurs de configuration. Vous allez découvrir comment organiser les paramètres à l’aide de clés hiérarchiques et authentifier votre application pour accéder aux données de configuration basées sur le cloud.

Tâches effectuées dans cet exercice :

* Créer une ressource d’Azure App Configuration
* Stockez les informations de configuration de la chaîne de connexion
* Créez une application console .NET pour récupérer les informations de configuration
* Nettoyer les ressources

Cet exercice dure environ **15** minutes.

## Créez une ressource Azure App Configuration et ajoutez les informations de configuration

Dans cette section de l’exercice, vous allez créer les ressources nécessaires dans Azure à l’aide d’Azure CLI.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Dans la barre d’outils Cloud Shell, dans le menu **Paramètres**, sélectionnez **Accéder à la version classique** (cela est nécessaire pour utiliser l’éditeur de code).

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire.

    ```
    az group create --name myResourceGroup --location eastus
    ```

1. La plupart des commandes nécessitent des noms uniques et utilisent les mêmes paramètres. La création de certaines variables réduira les modifications nécessaires aux commandes qui créent les ressources. Exécutez les commandes suivantes pour créer les variables nécessaires. Remplacez **myResourceGroup** par le nom que vous utilisez pour cet exercice. Si vous avez modifié l’emplacement à l’étape précédente, effectuez la même modification dans la variable **location**.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    appConfigName=appconfigname$RANDOM
    ```

1. Exécutez la commande suivante pour obtenir le nom de la ressource App Configuration. Notez le nom, vous en aurez besoin plus tard dans l’exercice.

    ```
    echo $appConfigName
    ```

1. Exécutez la commande suivante pour vous assurer que le fournisseur **Microsoft.AppConfiguration** est enregistré pour votre abonnement.

    ```
    az provider register --namespace Microsoft.AppConfiguration
    ```

1. L’inscription peut prendre quelques minutes. Exécutez la commande suivante pour vérifier l’état de l’enregistrement. Passez à l’étape suivante lorsque les résultats affichent **Enregistré**.

    ```
    az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
    ```

1. Exécutez la commande suivante pour créer une ressource Azure App Configuration. Cette opération peut prendre quelques minutes.

    ```
    az appconfig create --location $location \
        --name $appConfigName \
        --resource-group $resourceGroup
        --sku Free
    ```

    >**Conseil :** Si vous rencontrez des difficultés pour créer la ressource AppConfig à cause de restrictions de quota avec la valeur SKU **Gratuit**, utilisez **Développeur** à la place.
    

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour récupérer les informations de configuration, vous devez attribuer à votre utilisateur Microsoft Entra le rôle **Lecteur de données App Configuration**. 

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource de votre service App Configuration. L’ID de ressource définit l’étendue de l’attribution de rôle.

    ```
    resourceID=$(az appconfig show --resource-group $resourceGroup \
        --name $appConfigName --query id --output tsv)
    ```

1. Exécutez la commande suivante pour créer et attribuer le rôle **Lecteur de données App Configuration**.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "App Configuration Data Reader" \
        --scope $resourceID
    ```

Ensuite, ajoutez une chaîne de connexion d’espace réservé à App Configuration.

### Ajoutez des informations de configuration avec Azure CLI

Dans Azure App Configuration, une clé comme **Dev:conStr** est une clé hiérarchique ou une clé avec espace de noms. Les deux-points (:) servent de délimiteur pour créer une hiérarchie logique, où :

* **Dev** représente l’espace de noms ou le préfixe de l’environnement (indiquant que cette configuration est destinée à l’environnement de développement)
* **conStr** représente le nom de la configuration

Cette structure hiérarchique vous permet d’organiser les paramètres de configuration par environnement, fonctionnalité ou composant d’application, ce qui facilite la gestion et la récupération des paramètres associés.

Exécutez la commande suivante pour stocker la chaîne de connexion de l’espace réservé. 

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```

Cette commande retourne une sortie au format JSON. La dernière ligne contient la valeur en texte brut. 

```json
"value": "connectionString"
```

## Créez une application console .NET pour récupérer les informations de configuration

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir appconfig
    cd appconfig
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Identity** et **Microsoft.Extensions.Configuration.AzureAppConfiguration** au projet.

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
    ```

### Ajoutez le code pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Veillez à remplacer **YOUR_APP_CONFIGURATION_NAME** par le nom que vous avez noté précédemment, et parcourez les commentaires dans le code.

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Configuration.AzureAppConfiguration;
    using Azure.Identity;
    
    // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
    // with the name of your actual App Configuration service
    
    string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
    
    // Configure which authentication methods to use
    // DefaultAzureCredential tries multiple auth methods automatically
    DefaultAzureCredentialOptions credentialOptions = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a configuration builder to combine multiple config sources
    var builder = new ConfigurationBuilder();
    
    // Add Azure App Configuration as a source
    // This connects to Azure and loads configuration values
    builder.AddAzureAppConfiguration(options =>
    {
        
        options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
    });
    
    // Build the final configuration object
    try
    {
        var config = builder.Build();
        
        // Retrieve a configuration value by key name
        Console.WriteLine(config["Dev:conStr"]);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

## Se connecter à Azure et exécuter l’application

1. Dans le Cloud Shell, entrez la commande suivante pour vous connecter à Azure.

    ```
    az login
    ```

    **<font color="red">Vous devez vous connecter à Azure, même si la session Cloud Shell est déjà authentifiée.</font>**

    > **Remarque** :dans la plupart des scénarios, l’utilisation d’*az login* suffit. Toutefois, si vous avez des abonnements dans plusieurs locataires, vous devrez peut-être spécifier le locataire à l’aide du paramètre *--tenant*. Pour plus d’informations, consultez [Se connecter à Azure de manière interactive à l’aide d’Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Exécutez la commande suivante pour démarrer l’application console. L’application affichera la valeur **connectionString** que vous avez attribuée au paramètre **Dev:conStr** précédemment dans cet exercice.

    ```
    dotnet run
    ```

    L’application affichera la valeur **connectionString** que vous avez attribuée au paramètre **Dev:conStr** précédemment dans cet exercice.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
