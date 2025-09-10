---
lab:
  topic: Azure events and messaging
  title: Envoyez et recevez des messages depuis Azure Service Bus
  description: Découvrez comment envoyer et recevoir des messages avec Azure Service Bus à l’aide du kit de développement logiciel (SDK) .NET Azure.Messaging.ServiceBus.
---

# Envoyez et recevez des messages depuis Azure Service Bus

Dans cet exercice, vous allez créer et configurer des ressources Azure Service Bus, puis générer une application .NET pour envoyer et recevoir des messages à l’aide du kit de développement logiciel (SDK) **Azure.Messaging.ServiceBus**. Vous allez apprendre à approvisionner un espace de noms et une file d’attente Service Bus, à attribuer des autorisations et à interagir avec les messages par programmation. 

Tâches effectuées dans cet exercice :

* Créez des ressources Azure Service Bus
* Attribuez un rôle à votre nom d’utilisateur Microsoft Entra
* Créez une application console .NET pour envoyer et recevoir des messages
* Nettoyer les ressources

Cet exercice dure environ **30** minutes.

## Créez des ressources Azure Event Hubs

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
    namespaceName=svcbusns$RANDOM
    ```

1. Vous aurez besoin du nom attribué à l’espace de noms plus tard dans cet exercice. Exécutez la commande suivante et enregistrez le résultat.

    ```
    echo $namespaceName
    ```

### Créez un espace de noms et une file d’attente Azure Service Bus

1. Créez un espace de noms de messagerie Service Bus. La commande suivante crée un espace de noms à l’aide de la variable que vous avez créée précédemment. L’opération prend quelques minutes.

    ```bash
    az servicebus namespace create \
        --resource-group $resourceGroup \
        --name $namespaceName \
        --location $location
    ```

1. Maintenant qu’un espace de noms a été créé, vous devez créer une file d’attente pour stocker les messages. Exécutez la commande suivante pour créer une file d’attente nommée **myqueue**.

    ```bash
    az servicebus queue create --resource-group $resourceGroup \
        --namespace-name $namespaceName \
        --name myqueue
    ```

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour permettre à votre application d’envoyer et de recevoir des messages, attribuez à votre utilisateur Microsoft Entra le rôle **Propriétaire des données Azure Service Bus** au niveau de l’espace de noms Service Bus. Cela donne à votre compte d’utilisateur l’autorisation de gérer et d’accéder aux files d’attente et aux rubriques à l’aide d’Azure RBAC. Effectuez les étapes suivantes dans le Cloud Shell.

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource de l’espace de noms Service Bus. L’ID de la ressource définit l’étendue à laquelle le rôle sera attribué pour un espace de noms spécifique.

    ```
    resourceID=$(az servicebus namespace show --name $namespaceName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. Exécutez la commande suivante pour créer et attribuer le rôle **Propriétaire des données Azure Service Bus**.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Service Bus Data Owner" \
        --scope $resourceID
    ```

## Créez une application console .NET pour envoyer et recevoir des messages

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir svcbus
    cd svcbus
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Messaging.ServiceBus** et **Azure.Identity** au projet.

    ```
    dotnet add package Azure.Messaging.ServiceBus
    dotnet add package Azure.Identity
    ```

### Ajoutez le code de démarrage pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Veillez à bien relire les commentaires dans le code et remplacez **<YOUR-NAMESPACE>** par l’espace de noms Service Bus que vous avez enregistré précédemment.

    ```csharp
    using Azure.Messaging.ServiceBus;
    using Azure.Identity;
    using System.Timers;
    
    
    // TODO: Replace <YOUR-NAMESPACE> with your Service Bus namespace
    string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
    string queueName = "myQueue";
    
    
    // ADD CODE TO CREATE A SERVICE BUS CLIENT
    
    
    
    // ADD CODE TO SEND MESSAGES TO THE QUEUE
    
    
    
    // ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
    
    
    
    // Dispose client after use
    await client.DisposeAsync();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajoutez du code pour envoyer des messages à la file d’attente

Il est maintenant temps d’ajouter du code pour créer le client Service Bus et envoyer un lot de messages à la file d’attente.

1. Recherchez le commentaire **// ADD CODE TO CREATE A SERVICE BUS CLIENT** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a Service Bus client using the namespace and DefaultAzureCredential
    // The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
    ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));
    ```

1. Recherchez le commentaire **// ADD CODE TO SEND MESSAGES TO THE QUEUE** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Create a sender for the specified queue
    ServiceBusSender sender = client.CreateSender(queueName);
    
    // create a batch 
    using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();
    
    // number of messages to be sent to the queue
    const int numOfMessages = 3;
    
    for (int i = 1; i <= numOfMessages; i++)
    {
        // try adding a message to the batch
        if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
        {
            // if it is too large for the batch
            throw new Exception($"The message {i} is too large to fit in the batch.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of messages to the Service Bus queue
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
    }
    finally
    {
        // Calling DisposeAsync on client types is required to ensure that network
        // resources and other unmanaged objects are properly cleaned up.
        await sender.DisposeAsync();
    }
    
    Console.WriteLine("Press any key to continue");
    Console.ReadKey();
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis poursuivez l’exercice.

### Ajoutez du code pour traiter les messages dans la file d’attente

1. Recherchez le commentaire **// ADD CODE TO PROCESS MESSAGES FROM THE QUEUE** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Create a processor that we can use to process the messages in the queue
    ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());
    
    // Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
    // messages in the queue to process
    const int idleTimeoutMs = 3000;
    System.Timers.Timer idleTimer = new(idleTimeoutMs);
    idleTimer.Elapsed += async (s, e) =>
    {
        Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
        await processor.StopProcessingAsync();
    };
    
    try
    {
        // add handler to process messages
        processor.ProcessMessageAsync += MessageHandler;
    
        // add handler to process any errors
        processor.ProcessErrorAsync += ErrorHandler;
    
        // start processing 
        idleTimer.Start();
        await processor.StartProcessingAsync();
    
        Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
        // Wait for the processor to stop
        while (processor.IsProcessing)
        {
            await Task.Delay(500);
        }
        idleTimer.Stop();
        Console.WriteLine("Stopped receiving messages");
    }
    finally
    {
        // Dispose processor after use
        await processor.DisposeAsync();
    }
    
    // handle received messages
    async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");
    
        // Reset the idle timer on each message
        idleTimer.Stop();
        idleTimer.Start();
    
        // complete the message. message is deleted from the queue. 
        await args.CompleteMessageAsync(args.Message);
    }
    
    // handle any errors when receiving messages
    Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine(args.Exception.ToString());
        return Task.CompletedTask;
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

## Se connecter à Azure et exécuter l’application

1. Dans le volet de la ligne de commande Cloud Shell, entrez la commande suivante pour vous connecter à Azure.

    ```
    az login
    ```

    **<font color="red">Vous devez vous connecter à Azure, même si la session Cloud Shell est déjà authentifiée.</font>**

    > **Remarque** :dans la plupart des scénarios, l’utilisation d’*az login* suffit. Toutefois, si vous avez des abonnements dans plusieurs locataires, vous devrez peut-être spécifier le locataire à l’aide du paramètre *--tenant*. Pour plus d’informations, consultez [Se connecter à Azure de manière interactive à l’aide d’Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Exécutez la commande suivante pour démarrer l’application console. L’application s’interrompra à différentes étapes et vous invitera à appuyer sur une touche pour continuer. Cela vous permet de consulter les messages directement dans le portail Azure.

    ```
    dotnet run
    ```

    

1. Dans le portail Azure, accédez à l’espace de noms Service Bus que vous avez créé. 

1. Sélectionnez **myqueue** en bas de la fenêtre **Vue d’ensemble**.

1. Sélectionnez **Service Bus Explorer** dans le volet de navigation gauche.

1. Sélectionnez **Aperçu depuis le début** et les trois messages devraient apparaître après quelques secondes.

1. Dans le Cloud Shell, appuyez sur n’importe quelle touche pour continuer et l’application traitera les trois messages. 
 
1. Revenez au portail une fois que l’application a fini de traiter les messages. En sélectionnant à nouveau **Aperçu depuis le début**, vous constaterez qu’il n’y a aucun message dans la file d’attente.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.

