# Subir holamundoiOS a TestFlight con Jenkins + fastlane

> Escrito de memoria, sin consultar la documentación. Verifica en developer.apple.com los nombres que cambian entre versiones de Xcode (por ejemplo `method` en `ExportOptions.plist` y la ruta de instalación de los profiles).

Firma: certificado `.p12` importado en un keychain temporal. Autenticación con App Store Connect: API key `.p8`.

## 1. Datos que debes obtener de Apple

| Dato (nombre oficial) | Dónde sale | Para qué se usa |
|---|---|---|
| **Team ID** | developer.apple.com → Account → Membership details | `DEVELOPMENT_TEAM`, `teamID` |
| **Bundle ID** (explicit App ID) | Certificates, Identifiers & Profiles → Identifiers | `PRODUCT_BUNDLE_IDENTIFIER` / `CFBundleIdentifier` |
| **App record** | App Store Connect → Apps → New App (Name, Primary Language, Bundle ID, SKU) | Destino de la subida |
| **Issuer ID** | App Store Connect → Users and Access → Integrations → App Store Connect API | Autenticación de la API |
| **Key ID** | La misma pantalla, al crear la key | Autenticación de la API |
| **Private key `AuthKey_<KeyID>.p8`** | Se descarga una sola vez al crear la key | Firma del JWT |
| **Apple Distribution certificate** (`.cer`) | Certificates → Create → Apple Distribution, con un CSR | Identidad de firma |
| **`.p12`** (PKCS#12) con su contraseña | Se exporta desde Keychain Access: certificado más llave privada | Importarlo en el agente |
| **Provisioning profile "App Store Connect"** (`.mobileprovision`) | Profiles → Distribution → App Store Connect, ligado al App ID y al certificado | Firma del `.ipa` |

- La API key debe tener rol **App Manager** o **Admin**. Si es una *Team Key* (no *Individual*), necesita Issuer ID.
- Del profile necesitarás dos campos: **Name** y **UUID**.

## 2. Configuración del proyecto (build settings / Info.plist)

| Clave | Build setting equivalente | Nota |
|---|---|---|
| `CFBundleIdentifier` | `PRODUCT_BUNDLE_IDENTIFIER` | Debe coincidir con el App ID |
| `CFBundleShortVersionString` | `MARKETING_VERSION` | Versión visible, por ejemplo `1.0` |
| `CFBundleVersion` | `CURRENT_PROJECT_VERSION` | Build number: debe crecer en cada subida para la misma versión |
| `ITSAppUsesNonExemptEncryption` | `INFOPLIST_KEY_ITSAppUsesNonExemptEncryption` | `NO` evita el estado "Missing Compliance" |
| `CFBundleIconName` | `ASSETCATALOG_COMPILER_APPICON_NAME` | Requiere un `AppIcon` de 1024×1024 en `Assets.xcassets` |
| `MinimumOSVersion` | `IPHONEOS_DEPLOYMENT_TARGET` | Hoy es 27.2; los testers necesitan ese iOS o uno superior |
| `UIDeviceFamily` | `TARGETED_DEVICE_FAMILY` | `1,2,7` incluye iPhone, iPad y visionOS |
| `SUPPORTED_PLATFORMS` | – | Hoy incluye `macosx` y `xros`; para solo iOS déjalo en `iphoneos iphonesimulator` |
| `*UsageDescription` | `INFOPLIST_KEY_NS…UsageDescription` | Solo si la app usa cámara, ubicación, etc. |
| `PrivacyInfo.xcprivacy` | – | Privacy manifest, necesario si usas las required reason APIs |

## 3. Opciones de `xcodebuild` y `ExportOptions.plist`

**Archive (`xcodebuild archive`)**
- `CODE_SIGN_STYLE=Manual`
- `CODE_SIGN_IDENTITY="Apple Distribution"`
- `PROVISIONING_PROFILE_SPECIFIER=<Name del profile>`
- `DEVELOPMENT_TEAM=<Team ID>`
- `OTHER_CODE_SIGN_FLAGS="--keychain <ruta>"`

**`ExportOptions.plist` (`xcodebuild -exportArchive`)**
- `method` = `app-store-connect` (desde Xcode 15.3; antes era `app-store`)
- `teamID`
- `signingStyle` = `manual`
- `signingCertificate` = `Apple Distribution`
- `provisioningProfiles` = `{ <Bundle ID> : <Name del profile> }`
- `uploadSymbols` = `true`
- `stripSwiftSymbols` = `true`
- `destination` = `export` (para subir con fastlane después)

## 4. Operaciones de firma en el agente (`security`)

1. `security create-keychain` / `unlock-keychain` / `set-keychain-settings` crean un keychain temporal.
2. `security import <p12> -P <contraseña> -t cert -f pkcs12 -A` importa el certificado.
3. `security set-key-partition-list -S apple-tool:,apple: -s` evita que `codesign` pida permiso gráfico.
4. `security list-keychains -d user -s` añade el keychain a la búsqueda.
5. Copia el profile a `~/Library/MobileDevice/Provisioning Profiles/<UUID>.mobileprovision`. Xcode 16 y posteriores también usan `~/Library/Developer/Xcode/UserData/Provisioning Profiles/`. Para leer el UUID: `security cms -D -i <profile>`.

## 5. Nombres en fastlane

- `setup_ci`: crea el keychain temporal en CI.
- `import_certificate`: `certificate_path`, `certificate_password`, `keychain_name`, `keychain_password`.
- `install_provisioning_profile`: instala el `.mobileprovision`.
- `app_store_connect_api_key`: `key_id`, `issuer_id`, `key_filepath` (o `key_content`), `in_house: false`.
- `increment_build_number` y `latest_testflight_build_number`: para el `CFBundleVersion` creciente.
- `build_app` (gym): `scheme`, `export_method: "app-store"`, `export_options`, `codesigning_identity`.
- `upload_to_testflight` (pilot): `api_key`, `skip_waiting_for_build_processing`, `changelog`, `groups`, `distribute_external`, `uses_non_exempt_encryption`.

Variables de entorno útiles: `FASTLANE_SKIP_UPDATE_CHECK=1`, `FASTLANE_XCODEBUILD_SETTINGS_TIMEOUT=120`, `LC_ALL=en_US.UTF-8`, `LANG=en_US.UTF-8`.

## 6. Credentials de Jenkins

| ID sugerido | Tipo | Contenido |
|---|---|---|
| `ios-dist-cert-p12` | Secret file | El `.p12` |
| `ios-dist-cert-password` | Secret text | Su contraseña |
| `ios-appstore-profile` | Secret file | El `.mobileprovision` |
| `asc-api-key-p8` | Secret file | El `AuthKey_<KeyID>.p8` |
| `asc-key-id` | Secret text | Key ID |
| `asc-issuer-id` | Secret text | Issuer ID |
| `apple-team-id` | Secret text (o parámetro) | Team ID |

## 7. En App Store Connect, tras la subida

- **Export compliance:** lo resuelve `ITSAppUsesNonExemptEncryption`.
- **Internal Testing:** testers que ya son usuarios del equipo, sin revisión.
- **External Testing:** requiere Beta App Review, un Test Information (contacto, descripción) y un grupo.
- El procesamiento del build tarda entre minutos y una hora, y el build number no puede repetirse.
