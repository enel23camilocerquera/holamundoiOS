pipeline {
    agent { label 'macos' } // Nodo Jenkins con macOS y Xcode instalado

    parameters {
        string(name: 'PROJECT_NAME', defaultValue: 'holamundoiOS', description: 'Nombre del .xcodeproj / .xcworkspace (sin extensión)')
        string(name: 'SCHEME', defaultValue: 'holamundoiOS', description: 'Scheme de Xcode a compilar')
        choice(name: 'CONFIGURATION', choices: ['Release', 'Debug'], description: 'Configuración de build')
        choice(name: 'EXPORT_METHOD', choices: ['development', 'ad-hoc', 'app-store-connect', 'enterprise'], description: 'Método de exportación del .ipa')
        string(name: 'TEAM_ID', defaultValue: '', description: 'Apple Developer Team ID (10 caracteres)')
    }

    environment {
        BUILD_DIR    = "${WORKSPACE}/build"
        ARCHIVE_PATH = "${WORKSPACE}/build/${params.SCHEME}.xcarchive"
        EXPORT_PATH  = "${WORKSPACE}/build/ipa"
        LANG         = 'en_US.UTF-8'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
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
                sh '''
                    set -eo pipefail
                    rm -rf "$BUILD_DIR"
                    mkdir -p "$BUILD_DIR"

                    # Usa .xcworkspace si existe (CocoaPods), si no .xcodeproj
                    if [ -d "${PROJECT_NAME}.xcworkspace" ]; then
                        BUILD_TARGET="-workspace ${PROJECT_NAME}.xcworkspace"
                    else
                        BUILD_TARGET="-project ${PROJECT_NAME}.xcodeproj"
                    fi

                    # ExportOptions.plist
                    cat > "$BUILD_DIR/ExportOptions.plist" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>${EXPORT_METHOD}</string>
    <key>teamID</key>
    <string>${TEAM_ID}</string>
    <key>signingStyle</key>
    <string>automatic</string>
    <key>compileBitcode</key>
    <false/>
    <key>stripSwiftSymbols</key>
    <true/>
</dict>
</plist>
EOF

                    # 1. Compilar y archivar
                    xcodebuild $BUILD_TARGET \
                        -scheme "$SCHEME" \
                        -configuration "$CONFIGURATION" \
                        -sdk iphoneos \
                        -destination 'generic/platform=iOS' \
                        -archivePath "$ARCHIVE_PATH" \
                        DEVELOPMENT_TEAM="$TEAM_ID" \
                        -allowProvisioningUpdates \
                        clean archive

                    # 2. Exportar el .ipa
                    xcodebuild -exportArchive \
                        -archivePath "$ARCHIVE_PATH" \
                        -exportPath "$EXPORT_PATH" \
                        -exportOptionsPlist "$BUILD_DIR/ExportOptions.plist" \
                        -allowProvisioningUpdates
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
