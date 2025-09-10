---
lab:
  topic: Secure solutions in Azure
  title: Créer et récupérer des données secrètes depuis Azure Key Vault
  description: 'Découvrez comment créer un coffre de clés, créer et récupérer des secrets à l’aide d’Azure CLI, mais aussi par programmation.'
---

# Créer et récupérer des données secrètes depuis Azure Key Vault

Dans cet exercice, vous allez créer un coffre de clés Azure Key Vault, stocker des secrets à l’aide d’Azure CLI et créer une application console .NET capable de créer et de récupérer des secrets à partir du coffre de clés. Vous allez apprendre à configurer l’authentification, à gérer les secrets par programmation et à nettoyer les ressources une fois l’exercice terminé.  

Tâches effectuées dans cet exercice :

* Créez des ressources Azure Key Vault
* Stockez un secret dans un coffre de clés à l’aide d’Azure CLI
* Créez une application console .NET pour créer et récupérer des secrets
* Nettoyer les ressources

Cet exercice dure environ **30** minutes.

## Créez des ressources Azure Key Vault et ajoutez un secret

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
    keyVaultName=mykeyvaultname$RANDOM
    ```

1. Exécutez la commande suivante pour obtenir le nom du coffre de clés, puis enregistrez-le. Vous en aurez besoin plus tard dans l’exercice.

    ```
    echo $keyVaultName
    ```

1. Exécutez la commande suivante pour créer une ressource Azure Key Vault. Cette opération peut prendre quelques minutes.

    ```
    az keyvault create --name $keyVaultName \
        --resource-group $resourceGroup --location $location
    ```

### Attribuez un rôle à votre nom d’utilisateur Microsoft Entra

Pour créer et récupérer un secret, attribuez à votre utilisateur Microsoft Entra le rôle **Responsable des secrets du coffre de clés**. Cela donne à votre compte d’utilisateur l’autorisation de définir, supprimer et répertorier des secrets. Dans un scénario type, vous pouvez séparer les actions de création et de lecture en attribuant le rôle **Responsable des secrets du coffre de clés** à un groupe, et le rôle **Utilisateur des secrets du coffre de clés** (autorisé à récupérer et à répertorier les secrets) à un autre.

1. Exécutez la commande suivante pour récupérer le **userPrincipalName** de votre compte. Cela correspond à l’utilisateur auquel le rôle sera attribué.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Exécutez la commande suivante pour récupérer l’ID de ressource du coffre de clés. L’ID de la ressource définit l’étendue à laquelle le rôle sera attribué pour un coffre de clés spécifique.

    ```
    resourceID=$(az keyvault show --resource-group $resourceGroup \
        --name $keyVaultName --query id --output tsv)
    ```

1. Exécutez la commande suivante pour créer et attribuer le rôle **Responsable des secrets du coffre de clés**.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Key Vault Secrets Officer" \
        --scope $resourceID
    ```

Ensuite, ajoutez un secret au coffre de clés que vous avez créé.

### Ajoutez et récupérez un secret avec Azure CLI

1. Exécutez la commande suivante pour créer un secret. 

    ```
    az keyvault secret set --vault-name $keyVaultName \
        --name "MySecret" --value "My secret value"
    ```

1. Exécutez la commande suivante pour récupérer le secret afin de vérifier qu’il a bien été défini.

    ```
    az keyvault secret show --name "MySecret" --vault-name $keyVaultName
    ```

    Cette commande retourne une sortie au format JSON. La dernière ligne contient le mot de passe en texte brut. 

    ```json
    "value": "My secret value"
    ```

## Créez une application console .NET pour stocker et récupérer des secrets

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans le Cloud Shell.

>**Conseil :** Redimensionnez le Cloud Shell pour afficher plus d’informations et de code en faisant glisser la bordure supérieure. Vous pouvez également utiliser les boutons de réduction et d’agrandissement pour alterner entre le Cloud Shell et l’interface principale du portail.

1. Exécutez les commandes suivantes pour créer un répertoire contenant le projet et passez dans le répertoire du projet.

    ```
    mkdir keyvault
    cd keyvault
    ```

1. Créez l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Azure.Identity** et **Azure.Security.KeyVault.Secrets** au projet.

    ```
    dotnet add package Azure.Identity
    dotnet add package Azure.Security.KeyVault.Secrets
    ```

### Ajoutez le code de démarrage pour le projet

1. Pour commencer à modifier l’application, exécutez la commande suivante dans Cloud Shell :

    ```
    code Program.cs
    ```

1. Remplacez tout contenu existant par le code suivant. Veillez à remplacer **YOUR-KEYVAULT-NAME** par le nom réel de votre coffre de clés.

    ```csharp
    using Azure.Identity;
    using Azure.Security.KeyVault.Secrets;
    
    // Replace YOUR-KEYVAULT-NAME with your actual Key Vault name
    string KeyVaultUrl = "https://YOUR-KEYVAULT-NAME.vault.azure.net/";
    
    
    // ADD CODE TO CREATE A CLIENT
    
    
    
    // ADD CODE TO CREATE A MENU SYSTEM
    
    
    
    // ADD CODE TO CREATE A SECRET
    
    
    
    // ADD CODE TO LIST SECRETS
    
    
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajoutez le code pour terminer l’application

Il est maintenant temps d’ajouter du code pour terminer l’application.

1. Recherchez le commentaire **// ADD CODE TO CREATE A CLIENT** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Configure authentication options for connecting to Azure Key Vault
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create the Key Vault client using the URL and authentication credentials
    var client = new SecretClient(new Uri(KeyVaultUrl), new DefaultAzureCredential(options));
    ```

1. Recherchez le commentaire **// ADD CODE TO CREATE A MENU SYSTEM** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    // Main application loop - continues until user types 'quit'
    while (true)
    {
        // Display menu options to the user
        Console.Clear();
        Console.WriteLine("\nPlease select an option:");
        Console.WriteLine("1. Create a new secret");
        Console.WriteLine("2. List all secrets");
        Console.WriteLine("Type 'quit' to exit");
        Console.Write("Enter your choice: ");
    
        // Read user input and convert to lowercase for easier comparison
        string? input = Console.ReadLine()?.Trim().ToLower();
        
        // Check if user wants to exit the application
        if (input == "quit")
        {
            Console.WriteLine("Goodbye!");
            break;
        }
    
        // Process the user's menu selection
        switch (input)
        {
            case "1":
                // Call the method to create a new secret
                await CreateSecretAsync(client);
                break;
            case "2":
                // Call the method to list all existing secrets
                await ListSecretsAsync(client);
                break;
            default:
                // Handle invalid input
                Console.WriteLine("Invalid option. Please enter 1, 2, or 'quit'.");
                break;
        }
    }
    ```

1. Recherchez le commentaire **// ADD CODE TO CREATE A SECRET** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    async Task CreateSecretAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("\nCreating a new secret...");
            
            // Get the secret name from user input
            Console.Write("Enter secret name: ");
            string? secretName = Console.ReadLine()?.Trim();
    
            // Validate that the secret name is not empty
            if (string.IsNullOrEmpty(secretName))
            {
                Console.WriteLine("Secret name cannot be empty.");
                return;
            }
            
            // Get the secret value from user input
            Console.Write("Enter secret value: ");
            string? secretValue = Console.ReadLine()?.Trim();
    
            // Validate that the secret value is not empty
            if (string.IsNullOrEmpty(secretValue))
            {
                Console.WriteLine("Secret value cannot be empty.");
                return;
            }
    
            // Create a new KeyVaultSecret object with the provided name and value
            var secret = new KeyVaultSecret(secretName, secretValue);
            
            // Store the secret in Azure Key Vault
            await client.SetSecretAsync(secret);
    
            Console.WriteLine($"Secret '{secretName}' created successfully!");
            Console.WriteLine("Press Enter to continue...");
            Console.ReadLine();
        }
        catch (Exception ex)
        {
            // Handle any errors that occur during secret creation
            Console.WriteLine($"Error creating secret: {ex.Message}");
        }
    }
    ```

1. Recherchez le commentaire **// ADD CODE TO LIST SECRETS** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire le code et les commentaires.

    ```csharp
    async Task ListSecretsAsync(SecretClient client)
    {
        try
        {
            Console.Clear();
            Console.WriteLine("Listing all secrets in the Key Vault...");
            Console.WriteLine("----------------------------------------");
    
            // Get an async enumerable of all secret properties in the Key Vault
            var secretProperties = client.GetPropertiesOfSecretsAsync();
            bool hasSecrets = false;
    
            // Iterate through each secret property to retrieve full secret details
            await foreach (var secretProperty in secretProperties)
            {
                hasSecrets = true;
                try
                {
                    // Retrieve the actual secret value and metadata using the secret name
                    var secret = await client.GetSecretAsync(secretProperty.Name);
                    
                    // Display the secret information to the console
                    Console.WriteLine($"Name: {secret.Value.Name}");
                    Console.WriteLine($"Value: {secret.Value.Value}");
                    Console.WriteLine($"Created: {secret.Value.Properties.CreatedOn}");
                    Console.WriteLine("----------------------------------------");
                }
                catch (Exception ex)
                {
                    // Handle errors for individual secrets (e.g., access denied, secret not found)
                    Console.WriteLine($"Error retrieving secret '{secretProperty.Name}': {ex.Message}");
                    Console.WriteLine("----------------------------------------");
                }
            }
    
            // Inform user if no secrets were found in the Key Vault
            if (!hasSecrets)
            {
                Console.WriteLine("No secrets found in the Key Vault.");
            }
        }
        catch (Exception ex)
        {
            // Handle general errors that occur during the listing operation
            Console.WriteLine($"Error listing secrets: {ex.Message}");
        
        }
        Console.WriteLine("Press Enter to continue...");
        Console.ReadLine();
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

1. Exécutez la commande suivante pour démarrer l’application console. L’application affichera le système de menus de l’application. 

    ```
    dotnet run
    ```

1. Vous avez créé un secret au début de cet exercice, entrez **2** pour le récupérer et l’afficher.

1. Entrez **1** et saisissez un nom et une valeur de secret pour créer un nouveau secret.

1. Répertoriez à nouveau les secrets pour voir votre nouvel ajout.

Entrez **quit** une fois que vous avez terminé d’utiliser l’application.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
