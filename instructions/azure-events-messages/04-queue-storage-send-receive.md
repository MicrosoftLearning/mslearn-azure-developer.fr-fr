---
lab:
  topic: Azure events and messaging
  title: Envoyer et recevoir des messages à partir du stockage File d’attente Azure
  description: Découvrez comment envoyer et recevoir des messages à partir du stockage File d’attente Azure à l’aide du kit de développement logiciel (SDK) .NET Azure.StorageQueues.
---

# Envoyer et recevoir des messages à partir du stockage File d’attente Azure

Dans cet exercice, vous allez créer et configurer des ressources Stockage File d’attente Azure, puis générer une application .NET pour envoyer et recevoir des messages à l’aide du kit de développement logiciel (SDK) **Azure.Storage.Queues**. Vous allez apprendre à approvisionner des ressources de stockage, à gérer les messages en file d’attente et à nettoyer votre environnement une fois l’exercice terminé. 

Tâches effectuées dans cet exercice :

* Créer des ressources de stockage File d'attente Azure
* Attribuez un rôle à votre nom d’utilisateur Microsoft Entra
* Créez une application console .NET pour envoyer et recevoir des messages
* Nettoyer les ressources

Cet exercice dure environ **30** minutes.

## Créer des ressources de stockage File d'attente Azure

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
    storAcctName=storactname$RANDOM
    ```

1. Vous aurez besoin du nom attribué au compte de stockage plus tard dans cet exercice. Exécutez la commande suivante et enregistrez le résultat.

    ```
    echo $storAcctName
    ```

1. Exécutez la commande suivante pour créer un compte de stockage à l’aide de la variable que vous avez créée précédemment. L’opération prend quelques minutes.

    ```bash
    az storage account create --resource-group $resourceGroup \
        --name $storAcctName --location $location --sku Standard_LRS
    ```

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour permettre à votre application d’envoyer et de recevoir des messages, attribuez à votre utilisateur Microsoft Entra le rôle **Contributeur aux données en file d’attente du stockage**. Cela donne à votre compte d’utilisateur l’autorisation de créer des files d’attente et d’envoyer/recevoir des messages à l’aide d’Azure RBAC. Effectuez les étapes suivantes dans le Cloud Shell.

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource du compte de stockage. L’ID de la ressource définit l’étendue à laquelle le rôle sera attribué pour un espace de noms spécifique.

    ```
    resourceID=$(az storage account show --resource-group $resourceGroup \
        --name $storAcctName --query id --output tsv)
    ```

1. Exécutez la commande suivante pour créer et attribuer le rôle **Contributeur aux données en file d’attente du stockage**.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Queue Data Contributor" \
        --scope $resourceID
    ```

## Créez une application console .NET pour envoyer et recevoir des messages

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir queuestor
    cd queuestor
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Storage.Queues** et **Azure.Identity** au projet.

    ```
    dotnet add package Azure.Storage.Queues
    dotnet add package Azure.Identity
    ```

### Ajoutez le code de démarrage pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Veillez à bien relire les commentaires dans le code et remplacez **<YOUR-STORAGE-ACCT-NAME>** par le nom du compte de stockage que vous avez enregistré précédemment.

    ```csharp
    using Azure;
    using Azure.Identity;
    using Azure.Storage.Queues;
    using Azure.Storage.Queues.Models;
    using System;
    using System.Threading.Tasks;
    
    // Create a unique name for the queue
    // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
    string queueName = "myqueue-" + Guid.NewGuid().ToString();
    string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
    
    // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
    
    
    
    // ADD CODE TO SEND AND LIST MESSAGES
    
    
    
    // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
    
    
    
    // ADD CODE TO DELETE MESSAGES AND THE QUEUE
    
    
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajouter du code pour créer un client de file d’attente et créer une file d’attente

Il est maintenant temps d’ajouter du code pour créer le client de stockage de file d’attente et créer une file d’attente.

1. Recherchez le commentaire **// ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Instantiate a QueueClient to create and interact with the queue
    QueueClient queueClient = new QueueClient(
        new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
        new DefaultAzureCredential(options));
    
    Console.WriteLine($"Creating queue: {queueName}");
    
    // Create the queue
    await queueClient.CreateAsync();
    
    Console.WriteLine("Queue created, press Enter to add messages to the queue...");
    Console.ReadLine();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis poursuivez l’exercice.

### Ajoutez du code pour envoyer et répertorier les messages dans une file d’attente

1. Recherchez le commentaire **// ADD CODE TO SEND AND LIST MESSAGES** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Send several messages to the queue with the SendMessageAsync method.
    await queueClient.SendMessageAsync("Message 1");
    await queueClient.SendMessageAsync("Message 2");
    
    // Send a message and save the receipt for later use
    SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
    
    Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
    Console.ReadLine();
    
    // Peeking messages lets you view the messages without removing them from the queue.
    
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to update a message in the queue...");
    Console.ReadLine();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis poursuivez l’exercice.

### Ajoutez du code pour mettre à jour un message et répertorier les résultats

1. Recherchez le commentaire **// ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Update a message with the UpdateMessageAsync method and the saved receipt
    await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
    
    Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
    Console.ReadLine();
    
    
    // Peek messages from the queue to compare updated content
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to delete messages from the queue...");
    Console.ReadLine();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis poursuivez l’exercice.

### Ajoutez du code pour supprimer les messages et la file d’attente

1. Recherchez le commentaire **// ADD CODE TO DELETE MESSAGES AND THE QUEUE** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Delete messages from the queue with the DeleteMessagesAsync method.
    foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
    {
        // "Process" the message
        Console.WriteLine($"Deleting message: {message.MessageText}");
    
        // Let the service know we're finished with the message and it can be safely deleted.
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    Console.WriteLine("Messages deleted from the queue.");
    Console.WriteLine("\nPress Enter key to delete the queue...");
    Console.ReadLine();
    
    // Delete the queue with the DeleteAsync method.
    Console.WriteLine($"Deleting queue: {queueClient.Name}");
    await queueClient.DeleteAsync();
    
    Console.WriteLine("Done");
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

1. Développez **> Stockage de données** dans le menu de navigation gauche et sélectionnez **Files d’attente**.

1. Sélectionnez la file d’attente créée par l’application pour afficher les messages envoyés et surveiller l’activité de l’application.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.

