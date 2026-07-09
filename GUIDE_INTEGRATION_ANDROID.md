# Guide d'Intégration ID360 SDK Android

Ce guide vous accompagne pas à pas dans l'intégration du SDK ID360 dans votre application Android, de l'implémentation la plus simple (flux natif standard) aux cas d'usage avancés.

Pour télécharger le SDK : [releases](https://github.com/id360docaposte/id360docaposte.github.io/releases)
---

## 🌐 Intégration d'un parcours ID360 en utilisant le SDK (parcours clé en main)


Le SDK ouvre l'URL d'enrôlement ID360 dans une WebView sécurisée. L'application web pilote le parcours et déclenche les captures natives du téléphone via le pont JavaScript du SDK. L'application Android récupère ensuite le statut final de l'enrôlement.

### Étape 1 : Lancement de la WebView
Vous devez obtenir l'URL d'enrôlement depuis votre backend, puis la passer au SDK :

```kotlin
import com.docaposte.id360sdk.webview.ID360WebViewActivity

fun startWebViewFlow(enrollmentUrl: String) {
    val intent = ID360WebViewActivity.createIntent(
        context = this,
        url = enrollmentUrl, // URL du parcours ID360 fournie par votre backend
        language = "fr"
    )
    webViewLauncher.launch(intent)
}
```
> [!IMPORTANT]
> Ne modifiez pas l'URL fournie par votre backend. Le SDK gère lui-même la navigation interne.

### Étape 2 : Récupérer le statut final
Déclarez le launcher pour intercepter le signal de fin de parcours ou les éventuelles erreurs techniques :

```kotlin
import org.json.JSONObject

private val webViewLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    val payload = result.data?.getStringExtra(ID360WebViewActivity.RESULT_ENROLLMENT_PAYLOAD)
    
    if (!payload.isNullOrBlank()) {
        val status = JSONObject(payload).optString("status")
        // status possible : "OK", "KO", "FAILED", "CANCELED"
        // TODO: Traiter la fin de parcours selon le statut
        return@registerForActivityResult
    }

    // Gestion des erreurs matérielles ou système
    val errorCode = result.data?.getStringExtra(ID360WebViewActivity.RESULT_ERROR_CODE)
    val errorMessage = result.data?.getStringExtra(ID360WebViewActivity.RESULT_ERROR_MESSAGE)
    // Exemple : errorCode = "NFC_UNAVAILABLE" (si l'appareil n'a pas de puce NFC)
}
```

## 🔌 Intégration de la lecture NFC standalone (nécessite obligatoirement un appareil avec lecteur NFC et un document d'identité avec puce NFC)

Ce parcours est recommandé si vous souhaitez intégrer la lecture de document directement dans votre application en utilisant l'interface graphique par défaut du SDK.

Le SDK affiche ses propres écrans, gère la caméra et la puce NFC, puis vous renvoie un JSON contenant les données d'identité extraites.

### Étape 1 : Ajouter le SDK à votre projet
Dans votre fichier `settings.gradle.kts` :
```kotlin
include(":id360sdk")
```

Dans le fichier `build.gradle.kts` de votre module applicatif (ex: `:app`) :
```kotlin
dependencies {
    implementation(project(":id360sdk"))
}
```
> [!NOTE]
> Le SDK déclare automatiquement dans son manifest les permissions (Caméra et NFC) ainsi que les fonctionnalités matérielles requises. Vous n'avez aucune permission supplémentaire à déclarer dans votre manifeste principal.
> Si vous recevez le SDK sous forme d'archive `.aar` (release) au lieu du module source, la procédure est identique et l'API publique reste la même.

### Étape 2 : Lancer le flux et récupérer le résultat
Dans votre `Activity` ou `Fragment`, configurez le launcher standard d'Android pour lancer le flux de capture et traiter le JSON de sortie :

```kotlin
import android.app.Activity
import androidx.activity.result.contract.ActivityResultContracts
import com.docaposte.id360sdk.flow.ID360FlowActivity

// 1. Enregistrer le récepteur de résultat
private val nfcFlowLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        // Succès : récupération du JSON des données de la puce
        val nfcJson = result.data?.getStringExtra(ID360FlowActivity.RESULT_NFC_DATA)
        // TODO: Envoyer ce JSON à votre backend
    } else {
        // Annulation utilisateur ou erreur bloquante
        val error = result.data?.getStringExtra(ID360FlowActivity.RESULT_ERROR_MESSAGE)
        // TODO: Gérer l'erreur ou l'annulation
    }
}

// 2. Déclencher le flux
fun startStandardId360Flow() {
    val intent = ID360FlowActivity.createNfcReadIntent(
        context = this,
        language = "fr" // "fr", "en", etc.
    )
    nfcFlowLauncher.launch(intent)
}
```

### Étape 3 : Les données obtenues par défaut
Le JSON renvoyé dans `RESULT_NFC_DATA` contient les données d'identité "métier" prêtes à être envoyées à votre backend :
* **Identité** : `firstNames` (prénoms), `lastName` (nom), `gender` (genre), `nationality` (nationalité).
* **Document** : `documentNumber` (numéro), `birthDate` (date de naissance), `expiryDate` (date d'expiration).
* **Photo** : `faceImage` (photo d'identité encodée en Base64).

---

## ⚙️ Intégration des composants du SDK (pour une utilisation sur mesure)

Cette section documente les fonctionnalités d'intégration avancées du SDK pour les cas d'usage spécifiques.

### Paramètres de configuration (Optionnels)
Ces paramètres peuvent être ajoutés lors de l'appel aux méthodes de création d'intents :
* `retryThreshold` (NFC) : Nombre d'échecs NFC consécutifs avant de reproposer l'étape caméra MRZ (Par défaut : `3`).
* `keyId`, `masterKey` (NFC) : Clés pour la configuration PACE spécifique. À renseigner uniquement sur demande de vos équipes sécurité.
* `apiKey`, `apiUrl` (MRZ) : Identifiants pour l'upload d'images vers la plateforme d'enrôlement ID360 cloud.
* `documentName` (MRZ) : Nom du fichier sur le serveur d'enrôlement (Par défaut : `"scan"`).
* `uiCustomization` : Instance de `ID360UiCustomization` pour adapter les couleurs du SDK.

---

### Lecture NFC seule (composant IHM de lecture NFC)
Si vous connaissez déjà le numéro de document, la date de naissance et la date d'expiration (ex. saisis manuellement), vous pouvez sauter l'étape caméra :

```kotlin
val intent = ID360FlowActivity.createDirectNfcReadIntent(
    context = this,
    documentNumber = "AB1234567",
    dateOfBirth = "900115",    // Format YYMMDD (ex: 15 janv 1990 -> 900115)
    dateOfExpiry = "300115",   // Format YYMMDD
    documentType = "P",        // "P" (Passeport), "I" (Identité), etc.
    language = "fr"
)
nfcFlowLauncher.launch(intent)
```

---

### Lecture MRZ seule (Composant de lecture MRZ sans upload vers ID360)
Si vous souhaitez uniquement effectuer la capture caméra de la zone MRZ pour l'analyser localement sans interroger la puce NFC :

```kotlin
private val mrzFlowLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        val mrzJson = result.data?.getStringExtra(ID360FlowActivity.RESULT_MRZ_DATA)
        // Traiter les données MRZ décodées
    }
}

val intent = ID360FlowActivity.createMrzReadIntent(
    context = this,
    language = "fr"
)
mrzFlowLauncher.launch(intent)
```

---

### Lecture MRZ seule (Composant de lecture MRZ sans upload vers ID360)
Si vous utilisez la plateforme d'enrôlement ID360 et devez uploader le document sans lecture NFC :

```kotlin
val intent = ID360FlowActivity.createMrzReadIntent(
    context = this,
    apiKey = "votre-cle-api",
    apiUrl = "https://api.id360.docaposte.fr",
    documentName = "recto",
    language = "fr"
)
mrzFlowLauncher.launch(intent)
```
Le SDK gère l'upload automatique. S'il s'agit d'un document double face, il enchaîne la capture recto/verso et les charge sur l'endpoint : `{apiUrl}/enrollment/flow/document/{documentName}/`.

---

### Personnalisation visuelle du SDK
Vous pouvez passer un objet `ID360UiCustomization` pour adapter les écrans natifs (boutons, textes, icônes) à votre charte :

```kotlin
import com.docaposte.id360sdk.ID360UiCustomization

val customization = ID360UiCustomization(
    ui_button_color = "#0057B8",
    ui_button_text_color = "#FFFFFF",
    ui_text_color = "#0F172A"
)

val intent = ID360FlowActivity.createNfcReadIntent(
    context = this,
    language = "fr",
    uiCustomization = customization
)
```

---

### Mode Bas Niveau (Sans l'UI du SDK)
Si vous possédez votre propre interface de capture et souhaitez piloter nos moteurs :

#### 1. Parser une chaîne MRZ brute
```kotlin
import com.docaposte.id360sdk.utils.MrzParser

val mrzResult = MrzParser.parse(rawOcrText) // Renvoie un objet structuré
```

#### 2. Lire la puce NFC avec votre propre UI
```kotlin
import android.nfc.NfcAdapter
import android.nfc.Tag
import android.os.Bundle
import androidx.lifecycle.lifecycleScope
import com.docaposte.id360sdk.ID360SDK
import com.docaposte.id360sdk.MrzInfo
import com.docaposte.id360sdk.ScanResult
import kotlinx.coroutines.launch

override fun onResume() {
    super.onResume()
    val options = Bundle().apply {
        putInt(NfcAdapter.EXTRA_READER_PRESENCE_CHECK_DELAY, 250)
    }
    nfcAdapter?.enableReaderMode(
        this,
        { tag -> readTag(tag) },
        NfcAdapter.FLAG_READER_NFC_A or NfcAdapter.FLAG_READER_NFC_B or NfcAdapter.FLAG_READER_SKIP_NDEF_CHECK,
        options
    )
}

private fun readTag(tag: Tag) {
    val sdk = ID360SDK(this, "fr")
    val mrzInfo = MrzInfo(
        documentNumber = "AB1234567",
        dateOfBirth = "900115",
        dateOfExpiry = "300115",
        documentType = "P"
    )

    lifecycleScope.launch {
        sdk.readChip(tag, mrzInfo) { result ->
            when (result) {
                is ScanResult.Progress -> {
                    // Mettre à jour la progression : result.progress (0 à 100)
                }
                is ScanResult.Success -> {
                    val json = result.data.toJson()
                }
                is ScanResult.Error -> {
                    // Gérer l'erreur technique (ex. perte de connexion NFC)
                }
            }
        }
    }
}
```

#### 3. Réutiliser les écrans Compose individuels du SDK
Le SDK expose individuellement les composants graphiques `CameraPreviewScreen`, `ImagePreviewScreen` et `NfcReadingScreen`. Pour les intégrer dans votre propre navigation Compose, déclarez les dépendances Compose dans votre application :
```kotlin
implementation(platform("androidx.compose:compose-bom:2025.11.01"))
implementation("androidx.compose.material3:material3")
```

## ❓ Aide & Validation

### FAQ / Dépannage

#### Le SDK ne détecte pas le NFC sur mon téléphone de test.
* Vérifiez que le composant NFC est activé dans les réglages système du téléphone.
* Retirez les coques de protection contenant des éléments métalliques ou trop épaisses.
* Plaquez bien le document contre le dos de l'appareil (l'emplacement de l'antenne NFC varie d'un modèle à l'autre).

#### Est-il possible de tester sur un émulateur ?
Non. Le scan MRZ requiert la caméra arrière et la lecture NFC nécessite un matériel physique absent des ordinateurs de développement. Utilisez impérativement un téléphone physique.

#### Quelle est la différence entre une annulation et une erreur ?
Le SDK retourne `RESULT_CANCELED` pour ces deux cas. Vous devez analyser la chaîne de caractères renvoyée par `RESULT_ERROR_MESSAGE` pour différencier une annulation utilisateur (ex. clic sur retour) d'une erreur bloquante (ex. NFC indisponible).

#### Qui fournit l'URL d'enrôlement WebView IS360 ?
C'est votre serveur backend qui doit la générer via les API ID360 et la transmettre à l'application mobile pour initialiser la WebView.

---



### Prérequis Techniques
* **minSdk** : 24 (Android 7.0)
* **compileSdk** : 36
* **Kotlin** : 1.9+