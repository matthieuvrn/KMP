# Documentation Technique Complète - CryptoKMP

Ce document est un rapport technique exhaustif de l'application **CryptoKMP**. Il détaille l'intégralité des fichiers sources du code commun (`commonMain`), l'architecture, les flux de données et les technologies employées.

---

## 1. Vue d'ensemble
-   **Technologie** : Kotlin Multiplatform (KMP).
-   **UI** : Jetpack Compose Multiplatform.
-   **Architecture** : Clean Architecture + MVVM (Model-View-ViewModel).
-   **Injection de dépendances** : Koin.
-   **Réseau** : Ktor Client.
-   **Persistance** : Russhwolf Settings (pour le cache léger JSON).
-   **Sérialisation** : Kotlinx Serialization.

---

## 2. Analyse Détaillée des Fichiers (Dossier `commonMain`)

Les fichiers sont présentés par répertoire, suivant la structure du projet dans `org.example.project`.

### 2.1. Racine (`org.example.project`)

#### `App.kt`
*   **Rôle** : Point d'entrée de l'interface utilisateur partagée.
*   **Contenu** : Définit la fonction composable `App()`. Elle applique le `MaterialTheme` et appelle `AppNavigation()` qui gère le routing de toute l'application. C'est ce composant qui est appelé par les plateformes natives (Android `MainActivity` et Desktop `main.kt`).

#### `Greeting.kt`
*   **Rôle** : Classe de démonstration (souvent générée par le wizard KMP).
*   **Contenu** : Contient une fonction `greet()` qui retourne une chaîne "Hello, [Platform Name]!" en utilisant l'interface `Platform`. Elle sert principalement à vérifier que le code spécifique à la plateforme fonctionne.

#### `Platform.kt`
*   **Rôle** : Interface d'abstraction de la plateforme.
*   **Contenu** :
    *   `interface Platform` : Définit un contrat avec une propriété `name`.
    *   `expect fun getPlatform()` : Déclaration "expect", signifiant que chaque plateforme (Android, JVM) doit fournir une implémentation "actual" de cette fonction.

---

### 2.2. Data Layer (`data/`)

Cette couche gère la récupération et le stockage des données.

#### `data/local` (Persistance locale)

*   **`CryptoLocalDataSource.kt`**
    *   **Rôle** : Interface définissant les opérations de cache.
    *   **Méthodes** : `getCachedTopMarket()` (lire), `cacheTopMarket()` (écrire), `clearCache()`.
*   **`CryptoLocalDataSourceImpl.kt`**
    *   **Rôle** : Implémentation concrète du cache.
    *   **Fonctionnement** : Utilise `Settings` (stockage clé-valeur). Elle prend une liste d'objets `Coin`, les convertit en `CoinCache` (DTO local), puis les sérialise en JSON (string) pour les sauvegarder. Inversement pour la lecture.
*   **`SettingsFactory.kt`**
    *   **Rôle** : Wrapper "expect" pour créer une instance de `Settings` (SharedPrefs sur Android, Properties/Preferences sur Desktop).
*   **`model/CoinCache.kt`**
    *   **Rôle** : Modèle de données spécifique au stockage local.
    *   **Contenu** : Classe `CoinCache` annotée `@Serializable`. Contient aussi les fonctions d'extension `.toDomain()` et `.toCache()` pour mapper vers/depuis le modèle métier `Coin`. Cela permet de découpler le format de stockage du modèle utilisé dans l'app.

#### `data/network` (Client Réseau)

*   **`CoinGeckoApi.kt`**
    *   **Rôle** : Définition des appels API.
    *   **Contenu** :
        *   Interface `CoinGeckoApi` avec les méthodes suspendues `getMarkets` (liste) et `getMarketChart` (graphique).
        *   Classe `CoinGeckoApiImpl` : Implémente l'interface avec `HttpClient` de Ktor. Elle construit les requêtes HTTP (paramètres `vs_currency`, `page`, `sparkline`, etc.).
*   **`HttpClientFactory.kt`**
    *   **Rôle** : Usine pour créer le client HTTP Ktor.
    *   **Contenu** : Classe "expect" `HttpClientFactory`. La partie commune définit souvent la configuration JSON (`ContentNegotiation`, `ignoreUnknownKeys = true`), mais l'instanciation du moteur (OkHttp pour Android, CIO/Java pour Desktop) est spécifique à la plateforme.
*   **`SmokeTest.kt`**
    *   **Rôle** : Fichier utilitaire de test rapide.
    *   **Contenu** : Fonction `pingCoinGecko` qui fait un appel simple à l'endpoint `/ping` pour vérifier la connectivité.

#### `data/remote` (Source de données distante)

*   **`CryptoRemoteDataSource.kt`**
    *   **Rôle** : Interface abstraite pour accéder aux données distantes (indépendamment de l'implémentation API spécifique).
*   **`CryptoRemoteDataSourceImpl.kt`**
    *   **Rôle** : Implémentation qui orchestre les appels à `CoinGeckoApi`.
    *   **Fonctionnement** : Appelle l'API, reçoit des DTOs (`CoinMarketDto`), et les convertit immédiatement en modèles du domaine (`Coin`) grâce aux mappers. C'est la frontière entre le monde "API" et le monde "Domaine".
*   **`dto/CoinMarketDto.kt`**
    *   **Rôle** : Data Transfer Object pour la liste des marchés.
    *   **Contenu** : Classe Kotlin qui reflète **exactement** le JSON renvoyé par CoinGecko (champs `current_price`, `market_cap`, etc. mappés avec `@SerialName`).
*   **`dto/MarketChartDto.kt`**
    *   **Rôle** : DTO pour les données du graphique.
    *   **Contenu** : Contient une liste de listes de doubles (`prices: List<List<Double>>`), format un peu particulier de CoinGecko (Timestamp, Prix).
*   **`mapper/DtoMappers.kt`**
    *   **Rôle** : Convertisseurs.
    *   **Contenu** :
        *   `toDomain()` pour `CoinMarketDto` -> `Coin`.
        *   `toDomain()` pour `MarketChartDto` -> `List<PricePoint>`. Gère la conversion complexe du tableau `[Timestamp, Prix]` en objet `PricePoint` avec gestion des dates (`Instant`).

#### `data/repository` (Gestion centrale des données)

*   **`CryptoRepositoryImpl.kt`**
    *   **Rôle** : Implémentation du `CryptoRepository` (défini dans `domain`).
    *   **Fonctionnement** : C'est la source de vérité unique pour les Use Cases.
        *   Combine `RemoteDataSource` et `LocalDataSource`.
        *   Enveloppe les appels dans `runDomain` pour gérer les erreurs de manière uniforme (voir `DomainResult`).
        *   Gère la logique : "Récupérer depuis le réseau, si succès, mettre en cache".

---

### 2.3. Domain Layer (`domain/`)

C'est le cœur de l'application, contenant la logique métier pure, sans dépendance Android ou UI.

#### `domain/common`
*   **`DomainResult.kt`** : Classe scellée (`Success<T>` / `Error`) utilisée comme type de retour standard pour toutes les opérations métier. Permet de forcer la gestion des erreurs.
*   **`DomainError.kt`** (supposé dans le même fichier ou package) : Hiérarchie d'erreurs typées (`NetworkError`, `ApiError`, etc.).

#### `domain/model`
*   **`Coin.kt`** : L'entité principale représentant une crypto-monnaie dans l'application. Nettoyée de toute annotation de sérialisation.
*   **`PricePoint.kt`** : Modèle simple `(time: Instant, price: Double)` pour les graphiques.

#### `domain/repository`
*   **`CryptoRepository.kt`** : Interface définissant les opérations métier disponibles (`getTopMarket`, `getCoinChart`, `getCachedTopMarket`). Permet d'inverser la dépendance (les Use Cases dépendent de l'interface, pas de l'implémentation).

#### `domain/usecase` (Cas d'utilisation)
Chaque classe représente une action unitaire possible pour l'utilisateur.
*   **`GetTopMarketsUseCase.kt`** : Récupère la liste des cryptos. Gère la stratégie de cache "Intelligente" (retourne le cache si valide et pas de `forceRefresh`, sinon appelle le réseau et met à jour le cache).
*   **`GetCoinByIdUseCase.kt`** : Trouve une crypto spécifique par son ID. Cherche d'abord dans le cache mémoire/local pour éviter un appel réseau inutile, sinon recharge la liste globale (l'API gratuite limitant les appels détaillés, on réutilise souvent la liste globale).
*   **`GetCoinChartUseCase.kt`** : Récupère l'historique des prix pour une crypto donnée. Pas de cache implémenté ici (données temps réel).

---

### 2.4. DI Layer (`di/`)

*   **`AppModule.kt`**
    *   **Rôle** : Configuration de Koin.
    *   **Détail** :
        *   `dataModule` : Fournit les singletons techniques (`HttpClient`, `Settings`, `Api`, `DataSources`, `Repository`).
        *   `domainModule` : Fournit les Use Cases (factories).
        *   `presentationModule` : Fournit les ViewModels (`MarketViewModel`, `DetailViewModel`).
        *   `commonModule` : Aggrège le tout.

---

### 2.5. Presentation Layer (`presentation/`)

Couche UI utilisant Jetpack Compose et le pattern MVVM.

#### `presentation/component`
*   **`CoinListItem.kt`** : Composant UI réutilisable affichant une ligne de crypto (Rang, Image, Nom, Symbole, Prix, Variation). Gère les couleurs conditionnelles (Vert/Rouge) via `PriceChangePill`.

#### `presentation/mapper`
*   **`CoinUiMapper.kt`** : Convertit `Coin` (Domain) -> `CoinUiModel` (UI). Formate les prix (ajout du symbole $), les pourcentages, et détermine la couleur/icône de variation.
*   **`CoinDetailUiMapper.kt`** : Convertit `Coin` -> `CoinDetailUiModel`. Prépare toutes les données pour l'écran de détail (formatage complexe des gros nombres "M", "B", dates, supplies).

#### `presentation/model`
*   **`CoinUiModel.kt`** : Modèle optimisé pour l'affichage en liste (Champs String pré-formatés).
*   **`CoinDetailUiModel.kt`** : Modèle complet pour l'écran de détail, regroupant les infos par sections (`PriceInfo`, `MarketInfo`, `SupplyInfo`, `HistoricalInfo`).
*   **`MarketUiState.kt`** : État de l'écran liste (`Loading`, `Success(list)`, `Error`).
*   **`DetailUiState.kt`** : État de l'écran détail.

#### `presentation/theme`
*   **`CryptoColors.kt`** : Palette de couleurs sémantiques (Positif/Vert, Négatif/Rouge, Neutre/Gris) utilisée pour les variations de prix.

#### `presentation/navigation`
*   **`Navigation.kt`** :
    *   Définit les objets de route : `MarketRoute` (objet) et `DetailRoute` (data class avec `coinId`).
    *   `AppNavigation` : Le `NavHost` qui mappe ces routes aux écrans `MarketScreen` et `DetailScreen`. Gère le passage d'arguments (ID de la crypto).

#### `presentation/screen`
*   **`MarketScreen.kt`** : Écran principal.
    *   Affiche une liste (`LazyColumn`) ou des états de chargement/erreur.
    *   Possède un bouton "Rafraîchir" dans le header.
    *   Délègue les clics au `AppNavigation`.
*   **`DetailScreen.kt`** : Écran de détail.
    *   Utilise `Scaffold` pour la TopBar.
    *   Affiche les informations détaillées dans des "Cartes" (`SectionCard`) : Prix, Marché, Offre, Historique.
    *   Gère le scroll vertical.

#### `presentation/viewmodel`
*   **`MarketViewModel.kt`** :
    *   Expose `uiState` (StateFlow).
    *   `loadMarketData` : Appelle `GetTopMarketsUseCase`. Mappe les résultats en `UiModel`. Gère les cas d'erreur en messages lisibles.
    *   Fonction `refresh()` pour forcer le rechargement réseau.
*   **`DetailViewModel.kt`** :
    *   Prend `coinId` en paramètre (injecté via Koin `parametersOf`).
    *   Charge les détails d'une crypto spécifique via `GetCoinByIdUseCase`.

---

## 3. Fonctionnement de Koin (Injection de Dépendances)

Koin est le moteur qui relie toutes les briques.

1.  **Définition** : Dans `di/AppModule.kt`, on déclare le graphe de dépendances.
    *   `single` : Crée une instance unique (Singleton). Utilisé pour le Repository, l'API, la DB.
    *   `factory` : Crée une nouvelle instance à chaque demande. Utilisé pour les ViewModels et UseCases.
2.  **Initialisation** :
    *   Au démarrage de l'app (`CryptoApplication` sur Android, `main` sur Desktop), on appelle `startKoin { modules(commonModule) }`.
3.  **Injection** :
    *   **Dans les classes** : Injection par constructeur. Exemple : `class CryptoRepositoryImpl(private val api: CoinGeckoApi)`. Koin fournit automatiquement `api` car il sait comment le créer.
    *   **Dans Compose** : Via la fonction `koinInject()`. Exemple : `val viewModel = koinInject<MarketViewModel>()`.

---

## 4. Le Modèle MVVM (Model-View-ViewModel) pas à pas

Voici le cycle de vie complet d'une action utilisateur (ex: Lancer l'application) :

1.  **View (UI)** : `MarketScreen` est appelé. Il demande un `MarketViewModel` à Koin.
2.  **ViewModel** : `MarketViewModel` s'initialise (`init`). Il émet l'état `Loading` dans son `uiState`. Il lance une coroutine pour appeler `GetTopMarketsUseCase`.
3.  **Use Case** : `GetTopMarketsUseCase` est invoqué. Il demande au `CryptoRepository` les données.
4.  **Repository** :
    *   Vérifie s'il y a un cache local via `CryptoLocalDataSource`.
    *   Si non, appelle `CryptoRemoteDataSource`.
5.  **Data Source (Remote)** : `CryptoRemoteDataSource` appelle `CoinGeckoApi`.
6.  **API** : `CoinGeckoApi` fait la requête HTTP GET vers CoinGecko.
7.  **Retour des données** :
    *   Le JSON est reçu et parsé en `CoinMarketDto`.
    *   Converti en `Coin` (Domain) par le Mapper.
    *   Renvoyé au Repository.
    *   Le Repository sauvegarde ces données dans le cache (`CryptoLocalDataSource`).
    *   Le Repository renvoie un `DomainResult.Success(List<Coin>)` au Use Case, puis au ViewModel.
8.  **ViewModel (Mise à jour)** : Reçoit le succès. Il transforme la liste de `Coin` en liste de `CoinUiModel` (formaté pour l'affichage). Il met à jour `uiState` avec `Success`.
9.  **View (Rendu)** : Compose détecte le changement de `uiState`. Il recompose l'écran, remplaçant le loader par la `LazyColumn` remplie de `CoinListItem`.

---

## 5. Gestion des Erreurs
L'application gère les erreurs à plusieurs niveaux :
*   **Technique** : Exceptions HTTP/Parsing catchées dans `runDomain`.
*   **Domaine** : Conversion en objets `DomainError` (Network, Api, Cache, Unknown).
*   **UI** : Le ViewModel convertit ces erreurs techniques en messages utilisateur (ex: "Erreur de connexion réseau") et boolean `isNetworkError` pour afficher une UI adaptée (Bouton "Réessayer").
