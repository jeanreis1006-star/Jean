<#
Cria .github\workflows\build-apk-debug.yml, empacota em build-apk-debug.zip e gera o Base64.
Uso: execute este script no diretório raiz do clone do repositório.
Opcional: passe um nome de saída para o arquivo .b64 como parâmetro.
Ex: .\create-and-print-base64.ps1 -OutB64 build-apk-debug.zip.b64
#>

param(
    [string]$OutB64 = "build-apk-debug.zip.b64"
)

$ErrorActionPreference = 'Stop'

# Ensure workflows folder exists
$workflowDir = ".github\workflows"
if (-not (Test-Path -Path $workflowDir)) {
    New-Item -ItemType Directory -Path $workflowDir -Force | Out-Null
}

# Write YAML workflow (literal here-string, sem expansão de variáveis)
$workflowYaml = @'
name: Build Debug APK

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      ANDROID_SDK_ROOT: ${{ runner.tool_cache }}/android-sdk

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java 11
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '11'

      - name: Install Android SDK cmdline-tools and packages
        run: |
          sudo apt-get update
          sudo apt-get install -y unzip wget
          mkdir -p $ANDROID_SDK_ROOT/cmdline-tools
          wget https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip -O cmdline-tools.zip
          unzip -q cmdline-tools.zip -d $ANDROID_SDK_ROOT/cmdline-tools
          export PATH=$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools/bin:$PATH
          yes | $ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools/bin/sdkmanager --sdk_root=$ANDROID_SDK_ROOT "platform-tools" "platforms;android-33" "build-tools;33.0.2"

      - name: Grant gradlew execute
        run: |
          if [ -f ./gradlew ]; then chmod +x ./gradlew; fi
          if [ -f android/gradlew ]; then chmod +x android/gradlew; fi

      - name: Build Debug APK (root gradle or android/)
        run: |
          if [ -f ./gradlew ]; then ./gradlew assembleDebug; elif [ -f android/gradlew ]; then cd android && ./gradlew assembleDebug; else echo "No gradle wrapper found"; exit 1; fi

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: debug-apk
          path: |
            app/build/outputs/apk/debug/*.apk
            android/app/build/outputs/apk/debug/*.apk
            build/app/outputs/flutter-apk/*.apk
'@

$workflowPath = Join-Path $workflowDir "build-apk-debug.yml"
$workflowYaml | Out-File -FilePath $workflowPath -Encoding utf8 -Force
Write-Host "Arquivo escrito em: $workflowPath"

# Create ZIP
$zipPath = "build-apk-debug.zip"
if (Test-Path $zipPath) { Remove-Item $zipPath -Force }
Compress-Archive -Path $workflowPath -DestinationPath $zipPath -Force
Write-Host "ZIP criado em: $zipPath"

# Generate Base64 and save to file, also print to console
$b64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($zipPath))
$b64 | Tee-Object -FilePath $OutB64 | Out-Host

Write-Host ""
Write-Host "Base64 gravado em: $OutB64"
Write-Host "Para decodificar o base64 de volta para ZIP no PowerShell execute:"
Write-Host "[IO.File]::WriteAllBytes('build-apk-debug.zip',[Convert]::FromBase64String((Get-Content $OutB64 -Raw)))"
