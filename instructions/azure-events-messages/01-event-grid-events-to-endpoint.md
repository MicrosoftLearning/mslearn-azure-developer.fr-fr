---
lab:
  topic: Azure events and messaging
  title: Acheminez les événements vers un point de terminaison personnalisé avec Azure Event Grid
  description: Découvrez comment utiliser Azure Event Grid pour acheminer des événements vers un point de terminaison personnalisé.
---

# Acheminez les événements vers un point de terminaison personnalisé avec Azure Event Grid

Dans cet exercice, vous allez créer une rubrique Azure Event Grid et un point de terminaison d’application web, puis générer une application console .NET qui envoie des événements personnalisés à la rubrique Event Grid. Vous allez apprendre à configurer des abonnements à des événements, à vous authentifier auprès d’Event Grid et à vérifier que vos événements sont correctement acheminés vers le point de terminaison en les consultant dans l’application web.

Tâches effectuées dans cet exercice :

* Créez des ressources Azure Event Grid
* Activer un fournisseur de ressources Event Grid
* Créez une rubrique dans Event Grid
* Créer un point de terminaison de message
* Abonnez-vous à la rubrique
* Envoyez un événement avec une application console .NET
* Nettoyer les ressources

Cet exercice dure environ **30** minutes.

## Créez des ressources Azure Event Grid

Dans cette section de l’exercice, vous allez créer les ressources nécessaires dans Azure à l’aide d’Azure CLI.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Dans la barre d’outils Cloud Shell, dans le menu **Paramètres**, sélectionnez **Accéder à la version classique** (cela est nécessaire pour utiliser l’éditeur de code).

1. Créez un groupe de ressources pour les ressources nécessaires à cet exercice. Ignorez cette étape si vous disposez déjà d’un groupe de ressources que vous souhaitez utiliser. Remplacez **myResourceGroup** par un nom que vous souhaitez utiliser pour le groupe de ressources. Vous pouvez remplacer **eastus** par une région proche de vous si nécessaire.

    ```bash
    az group create --name myResourceGroup --location eastus
    ```

1. La plupart des commandes nécessitent des noms uniques et utilisent les mêmes paramètres. La création de certaines variables réduira les modifications nécessaires aux commandes qui créent les ressources. Exécutez les commandes suivantes pour créer les variables nécessaires. Remplacez **myResourceGroup** par le nom que vous utilisez pour cet exercice. Si vous avez modifié l’emplacement à l’étape précédente, effectuez la même modification dans la variable **location**.

    ```bash
    let rNum=$RANDOM
    resourceGroup=myResourceGroup
    location=eastus
    topicName="mytopic-evgtopic-${rNum}"
    siteName="evgsite-${rNum}"
    siteURL="https://${siteName}.azurewebsites.net"
    ```

### Activer un fournisseur de ressources Event Grid

Un fournisseur de ressources Azure est un service qui définit et gère des types spécifiques de ressources dans Azure. C’est ce qu’Azure utilise en arrière-plan lorsque vous déployez ou gérez des ressources. Enregistrez le fournisseur de ressources Event Grid avec la commande **az provider register**. 

```bash
az provider register --namespace Microsoft.EventGrid
```

L’inscription peut prendre quelques minutes. Vous pouvez vérifier l’état avec la commande suivante.

```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

> **Remarque :** Cette étape n’est nécessaire que pour les abonnements qui n’ont pas déjà utilisé Event Grid.

### Créez une rubrique dans Event Grid

Créez un sujet à l’aide de la commande **az eventgrid topic create**. Le nom doit être unique, car il fait partie de l’entrée DNS.  

```bash
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```

### Créer un point de terminaison de message

Avant de s’abonner à la rubrique personnalisée, nous devons créer le point de terminaison pour le message d’événement. En règle générale, le point de terminaison entreprend des actions en fonction des données d’événement. Le script suivant utilise une application web prédéfinie qui affiche les messages d’événement. La solution déployée comprend un plan App Service, une offre App Service Web Apps et du code source en provenance de GitHub.

1. Exécutez les commandes suivantes pour créer un point de terminaison de message. La commande **echo** affichera l’URL du site pour le point de terminaison.

    ```bash
    az deployment group create \
        --resource-group $resourceGroup \
        --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
        --parameters siteName=$siteName hostingPlanName=viewerhost
    
    echo "Your web app URL: ${siteURL}"
    ```

    > **Remarque :** Cette commande peut prendre plusieurs minutes.

1. Ouvrez un nouvel onglet dans votre navigateur et accédez à l’URL générée à la fin du script précédent pour vérifier que l’application web est en cours d’exécution. Vous devez voir le site sans messages affichés.

    > **Conseil :** Laissez le navigateur s’exécuter, il sert à afficher les mises à jour.

### Abonnez-vous à la rubrique

Vous vous abonnez à une rubrique Event Grid pour indiquer à Event Grid les événements qui vous intéressent, et où les envoyer. 

1. Abonnez-vous à une rubrique à l’aide de la commande **az eventgrid event-subscription create**. Le script suivant récupère l’ID d’abonnement de votre compte et l’utilise pour créer l’abonnement à l’événement.

    ```bash
    endpoint="${siteURL}/api/updates"
    topicId=$(az eventgrid topic show --resource-group $resourceGroup \
        --name $topicName --query "id" --output tsv)
    
    az eventgrid event-subscription create \
        --source-resource-id $topicId \
        --name TopicSubscription \
        --endpoint $endpoint
    ```

1. Affichez à nouveau votre application web, et notez qu’un événement de validation d’abonnement lui a été envoyé. Sélectionnez l’icône en forme d’œil pour développer les données d’événements. Event Grid envoie l’événement de validation pour que le point de terminaison puisse vérifier qu’il souhaite recevoir des données d’événement. L’application web inclut du code pour valider l’abonnement.

## Envoyez un événement avec une application console .NET

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```bash
    mkdir eventgrid
    cd eventgrid
    ```

1. Créez l’application console .NET.

    ```bash
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Messaging.EventGrid** et **dotenv.net** au projet.

    ```bash
    dotnet add package Azure.Messaging.EventGrid
    dotnet add package dotenv.net
    ```

### Configurez l’application console

Dans cette section, vous récupérez le point de terminaison de la rubrique et la clé d’accès afin de pouvoir les ajouter à un fichier **.env** qui contiendra ces secrets.

1. Exécutez les commandes suivantes pour récupérer l’URL et la clé d’accès de la rubrique que vous avez créée précédemment. Veillez à enregistrer ces valeurs.

    ```bash
    az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
    az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
    ```

1. Exécutez la commande suivante pour créer le fichier **.env** destiné à contenir les secrets, puis ouvrez-le dans l’éditeur de code.

    ```bash
    touch .env
    code .env
    ```

1. Ajoutez le code suivant au fichier **.env**. Remplacez **YOUR_TOPIC_ENDPOINT** et **YOUR_TOPIC_ACCESS_KEY** par les valeurs que vous avez enregistrées précédemment.

    ```
    TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
    TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

Il est maintenant temps de remplacer le code du modèle dans le fichier **Program.cs** à l’aide de l’éditeur dans Cloud Shell.

### Ajoutez le code pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```bash
    code Program.cs
    ```

1. Remplacez tout code existant par le code suivant. Veillez à bien relire les commentaires dans le code.

    ```csharp
    using dotenv.net; 
    using Azure.Messaging.EventGrid; 
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Start the asynchronous process to send an Event Grid event
    ProcessAsync().GetAwaiter().GetResult();
    
    async Task ProcessAsync()
    {
        // Retrieve Event Grid topic endpoint and access key from environment variables
        var topicEndpoint = envVars["TOPIC_ENDPOINT"];
        var topicKey = envVars["TOPIC_ACCESS_KEY"];
        
        // Check if the required environment variables are set
        if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
        {
            Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
            return;
        }
    
        // Create an EventGridPublisherClient to send events to the specified topic
        EventGridPublisherClient client = new EventGridPublisherClient
            (new Uri(topicEndpoint),
            new Azure.AzureKeyCredential(topicKey));
    
        // Create a new EventGridEvent with sample data
        var eventGridEvent = new EventGridEvent(
            subject: "ExampleSubject",
            eventType: "ExampleEventType",
            dataVersion: "1.0",
            data: new { Message = "Hello, Event Grid!" }
        );
    
        // Send the event to Azure Event Grid
        await client.SendEventAsync(eventGridEvent);
        Console.WriteLine("Event sent successfully.");
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

1. Exécutez la commande suivante dans le Cloud Shell pour démarrer l’application console. Le message **Événement envoyé avec succès.** s’affiche lorsque le message est envoyé.

    ```bash
    dotnet run
    ```

1. Affichez votre application web pour voir l’événement que vous venez d’envoyer. Sélectionnez l’icône en forme d’œil pour développer les données d’événements.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.