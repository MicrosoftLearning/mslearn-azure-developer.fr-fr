---
lab:
  topic: Azure Storage
  title: Créer des ressources de stockage Blob à l’aide de la bibliothèque de client .NET
  description: 'Découvrez comment utiliser la bibliothèque cliente .NET du stockage Azure pour créer des conteneurs, télécharger et répertorier des blobs, et supprimer des conteneurs.'
---

# Créer des ressources de stockage Blob à l’aide de la bibliothèque de client .NET

Dans cet exercice, vous allez créer un compte de stockage Azure et générer une application console .NET à l’aide de la bibliothèque cliente Azure Storage Blob afin de créer des conteneurs, télécharger des fichiers vers le stockage blob, répertorier les blobs et télécharger des fichiers. Vous allez apprendre à vous authentifier avec Azure, à effectuer des opérations de stockage Blob par programmation et à vérifier les résultats dans le portail Azure.

Tâches effectuées dans cet exercice :

* Préparer les ressources Azure
* Créez une application console pour créer et télécharger des données
* Exécutez l’application et vérifiez les résultats
* Nettoyer les ressources

Cet exercice dure environ **30** minutes.

## Création d’un compte de stockage Azure

Dans cette section de l’exercice, vous allez créer les ressources nécessaires dans Azure à l’aide d’Azure CLI.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Dans la barre d’outils Cloud Shell, dans le menu **Paramètres**, sélectionnez **Accéder à la version classique** (cela est nécessaire pour utiliser l’éditeur de code).

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus2** par une région proche de vous si nécessaire. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser.

    ```
    az group create --location eastus2 --name myResourceGroup
    ```

1. La plupart des commandes nécessitent des noms uniques et utilisent les mêmes paramètres. La création de certaines variables réduira les modifications nécessaires aux commandes qui créent les ressources. Exécutez les commandes suivantes pour créer les variables nécessaires. Remplacez **myResourceGroup** par le nom que vous utilisez pour cet exercice.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    accountName=storageacct$RANDOM
    ```

1. Exécutez les commandes suivantes pour créer le compte de stockage Azure. Chaque nom de compte doit être unique. La première commande crée une variable avec un nom unique pour votre compte de stockage. Enregistrez le nom de votre compte à partir de la sortie de la commande **echo**. 

    ```
    az storage account create --name $accountName \
        --resource-group $resourceGroup \
        --location $location \
        --sku Standard_LRS 
    
    echo $accountName
    ```

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour permettre à votre application de créer des ressources et des éléments, attribuez à votre utilisateur Microsoft Entra le rôle **Propriétaire des données de blobs de stockage**. Effectuez les étapes suivantes dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource du compte de stockage. L’ID de la ressource définit l’étendue à laquelle le rôle sera attribué pour un espace de noms spécifique.

    ```
    resourceID=$(az storage account show --name $accountName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. Exécutez la commande suivante pour créer et attribuer le rôle **Propriétaire des données de blobs de stockage**. Ce rôle vous donne les autorisations nécessaires pour gérer les conteneurs et les éléments.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Blob Data Owner" \
        --scope $resourceID
    ```

## Créez une application console .NET pour créer des conteneurs et des éléments

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir azstor
    cd azstor
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages requis dans l’application.

    ```
    dotnet add package Azure.Storage.Blobs
    dotnet add package Azure.Identity
    ```

1. Exécutez la commande suivante pour créer un dossier **données** dans votre projet. 

    ```
    mkdir data
    ```

Il est maintenant temps d’ajouter le code pour le projet.

### Ajoutez le code de démarrage pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Veillez à bien relire les commentaires dans le code.

    ```csharp
    using Azure.Storage.Blobs;
    using Azure.Storage.Blobs.Models;
    using Azure.Identity;
    
    Console.WriteLine("Azure Blob Storage exercise\n");
    
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Run the examples asynchronously, wait for the results before proceeding
    await ProcessAsync();
    
    Console.WriteLine("\nPress enter to exit the sample application.");
    Console.ReadLine();
    
    async Task ProcessAsync()
    {
        // CREATE A BLOB STORAGE CLIENT
        
    
    
        // CREATE A CONTAINER
        
    
    
        // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
        
    
    
        // UPLOAD THE FILE TO BLOB STORAGE
        
    
    
        // LIST BLOBS IN THE CONTAINER
        
    
    
        // DOWNLOAD THE BLOB TO A LOCAL FILE
        
    
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.


## Ajoutez du code pour terminer le projet

Tout au long du reste de l’exercice, vous ajoutez du code dans des zones spécifiques afin de créer l’application complète. 

1. Recherchez le commentaire **// CREATE A BLOB STORAGE CLIENT**, puis ajoutez le code suivant directement sous le commentaire. Le **BlobServiceClient** sert de point d’entrée principal pour la gestion des conteneurs et des blobs dans un compte de stockage. Le client utilise *DefaultAzureCredential* pour l’authentification. Veillez à remplacer **YOUR_ACCOUNT_NAME** par le nom que vous avez enregistré précédemment.

    ```csharp
    // Create a credential using DefaultAzureCredential with configured options
    string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
    
    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);
    
    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.

1. Recherchez le commentaire **// CREATE A CONTAINER**, puis ajoutez le code suivant directement sous le commentaire. La création d’un conteneur implique la création d’une instance de la classe **BlobServiceClient**, puis l’appel de la méthode **CreateBlobContainerAsync** pour créer le conteneur dans votre compte de stockage. Une valeur GUID est ajoutée au nom du conteneur pour s’assurer qu’il est unique. La méthode **CreateBlobContainerAsync** échoue si le conteneur existe déjà.

    ```csharp
    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();
    
    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);
    
    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.

1. Trouvez le commentaire **// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE**, puis ajoutez le code suivant directement sous le commentaire. Cela crée un fichier dans le répertoire de données qui est téléchargé vers le conteneur.

    ```csharp
    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);
    
    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.

1. Recherchez le commentaire **// UPLOAD THE FILE TO BLOB STORAGE**, puis ajoutez le code suivant directement sous le commentaire. Le code obtient une référence à un objet **BlobClient** en appelant la méthode **GetBlobClient** sur le conteneur créé dans la section précédente. Il télécharge ensuite un fichier local généré à l’aide de la méthode **UploadAsync**. Cette méthode crée l’objet blob s’il n’existe pas déjà, et le remplace s’il existe.

    ```csharp
    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);
    
    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
    
    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }
    
    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.

1. Recherchez le commentaire **// LIST BLOBS IN THE CONTAINER**, puis ajoutez le code suivant directement sous le commentaire. Vous répertoriez les blobs dans le conteneur à l’aide de la méthode **GetBlobsAsync**. Dans ce cas, un seul blob a été ajouté au conteneur. Il n’y a donc qu’un blob répertorié. 

    ```csharp
    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }
    
    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications, puis passez à l’étape suivante.

1. Recherchez le commentaire **// DOWNLOAD THE BLOB TO A LOCAL FILE**, puis ajoutez le code suivant directement sous le commentaire. Le code utilise la méthode **DownloadAsync** pour télécharger le blob créé précédemment vers votre système de fichiers local. L’exemple de code ajoute un suffixe « DOWNLOADED » au nom du blob afin que vous puissiez voir les deux fichiers dans le système de fichiers local. 

    ```csharp
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

## Se connecter à Azure et exécuter l’application

1. Dans le volet de la ligne de commande Cloud Shell, entrez la commande suivante pour vous connecter à Azure.

    ```
    az login
    ```

    **<font color="red">Vous devez vous connecter à Azure, même si la session Cloud Shell est déjà authentifiée.</font>**

    > **Remarque** :dans la plupart des scénarios, l’utilisation d’*az login* suffit. Toutefois, si vous avez des abonnements dans plusieurs locataires, vous devrez peut-être spécifier le locataire à l’aide du paramètre *--tenant*. Pour plus d’informations, consultez [Se connecter à Azure de manière interactive à l’aide d’Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Exécutez la commande suivante pour démarrer l’application console. L’application s’interrompra à plusieurs reprises pendant son exécution, jusqu’à ce que vous appuyiez sur une touche pour continuer. Cela vous permet de consulter les messages directement dans le portail Azure.

    ```
    dotnet run
    ```

1. Dans le portail Azure, accédez au compte de stockage Azure que vous avez créé. 

1. Développez **> Stockage de données** dans le menu de navigation gauche et sélectionnez **Conteneurs**.

1. Sélectionnez le conteneur créé par l’application pour afficher le blob qui a été téléchargé.

1. Exécutez les deux commandes ci-dessous pour accéder au répertoire **données** et afficher la liste des fichiers qui ont été téléversés et téléchargés.

    ```
    cd data
    ls
    ```

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet 

