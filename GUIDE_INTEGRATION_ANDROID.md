# Guide d'integration ID360SDK Android

Ce document s'adresse aux equipes qui integrent le SDK dans une application Android. Il decrit les points d'entree publics, les prerequis, les parcours d'integration les plus simples et les donnees renvoyees par le SDK.

Pour la documentation mainteneur du SDK, voir [README.md](README.md).

## Choisir le bon mode d'integration

| Votre besoin | Point d'entree recommande |
|---|---|
| Flux complet scan MRZ puis lecture NFC avec ecrans natifs | `ID360FlowActivity.createNfcReadIntent(...)` |
| Lecture NFC directe quand vous avez deja les donnees MRZ | `ID360FlowActivity.createDirectNfcReadIntent(...)` |
| Scan MRZ seul avec upload de l'image du document | `ID360FlowActivity.createMrzReadIntent(...)` |
| Webapp embarquee dans une WebView | `ID360WebViewActivity.createIntent(...)` |
| UI 100% personnalisee avec logique SDK | `ID360SDK` + `MrzParser` |
| Reutilisation des ecrans Compose du SDK dans votre navigation | `CameraPreviewScreen`, `ImagePreviewScreen`, `NfcReadingScreen` |

## Prerequis

| Element | Valeur |
|---|---|
| `minSdk` | 24 |
| `compileSdk` | 36 |
| Kotlin | 1.9+ |
| Jetpack Compose | requis si vous utilisez les composants UI du SDK |

Le SDK declare lui-meme les permissions et features necessaires dans son manifest. Vous n'avez pas de permissions Android supplementaires a ajouter dans l'application consommatrice.

## Ajouter le SDK a votre application

### Integration en module Gradle

Ajoutez le module dans `settings.gradle.kts` :

```kotlin
include(":id360sdk")
```

Ajoutez ensuite la dependance dans le module applicatif :

```kotlin
dependencies {
    implementation(project(":id360sdk"))

    // Requis uniquement si vous utilisez les ecrans Compose du SDK
    implementation(platform("androidx.compose:compose-bom:2025.11.01"))
    implementation("androidx.compose.material3:material3")
}
```

### Integration a partir d'un artefact distribue

Si votre equipe recoit un artefact `release` du SDK, integrez cette variante plutot qu'un build `debug`. L'API publique reste la meme: les exemples ci-dessous sont valables quel que soit le mode de distribution du SDK.

## Integration recommandee: flux cle en main

`ID360FlowActivity` est le point d'entree le plus simple pour une equipe applicative. Vous lancez une Activity et vous recupererez soit un JSON MRZ, soit un JSON NFC, soit un message d'erreur.

### 1. Flux complet MRZ -> NFC

```kotlin
import android.app.Activity
import androidx.activity.result.contract.ActivityResultContracts
import com.docaposte.id360sdk.flow.ID360FlowActivity

private val nfcFlowLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        val nfcJson = result.data?.getStringExtra(ID360FlowActivity.RESULT_NFC_DATA)
        // Envoyer ce JSON a votre backend ou le parser dans votre app.
    } else {
        val error = result.data?.getStringExtra(ID360FlowActivity.RESULT_ERROR_MESSAGE)
        // Gerer le cas d'annulation ou d'erreur bloquante.
    }
}

fun startId360Flow() {
    val intent = ID360FlowActivity.createNfcReadIntent(
        context = this,
        keyId = "your-key-id",
        masterKey = "base64-encoded-master-key",
        retryThreshold = 3,
        language = "fr"
    )
    nfcFlowLauncher.launch(intent)
}
```

Quand utiliser ce mode:

- vous voulez laisser le SDK gerer la camera, la lecture NFC et les ecrans de progression ;
- vous souhaitez recuperer un JSON final pret a transmettre a un backend ;
- vous ne voulez pas implementer vous-meme la logique MRZ/NFC.

### 2. Flux NFC direct sans etape camera

Utilisez ce mode si vous connaissez deja le numero de document, la date de naissance et la date d'expiration.

```kotlin
val intent = ID360FlowActivity.createDirectNfcReadIntent(
    context = this,
    documentNumber = "AB1234567",
    dateOfBirth = "900115",
    dateOfExpiry = "300115",
    documentType = "P",
    retryThreshold = 3,
    language = "fr"
)

nfcFlowLauncher.launch(intent)
```

Le flux saute completement l'etape MRZ si `documentNumber`, `dateOfBirth` et `dateOfExpiry` sont tous fournis.

### 3. Flux MRZ seul

Utilisez ce mode si vous voulez seulement capturer la MRZ et l'image du document, sans lecture NFC.

```kotlin
private val mrzFlowLauncher = registerForActivityResult(
    ActivityResultContracts.StartActivityForResult()
) { result ->
    if (result.resultCode == Activity.RESULT_OK) {
        val mrzJson = result.data?.getStringExtra(ID360FlowActivity.RESULT_MRZ_DATA)
    }
}

fun startMrzOnlyFlow() {
    val intent = ID360FlowActivity.createMrzReadIntent(
        context = this,
        apiKey = "your-api-key",
        apiUrl = "https://api.example.com",
        documentName = "scan",
        language = "fr"
    )
    mrzFlowLauncher.launch(intent)
}
```

## Integration WebView

`ID360WebViewActivity` permet d'embarquer une webapp et d'exposer les fonctions natives au JavaScript.

### Lancement

```kotlin
import com.docaposte.id360sdk.webview.ID360WebViewActivity

val intent = ID360WebViewActivity.createIntent(
    context = this,
    url = "https://your-webapp.com/flow",
    language = "fr"
)

startActivity(intent)
```

Si vous voulez fermer la WebView et afficher un message hote quand l'appareil n'est pas compatible NFC, lancez cette Activity avec `ActivityResultContracts.StartActivityForResult()` et lisez `ID360WebViewActivity.RESULT_ERROR_CODE` / `ID360WebViewActivity.RESULT_ERROR_MESSAGE`.

### API JavaScript recommandee

```javascript
window.id360.startNfcRead({
  keyId: "your-key-id",
  masterKey: "base64-encoded-master-key",
  retryThreshold: 3,
  language: "fr"
});

window.id360.startMrzRead({
  apiKey: "your-api-key",
  apiUrl: "https://api.example.com",
  documentName: "scan",
  language: "fr"
});

function onNfcReadSuccess(nfcData) {
  console.log("NFC data", nfcData);
}

function onMrzReadSuccess(mrzData) {
  console.log("MRZ data", mrzData);
}

function onId360Error(error) {
    console.error(error.code, error.message);
}
```

`startNfcRead` reutilise automatiquement une MRZ deja mise en cache par `startMrzRead`. Si aucune MRZ n'est disponible, le SDK effectue d'abord un pre-check MRZ pour detecter l'absence de puce.

- document sans puce NFC : la webapp recoit `onId360Error({ code: "DOCUMENT_WITHOUT_NFC_CHIP", message })` ;
- appareil sans NFC ou NFC desactive : sur Android, l'Activity WebView peut retourner `RESULT_CANCELED` avec `RESULT_ERROR_CODE = "NFC_UNAVAILABLE"` pour que l'application hote gere elle-meme le retour a l'accueil.

## Integration avancee sans UI SDK

Choisissez ce mode si votre equipe gere deja sa propre experience utilisateur et veut seulement reutiliser le parsing MRZ ou la lecture NFC.

### Parser un texte MRZ

```kotlin
import com.docaposte.id360sdk.utils.MrzParser

val mrzResult = MrzParser.parse(rawOcrText)
```

### Lire la puce NFC avec votre propre UI

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
                is ScanResult.Progress -> Unit
                is ScanResult.Success -> {
                    val json = result.data.toJson()
                }
                is ScanResult.Error -> Unit
            }
        }
    }
}
```

Dans ce mode, c'est votre application qui doit activer le reader mode NFC et gerer la progression de lecture.

## Donnees renvoyees par le SDK

### Resultat NFC

Le flux renvoie `RESULT_NFC_DATA`, une chaine JSON issue de `ReadData.toJson()`. Vous y trouverez notamment:

- l'identite du porteur: `firstNames`, `lastName`, `gender`, `nationality` ;
- les donnees documentaires: `documentNumber`, `birthDate`, `expiryDate` ;
- les medias: `faceImage` en Base64 ;
- les data groups bruts: `dg1Raw`, `dg2Raw`, `dg11Raw`, `dg12Raw`, `dg13Raw`, `dg14Raw`, `dg15Raw`, `sod` ;
- des elements techniques de securite: `chipAuthStatus`, `activeAuthStatus`, `pacePICCPublicKey`, `apduTrace86`, `bacRndIFD`, `bacKIFD`.

### Resultat MRZ

Le flux renvoie `RESULT_MRZ_DATA`, une chaine JSON avec au minimum:

- `documentNumber`
- `documentType`
- `countryCode`
- `dateOfBirth`
- `dateOfExpiry`
- `hasRfidChip`
- `nfcAvailable`
- `documentName`

## Checklist d'integration

Avant de livrer l'integration, verifiez les points suivants:

- le SDK est ajoute dans le module applicatif en variante `release` pour une distribution externe ;
- l'application sait recuperer `RESULT_NFC_DATA`, `RESULT_MRZ_DATA` et `RESULT_ERROR_MESSAGE` ;
- le parcours choisi est clair: flux complet, flux NFC direct, MRZ seul ou WebView ;
- si vous utilisez le mode sans UI, votre application active bien le reader mode NFC ;
- si vous utilisez la personnalisation runtime, vous passez les couleurs via `ID360UiCustomization`.

## Questions frequentes

### Que se passe-t-il si le NFC n'est pas disponible?

Si le NFC est requis mais indisponible ou desactive sur l'appareil, le flux retourne `RESULT_CANCELED` avec `RESULT_ERROR_MESSAGE`.

### Peut-on reutiliser les ecrans du SDK dans une navigation Compose existante?

Oui. Les composables `CameraPreviewScreen`, `ImagePreviewScreen` et `NfcReadingScreen` peuvent etre integres dans votre navigation, a condition d'ajouter les dependances Compose cote application.
