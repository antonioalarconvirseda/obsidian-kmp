# kotlinx.serialization

## Contexto
TerrariaWiki recibe y envía JSON. Necesitamos un parser que:
- Funcione con `data class` de Kotlin sin escribir código a mano.
- Sea multiplataforma (futuro KMP).
- No use reflexión (mejor para tamaño de APK y arranque en frío).
- Tolere campos nuevos o faltantes de la API externa.

## Decisión
Adoptar **kotlinx.serialization 1.7.x** con el plugin del compilador.

Configuración del JSON:
```kotlin
private val terrariaJson = Json {
    ignoreUnknownKeys = true   // API puede añadir campos sin romper la app
    isLenient = true           // acepta JSON con comillas simples, etc.
    explicitNulls = false      // no serializa nulls (payload más pequeño)
    encodeDefaults = true      // serializa defaults si vienen en la data class
}
```

## Implementación Kotlin

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.serialization)
}

// Cualquier DTO:
@Serializable
data class ItemDto(
    val name: String = "",
    val type: String = "",
    val rare: String = "0",
    @SerialName("tooltip") val tooltip: String? = null,
    @SerialName("listcat") val listCat: String? = null,
    val image: String? = null
)
```

```kotlin
// Uso con Ktor:
install(ContentNegotiation) {
    json(terrariaJson)
}

val response: CargoResponse<ItemDto> = client.get("...") { ... }.body()
```

### `@SerialName` para mapear snake_case → camelCase

MediaWiki devuelve `listcat`, `usetime`, `itemid`, etc. (todo minúsculas, sin separadores). En Kotlin preferimos camelCase. `@SerialName("listcat")` resuelve:

```kotlin
@SerialName("listcat") val listCat: String? = null
```

## Alternativas descartadas
- **Gson:** usa reflexión, no KMP-friendly, mantenimiento停了.
- **Moshi:** excelente, pero multiplataforma más limitado que kotlinx.serialization.
- **Jackson:** demasiado pesado para Android.

## Riesgos & mitigación
- **Riesgo:** olvidarse de poner el plugin `kotlin.serialization` en `build.gradle.kts` y los DTOs no compilan. **Mitigación:** el plugin está en el version catalog (`libs.plugins.kotlin.serialization`) y aplicado en el módulo `app`.
- **Riesgo:** un campo nuevo en la API no se deserializa. **Mitigación:** `ignoreUnknownKeys = true` ya lo cubre; los campos añadidos al DTO más adelante se pueden mapear sin breaking change.
- **Riesgo:** R8/ProGuard puede borrar los serializers generados. **Mitigación:** reglas ProGuard en `proguard-rules.pro`:
  ```
  -keepattributes *Annotation*, InnerClasses
  -keepclassmembers class **$$serializer { *; }
  -keepclassmembers class * {
      @kotlinx.serialization.Serializable <fields>;
  }
  ```
