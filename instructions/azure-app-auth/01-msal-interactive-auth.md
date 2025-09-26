---
lab:
  topic: Azure authentication and authorization
  title: Mettre en œuvre l’authentification interactive avec MSAL.NET
  description: Découvrez comment implémenter l’authentification interactive à l’aide du kit de développement logiciel (SDK) MSAL.NET et acquérir un jeton.
---

# Mettre en œuvre l’authentification interactive avec MSAL.NET

Dans cet exercice, vous allez enregistrer une application dans Microsoft Entra ID, puis créer une application console .NET qui utilise MSAL.NET pour effectuer une authentification interactive et obtenir un jeton d’accès pour Microsoft Graph. Vous allez apprendre à configurer les champs d’application de l’authentification, à gérer le consentement des utilisateurs et à voir comment les jetons sont mis en cache pour les exécutions suivantes. 

Tâches effectuées dans cet exercice :

* Inscrire une application avec la plateforme d’identités Microsoft
* Créez une application console .NET qui implémente la classe **PublicClientApplicationBuilder** pour configurer l’authentification.
* Obtenez un jeton de manière interactive à l’aide de l’autorisation Microsoft Graph **user.read**.

Cet exercice dure environ **15** minutes.

## Avant de commencer

Pour compléter cet exercice, vous avez besoin de :

* Un abonnement Azure. Si vous n’en avez pas, vous pouvez [vous inscrire](https://azure.microsoft.com/).

* [Visual Studio Code](https://code.visualstudio.com/) sur l’une des [plateformes prises en charge](https://code.visualstudio.com/docs/supporting/requirements#_platforms).

* [.NET 8](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) ou d’une version ultérieure.

* [Kit de développement C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit) pour Visual Studio Code.

## Inscrire une nouvelle application

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.

1. Dans le portail, recherchez et sélectionnez **Enregistrements d’applications**. 

1. Sélectionnez **+ Nouvel enregistrement**, puis, lorsque la page **Enregistrer une application** s’affiche, entrez les informations d’enregistrement de votre application :

    | Champ | Valeur |
    |--|--|
    | **Nom** | Entrez `myMsalApplication`  |
    | **Types de comptes pris en charge** | Sélectionnez **Comptes dans cet annuaire organisationnel uniquement** |
    | **URI de redirection (facultatif)** | Sélectionnez **Client public/natif (mobile & bureau)** et entrez `http://localhost` dans la case à droite. |

1. Sélectionnez **Inscrire**. Microsoft Entra ID attribue un ID d’application unique (client) à votre application, et vous êtes redirigé vers la page **Vue d’ensemble** de votre application. 

1. Dans la section **Essentiels** de la page **Vue d’ensemble**, enregistrez l’**ID d’application (client)** et l’**ID du répertoire (locataire)**. Ces informations sont nécessaires pour l’application.

    ![Capture d’écran montrant l’emplacement des champs à copier.](./media/01-app-directory-id-location.png)
 
## Créer une application console .NET pour obtenir un jeton

Maintenant que les ressources nécessaires sont déployées sur Azure, l’étape suivante consiste à configurer l’application console. Les étapes suivantes sont effectuées dans votre environnement local.

1. Créez un dossier nommé **authapp**, ou un nom de votre choix, pour le projet.

1. Lancez **Visual Studio Code**, sélectionnez **Fichier > Ouvrir le dossier...** et sélectionnez le dossier du projet.

1. Sélectionnez **Afficher > Terminal** pour ouvrir un terminal.

1. Exécutez la commande suivante dans le terminal VS Code pour créer l’application console .NET.

    ```
    dotnet new console
    ```

1. Exécutez les commandes suivantes pour ajouter les packages **Microsoft.Identity.Client** et **dotenv.net** au projet.

    ```
    dotnet add package Microsoft.Identity.Client
    dotnet add package dotenv.net
    ```

### Configurez l’application console

Dans cette section, vous allez créer et modifier un fichier **.env** pour y stocker les secrets que vous avez notés précédemment. 

1. Sélectionnez **Fichier > Nouveau fichier...** et créez un fichier nommé *.env* dans le dossier du projet.

1. Ouvrez le fichier **.env** et ajoutez le code suivant. Remplacez **YOUR_CLIENT_ID** et **YOUR_TENANT_ID** par les valeurs que vous avez enregistrées précédemment.

    ```
    CLIENT_ID="YOUR_CLIENT_ID"
    TENANT_ID="YOUR_TENANT_ID"
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajoutez le code de démarrage pour le projet

1. Ouvrez le fichier *Program.cs* et remplacez tout contenu existant par le code suivant. Veillez à bien relire les commentaires dans le code.

    ```csharp
    using Microsoft.Identity.Client;
    using dotenv.net;
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Retrieve Azure AD Application ID and tenant ID from environment variables
    string _clientId = envVars["CLIENT_ID"];
    string _tenantId = envVars["TENANT_ID"];
    
    // ADD CODE TO DEFINE SCOPES AND CREATE CLIENT 
    
    
    
    // ADD CODE TO ACQUIRE AN ACCESS TOKEN
    
    
    ```

1. Appuyez sur **ctrl+s** pour enregistrer vos modifications.

### Ajoutez le code pour terminer l’application

1. Recherchez le commentaire **// ADD CODE TO DEFINE SCOPES AND CREATE CLIENT** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire les commentaires dans le code.

    ```csharp
    // Define the scopes required for authentication
    string[] _scopes = { "User.Read" };
    
    // Build the MSAL public client application with authority and redirect URI
    var app = PublicClientApplicationBuilder.Create(_clientId)
        .WithAuthority(AzureCloudInstance.AzurePublic, _tenantId)
        .WithDefaultRedirectUri()
        .Build();
    ```

1. Recherchez le commentaire **// ADD CODE TO ACQUIRE AN ACCESS TOKEN** et ajoutez le code suivant directement après le commentaire. Veillez à bien relire les commentaires dans le code.

    ```csharp
    // Attempt to acquire an access token silently or interactively
    AuthenticationResult result;
    try
    {
        // Try to acquire token silently from cache for the first available account
        var accounts = await app.GetAccountsAsync();
        result = await app.AcquireTokenSilent(_scopes, accounts.FirstOrDefault())
                    .ExecuteAsync();
    }
    catch (MsalUiRequiredException)
    {
        // If silent token acquisition fails, prompt the user interactively
        result = await app.AcquireTokenInteractive(_scopes)
                    .ExecuteAsync();
    }
    
    // Output the acquired access token to the console
    Console.WriteLine($"Access Token:\n{result.AccessToken}");
    ```

1. Appuyez sur **ctrl+s** pour enregistrer le fichier, puis sur **ctrl+q** pour quitter l’éditeur.

## Exécution de l'application

Maintenant que l’application est terminée, il est temps de l’exécuter. 

1. Lancez l’application en exécutant la commande suivante :

    ```
    dotnet run
    ```

1. L’application ouvre le navigateur par défaut, qui vous invite à sélectionner le compte avec lequel vous souhaitez vous authentifier. Si plusieurs comptes sont répertoriés, sélectionnez celui qui est associé au locataire utilisé dans l’application.

1. Si c’est la première fois que vous vous authentifiez auprès de l’application enregistrée, vous recevrez une notification **Autorisations demandées** vous demandant d’autoriser l’application à vous connecter, à lire votre profil et à conserver l’accès aux données auxquelles vous lui avez accordé l’accès. Sélectionnez **Accepter**.

    ![Capture d’écran montrant la notification des autorisations demandées](./media/01-granting-permission.png)

1. Vous devriez voir des résultats similaires à l’exemple ci-dessous dans la console.

    ```
    Access Token:
    eyJ0eXAiOiJKV1QiLCJub25jZSI6IlZF.........
    ```

1. Lancez l’application une deuxième fois et constatez que vous ne recevez plus la notification **Autorisations demandées**. L’autorisation que vous avez accordée précédemment a été mise en cache.

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
