pipeline {
    agent { label 'macos' } // Nodo Jenkins con macOS y Xcode instalado

    parameters {
        string(name: 'PROJECT_NAME', defaultValue: 'holamundoiOS', description: 'Nombre del .xcodeproj / .xcworkspace (sin extensión)')
        string(name: 'SCHEME', defaultValue: 'holamundoiOS', description: 'Scheme de Xcode a compilar')
        choice(name: 'CONFIGURATION', choices: ['Release', 'Debug'], description: 'Configuración de build')
        string(name: 'DEVELOPER_DIR', defaultValue: '/Applications/Xcode.app/Contents/Developer', description: 'Ruta del Xcode a usar en el agente')
    }

    environment {
        BUILD_DIR     = "${WORKSPACE}/build"
        ARCHIVE_PATH  = "${WORKSPACE}/build/${params.SCHEME}.xcarchive"
        DEVELOPER_DIR = "${params.DEVELOPER_DIR}"
        LANG          = 'en_US.UTF-8'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Generate IPA') {
            steps {
                // Compilación sin firma: no requiere certificados ni Apple ID.
                // El .ipa resultante NO está firmado (no se puede instalar ni subir a tienda).
                sh '''
                    set -eo pipefail
                    xcodebuild -version

                    rm -rf "$BUILD_DIR"
                    mkdir -p "$BUILD_DIR"

                    # Usa .xcworkspace si existe (CocoaPods), si no .xcodeproj
                    if [ -d "${PROJECT_NAME}.xcworkspace" ]; then
                        BUILD_TARGET="-workspace ${PROJECT_NAME}.xcworkspace"
                    else
                        BUILD_TARGET="-project ${PROJECT_NAME}.xcodeproj"
                    fi

                    # 1. Compilar y archivar sin firma
                    xcodebuild $BUILD_TARGET \
                        -scheme "$SCHEME" \
                        -configuration "$CONFIGURATION" \
                        -sdk iphoneos \
                        -destination 'generic/platform=iOS' \
                        -archivePath "$ARCHIVE_PATH" \
                        CODE_SIGNING_ALLOWED=NO \
                        CODE_SIGNING_REQUIRED=NO \
                        CODE_SIGN_IDENTITY="" \
                        clean archive

                    # 2. Empaquetar el .ipa (Payload/<app>.app -> .ipa)
                    APP_PATH=$(find "$ARCHIVE_PATH/Products/Applications" -maxdepth 1 -name "*.app" | head -n 1)
                    [ -n "$APP_PATH" ] || { echo "No se encontró el .app en el archive"; exit 1; }

                    mkdir -p "$BUILD_DIR/ipa/Payload"
                    cp -R "$APP_PATH" "$BUILD_DIR/ipa/Payload/"
                    (cd "$BUILD_DIR/ipa" && zip -qry "$SCHEME.ipa" Payload && rm -rf Payload)

                    ls -lh "$BUILD_DIR/ipa"
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build/ipa/*.ipa', fingerprint: true
                }
            }
        }
    }

    post {
        cleanup {
            sh 'rm -rf "$ARCHIVE_PATH"'
        }
    }
}
