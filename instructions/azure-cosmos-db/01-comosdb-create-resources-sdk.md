---
lab:
  topic: Azure Cosmos DB
  title: Créer des ressources dans Azure Cosmos DB for NoSQL à l’aide de .NET
  description: "Découvrez comment créer des ressources de base de données et de conteneur dans Azure\_Cosmos\_DB avec le kit de développement logiciel (SDK) Microsoft .NET\_v3."
---

# Créer des ressources dans Azure Cosmos DB for NoSQL à l’aide de .NET

Dans cet exercice, vous allez créer un compte Azure Cosmos DB et générer une application console .NET qui utilise le Kit de développement logiciel (SDK) Microsoft Azure Cosmos DB pour créer une base de données, un conteneur et un exemple d’élément. Vous allez apprendre à configurer l’authentification, à effectuer des opérations sur la base de données par programmation et à vérifier vos résultats dans le portail Azure.

Tâches effectuées dans cet exercice :

* Création d’un compte Azure Cosmos DB
* Créez une application console qui crée une base de données, un conteneur et un élément
* Exécuter l’application console et vérifier les résultats

Cet exercice dure environ **30** minutes.

## Création d’un compte Azure Cosmos DB

Dans cette section de l’exercice, vous allez créer un groupe de ressources et un compte Azure Cosmos DB. Vous enregistrez également le point de terminaison et la clé d’accès pour le compte.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Dans la barre d’outils Cloud Shell, dans le menu **Paramètres**, sélectionnez **Accéder à la version classique** (cela est nécessaire pour utiliser l’éditeur de code).

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. La plupart des commandes nécessitent des noms uniques et utilisent les mêmes paramètres. La création de certaines variables réduira les modifications nécessaires aux commandes qui créent les ressources. Exécutez les commandes suivantes pour créer les variables nécessaires. Remplacez **myResourceGroup** par le nom que vous utilisez pour cet exercice.

    ```
    resourceGroup=myResourceGroup
    accountName=cosmosexercise$RANDOM
    ```

1. Exécutez les commandes suivantes pour créer le compte Azure Cosmos DB. Chaque nom de compte doit être unique. 

    ```
    az cosmosdb create --name $accountName \
        --resource-group $resourceGroup
    ```

1.  Exécutez la commande suivante pour récupérer le **documentEndpoint** pour le compte Azure Cosmos DB. Enregistrez le point de terminaison à partir des résultats de la commande, il est nécessaire plus loin dans l’exercice.

    ```
    az cosmosdb show --name $accountName \
        --resource-group $resourceGroup \
        --query "documentEndpoint" --output tsv
    ```

1. Récupérez la clé primaire du compte à l’aide de la commande suivante. Enregistrez la clé primaire à partir des résultats de la commande, elle sera nécessaire plus tard dans l’exercice.

    ```
    az cosmosdb keys list --name $accountName \
        --resource-group $resourceGroup \
        --query "primaryMasterKey" --output tsv
    ```

## Créez des ressources de données et un élément à l’aide d’une application console .NET

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Créez un dossier pour le projet et accédez à ce dossier.

    ```bash
    mkdir cosmosdb
    cd cosmosdb
    ```

1. Créez l’application console .NET.

    ```bash
    dotnet new console
    ```

### Configurez l’application console

1. Exécutez les commandes suivantes pour ajouter les packages **Microsoft.Azure.Cosmos**, **Newtonsoft.Json** et **dotenv.net** au projet.

    ```bash
    dotnet add package Microsoft.Azure.Cosmos --version 3.*
    dotnet add package Newtonsoft.Json --version 13.*
    dotnet add package dotenv.net
    ```

1. Exécutez la commande suivante pour créer le fichier **.env** destiné à contenir les secrets, puis ouvrez-le dans l’éditeur de code.

    ```bash
    touch .env
    code .env
    ```

1. Ajoutez le code suivant au fichier **.env**. Remplacez **YOUR_DOCUMENT_ENDPOINT** et **YOUR_ACCOUNT_KEY** par les valeurs que vous avez enregistrées précédemment.

    ```
    DOCUMENT_ENDPOINT="YOUR_DOCUMENT_ENDPOINT"
    ACCOUNT_KEY="YOUR_ACCOUNT_KEY"
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

Il est maintenant temps de remplacer le code du modèle dans le fichier **Program.cs** à l’aide de l’éditeur dans Cloud Shell.

### Ajouter le code de démarrage du projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```bash
    code Program.cs
    ```

1. Remplacez tout code existant par l’extrait de code suivant. 

    Le code fournit la structure globale de l’application. Passez en revue les commentaires dans le code pour comprendre comment il fonctionne. Pour terminer l’application, vous ajoutez du code dans les zones spécifiées plus tard dans l’exercice. 

    ```csharp
    using Microsoft.Azure.Cosmos;
    using dotenv.net;
    
    string databaseName = "myDatabase"; // Name of the database to create or use
    string containerName = "myContainer"; // Name of the container to create or use
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    string cosmosDbAccountUrl = envVars["DOCUMENT_ENDPOINT"];
    string accountKey = envVars["ACCOUNT_KEY"];
    
    if (string.IsNullOrEmpty(cosmosDbAccountUrl) || string.IsNullOrEmpty(accountKey))
    {
        Console.WriteLine("Please set the DOCUMENT_ENDPOINT and ACCOUNT_KEY environment variables.");
        return;
    }
    
    // CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY
    
    
    try
    {
        // CREATE A DATABASE IF IT DOESN'T ALREADY EXIST
    
    
        // CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY
    
    
        // DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER
    
    
        // ADD THE ITEM TO THE CONTAINER
    
    
    }
    catch (CosmosException ex)
    {
        // Handle Cosmos DB-specific exceptions
        // Log the status code and error message for debugging
        Console.WriteLine($"Cosmos DB Error: {ex.StatusCode} - {ex.Message}");
    }
    catch (Exception ex)
    {
        // Handle general exceptions
        // Log the error message for debugging
        Console.WriteLine($"Error: {ex.Message}");
    }
    
    // This class represents a product in the Cosmos DB container
    public class Product
    {
        public string? id { get; set; }
        public string? name { get; set; }
        public string? description { get; set; }
    }
    ```

Ensuite, vous ajoutez du code dans des zones spécifiques des projets pour créer : le client, la base de données, le conteneur, et ajoutez un échantillon au conteneur.

### Ajouter du code pour créer le client et effectuer des opérations 

1. Ajoutez le code suivant dans l’espace après le commentaire **// CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY**. Ce code définit le client utilisé pour se connecter à votre compte Azure Cosmos DB.

    ```csharp
    CosmosClient client = new(
        accountEndpoint: cosmosDbAccountUrl,
        authKeyOrResourceToken: accountKey
    );
    ```

    >Remarque : il est recommandé d’utiliser **DefaultAzureCredential** de la bibliothèque *Azure Identity*. Selon la configuration de votre abonnement, cela peut nécessiter des configurations supplémentaires dans Azure. 

1. Ajoutez le code suivant dans l’espace après le commentaire **// CREATE A DATABASE IF IT DOESN'T ALREADY EXIST**. 

    ```csharp
    Database database = await client.CreateDatabaseIfNotExistsAsync(databaseName);
    Console.WriteLine($"Created or retrieved database: {database.Id}");
    ```

1. Ajoutez le code suivant dans l’espace après le commentaire **// CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY**. 

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync(
        id: containerName,
        partitionKeyPath: "/id"
    );
    Console.WriteLine($"Created or retrieved container: {container.Id}");
    ```

1. Ajoutez le code suivant dans l’espace après le commentaire **// DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER**. Cela définit l’élément ajouté au conteneur.

    ```csharp
    Product newItem = new Product
    {
        id = Guid.NewGuid().ToString(), // Generate a unique ID for the product
        name = "Sample Item",
        description = "This is a sample item in my Azure Cosmos DB exercise."
    };
    ```

1. Ajoutez le code suivant dans l’espace après le commentaire **// ADD THE ITEM TO THE CONTAINER**. 

    ```csharp
    ItemResponse<Product> createResponse = await container.CreateItemAsync(
        item: newItem,
        partitionKey: new PartitionKey(newItem.id)
    );

    Console.WriteLine($"Created item with ID: {createResponse.Resource.id}");
    Console.WriteLine($"Request charge: {createResponse.RequestCharge} RUs");
    ```

1. Maintenant que le code est terminé, enregistrez votre progression à l’aide de **ctrl + s** pour enregistrer le fichier, puis **ctrl + q** pour quitter l’éditeur.

1. Exécutez la commande suivante dans Cloud Shell pour rechercher d’éventuelles erreurs dans le projet. Si des erreurs s’affichent, ouvrez le fichier *Program.cs* dans l’éditeur et recherchez des erreurs de code manquant ou de collage.

    ```
    dotnet build
    ```

Maintenant que le projet est terminé, il est temps d’exécuter l’application et de vérifier les résultats dans le portail Azure.

## Exécuter l’application et vérifier les résultats

1. Exécutez la commande `dotnet run` si vous êtes dans Cloud Shell. La sortie doit être similaire à l’exemple suivant.

    ```
    Created or retrieved database: myDatabase
    Created or retrieved container: myContainer
    Created item: c549c3fa-054d-40db-a42b-c05deabbc4a6
    Request charge: 6.29 RUs
    ```

1. Dans le portail Azure, accédez à la ressource Azure Cosmos DB que vous avez créée précédemment. Sélectionnez **Explorateur de données** dans le volet de navigation de gauche. Dans l’**Explorateur de données**, sélectionnez **myDatabase**, puis développez **myContainer**. Vous pouvez afficher l’élément que vous avez créé en sélectionnant **Éléments**.

    ![Capture d’écran montrant l’emplacement de Éléments dans l’Explorateur de données.](./media/01/cosmos-data-explorer.png)

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
