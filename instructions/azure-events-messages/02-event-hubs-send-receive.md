---
lab:
  topic: Azure events and messaging
  title: Envoyer et récupérer des événements à partir d’Azure Event Hubs
  description: Découvrez comment envoyer et récupérer des événements à partir d’Azure Event Hubs avec le kit de développement logiciel (SDK) .NET Azure.Messaging.EventHubs.
---

# Envoyer et récupérer des événements à partir d’Azure Event Hubs

Dans cet exercice, vous créez des ressources Azure Event Hubs et développez une application console .NET pour envoyer et recevoir des événements à l’aide du kit de développement logiciel (SDK) **Azure.Messaging.EventHubs**. Vous découvrez comment approvisionner des ressources cloud, interagir avec Event Hubs et nettoyer votre environnement une fois terminé.

Tâches effectuées dans cet exercice :

* Créer un groupe de ressources
* Créez des ressources Azure Event Hubs
* Créez une application console .NET pour envoyer et récupérer des événements
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
    namespaceName=eventhubsns$RANDOM
    ```

### Créer un espace de noms et un hub d’événements Azure Event Hubs

Un espace de noms Azure Event Hubs est un conteneur logique pour les ressources de hub d’événements au sein d’Azure. Il fournit un conteneur à portée unique dans lequel vous pouvez créer un ou plusieurs hubs d’événements, qui sont utilisés pour ingérer, traiter et stocker de grands volumes de données d’événements. Les instructions suivantes sont exécutées dans le Cloud Shell. 

1. Exécutez la commande suivante pour créer un espace de noms Event Hub.

    ```
    az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
    ```

1. Exécutez la commande suivante pour créer un hub d’événements nommé **myEventHub** dans l’espace de noms Event Hubs. 

    ```
    az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
      --namespace-name $namespaceName
    ```

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour permettre à votre application d’envoyer et de recevoir des messages, attribuez à votre utilisateur Microsoft Entra le rôle **Propriétaire des données Azure Event Hubs** au niveau de l’espace de noms Event Hubs. Cela donne à votre compte d’utilisateur l’autorisation de gérer et d’accéder aux files d’attente et aux rubriques à l’aide d’Azure RBAC. Effectuez les étapes suivantes dans le Cloud Shell.

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource de l’espace de noms Event Hubs. L’ID de la ressource définit l’étendue à laquelle le rôle sera attribué pour un espace de noms spécifique.

    ```
    resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
        --name $namespaceName --query id --output tsv)
    ```
1. Exécutez la commande suivante pour créer et attribuer le **Propriétaire de données Azure Event Hubs**, qui vous donne l’autorisation d’envoyer et de récupérer des événements.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Event Hubs Data Owner" \
        --scope $resourceID
    ```

## Envoyez et récupérez des événements avec une application console .NET

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir eventhubs
    cd eventhubs
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Messaging.EventHubs** et **Azure.Identity** au projet.

    ```
    dotnet add package Azure.Messaging.EventHubs
    dotnet add package Azure.Identity
    ```

Il est maintenant temps de remplacer le code du modèle dans le fichier **Program.cs** à l’aide de l’éditeur dans Cloud Shell.

### Ajoutez le code de démarrage pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Assurez-vous de lire les commentaires dans le code et remplacez **YOUR_EVENT_HUB_NAMESPACE** par votre espace de noms de hub d’événements.

    ```csharp
    using Azure.Messaging.EventHubs;
    using Azure.Messaging.EventHubs.Producer;
    using Azure.Messaging.EventHubs.Consumer;
    using Azure.Identity;
    using System.Text;
    
    // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
    string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
    string eventHubName = "myEventHub"; 
    
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Number of events to be sent to the event hub
    int numOfEvents = 3;
    
    // CREATE A PRODUCER CLIENT AND SEND EVENTS
    
    
    
    // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
    
    
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajoutez le code pour terminer l’application

Dans cette section, vous ajoutez du code pour créer les clients producteurs et consommateurs afin d’envoyer et de recevoir des événements.

1. Recherchez le commentaire **// CREATE A PRODUCER CLIENT AND SEND EVENTS** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire les commentaires dans le code.

    ```csharp
    // Create a producer client to send events to the event hub
    EventHubProducerClient producerClient = new EventHubProducerClient(
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    // Create a batch of events 
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    
    
    // Adding a random number to the event body and sending the events. 
    var random = new Random();
    for (int i = 1; i <= numOfEvents; i++)
    {
        int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
        string eventBody = $"Event {randomNumber}";
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
        {
            // if it is too large for the batch
            throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of events to the event hub
        await producerClient.SendAsync(eventBatch);
    
        Console.WriteLine($"A batch of {numOfEvents} events has been published.");
        Console.WriteLine("Press Enter to retrieve and print the events...");
        Console.ReadLine();
    }
    finally
    {
        await producerClient.DisposeAsync();
    }
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

1. Recherchez le commentaire **// CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire les commentaires dans le code.

    ```csharp
    // Create an EventHubConsumerClient
    await using var consumerClient = new EventHubConsumerClient(
        EventHubConsumerClient.DefaultConsumerGroupName,
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    Console.Clear();
    Console.WriteLine("Retrieving all events from the hub...");
    
    // Get total number of events in the hub by summing (last - first + 1) for all partitions
    // This count is used to determine when to stop reading events
    long totalEventCount = 0;
    string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
    foreach (var partitionId in partitionIds)
    {
        PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
        if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
        {
            totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
        }
    }
    
    // Start retrieving events from the event hub and print to the console
    int retrievedCount = 0;
    await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
    {
        if (partitionEvent.Data != null)
        {
            string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
            Console.WriteLine($"Retrieved event: {body}");
            retrievedCount++;
            if (retrievedCount >= totalEventCount)
            {
                Console.WriteLine("Done retrieving events. Press Enter to exit...");
                Console.ReadLine();
                return;
            }
        }
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

1. Lancez l’application en exécutant la commande suivante :

    ```
    dotnet run
    ```

    Après quelques secondes, vous devriez voir la sortie similaire à l’exemple suivant :
    
    ```
    A batch of 3 events has been published.
    Press Enter to retrieve and print the events...
    
    Retrieving all events from the hub...
    Retrieved event: Event 4
    Retrieved event: Event 96
    Retrieved event: Event 74
    Done retrieving events. Press Enter to exit...
    ```

L’application envoie toujours trois événements au hub, mais elle récupère tous les événements présents dans le hub. Si vous exécutez l’application plusieurs fois, un nombre croissant d’événements sera récupéré. Les nombres aléatoires utilisés pour la création d’événements vous aident à identifier les différents événements.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées. 
