# Setup de Entorno — TerrariaWiki

Estado del entorno verificado el 2026-07-25, durante la fase de planificación (GLM-5.2).

## Sistema operativo
- **macOS 26.5.1** (`darwin`, `arm64` = Apple Silicon)

## Android Studio
- Instalado en `/Applications/Android Studio.app`
- Última versión estable descargada de https://developer.android.com/studio
- Trae: JDK, SDK, Platform-tools, AVD Manager, emulador, etc.

## JDK
- **OpenJDK 21.0.10** bundled con Android Studio
- Path: `/Applications/Android Studio.app/Contents/jbr/Contents/Home`
- Suficiente para Kotlin 2.0+ y AGP 8.x

## Android SDK
- Path: `~/Library/Android/sdk`
- **Platforms instaladas:** `android-35`, `android-36`
- **Build-tools instaladas:** `35.0.0`, `36.1.0`
- Decisión: `compileSdk = 35`, `targetSdk = 35`, `minSdk = 26`

### Por qué minSdk = 26 (Android 8.0)
- Cubre ~99% de dispositivos activos en 2026.
- Permite usar APIs modernas (Compose, coroutines, Flow) sin polyfills.
- `targetSdk = 35` es la última estable recomendada por Google a fecha de hoy.

## ADB / Platform-tools
- `~/Library/Android/sdk/platform-tools/adb`
- **No está en `$PATH` por defecto** en este Mac → usar path absoluto o añadir a PATH.

## Kotlin
- Plugin del IDE en `/Applications/Android Studio.app/Contents/plugins/Kotlin`
- Versión del compilador se gestiona vía `gradle/libs.versions.toml`

## Git
- `git config --global user.name = AAV`
- `git config --global user.email = antonioalarconvirseda@hotmail.com`
- Protocolo: HTTPS (no SSH)

## GitHub CLI (`gh`)
- Instalado en `/opt/homebrew/bin/gh`
- Autenticado como **antonioalarconvirseda**
- Scopes: `gist`, `read:org`, `repo`, `workflow`
- Suficiente para crear el repo público y hacer push con `gh repo create --push`

## Obsidian
- Bóveda en `/Users/aav/gits/obsidian-kmp/Kotlin Multiplatform/`
- Esta bóveda contiene las notas de documentación de TerrariaWiki bajo `TerrariaWiki/`

---

## Conexión del dispositivo físico (cuando el usuario decida)

Pasos exactos para un móvil **Android 12+**:

1. **Activar Opciones de desarrollador:**
   - Ajustes → Acerca del teléfono
   - Pulsar **7 veces** en "Número de compilación" hasta que aparece el mensaje "Ahora eres un desarrollador".

2. **Activar Depuración USB:**
   - Ajustes → Sistema → Opciones de desarrollador
   - Activar **Depuración USB**.

3. **Conectar por USB al Mac.**

4. **Verificar conexión desde terminal:**
   ```bash
   ~/Library/Android/sdk/platform-tools/adb devices
   ```
   Debe mostrar el dispositivo como `device` (no `unauthorized` ni `offline`).

5. **Aceptar huella RSA en el móvil:**
   - La primera vez aparece un diálogo "¿Permitir depuración USB desde este equipo?"
   - Marcar "Permitir siempre" y aceptar.

6. **Confirmar en Android Studio:**
   - Menú Run → Select Device → debe aparecer el modelo del móvil.

### Notas por fabricante

| Fabricante | Extra a activar |
|---|---|
| Xiaomi / MIUI / HyperOS | "Instalar vía USB" en opciones de desarrollador (requiere cuenta Mi activa). |
| Realme / ColorOS | Igual que Xiaomi; además desactivar "Verificación de aplicaciones vía USB". |
| OnePlus / OxygenOS | 7 toques en "Número de compilación" + cuenta OnePlus activa. |
| Pixel / Samsung / ASUS | Flujo estándar sin extras. |

### Para instalar el APK debug

```bash
# desde la raíz del proyecto
./gradlew :app:installDebug

# o manualmente
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

---

## Validación previa realizada (GLM-5.2)
- `adb version` → OK
- `~/Library/Android/sdk/platform-tools/adb devices` → lista vacía (sin móvil conectado todavía, esperado)
- Listado de `platforms/`, `build-tools/` → presente
- `java -version` bundled → 21.0.10
- `gh auth status` → logged in como `antonioalarconvirseda`
