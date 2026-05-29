# Analyse d'architecture — Xiaomi Home (米家) Plugin System

**Application**: com.xiaomi.smarthome  
**Version**: 11.5.515 (classes2.dex / classes12.dex)  
**Date d'analyse**: Mai 2026  
**Outil**: jadx-gui

---

## 1. Introduction

Cette analyse couvre l'architecture complète du système de plugins de l'application Xiaomi Home (米家). L'application utilise un système de plugins dynamiques téléchargés à la demande pour chaque modèle d'appareil connecté. Les plugins peuvent être de quatre types : **RN** (React Native), **MPK** (natif), **APK**, ou **H5**.

---

## 2. Chemin de stockage racine

**Classe**: `h7c` (FileUtils) — `df/000010df.java:450`  
**Méthode**: `OooOoo(Context context)` → `context.getFilesDir().getPath()`

```
/data/data/com.xiaomi.smarthome/files/
```

Fallback manuel si `getFilesDir()` échoue : construction du chemin `/data/data/<package>/files`.

Cette valeur est mise en cache dans **`qn9.OooO00o`** (PluginPackageUtils) via `qn9.OooO0Oo(context)` — `a5/000027a5.java:403`.  
Le répertoire est **interne** à l'application, **pas** sur le stockage externe. Non accessible sans root sur Android 11+.

---

## 3. Arborescence complète des répertoires

```
/data/data/com.xiaomi.smarthome/files/
└── plugin/
    ├── download/          ← Plugins téléchargés (bruts)
    ├── install/
    │   ├── mpk/           ← Plugins MPK installés (natifs)
    │   │   └── <plugin_id>/
    │   ├── rn/            ← Plugins RN installés (React Native)
    │   │   └── <plugin_id>/
    │   │       ├── conf.json
    │   │       ├── _conf.json
    │   │       ├── project.json
    │   │       ├── bundle.sign
    │   │       ├── bundle.cert
    │   │       ├── android/       ← si conf.json existe
    │   │       │   └── sdk.bundle
    │   │       └── _android/      ← si conf.json absent
    │   │           └── sdk.bundle
    │   └── lib/           ← Bibliothèques partagées
    ├── tmp/               ← Répertoire temporaire
    ├── rnsdk/             ← RN SDK installé
    │   └── _config/       ← Configuration du SDK
    └── downloading/       ← Téléchargements en cours
```

### 3.1 Correspondance des constantes

| Chemin | Variable | Constante | Valeur |
|--------|----------|-----------|--------|
| `plugin/download/` | `pn9.OooOO0O` | `RN_DOWNLOAD_DIR_KEY` | `"download"` |
| `plugin/install/mpk/` | `pn9.OooOOO0` | `PLUGIN_TYPE_MPK` | `"mpk"` |
| `plugin/install/rn/` | `pn9.OooOOO` | `PLUGIN_TYPE_RN` | `"rn"` |
| `plugin/install/rn/<pid>/_temp/` | `pn9.OooOOOO` | — | `"_temp"` |
| `plugin/tmp/` | `pn9.OooOO0o` | — | `"tmp"` |
| `plugin/rnsdk/` | `pn9.OooOOOo` | `RN_SDK_INSTALL_PATH_SUFFIX` | `/plugin/rnsdk` |
| `plugin/rnsdk/_config/` | `pn9.OooOOo0` | `RN_SDK_CONFIG_FILE_PATH_SUFFIX` | `/plugin/rnsdk/_config` |
| `plugin/install/lib/` | `pn9.OooOOo` | — | `"lib"` |

**Source**: Initialisation dans `pn9.<init>` → `54/00002554.java:405-428`

### 3.2 Répertoire de téléchargement (`plugin/download/`)

Fichiers nommés `<package_id>.<ext>` :
- `<id>.rn` → plugin RN (zip)
- `<id>.mpk` → plugin MPK
- `<id>.apk` → plugin APK
- `<id>.h5` → plugin H5

### 3.3 Répertoire d'installation RN (`plugin/install/rn/<plugin_id>/`)

Contient le bundle React Native décompressé :

| Fichier | Constante | Rôle |
|---------|-----------|------|
| `conf.json` | `RN_CONF_A = "conf.json"` | Configuration du plugin |
| `_conf.json` | `RN_CONF_B = "_conf.json"` | Configuration alternative |
| `project.json` | `RN_PROJECT_JSON = "project.json"` | Manifeste du projet RN |
| `bundle.sign` | — | Signature ECDSA (SHA256withECDSA) |
| `bundle.cert` | — | Certificat PEM |
| `android/sdk.bundle` | `RN_DIR_A = "android"` | Bundle RN (si conf.json existe) |
| `_android/sdk.bundle` | `RN_DIR_B = "_android"` | Bundle RN (fallback) |

La sélection `android/` vs `_android/` est faite par `PluginPackageUtils.OooO0o()` (`a5/000027a5.java:416`) :
1. Si `conf.json` existe → utilise `android/`
2. Sinon, si le chemin contient `"rnsdk"`, `"download"`, ou `"install"` → utilise `_android/`
3. Sinon → utilise `android/`

---

## 4. PluginPackageInfo — Structure de données

**Fichier**: `c6/000118c6.java`  
**Implements**: `Parcelable`  
**Package**: `com.xiaomi.smarthome.core.entity.plugin`

| Champ | Getter | Type | Description |
|-------|--------|------|-------------|
| `OooOoo0` | `getPluginId()` | `long` | Identifiant unique du plugin |
| `OooOoo` | `getPackageId()` | `long` | Identifiant du package |
| `OooOooO` | `getPackagePath()` | `String` | Chemin absolu sur disque |
| `OooOooo` | `getVersion()` | `int` | Numéro de version |
| `Oooo000` | `getMinApiLevel()` | `int` | API level minimale requise |
| `Oooo00O` | `getDeveloperId()` | `long` | Identifiant développeur |
| `Oooo00o` | `getPackageType()` | `String` | Type (`"rn"`, `"mpk"`, `"apk"`, `"h5"`) |
| `Oooo0` | `getPlatform()` | `String` | Plateforme |
| `Oooo0O0` | `getPackageName()` | `String` | Nom du package |
| `Oooo0OO` | `getModelList()` | `List<String>` | Liste des modèles compatibles |
| `Oooo0o0` | — | `boolean` | Support widget |
| `Oooo0o` | — | `boolean` | Support App AV |
| `Oooo0oO` | `getLastModifiedTime()` | `long` | Date de dernière modification |

### Sérialisation Parcel (writeToParcel, line 258) :
```java
writeLong(pluginId);
writeLong(packageId);
writeString(packagePath);
writeInt(version);
writeInt(minApiLevel);
writeLong(developerId);
writeString(packageType);
writeString(platform);
writeString(packageName);
writeStringList(modelList);
writeInt(isSupportWidget);
writeInt(isSupportAppAV);
writeLong(lastModifiedTime);
```

---

## 5. Caches en mémoire

**Classe**: `pn9` (PluginPackageManager) — `54/00002554.java`
**Instance**: Singleton via `pn9.OooOO0O()`

### 5.1 Maps de cache

| Map | Type | Clé | Valeur | Contenu |
|-----|------|-----|--------|---------|
| `OooO0oO` | `ConcurrentHashMap` | `String` (model) | `PluginPackageInfo` | Plugins **installés** |
| `OooO0oo` | `ConcurrentHashMap` | `String` (model) | `PluginPackageInfo` | Plugins **téléchargés** |

### 5.2 Méthodes d'accès

```java
// Récupérer le plugin installé pour un modèle
OooO(String model) → OooO0oO.get(model)

// Récupérer le plugin téléchargé pour un modèle
OooO0oo(String model) → OooO0oo.get(model)

// Ajouter un plugin téléchargé au cache
OooO0O0(PluginPackageInfo) → parcourt getModelList(), put dans OooO0oo

// Ajouter un plugin installé au cache
OooO0OO(PluginPackageInfo) → parcourt getModelList(), put dans OooO0oO

// Construire JSON pour requête API
OooO0o(String model) → {model, package_id, type}
```

### 5.3 Initialisation des caches

Au démarrage (`54/00002554.java:435-445`) :
1. `OooO0oO.clear()` puis scan du répertoire `install/` (via `OooO0OO()`)
2. `OooO0oo.clear()` puis scan du répertoire `download/` (via `OooO0oo()`)
3. Nettoyage des vieux SDKs (versions obsolètes dans `plugin/rnsdk/`)

Le scan examine les fichiers dans `install/rn/<plugin_id>/` et `download/<package_id>.{rn,mpk,apk,h5}`, parse les noms pour extraire les IDs, et construit les `PluginPackageInfo`.

---

## 6. Persistance des métadonnées

**Fichier**: `SharedPreferences` nommé `"plugin_record_list.parcel2"`  
**Constante**: `PluginPackageConstant.PLUGIN_RECORD_LIST_PREF`  
**Ouvert par**: `gq9` (PluginUpdateManager) — `c4/00000fc4.java:45`

### 6.1 Format des clés

`"rn_sdk_timestamp_<sdkId>"` — généré par `pn9.OooO00o(long)` (`54/00002554.java:63`)

### 6.2 Format des valeurs

| Préfixe | Signification |
|---------|---------------|
| `"!"` + apiLevel | SDK **interdit** si l'API level de l'app est trop bas |
| `"#"` + timestamp | SDK **disponible** avec timestamp de téléchargement |

La gestion est dans `gq9.OooOO0o()` (`c4/00000fc4.java:265-320`) :
1. Valeur `"!"` → comparer avec `lp9.OooO0O0` ; si app trop vieille, refuser
2. Valeur `"#"` → vérifier si SDK existe déjà sur disque via `pn9.OooO0Oo()`
3. Si aucune des deux → déclencher téléchargement

---

## 7. API Réseau

### 7.1 `POST /v2/device/check_device_version`

**Utilisation**: Vérification des mises à jour firmware et plugin  
**Fichiers**: `53/00012453.java:74`, `a8/000025a8.java:277`, `07/00011707.java:323`

**Requête** :
```json
{
  "did": "<device did>",
  "platform": "android",
  "app_level": "<versionCode de l'app>",
  "plugin_level": "<version du plugin installé, ou '96'>"
}
```

**Transport**: `xr2.OooO0oo(ArrayList, KeyValuePair("data", json.toString()), "POST", url, ArrayList)` → `NetRequest` → `ggc.OooO00o().OooO0O0()` avec RC4.

### 7.2 `POST /v2/plugin/fetch_plugin` — **API principale de téléchargement**

**Utilisation**: Récupérer les URLs de téléchargement des plugins  
**Fichier**: `32/00011d32.java:383`

**Requête** :
```json
{
  "is_car_mihome": false,
  "latest_req": {
    "api_level": <app api level>,
    "app_platform": "android",
    "region": "<serveur régional>",
    "plugins": [
      {
        "model": "segway.ninebot.es1",
        "package_id": <id>,
        "type": ""
      }
    ],
    "package_type": ""
  },
  "backup_req": {
    "plugins": [
      {
        "model": "segway.ninebot.es1",
        "package_id": <id>,
        "type": ""
      }
    ],
    "api_level": <app api level>,
    "app_platform": "phone"
  }
}
```

### 7.3 `POST /v2/plugin/fetch_sdk`

**Utilisation**: Mise à jour du SDK React Native  

**Requête** :
```json
{
  "platform": "android",
  "sdk_version": [<versions actuellement installées>]
}
```

### 7.4 Autres endpoints plugin

| Endpoint | Usage | Fichier |
|----------|-------|---------|
| `POST /v2/plugin/get_support_models` | Récupérer la liste des modèles standard | `10/00001017.java:86` |
| `POST /v2/plugin/category_standard_model` | Catégorie de modèle standard | `ab/000127ab.java:83` |
| `POST /v2/plugin/gen_presigned_url` | Générer URL présignée (upload) | `3c/00003b3c.java:159` |
| `POST /v2/plugin/flag_upload_status` | Signaler statut upload | `3c/00003b3c.java:106` |

### 7.5 Pipeline réseau

```
JSONObject → toString() → KeyValuePair("data", ...) → xr2.OooO0oo() → NetRequest
  → ggc.OooO00o().OooO0O0(NetRequest, Callback)
  → Chiffrement RC4 (Crypto.RC4)
  → Envoi HTTP POST
```

---

## 8. Sécurité du bundle React Native

### 8.1 Algorithmes

| Propriété | Constante | Valeur |
|-----------|-----------|--------|
| Hash | `RN_BUNDLE_HASH_ALGORITHM` | `"SHA-256"` |
| Signature | `RN_BUNDLE_SIGNATURE_ALGORITHM` | `"SHA256withECDSA"` |
| Format certificat | `RN_CERT_FLAG` | `"-----BEGIN "` |
| Longueur min signature | — | 1024 octets |

### 8.2 Processus de vérification

**Méthode**: `qn9.OooO00o(String path)` (`a5/000027a5.java:48`)

1. Résoudre le chemin réel via `OooO0o(str)` (sélection `android`/`_android`)
2. Lire `project.json` depuis le disque
3. Charger le certificat depuis `bundle.cert`
4. Charger la signature depuis `bundle.sign`
5. Vérifier la signature ECDSA du `project.json` avec le certificat
6. Vérifier le hash SHA-256 du contenu du bundle

Cette vérification est effectuée :
- Au moment du scan des plugins (`scanDownloadedPackageInternal`)
- Avant l'installation d'un plugin téléchargé
- La méthode `qn9.OooOO0O(pluginPackageInfo)` valide le package

---

## 9. Classes clés et leurs responsabilités

| Classe | Fichier | Rôle |
|--------|---------|------|
| `pn9` | `54/00002554.java` | **PluginPackageManager** — singleton, gestion des caches, scan, init |
| `qn9` | `a5/000027a5.java` | **PluginPackageUtils** — utilitaires (paths, vérif signature) |
| `gq9` | `c4/00000fc4.java` | **PluginUpdateManager** — téléchargement, mise à jour, SharedPrefs |
| `qp9` | `c6/000027c6.java` | **PluginUpdateApi** — appels API réseau |
| `tp9` (tm9) | — | **PluginDownExec** — exécution des téléchargements |
| `h7c` | `df/000010df.java` | **FileUtils** — opérations fichiers, getAppFilesDir |
| `PluginPackageInfo` | `c6/000118c6.java` | Structure de données du plugin (Parcelable) |
| `PluginPackageConstant` | `7a/0001397a.java` | Constantes (chemins, types RN/MPK/etc.) |
| `DefaultPluginFileManager` | `8e/0000348e.java` | Gestion fichiers plugins installés |
| `xr2` | — | Construction des `NetRequest` |
| `ggc` | — | Client HTTP avec chiffrement RC4 |
| `lp9` | — | API level de l'application |
| `fhe` | — | Téléchargement via `DownloadManager` système |
| `v38` | — | Contexte global de l'application |

---

## 10. Cycle de vie d'un plugin

### 10.1 Déclenchement

L'utilisateur clique sur un appareil → `DeviceRenderer.onItemClick()` (`32/00011d32.java`)

### 10.2 Vérification du plugin installé

```java
pn9.OooOO0O().OooO(model)  // chercher dans OooO0oO (installé)
```

Si trouvé → lancer l'activité RN directement.
Si non trouvé → vérifier dans `OooO0oo` (téléchargé).

### 10.3 Fetch des infos de mise à jour

```java
gq9.OooO0oO(String model, Callback) → qp9.OooO0o0(JSONArray, int, Callback)
  → NetRequest POST /v2/plugin/fetch_plugin
  → Réponse : {package_id, download_url, version, ...}
```

### 10.4 Téléchargement

```java
tm9.OooO0O0(um9 downloadTask)  // PluginDownExec
  → fhe (DownloadManager système ou interne)
  → Fichier : <base>/plugin/download/<package_id>.rn
```

### 10.5 Installation

```java
pn9.OooO0oO()  // Installation du plugin
  → Extraction du .rn (zip) → plugin/install/rn/<plugin_id>/
  → Vérification de la signature ECDSA
  → Mise à jour du cache OooO0oO
  → Nettoyage du fichier .rn téléchargé
```

### 10.6 Exécution

```java
PluginRNActivity / RNRuntime  // Charge le bundle depuis
  <base>/plugin/install/rn/<plugin_id>/<_android|android>/sdk.bundle
```

---

## 11. Le plugin Ninebot

### 11.1 Identification

- **Modèle**: `"segway.ninebot.es1"` (et variantes : `es2`, `es4`, `max`, `g30`, etc.)
- **Type de plugin**: **RN** (React Native) — confirmé par `OooOOO0()` (isRNPlugin)
- **Plugin ID**: Attribué par le serveur, associé au modèle

### 11.2 Emplacement

```
/data/data/com.xiaomi.smarthome/files/plugin/
├── download/<package_id>.rn          ← Archive téléchargée
└── install/rn/<plugin_id>/           ← Contenu extrait
    ├── conf.json
    ├── project.json
    ├── bundle.sign
    ├── bundle.cert
    └── _android/sdk.bundle           ← Le vrai bundle RN
```

### 11.3 Récupération sans root

Le répertoire `/data/data/com.xiaomi.smarthome/files/` est un répertoire de stockage **interne**. Il n'est pas accessible via MTP ou ADB sans root sur Android 11+.

Méthodes potentielles (limitées) :
1. **Sauvegarde ADB** : `adb backup -f backup.ab com.xiaomi.smarthome` (souvent désactivée par les OEMs sur Android 12+)
2. **Bug report** : Peut capturer les noms de fichiers, pas le contenu
3. **Root** : Accès complet
4. **Custom recovery** : Extraction via TWRP

---

## 12. Codes et constantes importantes

### 12.1 Types de plugins

```java
PLUGIN_TYPE_RN  = "rn"    // React Native
PLUGIN_TYPE_MPK = "mpk"   // Natif
PLUGIN_TYPE_APK = "apk"   // APK Android
PLUGIN_TYPE_H5  = "h5"    // WebView H5
PLUGIN_TYPE_RN_SDK = "rnsdk"
```

### 12.2 Statuts SDK

```java
RN_SDK_UNKNOWN     = 0
RN_SDK_FAILURE     = 1
RN_SDK_UPDATING    = 2
RN_SDK_UPDATED     = 3
RN_SDK_DOWNLOADING = 4
RN_SDK_DOWNLOADED  = 5
RN_SDK_INSTALLING  = 6
RN_SDK_INSTALLED   = 7
RN_SDK_FORBIDDEN   = 10
```

### 12.3 Identifiants valides

```java
IS_VALID_RN_PACKAGE_ID(long): j2 >= 1000000
IS_VALID_RN_PLUGIN_ID(long):  j2 >= 1000000
```

### 12.4 Noms de fichiers spéciaux

```java
RN_CONF_A      = "conf.json"
RN_CONF_B      = "_conf.json"
RN_DIR_A       = "android"
RN_DIR_B       = "_android"
RN_PROJECT_JSON = "project.json"
```

---

## 13. Flux d'initialisation au démarrage

```
Application.onCreate()
  → pn9.<init>()              // PluginPackageManager
    → qn9.OooO0Oo(context)    // getAppFilesDir (caché)
    → Initialisation des chemins (download, install, tmp, rnsdk, lib)
    → Clear caches OooO0oO + OooO0oo
    → scanDownloadedPackageInternal()  // Scanne download/
    → scanInstalledPackageInternal()   // Scanne install/
    → Nettoyage vieux SDKs dans rnsdk/
  
  → gq9.<init>()               // PluginUpdateManager
    → Ouverture SharedPreferences "plugin_record_list.parcel2"
    → Initialisation du PluginDownExec (tm9)
    → Vérification des SDKs à mettre à jour
```

---

## 14. Gestion des erreurs et logging

L'application utilise un système de logging interne via `cg8` avec tracing de performance :
```java
cg8.OooOOo(jCurrentTimeMillis, "com.xiaomi.smarthome.server.PluginPackageManager->init");
```

Les logs plugins utilisent `w38.OooOO0(LogType.PLUGIN, tag, ...)`.

Les erreurs de plugin sont encapsulées dans `PluginError` (package `com.xiaomi.smarthome.plugin.update`), avec des codes d'erreur comme :
- `-1` : Erreur générique
- `-2` : Service core non enregistré
- `DOWNLOADERROR_SDKVERSION` : Version SDK trop ancienne

---

## 15. Architecture orientée performance

Chaque méthode mesurable est chronométrée :
```java
long t = System.currentTimeMillis();
// ... opération ...
com.trace.slowmethod.OooO0O0.OooO0O0(System.currentTimeMillis() - t, "method_signature");
```

Les opérations lourdes (scan, téléchargement) sont déléguées à des threads de fond via :
- `kz7` (HandlerThread personnalisé)
- `xd7` (Handler pour tâches différées)
- `Schedulers.io()` (RxJava)

Le Handler `OooO0Oo` (main thread) délivre les callbacks UI pendant que `OooO0o` (worker thread) exécute les opérations plugins.

---

## 16. Relations entre classes

```
pn9 (PluginPackageManager)
  ├── OooO0oO : Map<model, PluginPackageInfo> (installé)
  ├── OooO0oo : Map<model, PluginPackageInfo> (téléchargé)
  ├── OooO(String model) → get install info
  ├── OooO0oo(String model) → get download info
  ├── OooO0o(String model) → JSON pour API
  ├── OooO0oO(int, String, long, String, PluginPackageInfo) → delete old
  └── init() → scan, nettoyage

gq9 (PluginUpdateManager)
  ├── OooO0o0 : SharedPreferences (plugin_record_list.parcel2)
  ├── OooO00o : tm9 (PluginDownExec)
  ├── OooO0o : Map<Long, fq9> (tâches SDK en cours)
  ├── OooO0oO(String model, Callback) → fetch latest plugin
  ├── OooOO0o(long sdkId, um9 task) → start/load RN SDK
  └── OooOO0O(um9 task) → téléchargement effectif

qp9 (PluginUpdateApi)
  ├── OooO0o(JSONArray, int, Callback) → POST /v2/plugin/fetch_plugin
  └── OooO0o(JSONArray, Callback) → POST /v2/plugin/fetch_sdk

qm9 (PluginPackageUtils - qn9)
  ├── OooO0Oo(Context) → getAppFilesDir
  ├── OooO0o(String) → resolve android/_android path
  └── OooO00o(String) → check ECDSA signature
```

---

## 17. Glossaire des obfuscations

| Nom obfusqué | Nom réel probable | Fichier |
|-------------|-------------------|---------|
| `pn9` | `PluginPackageManager` | `54/00002554.java` |
| `qn9` | `PluginPackageUtils` | `a5/000027a5.java` |
| `gq9` | `PluginUpdateManager` | `c4/00000fc4.java` |
| `qp9` | `PluginUpdateApi` | `c6/000027c6.java` |
| `tm9` | `PluginDownExec` | — |
| `h7c` | `FileUtils` | `df/000010df.java` |
| `lp9` | `SdkSetting` (API level) | — |
| `v38` | `GlobalContext` | — |
| `xr2` | `NetRequestBuilder` | — |
| `ggc` | `HttpClient` | — |
| `fhe` | `DownloadManagerWrapper` | — |
| `cg8` | `TraceLog` (logging perf) | — |
| `w38` | `PluginLogger` | — |
| `hq9` | `ServerConfig` | — |
| `d4d` | `AppConfig` | — |

---

## 18. Recherche des modèles Ninebot

### 18.1 Modèles internes identifiés

Les modèles Ninebot utilisés dans le code ne sont PAS `segway.ninebot.es1` comme supposé initialement. Les vrais modèles internes sont :

| Modèle | Fichier | Usage |
|--------|---------|-------|
| `"ninebot.scooter.v1"` | `5e/00011c5e.java`, `65/00011c65.java`, `d7/000012d7.java` | Trottinette Ninebot (ES1/ES2/ES4) |
| `"ninebot.scooter.v2"` | Mêmes fichiers | Trottinette Ninebot (ES3/MAX/G30) |
| `"ninebot.balscooter.v1"` | `50/00002150.java`, `c2/000033c2.java`, `10/00002810.java` | Ninebot Balance Scooter |

Aucune référence à `es1`, `es2`, `es4`, ou `g30` n'a été trouvée comme chaîne littérale dans le code. Les modèles sont abstraits par les identifiants `v1`, `v2`.

### 18.2 Classe dédiée : `BleMatchNinebotImageFragment`

**Fichier**: `65/00011c65.java` (BleMatchNinebotImageFragment)  
**Package**: `com.xiaomi.smarthome.ble`  
**Rôle**: Fragment UI pour l'appairage BLE des trottinettes Ninebot

Points clés :
- Ligne 64 : `public class BleMatchNinebotImageFragment extends BleMatchBaseFragment`
- Ligne 190-197 : utilise les modèles `ninebot.scooter.v1` / `ninebot.scooter.v2`
- Affiche des images de appairage spécifiques (`pair_image_ninebot_preview`, etc.)
- Classe avec 9 classes internes anonymes (gestion du scan BLE, connexion, etc.)

Les modèles Ninebot sont traités comme des devices BLE standards nécessitant un plugin RN pour l'interface utilisateur complète.

### 18.3 Traces potentielles dans SharedPreferences / MMKV / SQLite

**SharedPreferences `plugin_record_list.parcel2`** :
- Clés : `"rn_sdk_timestamp_<sdkId>"` (concerne les SDKs, pas les plugins individuels)
- Ne stocke PAS le plugin_id du Ninebot directement

**MMKV — IDs utilisés dans l'application** :
| ID MMKV | Usage |
|---------|-------|
| `"globaldynamicsetting"` | Paramètres globaux dynamiques |
| `"std.plugin.launch_config"` | Configuration de lancement des plugins |
| `"car_model_allow_cache"` | Modèles de voiture autorisés |
| `"rn_debug_enable"`, `"rn_debug_model"`, `"rn_debug_package_name"` | Debug RN (MMKV défaut) |

**SQLite** : Aucune base SQLite dédiée au stockage des métadonnées de plugins n'a été identifiée.

### 18.4 Conclusion Phase 1

Le `plugin_id` du Ninebot n'est pas stocké de façon persistante lisible directement. Il est :
1. Récupéré depuis le serveur via `POST /v2/plugin/fetch_plugin`
2. Conservé en mémoire dans `pn9.OooO0oO` (cache installé) et `pn9.OooO0oo` (cache téléchargé)
3. Accessible via `pn9.OooO0o(String model)` qui retourne `{model, package_id, type}`

Le seul moyen d'obtenir le `plugin_id` est d'intercepter la réponse réseau ou d'inspecter la structure de répertoires `install/rn/<plugin_id>/`.

---

## 19. Format de la réponse `fetch_plugin`

### 19.1 Classe de requête

**Fichier**: `32/00011d32.java` (DeviceRenderer$5.OooO0OO)  
**Endpoint**: `POST /v2/plugin/fetch_plugin`  
**Transport**: RC4 (ggc = SmartHomeRc4Api)

### 19.2 Structure des modèles de réponse

**Modèle de tâche de téléchargement interne (`um9`)** :

| Champ obfusqué | Type | Signification probable |
|----------------|------|------------------------|
| `OooO0OO` | `String` | `model` |
| `OooO00o` | `int` | `taskId` |
| `OooO0Oo` | `long` | `pluginId` |
| `OooO0o0` | `long` | `packageId` |
| `OooO0oO` | `String` | `packageType` |
| `OooO0oo` | `String` | `packageSafeUrl` |
| `OooOO0` | `String` | `packageUrl` |
| `OooOO0O` | `long` | `packageLength` |
| `OooOO0o` | `boolean` | `isForce` |
| `OooOOO0` | `int` | `version` |

**Modèle externe (`PluginDownloadTask`)** — `c6/000118c6.java` :

| Champ | Type | Signification |
|-------|------|---------------|
| `OooOooo` | `long` | `packageId` |
| `Oooo00O` | `int` | (taskId probable) |
| `Oooo00o` | `String` | — |
| `Oooo0` | `String` | — |
| `Oooo0O0` | `int` | — |
| `OooOoo0` | `String` | `model` |
| `OooOoo` | `int` | `version` |
| `OooOooO` | `String` | `packageType` |

**Modèle de module RN externe (`yya`)** — `30/00003d30.java` :

| Champ JSON | Champ Java | Type |
|------------|------------|------|
| `"module_name"` | `module_name` | `String` |
| `"md5"` | `md5` | `String` |
| `"download_url"` | `download_url` | `String` |
| `"app_abi"` | `app_abi` | `String` |

### 19.3 Réponse non décompilée

La méthode principale de parsing `up9.onSuccess(Object)` (dans `PluginUpdateManager$2`) contient **832 instructions** et n'a PAS été décompilée par jadx. C'est dans cette méthode que :
- La réponse JSON du serveur est parsée
- Les champs `package_url`, `safe_package_url`, `plugin_id`, `package_id`, `version` sont extraits
- Les objets `um9` (tâches de téléchargement) sont construits

### 19.4 Champs JSON identifiés indirectement

Par analyse croisée des modèles et des appels :

```json
// Réponse probable de fetch_plugin
{
  "plugins": [
    {
      "plugin_id": <long>,
      "package_id": <long>,
      "version": <int>,
      "package_url": "<URL de téléchargement>",
      "safe_package_url": "<URL de backup>",
      "package_length": <long>,
      "package_type": "rn",
      "is_force": false,
      "md5": "<checksum>"
    }
  ]
}
```

Le flux est :
1. `gq9.OooO0oO(String model, Callback)` → construit la requête
2. `qp9.OooO0o0(JSONArray, int, Callback)` → envoie via `ggc`
3. Callback → parse la réponse → construit `um9`
4. `tm9.OooO0O0(um9)` → exécute le téléchargement via `fhe` (DownloadManager)
5. Fichier → `<base>/plugin/download/<package_id>.rn`
6. Installation → extraction vers `<base>/plugin/install/rn/<plugin_id>/`

---

## 20. Authentification et chiffrement

### 20.1 Architecture d'authentification

Tous les appels API vers Xiaomi passent par un système à 3 couches :

```
Couche 1: Auth (cookies)
  └── serviceToken + cUserId + userId + yetAnotherServiceToken
      
Couche 2: Chiffrement (selon mode)
  ├── Crypto.RC4  → ggc (SmartHomeRc4Api)   ← utilisé par défaut
  ├── Crypto.NONE → bgc (SmartHomeOpenApi)   ← basique
  └── Crypto.HTTPS→ tec (SmartHomeHttpsApi)  ← HTTPS only

Couche 3: Transport
  └── OkHttpClient avec MijiaCookie
```

### 20.2 Cookies requis

Les cookies suivants sont automatiquement attachés par `MijiaCookie` (`31/00011931.java`) à chaque requête :

| Cookie | Source | Description |
|--------|--------|-------------|
| `serviceToken` | `MiServiceTokenInfo.OooO00o()` | Token d'authentification principal |
| `cUserId` | `MiServiceTokenInfo.OooO0o()` | User ID chiffré |
| `yetAnotherServiceToken` | `MiServiceTokenInfo.OooO00o()` | Token secondaire |
| `userId` | `LoginManager` | ID utilisateur clair |
| `PassportDeviceId` | Device ID | Identifiant de l'appareil |
| `channel` | config | Canal de distribution |
| `country_code` | config | Code pays |
| `locale` | config | Locale |

**Classe**: `MijiaCookie.OooO0OO(MiServiceTokenInfo, String)` — ligne 75-117

### 20.3 Flux d'authentification complet

```
Application.onCreate()
  → LoginManager.login() ou restoreSession()
    → OAuth2 / passport.api → {serviceToken, cUserId, userId, ssecurity}
    
      ↓
      
Appel API (ex: fetch_plugin)
  → CoreApi.sendSmartHomeRequest(netRequest, crypto, callback)
    → CoreApiStub vérifie le process calling (Binder.getCallingPid())
    → Dispatch selon crypto :
        RC4 → ggc.OooO00o().OooO0O0(request, callback)
        
          ↓
          
ggc (SmartHomeRc4Api)
  → OkHttpClient avec interceptors
    → MijiaCookie (CookieManager)
      → Attache : serviceToken, cUserId, userId, etc.
    → Chiffrement du body "data" avec RC4
    → Envoi HTTP POST
```

### 20.4 Possibilité de répliquer sans authentification

**Sans les tokens Xiaomi, il est impossible de :**
1. Déchiffrer le body RC4 (clé dérivée du `ssecurity`)
2. Passer l'authentification par cookie
3. Obtenir une réponse valide du serveur

**La reproduction de `POST /v2/plugin/fetch_plugin` nécessite :**
- `serviceToken` (valide, non expiré)
- `cUserId`
- `userId`
- `PassportDeviceId`
- `ssecurity` (pour le RC4)

**Clé RC4** : Dérivée du `ssecurity` (secret partagé obtenu lors du login OAuth). Sans le `ssecurity`, le body chiffré ne peut pas être déchiffré.

### 20.5 Classes clés pour le réseau

| Classe | Fichier | Rôle |
|--------|---------|------|
| `Crypto` (enum) | `bd/000118bd.java` | Modes NONE/RC4/HTTPS |
| `ggc` (SmartHomeRc4Api) | `26/00000f26.java` | Client HTTP avec RC4 |
| `bgc` (SmartHomeOpenApi) | `8e/0000038e.java` | Client HTTP sans chiffrement |
| `tec` (SmartHomeHttpsApi) | — | Client HTTPS |
| `MijiaCookie` | `31/00011931.java` | Gestionnaire de cookies auth |
| `pec` (CookieManager base) | `c5/000024c5.java` | Classe de base cookie |
| `NetRequest` | `c0/000118c0.java` | Structure de requête |
| `KeyValuePair` | `be/000118be.java` | Paire clé-valeur pour params |
| `xr2` | `16/00003816.java` | Builder de NetRequest |
| `qme` (Builder) | `9a/0000279a.java` | Builder interne |
| `v38` (MiJiaEnvironment) | `ed/000031ed.java` | Contexte global |
| `h09` (OkHttpConfig) | `65/00001065.java` | Configuration OkHttp |
| `MiServiceTokenInfo` | — | Contient serviceToken, cUserId, ssecurity |
| `LoginManager` | — | Gère la session utilisateur |

---

## 21. Structure du bundle React Native Xiaomi

### 21.1 Moteur JavaScript

Le bundle RN Xiaomi utilise **V8** comme moteur JavaScript principal :

```java
// 3c/0001263c.java:181
public V8ExecutorFactory o00Ooo;

// 3c/0001263c.java:792
V8ExecutorFactory v8ExecutorFactory = new V8ExecutorFactory();
```

| Moteur | Bibliothèque | Utilisé ? |
|--------|-------------|-----------|
| **V8** | `V8ExecutorFactory` | **OUI** — moteur principal |
| Hermes | `libhermes.so`, `libhermes-executor-release.so` | Présent mais PAS utilisé par défaut |
| JSC | `libjscexecutor.so`, `JSCManagerV2` | Présent mais PAS utilisé par défaut |

### 21.2 Fichiers du bundle

Un bundle RN Xiaomi installé contient :

```
<base>/plugin/install/rn/<plugin_id>/
├── project.json              ← Manifeste (toujours présent)
├── main.bundle               ← Bundle JS principal (V8, texte)
├── sdk.bundle                ← Bundle de fallback (si main.bundle absent)
├── bundle.sign               ← Signature ECDSA
└── bundle.cert               ← Chaîne de certificats X.509
```

**Ordre de chargement** (RNRuntime.loadScriptCache, `3c/0001263c.java:1290-1386`) :
1. Cherche `main.bundle` dans `android/` ou `_android/`
2. Si absent, cherche dans les feature-bundles
3. Si toujours absent, cherche `sdk.bundle`
4. Vérifie le type "indexed-ram-bundle" (format RAM bundle)

### 21.3 Format du bundle

Le bundle est un **fichier JavaScript texte brut** (pas du bytecode Hermes) :
- V8 exécute du JS texte, pas du bytecode propriétaire
- Le bundle peut être lu directement avec n'importe quel éditeur de texte
- La structure est un bundle React Native standard :
  ```javascript
  // modules du bundle
  __d(function(...) { ... });  // définition de module
  __r(moduleId);               // require
  // code du bundle ...
  ```

### 21.4 Décompilation possible

| Outil | Bundle type | Compatible |
|-------|-------------|------------|
| `react-native-decompiler` | JS texte (V8) | **OUI** |
| Décompilateur Hermes | Bytecode `.hbc` | Non applicable (V8) |
| Lecture directe (cat/less) | JS texte | **OUI** |
| Beautify JS | JS texte | **OUI** |

**Le bundle Ninebot est un fichier JS texte standard.** Il n'est pas compilé en bytecode Hermes. Tout décompilateur JS standard peut être utilisé.

### 21.5 Compression

Le fichier `.rn` téléchargé est une archive **Zip** standard :
```
<package_id>.rn  ← fichier Zip
├── project.json
├── main.bundle (ou sdk.bundle)
├── bundle.sign
└── bundle.cert
```

Méthodes de décompression utilisées dans le code :
- `ZipFileUtils.doUnzip()` — `fb/000029fb.java:529`
- `YRNFileUtils.unZipFile()` — `32/00014b32.java:308`
- `MihomeJsBridge.unzip` — `8f/00002d8f.java:111`

### 21.6 Vérification d'intégrité

Tout bundle téléchargé est vérifié par signature ECDSA avant installation :
```
SHA-256(project.json) + SHA-256(main.bundle)
  → signé avec clé privée du développeur
  → vérifié avec bundle.cert → chaîne jusqu'à MijiaRootCert.der
```

Cette vérification est effectuée dans `qn9.OooO00o(String)` (`a5/000027a5.java:48`).

---

## 22. Comparaison des stratégies d'obtention du bundle

### 22.1 Analyse détaillée des 4 méthodes

#### A. Extraction depuis un téléphone rooté

| Critère | Évaluation |
|---------|------------|
| **Difficulté** | Faible (1/5) — `cp -r /data/data/.../files/plugin/install/rn/<id>/_android/sdk.bundle` |
| **Temps** | 5-15 minutes |
| **Probabilité de succès** | 100% (avec root) |
| **Prérequis** | Téléphone rooté + Xiaomi Home + Ninebot appairé |
| **Avantages** | Accès complet à tous les fichiers ; pas de reverse engineering |
| **Inconvénients** | Nécessite un téléphone rooté ; le bundle peut être signé différemment selon la version |

#### B. Interception MITM

| Critère | Évaluation |
|---------|------------|
| **Difficulté** | Très élevée (5/5) — le trafic est chiffré en RC4 |
| **Temps** | 2-8 heures (installation du certificat + contournement pinning + reverse RC4) |
| **Probabilité de succès** | 30-50% |
| **Prérequis** | Proxy MITM (Burp/Charles) + contournement SSL pinning + reverse de la clé RC4 |
| **Avantages** | Pas besoin de root ; capture en temps réel |
| **Inconvénients** | Le trafic est chiffré en RC4 (pas seulement HTTPS) ; nécessite de hooker la clé RC4 avec Frida ; le certificat Xiaomi peut avoir du pinning |

#### C. Instrumentation Frida

| Critère | Évaluation |
|---------|------------|
| **Difficulté** | Moyenne (3/5) |
| **Temps** | 30 minutes à 2 heures |
| **Probabilité de succès** | 90% |
| **Prérequis** | Frida + USB debugging + APK avec Frida gadget (sans root) ou Frida server (root) |
| **Avantages** | **Fonctionne sans root** (avec Frida gadget injecté dans l'APK) ; peut hooker les opérations fichier, réseau, ou les deux |
| **Inconvénients** | Doit repackager l'APK si pas de root ; le hook réseau nécessite de comprendre le RC4 |

**Script Frida recommandé** (le plus simple) :
```javascript
// Hook FileOutputStream pour capturer sdk.bundle lors de l'écriture
var FileOutputStream = Java.use('java.io.FileOutputStream');
FileOutputStream.$init.overload('java.io.File').implementation = function(file) {
    if (file.getPath().indexOf('sdk.bundle') !== -1 ||
        file.getPath().indexOf('main.bundle') !== -1) {
        console.log('[WRITE] ' + file.getPath());
        // Lire le chemin et capturer plus tard
    }
    return this.$init(file);
};
```

**Alternative** — hooker le `ZipFile` pour capturer l'extraction :
```javascript
var ZipFile = Java.use('java.util.zip.ZipFile');
ZipFile.$init.overload('java.lang.String').implementation = function(path) {
    if (path.indexOf('.rn') !== -1) {
        console.log('[ZIP] ' + path);
        // Le bundle peut être extrait depuis cette archive
    }
    return this.$init(path);
};
```

#### D. Rejeu de l'API fetch_plugin

| Critère | Évaluation |
|---------|------------|
| **Difficulté** | Élevée (4/5) |
| **Temps** | 2-4 heures |
| **Probabilité de succès** | 60% |
| **Prérequis** | Tokens Xiaomi valides (serviceToken, cUserId, userId, ssecurity) + implémentation RC4 compatible |
| **Avantages** | Pas besoin de téléphone physique après extraction des tokens |
| **Inconvénients** | Les tokens expirent ; le RC4 nécessite le `ssecurity` ; les tokens sont liés à l'appareil (PassportDeviceId) ; le serveur peut rejeter les requêtes non standards |

### 22.2 Tableau comparatif

| Méthode | Difficulté | Temps | Succès | Root ? | Notes |
|---------|-----------|-------|--------|--------|-------|
| **A. Root** | ★☆☆☆☆ | 15 min | 100% | OUI | Extraction directe du fichier |
| **B. MITM** | ★★★★★ | 4h+ | 30% | NON | Bloqué par RC4 + pinning |
| **C. Frida** | ★★★☆☆ | 1-2h | 90% | NON* | *Frida gadget sans root, Frida server avec root |
| **D. API replay** | ★★★★☆ | 3h | 60% | NON | Nécessite tokens + implémentation RC4 |

### 22.3 Recommandation finale

**La méthode la plus rapide et fiable pour obtenir le bundle React Native Ninebot est :**

> **1er choix** : **Frida sans root** (injection Frida gadget dans l'APK) — hooker `java.io.File` ou `java.util.zip.ZipFile` pour capturer `sdk.bundle` / `main.bundle` lors de son chargement par l'application.
>
> **2ème choix** : **Extraction via téléphone rooté** (copie directe) — si un appareil rooté est disponible.
>
> **3ème choix** : **ADB backup** (`adb backup -f backup.ab com.xiaomi.smarthome`) — si l'application l'autorise (test rapide).

### 22.4 Procédure recommandée (Frida + hook fichier)

Étape par étape :

```
1. Prérequis :
   - Téléphone Android avec USB debugging activé
   - ADB installé
   - Appareil Ninebot déjà appairé dans Xiaomi Home

2. Option A — Avec root :
   adb push frida-server /data/local/tmp/
   adb shell /data/local/tmp/frida-server &
   frida -U -l capture_bundle.js com.xiaomi.smarthome

3. Option B — Sans root (Frida gadget) :
   apktool d com.xiaomi.smarthome.apk
   Copier frida-gadget.so dans lib/<arch>/
   Ajouter <uses-permission android:name="android.permission.INTERNET"/>
   apktool b com.xiaomi.smarthome
   jarsigner ...
   adb install ...

4. Hook Frida (capture_bundle.js) :
   - Hooker ZipFile.$init(path) pour capturer les chemins .rn
   - Hooker FileOutputStream.$init(file) pour capturer sdk.bundle
   - Hooker ZipEntry.getInputStream() pour lire le contenu
   - Déclencher l'ouverture du plugin Ninebot dans l'app
   - Le bundle est capturé automatiquement

5. Analyser le bundle :
   - Le fichier capturé est un JS texte (V8)
   - Chercher les fonctions : getSpeed(), getBattery(), getMileage()
   - react-native-decompiler ou lecture directe
```

### 22.5 Accès aux propriétés batterie/vitesse/kilométrage

Une fois le `sdk.bundle` ou `main.bundle` obtenu :
1. Le bundle est du **JavaScript React Native standard** (V8, pas Hermes)
2. Chercher les fonctions BLE/MiOT natives :
   - `getSpeed()` / `speed` / `current_speed`
   - `getBattery()` / `battery` / `battery_level`
   - `getMileage()` / `mileage` / `total_mileage` / `odometer`
   - `getRange()` / `range` / `estimated_range`
3. Ces propriétés sont généralement exposées via le bridge RN → `NativeModules` → `BleDevice` / `Device`
4. Le format étant du JS texte, tout éditeur de texte ou beautifier JS peut être utilisé pour l'analyse

### 22.6 Note importante sur le modèle Ninebot

Le modèle à utiliser dans les requêtes API est **`"ninebot.scooter.v1"`** (ou `"ninebot.scooter.v2"`), **pas** `"segway.ninebot.es1"`. Le `package_id` / `plugin_id` est attribué dynamiquement par le serveur et peut varier selon la version du firmware de la trottinette et la région du compte Xiaomi.
