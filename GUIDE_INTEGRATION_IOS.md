---
layout: default
title: Guide d'intégration iOS
---

# Guide d'integration ID360SDK iOS

Ce document s'adresse aux equipes qui integrent le SDK dans une application iOS. Il se concentre sur l'usage du SDK, pas sur son developpement interne.

Pour la documentation mainteneur du SDK, voir [README.md](README.md).

## Choisir le bon mode d'integration

| Votre besoin | Point d'entree recommande |
|---|---|
| Flux complet scan MRZ puis lecture NFC avec ecrans SwiftUI | `ID360FlowView` avec `ID360FlowConfig.nfcRead(...)` |
| Lecture NFC directe avec donnees MRZ deja connues | `ID360FlowView` avec `ID360FlowConfig.directNfcRead(...)` |
| Scan MRZ seul avec upload du document | `ID360FlowView` avec `ID360FlowConfig.mrzRead(...)` |
| Webapp embarquee dans une WKWebView | `ID360WebViewView` |
| UI 100% personnalisee avec logique SDK | `ID360NfcReader` + `MrzParser` |

## Prerequis

| Element | Valeur |
|---|---|
| iOS minimum | 16.0 |
| Swift | 5.9+ |
| Xcode | 15.0+ |

## Ajouter le SDK a votre application

### Option 1: Swift Package Manager local

Dans Xcode:

1. Ouvrez `File > Add Package Dependencies...`
2. Choisissez `Add Local...`
3. Selectionnez le dossier `ID360SDK`
4. Ajoutez le package a la target de votre application

Dans un `Package.swift`, la dependance s'ecrit ainsi:

```swift
dependencies: [
    .package(path: "../ID360SDK")
]
```

### Option 2: integration binaire pour une autre equipe

Si votre equipe recoit les binaires deja generes, ajoutez ces trois artefacts dans le projet Xcode:

- `ID360SDK.xcframework`
- `Lottie.xcframework`
- `OpenSSL.xcframework`

Dans la target applicative, verifiez qu'ils sont presents dans `Frameworks, Libraries and Embedded Content` avec `Embed & Sign`.

Ensuite, importez le module dans votre code:

```swift
import ID360SDK
```

## Permissions et capabilities

Ajoutez les cles suivantes dans `Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>L'acces a la camera est necessaire pour scanner les documents d'identite</string>

<key>NFCReaderUsageDescription</key>
<string>Le NFC est necessaire pour lire la puce de votre document d'identite</string>

<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
    <string>A0000002471001</string>
    <string>A0000002472001</string>
</array>
```

Activez egalement la capability `Near Field Communication Tag Reading` dans Xcode. Le mode `TAG` est obligatoire. Le mode `PACE` est recommande pour les documents eID.

## Integration recommandee: flux cle en main

Le point d'entree le plus simple est `ID360FlowView`. Vous lui passez une configuration, puis vous ecoutez les evenements de resultat.

### 1. Flux complet MRZ -> NFC

```swift
import ID360SDK
import SwiftUI

struct KycView: View {
    var body: some View {
        ID360FlowView(
            config: .nfcRead(
                keyId: "your-key-id",
                masterKey: "base64-encoded-master-key",
                retryThreshold: 3,
                language: "fr"
            )
        ) { event in
            switch event {
            case .finishedWithNfc(let json):
                print("NFC JSON:", json)
            case .finishedWithError(let message):
                print("Erreur:", message)
            default:
                break
            }
        }
    }
}
```

Quand utiliser ce mode:

- vous voulez le parcours complet gere par le SDK ;
- vous travaillez deja en SwiftUI ou acceptez de presenter une vue SwiftUI ;
- vous voulez recuperer un JSON final plutot qu'orchestrer vous-meme les etapes.

### 2. Flux NFC direct sans camera

```swift
ID360FlowView(
    config: .directNfcRead(
        documentNumber: "AB1234567",
        dateOfBirth: "900115",
        dateOfExpiry: "300115",
        documentType: "P",
        retryThreshold: 3,
        language: "fr"
    )
) { event in
    if case .finishedWithNfc(let json) = event {
        print(json)
    }
}
```

Ce mode saute l'etape MRZ quand `documentNumber`, `dateOfBirth` et `dateOfExpiry` sont fournis.

### 3. Flux MRZ seul

```swift
ID360FlowView(
    config: .mrzRead(
        apiKey: "your-api-key",
        apiUrl: "https://api.example.com",
        documentName: "scan",
        language: "fr"
    )
) { event in
    if case .finishedWithMrz(let json) = event {
        print(json)
    }
}
```

### 4. Personnalisation UI optionnelle

```swift
let customization = ID360UiCustomization(
    ui_button_color: "#0057B8",
    ui_button_text_color: "#FFFFFF",
    ui_text_color: "#0F172A"
)

let config = ID360FlowConfig.nfcRead(
    retryThreshold: 3,
    language: "fr",
    uiCustomization: customization
)
```

La customisation s'applique surtout aux ecrans autres que la capture camera. `ui_icon_color` pilote la barre de progression NFC.

## Integration WebView

Utilisez `ID360WebViewView` si votre equipe integre le parcours KYC dans une webapp.

### Presentation de la vue

```swift
import ID360SDK

ID360WebViewView(
    url: "https://your-webapp.com/flow",
    language: "fr",
    onIncompatibleNfcDevice: { message in
        // Fermez votre presentation et affichez un message hote.
        print(message)
    }
)
```

### API JavaScript cote webapp

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

Le SDK memorise automatiquement les donnees MRZ d'un `startMrzRead` pour reutiliser ces valeurs lors du `startNfcRead` suivant.

Si aucune MRZ n'est disponible, `startNfcRead` effectue d'abord un pre-check MRZ.

- document sans puce NFC : la webapp recoit `onId360Error({ code: "DOCUMENT_WITHOUT_NFC_CHIP", message })` ;
- appareil sans NFC ou NFC desactive : si vous fournissez `onIncompatibleNfcDevice`, la vue vous notifie directement pour gerer le retour a votre propre ecran ; sinon la webapp recoit `onId360Error({ code: "NFC_UNAVAILABLE", message })`.

## Integration avancee sans UI SDK

Choisissez cette approche si vous avez deja vos propres ecrans et souhaitez seulement reutiliser le coeur MRZ/NFC.

### Parser du texte MRZ

```swift
import ID360SDK

let result = MrzParser.parse(rawOcrText)
```

### Lire la puce NFC avec votre propre UI

```swift
import ID360SDK

let reader = ID360NfcReader(language: "fr")
let mrzInfo = MrzInfo(
    documentNumber: "AB1234567",
    dateOfBirth: "900115",
    dateOfExpiry: "300115",
    documentType: "P"
)

reader.readChip(mrzInfo: mrzInfo) { result in
    switch result {
    case .progress(let message, let progress):
        print(message, progress)
    case .success(let data):
        print(data.toJson())
    case .error(let message):
        print(message)
    }
}
```

Sur iOS, ce mode declenche la feuille systeme NFC au moment de `readChip`.

## Evenements et donnees renvoyees

### Evenements `FlowEvent`

| Evenement | Contenu |
|---|---|
| `.finishedWithNfc(String)` | JSON du resultat NFC |
| `.finishedWithMrz(String)` | JSON du resultat MRZ |
| `.finishedWithError(String)` | message d'erreur exploitable par l'app |
| `.finished` | fin de parcours sans donnees |

### Resultat NFC

Le JSON NFC contient notamment:

- `firstNames`, `lastName`, `gender`, `nationality` ;
- `documentNumber`, `birthDate`, `expiryDate` ;
- `faceImage` en Base64 ;
- les data groups bruts `dg1Raw`, `dg2Raw`, `dg11Raw`, `dg12Raw`, `dg13Raw`, `dg14Raw`, `dg15Raw`, `sod` ;
- des champs techniques comme `chipAuthStatus`, `activeAuthStatus`, `pacePICCPublicKey`, `apduTrace86`.

### Resultat MRZ

Le JSON MRZ contient au minimum:

- `documentNumber`
- `documentType`
- `countryCode`
- `dateOfBirth`
- `dateOfExpiry`
- `hasRfidChip`
- `nfcAvailable`
- `documentName`

## Checklist d'integration

- le SDK est bien ajoute via SPM local ou via les trois `xcframework` attendus ;
- `Info.plist` contient les descriptions camera et NFC ;
- la capability `Near Field Communication Tag Reading` est activee ;
- l'application sait traiter `.finishedWithNfc`, `.finishedWithMrz` et `.finishedWithError` ;
- si vous utilisez le mode sans UI, votre parcours gere l'etat de progression et les erreurs ;
- si vous utilisez la customisation runtime, vous passez les couleurs via `ID360UiCustomization`.

## Questions frequentes

### Que faire si l'equipe n'utilise pas SwiftUI?

Le chemin le plus simple reste de presenter `ID360FlowView` ou `ID360WebViewView` depuis un `UIHostingController`. Si vous ne voulez pas exposer de vue SwiftUI, utilisez l'API coeur `ID360NfcReader` et votre propre UI UIKit.

### Peut-on integrer le SDK sans lecture NFC?

Oui. Le mode `.mrzRead(...)` permet de faire uniquement le scan MRZ et l'upload du document.
