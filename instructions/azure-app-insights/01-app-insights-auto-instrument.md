---
lab:
  topic: Application Insights
  title: Surveiller une application avec l’auto-instrumentation
  description: 'Découvrez comment surveiller une application dans Application Insights sans modifier le code en configurant l’auto-instrumentation '
---

# Surveiller une application avec l’auto-instrumentation

Dans cet exercice, vous allez créer une application web Azure App Service avec Application Insights activé, configurer l’auto-instrumentation sans modifier le code, créer et déployer une application Blazor, puis afficher les métriques de l’application et les données d’erreur dans Application Insights. Mettre en œuvre une surveillance et une observabilité complètes des applications, sans avoir à modifier votre code, simplifie les déploiements et les migrations.

Tâches effectuées dans cet exercice :

* Créez une ressource d’application web avec Application Insights activé
* Configurez l’instrumentation pour l’application web.
* Créez une application Blazor et déployez-la dans la ressource d’application web.
* Consultez l’activité des applications dans Application Insights
* Nettoyer les ressources

Cet exercice dure environ **20** minutes.

## Créer des ressources dans Azure

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Sélectionnez l’option **+ Créer une ressource** située dans le titre **Services Azure** en haut de la page d’accueil. 
1. Dans la barre de recherche **Rechercher dans la Place de marché**, entrez *Application web* et appuyez sur **Entrée** pour commencer la recherche.
1. Dans la vignette Application web, sélectionnez la liste déroulante **Créer**, puis sélectionnez **Application web**.

    ![Capture d’écran de la vignette Application web.](./media/create-web-app-tile.png)

La sélection de **Créer** ouvre un modèle avec quelques onglets pour renseigner des informations sur votre déploiement. Les étapes suivantes vous guident tout au long des modifications à apporter dans les onglets pertinents.

1. Renseignez l’onglet **Informations de base** en utilisant les informations contenues dans le tableau suivant :

    | Setting | Action |
    |--|--|
    | **Abonnement** | Conservez les valeurs par défaut. |
    | **Groupe de ressources** | Sélectionnez Créer nouveau, entrez `rg-WebApp`, puis sélectionnez OK. Vous pouvez également sélectionner un groupe de ressources existant si vous préférez. |
    | **Nom** | Entrez un nom unique, par exemple **YOUR-INITIALS-monitorapp**. Remplacez **YOUR-INITIALS** par vos initiales ou une autre valeur. Le nom doit être unique, ce qui peut nécessiter quelques modifications. |
    | Curseur sous le paramètre **Nom** | Sélectionnez le curseur pour le désactiver. Ce curseur apparaît uniquement dans certaines configurations Azure. |
    | **Publier** | Sélectionnez l’option **Code**. |
    | **Pile d’exécution** | Sélectionnez **.NET 8 (LTS)** dans le menu déroulant. |
    | **Système d’exploitation** | Sélectionnez **Windows**. |
    | **Région** | Conservez la sélection par défaut ou choisissez une région proche de vous. |
    | **Plan Windows** | Conservez la sélection par défaut. |
    | **Plan tarifaire** | Sélectionnez la liste déroulante et choisissez le plan **F1 gratuit**. |

1. Sélectionnez ou accédez à l’onglet **Surveiller + sécuriser**, puis entrez les informations dans le tableau suivant :

    | Setting | Action |
    |--|--|
    | **Activer Application Insights** | Sélectionnez **Oui**. |
    | **Application Insights** | Sélectionnez **Créer nouveau** et une boîte de dialogue apparaîtra. Entrez `autoinstrument-insights` dans le champ **Nom** de la boîte de dialogue. Sélectionnez ensuite **OK** pour accepter le nom. |
    | **Espace de travail** | Entrez `Workspace` si le champ n’est pas déjà rempli et verrouillé. |

1. Sélectionnez **Passer en revue + créer** et passez en revue les détails de votre déploiement. Sélectionnez ensuite **Créer** pour créer les ressources.

Le déploiement prendra quelques minutes. Une fois terminé, sélectionnez le bouton **Accéder à la ressource**.

### Configurez les paramètres d’instrumentation

Pour activer la surveillance sans modifier votre code, vous devez configurer l’instrumentation de votre application au niveau du service.

1. Dans le menu de navigation gauche, développez **Surveillance** et sélectionnez **Application Insights**.

1. Recherchez la section **Instrumenter votre application** et sélectionnez **.NET Core**.

1. Sélectionnez **Recommandé** dans la section **Niveau de collection**.

1. Sélectionnez **Appliquer**, puis confirmez les modifications.

1. Dans le menu de navigation gauche, sélectionnez **Vue d’ensemble**.

## Créez et déployez une application Blazor

Dans cette partie de l’exercice, vous allez créer une application Blazor dans le Cloud Shell et la déployer dans l’application web que vous avez créée. Toutes les étapes de cette section sont réalisées dans le Cloud Shell.

1. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***Bash***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure. Si vous êtes invité à sélectionner un compte de stockage pour conserver vos fichiers, sélectionnez **Aucun compte de stockage requis**, votre abonnement, puis sélectionnez **Appliquer**.

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *PowerShel*, basculez-le vers ***Bash***.

1. Exécutez les commandes suivantes pour créer un répertoire pour l’application Blazor et passer au répertoire.

    ```
    mkdir blazor
    cd blazor
    ```

1. Exécutez la commande suivante pour créer une nouvelle application Blazor dans le dossier.

    ```
    dotnet new blazor
    ```

1. Exécutez la commande suivante pour générer l’application afin de vous assurer qu’aucun problème n’est apparu lors de la création.

    ```
    dotnet build
    ```

### Déployer l’application sur App Service

Pour déployer l’application, vous devez d’abord la publier à l’aide de la commande **dotnet publish**, puis créer un fichier *.zip* pour le déploiement.

1. Exécutez la commande suivante pour publier l’application dans un répertoire de *publication*.

    ```
    dotnet publish -c Release -o ./publish
    ```

1. Exécutez les commandes suivantes pour créer un fichier *.zip* de l’application publiée. Le fichier *.zip* se trouvera dans le répertoire racine de l’application.

    ```
    cd publish
    zip -r ../app.zip .
    cd ..
    ```

1. Exécutez la commande suivante pour déployer l’application sur App Service. Remplacez **YOUR-WEB-APP-NAME** et **YOUR-RESOURCE-GROUP** par les valeurs que vous avez utilisées lors de la création des ressources App Service précédemment dans cet exercice.

    ```
    az webapp deploy --name YOUR-WEB-APP-NAME \
        --resource-group YOUR-RESOURCE-GROUP \
        --src-path ./app.zip
    ```

1. Une fois le déploiement terminé, sélectionnez le lien dans le champ **Domaine par défaut** situé dans la section **Essentiels** pour ouvrir l’application dans un nouvel onglet de votre navigateur.

Il est maintenant temps de consulter certaines métriques d’application de base dans Application Insights. Ne fermez pas cet onglet, vous en aurez besoin pour le reste de l’exercice.

## Voir les métriques dans Application Insights

Revenez à l’onglet du portail Azure et accédez à la ressource Application Insights que vous avez créée précédemment. L’onglet **Vue d’ensemble** affiche quelques graphiques de base :

* Requêtes échouées
* Temps de réponse du serveur
* Demandes du serveur
* Disponibilité

Dans cette section, vous allez effectuer certaines actions dans l’application web, puis revenir à cette page pour afficher l’activité. Le rapport d’activités est retardé, quelques minutes peuvent donc s’écouler avant qu’il n’apparaisse dans les graphiques.

Effectuez les étapes suivantes dans l’application web.

1. Naviguez entre les options de navigation **Accueil**, **+ Compteur** et **Météo** dans le menu de l’application web.

1. Actualisez plusieurs fois la page web pour générer les données **Temps de réponse du serveur** et **Requêtes serveur**.

1. Pour créer des erreurs, sélectionnez le bouton **Accueil**, puis ajoutez **/failures** à l’URL. Cet itinéraire n’existe pas dans l’application web et générera une erreur. Actualisez la page plusieurs fois pour générer des données d’erreur.

1. Revenez à l’onglet où Application Insights est en cours d’exécution et attendez une minute ou deux pour que les informations apparaissent dans les graphiques. 

1. Dans le volet de navigation de gauche, développez la section **Examiner** et sélectionnez **Échecs**. Le nombre de requêtes ayant échoué s’affiche, ainsi que des informations plus détaillées sur les codes de réponse correspondant à ces échecs.

Explorez les autres options de rapports pour avoir une idée des types d’informations disponibles. 

## Nettoyer les ressources

Maintenant que vous avez terminé l’exercice, vous devez supprimer les ressources cloud que vous avez créées pour éviter une utilisation inutile des ressources.

1. Dans votre navigateur, accédez au portail Azure [https://portal.azure.com](https://portal.azure.com) et connectez-vous en utilisant vos informations d’identification Azure.
1. Accédez au groupe de ressources que vous avez créé et affichez le contenu des ressources utilisées dans cet exercice.
1. Dans la barre d’outils, sélectionnez **Supprimer le groupe de ressources**.
1. Entrez le nom du groupe de ressources et confirmez que vous souhaitez le supprimer.

> **ATTENTION :** La suppression d’un groupe de ressources entraîne la suppression de toutes les ressources qu’il contient. Si vous avez choisi un groupe de ressources existant pour cet exercice, toutes les ressources existantes qui ne relèvent pas du champ d’application de cet exercice seront également supprimées.
