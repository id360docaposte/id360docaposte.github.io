# Guide d'Intégration ID360 SDK iOS

Ce guide vous accompagne pas à pas dans l'intégration du SDK ID360 dans votre application iOS, de l'implémentation la plus simple (flux natif standard) aux cas d'usage avancés.

Pour télécharger le SDK : [releases](https://github.com/id360docaposte/id360docaposte.github.io/releases)

---

## 🌐 Intégration d'un parcours ID360 en utilisant le SDK (parcours clé en main)

Le SDK ouvre l'URL d'enrôlement ID360 dans une WebView sécurisée (`ID360WebViewView`). L'application web pilote le parcours et déclenche les captures natives du téléphone via le pont JavaScript du SDK. L'application iOS intercepte ensuite la réussite ou l'éventuelle incompatibilité du matériel.

### Étape 1 : Lancement de la WebView
Vous devez obtenir l'URL d'enrôlement depuis votre backend, puis la passer au SDK :

```swift
import ID360SDK
import SwiftUI

struct EnrollmentWebView: View {
    let enrollmentURL = "https://id360.example.com/parcours/..." // Fournie par votre backend
    
    var body: some View {
        ID360WebViewView(
            url: enrollmentURL,
            language: "fr",
            onEnrollmentFinished: { payload in
                // Le parcours web s'est terminé avec succès
                // payload contient par exemple : {"status":"OK"}
                print("Payload :", payload)
            },
            onIncompatibleNfcDevice: { message in
                // Gérer le cas des iPhones incompatibles ou sans NFC
                print("Incompatible NFC :", message)
            }
        )
    }
}
```
> [!IMPORTANT]
> Ne modifiez pas l'URL fournie par votre backend (par exemple en y concaténant des sous-chemins comme `/capture`). Le SDK gère lui-même la navigation interne.

> **Pré-check WebView** : quand aucune MRZ n'est en cache, `startNfcRead` lance d'abord un scan MRZ pour remplir les données documentaires et distinguer clairement deux cas.
> - document sans puce NFC : la webapp reçoit `onId360Error({ code: "DOCUMENT_WITHOUT_NFC_CHIP", message })` ;
> - appareil sans NFC ou NFC désactivé : si vous fournissez `onIncompatibleNfcDevice`, la vue vous notifie directement pour que vous fermiez la WebView et reveniez à votre propre écran ; sinon la webapp reçoit `onId360Error({ code: "NFC_UNAVAILABLE", message })`.
>
> Note : un appel à `startMrzRead` (utilisé notamment par les parcours pre-NFC) renvoie `hasRfidChip` en tenant compte de la disponibilité NFC de l'appareil (`hasRfidChip = hasChip && nfcAvailable`). Ainsi, un appareil sans NFC sera traité de la même façon qu'un document sans puce.

---

## 🔌 Intégration de la lecture NFC standalone (nécessite obligatoirement un appareil avec lecteur NFC et un document d'identité avec puce NFC)

Ce parcours est recommandé si vous souhaitez intégrer la lecture de document directement dans votre application en utilisant l'interface graphique par défaut du SDK. 

Le SDK affiche ses propres écrans, gère la caméra et la puce NFC, puis vous renvoie un JSON contenant les données d'identité extraites.

### Étape 1 : Ajouter le SDK à Xcode
Dans Xcode :
1. Allez dans `File > Add Package Dependencies...`
2. Cliquez sur le bouton `Add Local...` en bas.
3. Sélectionnez le dossier contenant le SDK (`ID360SDK`).
4. Ajoutez le package à la cible (target) principale de votre application.

> **Alternative (via Package.swift) :**
> ```swift
> dependencies: [
>     .package(path: "../ID360SDK")
> ]
> ```

> [!NOTE]
> Si votre équipe reçoit les binaires précompilés, ajoutez les fichiers `ID360SDK.xcframework`, `Lottie.xcframework` et `OpenSSL.xcframework` dans la section `Frameworks, Libraries and Embedded Content` de votre cible avec l'option `Embed & Sign` sélectionnée.

### Étape 2 : Configurer les permissions
Ajoutez les clés suivantes dans le fichier `Info.plist` de votre projet principal :

```xml
<key>NSCameraUsageDescription</key>
<string>L'accès à la caméra est nécessaire pour scanner les documents d'identité</string>

<key>NFCReaderUsageDescription</key>
<string>Le NFC est nécessaire pour lire la puce de votre document d'identité</string>

<key>com.apple.developer.nfc.readersession.iso7816.select-identifiers</key>
<array>
    <string>A0000002471001</string>
    <string>A0000002472001</string>
</array>
```

Dans l'onglet **Signing & Capabilities** de votre cible Xcode, cliquez sur `+ Capability` et ajoutez **Near Field Communication Tag Reading** (en vous assurant que le mode `TAG` est actif, le mode `PACE` est également recommandé pour les documents eID).

### Étape 3 : Lancer la vue et récupérer le résultat
Le SDK propose des composants SwiftUI prêts à l'emploi. Intégrez `ID360FlowView` dans votre hiérarchie de vues :

```swift
import SwiftUI
import ID360SDK

struct KycView: View {
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        ID360FlowView(
            config: .nfcRead(language: "fr")
        ) { event in
            switch event {
            case .finishedWithNfc(let json):
                print("Données NFC récupérées :", json)
                // TODO: Envoyer ce JSON à votre backend
                dismiss()
                
            case .finishedWithError(let message):
                print("Erreur :", message)
                // TODO: Gérer l'erreur
                
            case .finished:
                // Annulation ou fin sans données
                dismiss()
                
            default:
                break
            }
        }
    }
}
```

### Étape 3 : Les données obtenues par défaut
Le JSON renvoyé dans `.finishedWithNfc` contient les données d'identité "métier" prêtes à être envoyées à votre backend :
* **Identité** : `firstNames` (prénoms), `lastName` (nom), `gender` (genre), `nationality` (nationalité).
* **Document** : `documentNumber` (numéro), `birthDate` (date de naissance), `expiryDate` (date d'expiration).
* **Photo** : `faceImage` (photo d'identité encodée en Base64).

---

## ⚙️ Intégration des composants du SDK (pour une utilisation sur mesure)

Cette section documente les fonctionnalités d'intégration avancées du SDK pour les cas d'usage spécifiques.

### Paramètres de configuration (Optionnels)
Ces paramètres peuvent être ajoutés lors de l'appel aux méthodes de configuration de `ID360FlowConfig` :
* `retryThreshold` (NFC) : Nombre d'échecs NFC consécutifs avant de reproposer l'étape caméra MRZ (Par défaut : `3`).
* `keyId`, `masterKey` (NFC) : Clés pour la configuration PACE spécifique. À renseigner uniquement sur demande de vos équipes sécurité.
* `apiKey`, `apiUrl` (MRZ) : Identifiants pour l'upload d'images vers la plateforme d'enrôlement ID360 cloud.
* `documentName` (MRZ) : Nom du fichier sur le serveur d'enrôlement (Par défaut : `"scan"`).
* `uiCustomization` : Instance de `ID360UiCustomization` pour adapter les couleurs du SDK.

---

### Lancement NFC direct (MRZ déjà connue)
Si vous connaissez déjà le numéro de document, la date de naissance et la date d'expiration (ex. saisis manuellement), vous pouvez sauter l'étape caméra :

```swift
ID360FlowView(
    config: .directNfcRead(
        documentNumber: "AB1234567",
        dateOfBirth: "900115",    // Format YYMMDD (ex: 15 janv 1990 -> 900115)
        dateOfExpiry: "300115",   // Format YYMMDD
        documentType: "P",        // "P" (Passeport), "I" (Identité), etc.
        language: "fr"
    )
) { event in
    if case .finishedWithNfc(let json) = event {
        print(json)
    }
}
```

---

### Lecture MRZ seule (Sans NFC ni upload)
Si vous souhaitez uniquement effectuer la capture caméra de la zone MRZ pour l'analyser localement sans interroger la puce NFC :

```swift
ID360FlowView(
    config: .mrzRead(language: "fr")
) { event in
    if case .finishedWithMrz(let json) = event {
        print(json)
    }
}
```

---

### Lecture MRZ avec upload vers la plateforme ID360
Si vous utilisez la plateforme d'enrôlement ID360 et devez uploader le document sans lecture NFC :

```swift
ID360FlowView(
    config: .mrzRead(
        apiKey: "votre-cle-api",
        apiUrl: "https://api.id360.docaposte.fr",
        documentName: "scan",
        language: "fr"
    )
) { event in
    if case .finishedWithMrz(let json) = event {
        print(json)
    }
}
```
Le SDK gère l'upload automatique. S'il s'agit d'un document double face, il enchaîne la capture recto/verso et les charge sur l'endpoint : `{apiUrl}/enrollment/flow/document/{documentName}/`.

---

### Personnalisation visuelle du SDK
Vous pouvez passer un objet `ID360UiCustomization` pour adapter les écrans natifs (boutons, textes, icônes) à votre charte :

```swift
let customization = ID360UiCustomization(
    ui_button_color: "#0057B8",
    ui_button_text_color: "#FFFFFF",
    ui_text_color: "#0F172A"
)

let config = ID360FlowConfig.nfcRead(
    language: "fr",
    uiCustomization: customization
)
```

---

### Intégration avec UIKit (Storyboard / Swift classique)
Si votre projet n'est pas structuré en SwiftUI, vous pouvez encapsuler nos composants dans un `UIHostingController` :

```swift
import UIKit
import SwiftUI
import ID360SDK

class KycViewController: UIViewController {
    
    func startVerificationFlow() {
        let kycView = ID360FlowView(config: .nfcRead(language: "fr")) { [weak self] event in
            switch event {
            case .finishedWithNfc(let json):
                self?.handleNfcSuccess(json)
            case .finishedWithError(let error):
                self?.handleError(error)
            case .finished:
                self?.dismiss(animated: true)
            default:
                break
            }
        }
        
        let hostingController = UIHostingController(rootView: kycView)
        hostingController.modalPresentationStyle = .fullScreen
        self.present(hostingController, animated: true)
    }
    
    private func handleNfcSuccess(_ json: String) {
        // Traitement du résultat
        self.dismiss(animated: true)
    }
    
    private func handleError(_ error: String) {
        // Affichage de l'alerte
    }
}
```

---

### Mode Bas Niveau (Sans l'UI du SDK)
Si vous possédez votre propre interface utilisateur et souhaitez piloter notre moteur NFC :

#### 1. Parser une chaîne MRZ brute
```swift
import ID360SDK

let result = MrzParser.parse(rawOcrText) // Renvoie un objet structuré
```

#### 2. Lire la puce NFC avec votre propre UI
```swift
import ID360SDK

let reader = ID360NfcReader(language: "fr")
let mrzInfo = MrzInfo(
    documentNumber: "AB1234567",
    dateOfBirth: "900115",
    dateOfExpiry: "300115",
    documentType: "P"
)

// Déclenche l'affichage de la feuille système NFC d'Apple
reader.readChip(mrzInfo: mrzInfo) { result in
    switch result {
    case .progress(let statusMessage, let percentage):
        // Mettre à jour votre UI de chargement (0 à 100 %)
        print("\(statusMessage) : \(percentage)%")
        
    case .success(let data):
        let jsonString = data.toJson()
        // Succès : envoyer le JSON au backend
        
    case .error(let errorMessage):
        // Échec de la lecture
        print("Erreur de lecture :", errorMessage)
    }
}
```

---

### Description complète du JSON NFC (Sécurité & Audit)
En plus des données métier (Identité, Document, Photo), le JSON NFC complet inclut les données brutes nécessaires à des vérifications cryptographiques poussées par votre serveur backend :
* **Data Groups bruts** : `dg1Raw`, `dg2Raw`, `dg11Raw`, etc.
* **Éléments de sécurité** : `chipAuthStatus`, `activeAuthStatus`, `pacePICCPublicKey`, `apduTrace86`.

---

## ❓ Aide & Validation

### FAQ / Dépannage

#### Le scan NFC ne se lance pas ou échoue systématiquement.
* Avez-vous configuré les clés NFC dans le fichier `Info.plist` (voir étape 2) ?
* La capability `Near Field Communication Tag Reading` est-elle bien activée dans Xcode ?
* Retirez les coques de protection qui peuvent perturber le signal NFC.

#### Comment tester sur le simulateur ?
Non. Le simulateur d'Apple ne supporte pas l'appareil photo pour l'OCR de la MRZ, ni les composants matériels NFC. Utilisez impérativement un iPhone physique.

#### Mon application cible une version d'iOS plus ancienne.
Le SDK requiert au minimum iOS 16.0.

#### Qui fournit l'URL d'enrôlement WebView ?
C'est votre serveur backend qui doit la générer via les API ID360 et la transmettre à l'application mobile pour initialiser la WebView.

---

### Checklist avant mise en production
- [ ] Les dépendances ont été intégrées via SPM ou via l'ajout des trois frameworks attendus (`ID360SDK`, `Lottie`, `OpenSSL`).
- [ ] Le fichier `Info.plist` contient bien les descriptions caméra et NFC ainsi que les identifiants d'applications ISO7816.
- [ ] La capability `Near Field Communication Tag Reading` est active au niveau du projet Xcode.
- [ ] L'application traite correctement tous les événements retournés par `ID360FlowView`.
- [ ] Les tests ont été menés avec succès sur des iPhones physiques avec de vrais documents à puce.

---

### Prérequis Techniques
* **iOS Cible Minimum** : iOS 16.0
* **Swift** : version 5.9+
* **Xcode** : version 15.0+
