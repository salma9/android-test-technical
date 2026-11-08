# AndroidRecruitmentTestApp

Application Android affichant les albums de
[technical-test.json](https://static.leboncoin.fr/img/shared/technical-test.json), disponible
hors ligne, avec gestion des favoris et un écran de détail.

Énoncé : [leboncoin/android-technical-test](https://github.com/leboncoin/android-technical-test)

---

## Sommaire

1. [Lancer le projet](#1-lancer-le-projet)
2. [Fonctionnalités](#2-fonctionnalités)
3. [Architecture](#3-architecture)
4. [Modules](#4-modules)
5. [Patterns appliqués](#5-patterns-appliqués)
6. [Choix des librairies](#6-choix-des-librairies)
7. [Bugs et pièges corrigés](#7-bugs-et-pièges-corrigés)
8. [Performances](#8-performances)
9. [Changements de configuration](#9-changements-de-configuration)
10. [Tests](#10-tests)
11. [Évolutions planifiées](#11-évolutions-planifiées)

---

## 1. Lancer le projet

- Android Studio (dernière version stable), JDK 17+
- minSdk 24, targetSdk / compileSdk 36

```bash
./gradlew assembleDebug          # construire l'application
./gradlew testDebugUnitTest      # lancer tous les tests unitaires
```

## 2. Fonctionnalités

- **Liste des albums** paginée, avec *pull-to-refresh*.
- **Hors ligne** : les données sont stockées en base locale et restent disponibles sans réseau,
  y compris après un redémarrage de l'application. Les miniatures déjà vues sont en cache disque.
- **Favoris** : ajout / retrait depuis la liste ou le détail, persistés localement ; filtre
  « Favoris » dans la liste.
- **Écran de détail** : image en grand, titre, numéros d'album et de piste, favori.
- **Gestion d'erreur** : erreur réseau / serveur / inconnue. Si des données sont en cache, un
  *snackbar* s'affiche par-dessus ; sinon un écran d'erreur avec bouton « Réessayer ».
- Thème clair / sombre (suit le système), textes en français et en anglais, geste retour prédictif.

## 3. Architecture

L'application suit les recommandations officielles d'architecture Android
([guide to app architecture](https://developer.android.com/topic/architecture)) : couches
**UI → Domain → Data**, flux de données unidirectionnel, et une approche **offline-first**.

```
                 ┌──────────────────────────────┐
                 │             :app             │  Hilt, navigation, thème
                 └──────┬────────────────┬──────┘
                        │                │
           ┌────────────▼───┐      ┌─────▼────────────┐
           │ :feature:albums│      │ :feature:details │  UI (Compose) + ViewModel
           └────────┬───────┘      └───────┬──────────┘
                    │  :core:ui  :core:analytics
                    └────────┬─────────────┘
                     ┌───────▼────────┐
                     │  :core:domain  │  modèles + interface AlbumRepository (Kotlin pur)
                     └───────▲────────┘
                     ┌───────┴────────┐
                     │   :core:data   │  OfflineFirstAlbumRepository
                     └───┬────────┬───┘
            ┌────────────▼──┐  ┌──▼──────────────┐
            │:core:database │  │  :core:network  │
            │    (Room)     │  │ (Retrofit/OkHttp)│
            └───────────────┘  └─────────────────┘
```

### Offline-first : la base de données est la seule source de vérité

```
  UI ──observe──▶ Room (PagingSource / Flow) ◀──replaceAll── refresh() ◀── API
```

- L'UI **n'observe que Room**. Elle ne lit jamais directement la réponse réseau.
- `refresh()` télécharge le JSON puis **remplace le cache en une seule transaction**
  (`AlbumDao.replaceAll`) : l'écran ne passe jamais par un état vide intermédiaire, et un
  rafraîchissement en échec ne vide jamais l'écran.
- Les **favoris sont dans une table séparée** (`favorites`) : remplacer le cache des albums ne
  les efface pas. Le statut favori est calculé en SQL (`EXISTS` / `JOIN`) et tout changement est
  propagé automatiquement à la liste et au détail.

### Règles de dépendances

- Les features **ne dépendent jamais l'une de l'autre** : elles exposent des callbacks
  (`onAlbumClick`, `onBackClick`) et c'est `:app` qui décide de la navigation.
- Les features ne connaissent que `:core:domain` (l'interface `AlbumRepository`), jamais Room ni
  Retrofit. L'implémentation est fournie par Hilt depuis `:core:data`.
- `:core:domain` est un module **Kotlin pur** (sans framework Android).

## 4. Modules

| Module | Rôle |
|---|---|
| `:app` | Assemble l'application : `@HiltAndroidApp`, `MainActivity` unique, `AppNavHost`, thème Spark, `ImageLoader` Coil partagé. |
| `:feature:albums` | Liste paginée, pull-to-refresh, filtre favoris, états vide / erreur. `AlbumsViewModel` + écrans Compose. |
| `:feature:details` | Écran de détail. `AlbumDetailViewModel` reçoit l'id via injection assistée Hilt. |
| `:core:domain` | Modèles métier (`Album`, `DataError`, `RefreshResult`) et contrat `AlbumRepository`. |
| `:core:data` | `OfflineFirstAlbumRepository` : synchronisation réseau → base, mapping, conversion des exceptions en erreurs métier. |
| `:core:database` | Room : entités `AlbumEntity`, `FavoriteEntity`, DAO, schéma exporté (`schemas/`) pour les futures migrations. |
| `:core:network` | Retrofit + kotlinx.serialization, `OkHttpClient` partagé, intercepteur User-Agent, logs en debug uniquement. |
| `:core:ui` | Composants partagés basés sur Spark : image avec placeholder, bouton favori, états chargement / message. |
| `:core:analytics` | Interface `AnalyticsHelper` indépendante du fournisseur ; implémentation Logcat. |
| `:core:common` | Dispatchers injectables (`@Dispatcher(IO / Default)`). |
| `:core:testing` | Fakes partagés (`FakeAlbumRepository`, `TestAnalyticsHelper`), `MainDispatcherRule`, données de test. |
| `build-logic` | Plugins de convention Gradle : configuration commune (SDK, Kotlin, Compose, Hilt, Room, dépendances de test) centralisée en un seul endroit. |

**Pourquoi ce découpage ?** Compilation incrémentale et parallèle plus rapide, frontières
claires (une feature ne peut pas accéder à la base par erreur), chaque module testable isolément,
et ajout d'une nouvelle feature en quelques lignes grâce au plugin `leboncoin.android.feature`.

## 5. Patterns appliqués

- **MVVM + UDF** : le ViewModel expose un `StateFlow<UiState>` immuable ; l'UI envoie des
  événements (callbacks), jamais de mutation directe.
- **Repository** avec interface dans le domaine et implémentation dans la data (inversion de
  dépendances).
- **Offline-first / Single source of truth** (voir section 3).
- **Écran stateful / stateless** : `AlbumsScreen(viewModel)` collecte l'état,
  `AlbumsScreen(uiState, ...)` ne fait qu'afficher. L'écran est donc prévisualisable et testable
  sans Hilt.
- **Injection de dépendances** avec Hilt, y compris l'**injection assistée** pour passer l'id de
  l'album au ViewModel de détail sans manipuler de `Bundle`.
- **Navigation type-safe** (routes `@Serializable`) dans une **single activity**.
- **Mappers** explicites entre couches : `NetworkAlbum` → `AlbumEntity` → `Album`.
- **Erreurs typées** : les exceptions sont converties en `DataError` (`Network`, `Server`,
  `Unknown`) ; `CancellationException` est toujours relancée.
- **Plugins de convention Gradle** et **version catalog** (`libs.versions.toml`).

## 6. Choix des librairies

| Librairie | Pourquoi |
|---|---|
| **Jetpack Compose** + **Spark** | UI déclarative ; Spark est le design system de leboncoin, déjà présent dans le projet. |
| **Hilt** | Standard Android pour l'injection de dépendances, intégré à ViewModel / Navigation, vérifié à la compilation. Remplace le *service locator* écrit à la main. |
| **Room** | Persistance SQLite avec requêtes vérifiées à la compilation, `Flow` réactifs, intégration Paging, migrations. |
| **Paging 3** | ~5000 éléments : chargement par pages depuis Room plutôt que tout en mémoire. |
| **Retrofit + OkHttp** | Standard pour le réseau. Un seul `OkHttpClient` partagé entre l'API et les images. |
| **kotlinx.serialization** | Sans réflexion, pensé pour Kotlin, déjà utilisé par Navigation pour les routes. |
| **Coil 3** | Chargement d'images orienté Compose et coroutines, cache mémoire + disque (miniatures disponibles hors ligne). |
| **Navigation Compose** | Navigation type-safe, gestion de la pile et du retour. |
| **Coroutines / Flow** | Asynchronisme structuré, annulé avec le cycle de vie du ViewModel. |
| **JUnit4, Robolectric, Turbine, kotlinx-coroutines-test, MockWebServer** | Tests unitaires rapides sur la JVM, y compris Room et Compose (Robolectric), tests de Flow (Turbine), tests réseau sans vrai serveur (MockWebServer). |
| **LeakCanary** (debug) | Détection des fuites mémoire pendant le développement. |

## 7. Bugs et pièges corrigés

| # | Problème dans le code initial | Conséquence | Correction |
|---|---|---|---|
| 1 | `DetailsActivity` déclarée `exported="true"` avec un intent-filter `MAIN`/`LAUNCHER` | Deux icônes dans le lanceur ; l'écran de détail pouvait être ouvert seul, sans album | Activité supprimée, remplacée par une destination Navigation Compose |
| 2 | `GlobalScope.launch` dans le ViewModel | Requêtes jamais annulées, fuite de la coroutine après la destruction de l'écran | `viewModelScope` |
| 3 | `catch (_: Exception) { /* TODO */ }` | Erreurs avalées : écran vide sans explication, et `CancellationException` interceptée | Erreurs converties en `DataError` et affichées ; `CancellationException` relancée |
| 4 | `MutableSharedFlow` sans *replay* + `LaunchedEffect(Unit) { loadAlbums() }` | Données perdues et rechargées à chaque rotation ; émission perdue si personne n'écoute | Données dans Room, Paging `cachedIn(viewModelScope)`, chargement déclenché une seule fois dans `init` |
| 5 | `AnalyticsHelper.initialize(this)` garde une référence à l'`Activity` dans un objet de longue durée | Fuite mémoire de l'activité | Nouvelle interface `AnalyticsHelper` sans `Context`, injectée par Hilt |
| 6 | `if (!BuildConfig.DEBUG)` pour activer `HttpLoggingInterceptor.Level.BODY` | Logs complets **en release** et aucun en debug (condition inversée) | `if (BuildConfig.DEBUG)`, et uniquement sur le client de l'API (pas sur les images) |
| 7 | `modifier` du composable réutilisé sur l'`AsyncImage` | Le padding / la hauteur de la carte appliqués deux fois à l'image | `Modifier` neuf pour l'image |
| 8 | `ChipTinted` utilisés pour afficher album / piste | Chips cliquables sans action : annoncées comme boutons par TalkBack, et interceptaient le clic sur la carte | Remplacés par des `Tag` non interactifs |
| 9 | `analyticsHelper.trackScreenView(...)` dans `onCreate` | Événement envoyé à chaque rotation | Envoyé une seule fois depuis le ViewModel |
| 10 | Test `AlbumsViewModelTest` appelant un constructeur inexistant (`Logger` en paramètre) | Le projet de tests ne compilait pas | Tests réécrits pour chaque module (voir section 10) |
| 11 | Aucune persistance, aucun favori, écran de détail « work in progress » | Exigences de l'énoncé non remplies | Room, table `favorites`, écran de détail complet |
| 12 | `AppDependencies` (*service locator* manuel) et `ViewModelProvider.Factory` écrites à la main | Code répétitif, pas de portée ni de vérification à la compilation | Hilt |
| 13 | Un `OkHttpClient` pour l'API et un autre implicite pour Coil ; User-Agent ajouté à la main dans chaque requête d'image | Deux pools de connexions, code dupliqué | Un client partagé, User-Agent ajouté par un intercepteur |
| 14 | SDK 35, JVM 11, configuration Gradle dupliquée dans chaque module | Configuration hétérogène et difficile à faire évoluer | SDK 36, JVM 17, plugins de convention dans `build-logic` |

## 8. Performances

- **Pagination** (pages de 50) depuis Room : seule une petite partie des ~5000 albums est en
  mémoire.
- **Clés stables** et `contentType` dans la `LazyColumn` : moins de recompositions, position de
  défilement conservée quand les données changent.
- **Mapping des ~5000 entrées sur `Dispatchers.Default`**, écriture en base dans une seule
  transaction.
- **Initialisation paresseuse** d'OkHttp et de l'`ImageLoader` (`dagger.Lazy`) : rien n'est
  construit sur le thread principal au démarrage.
- **Un seul client HTTP** partagé (pool de connexions et cache disque communs).
- Cache mémoire + disque des images par Coil, avec placeholder.
- **R8 activé en release** (`isMinifyEnabled` + `isShrinkResources`) : code réduit, optimisé et
  obfusqué, ressources inutilisées supprimées. L'APK passe d'environ 24,7 Mo (debug) à 2,5 Mo.
  Retrofit 3, kotlinx.serialization, Room, Hilt et Navigation fournissent leurs propres règles
  ProGuard : aucune règle `keep` supplémentaire n'est nécessaire. Le build release est signé avec
  la clé de debug pour pouvoir être installé localement.
- Fichier de stabilité Compose (`compose_stability.conf`) pour que les modèles du domaine soient
  considérés stables et ne provoquent pas de recompositions inutiles.

## 9. Changements de configuration

- Tout l'état est dans les **ViewModels** (survivent à la rotation) ; les pages déjà chargées sont
  conservées via `cachedIn(viewModelScope)`.
- Le filtre « favoris » est sauvegardé dans le **`SavedStateHandle`** : il survit aussi à la mort
  du processus.
- Le rafraîchissement et l'événement analytics sont lancés une seule fois (`init` du ViewModel),
  pas à chaque recréation de l'écran.
- La pile de navigation est restaurée par Navigation Compose.
- Mise en page défilable et largeur limitée sur l'écran de détail pour le paysage et les grands
  écrans.

## 10. Tests

34 tests unitaires, exécutés sur la JVM (`./gradlew testDebugUnitTest`) :

| Module | Ce qui est testé |
|---|---|
| `:core:network` | `AlbumApiTest` : parsing du JSON et erreurs HTTP avec MockWebServer. |
| `:core:database` | `AlbumDaoTest` : requêtes Room (pagination, filtre favoris, remplacement du cache sans perte des favoris, observation d'un album) sur une base en mémoire (Robolectric). |
| `:core:data` | `OfflineFirstAlbumRepositoryTest` : synchronisation, conservation des favoris et du cache en cas d'échec, conversion des erreurs. |
| `:feature:albums` | `AlbumsViewModelTest` (états, rafraîchissement, filtre, favoris, analytics) et `AlbumsScreenTest` (UI Compose avec Robolectric). |
| `:feature:details` | `AlbumDetailViewModelTest` (Turbine) et `AlbumDetailScreenTest` (UI Compose). |

Les ViewModels sont testés avec un `FakeAlbumRepository` partagé (`:core:testing`) plutôt qu'avec
des mocks : les tests vérifient le comportement et non les appels internes.

## 11. Évolutions planifiées

### Ce qui prépare déjà les évolutions

L'architecture actuelle a été pensée pour que les évolutions ci-dessous soient **localisées** :

| Point d'extension | Ce qu'il permet |
|---|---|
| Interface `AlbumRepository` dans `:core:domain` | Changer la source de données (backend paginé, sync incrémentale) sans toucher aux features. |
| Interface `AnalyticsHelper` | Brancher un vrai fournisseur sans toucher aux features. |
| Plugin `leboncoin.android.feature` | Créer une nouvelle feature en un fichier `build.gradle.kts` de quelques lignes. |
| Schéma Room exporté (`core/database/schemas/`) | Écrire et tester des migrations sans perdre les données ni les favoris des utilisateurs. |
| Features indépendantes, navigation dans `:app` | Ajouter, réordonner ou masquer un écran sans modifier les autres. |

### Feuille de route

Les évolutions sont regroupées en trois phases. Chaque phase ne commence que lorsque la
précédente est en place : on sécurise d'abord la qualité, puis l'expérience hors ligne, puis les
nouvelles fonctionnalités.

#### Phase 1 : fiabiliser et mesurer (court terme, ~1 sprint)

| Évolution | Pourquoi | Impact sur le code |
|---|---|---|
| **CI** (GitHub Actions) : build, tests unitaires, lint, Detekt / ktlint à chaque pull request | Aucune régression ne doit atteindre `dev` ; les règles de style sont vérifiées automatiquement | Fichier de workflow + plugin de convention pour Detekt |
| **Signature release** par la CI (clé dans les secrets) | Le build release utilise aujourd'hui la clé de debug | `signingConfigs` dans `:app` |
| **Baseline Profile** + Macrobenchmark | Mesurer puis accélérer le démarrage et le premier défilement | Nouveau module `:baselineprofile` |
| **Crash reporting** (Crashlytics ou Sentry) | Connaître les crashs en production | Implémentation dans `:app` uniquement |

#### Phase 2 : améliorer l'expérience hors ligne (moyen terme, ~1 à 2 sprints)

| Évolution | Pourquoi | Impact sur le code |
|---|---|---|
| **Indicateur de connectivité** (bandeau « hors ligne ») | L'utilisateur doit savoir qu'il consulte des données en cache | `NetworkMonitor` dans `:core:data`, observé par les ViewModels |
| **Synchronisation en arrière-plan** (WorkManager, réseau requis) | Données à jour à l'ouverture, sans attendre le rafraîchissement | Nouveau module `:sync`, appelle `AlbumRepository.refresh()` |
| **Rafraîchissement incrémental** (`ETag` / `If-None-Match`, ou API paginée + `RemoteMediator`) | Le JSON complet (~5000 éléments) est aujourd'hui retéléchargé à chaque fois | Limité à `:core:network` et `:core:data` ; les features ne changent pas |
| **Synchronisation des favoris** avec un compte utilisateur | Retrouver ses favoris sur un autre appareil | Colonne « à synchroniser » dans `favorites` + migration Room |

#### Phase 3 : nouvelles fonctionnalités (long terme)

| Évolution | Pourquoi | Impact sur le code |
|---|---|---|
| **Recherche** par titre | Retrouver un album parmi ~5000 | Nouveau module `:feature:search` ; table FTS dans Room + migration |
| **Regroupement par album** | Les titres sont aujourd'hui une liste plate | Requête Room groupée, nouvel écran ou en-têtes de section |
| **Grands écrans** (tablette, pliable) : liste et détail côte à côte | Mieux utiliser l'espace | `ListDetailPaneScaffold` dans `:app`, les écrans existants sont réutilisés tels quels |
| **Partage** d'un album | Fonction sociale attendue | Callback dans `:feature:details`, intent dans `:app` |
| **Analytics réel** (Firebase, Amplitude…) | Mesurer l'usage des fonctionnalités | Nouvelle implémentation de `AnalyticsHelper`, liée dans le module Hilt |

### Qualité continue (à chaque phase)

- **Tests** : chaque évolution arrive avec ses tests unitaires. On ajoute des tests de capture
  d'écran (Roborazzi) pour `:core:ui`, et des tests instrumentés pour les parcours principaux.
- **Migrations Room** : toute modification du schéma est accompagnée d'une migration et d'un test
  de migration (`MigrationTestHelper`). On ne supprime jamais la base de l'utilisateur.
- **Feature flags** pour les fonctionnalités importantes : livraison progressive et retour
  arrière possible sans nouvelle version.
- **Décisions documentées** : chaque choix d'architecture important est consigné dans un court
  document (ADR) à côté du code.
