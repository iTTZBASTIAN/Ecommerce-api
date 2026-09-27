# Plan de Mejoramiento PM1: registro de correcciones

**Programa:** Análisis y Desarrollo de Software (228118), ficha 3223876
**Proyecto:** App Android de e-commerce (Kotlin + Jetpack Compose) conectada a la API REST en Python (Flask)
**Documento local:** este archivo no se sube al repositorio (está excluido en `.git/info/exclude`).

Este documento registra, paso a paso, **qué se corrigió, dónde, cómo era antes, cómo quedó y por qué**,
en el orden en que se trabajó: **Ubicaciones → Roles → Perfil → Reto extra (carrito y pedidos)**.

> **Nota sobre los números de línea**
> - **"Antes"** es la línea en el código original del repositorio (`main` del instructor).
> - **"Ahora"** es la línea en el código corregido.
> - Cada cambio lleva en el código un comentario `[ERROR X]` con el antes, el cambio y el porqué;
>   se pueden buscar en Android Studio con `Ctrl + Shift + F`.

---

## Índice

1. [Diagnóstico general: la causa común](#1-diagnóstico-general-la-causa-común)
2. [Puesta en marcha: backend y conexión del celular](#2-puesta-en-marcha-backend-y-conexión-del-celular)
3. [Reto 1: Gestión de Ubicaciones (errores 1 a 8)](#3-reto-1-gestión-de-ubicaciones)
4. [Reto 2A: Gestión de Usuarios y cambio de rol (errores U1 a U6)](#4-reto-2a-gestión-de-usuarios-y-cambio-de-rol)
5. [Reto 2B: Mi Perfil (errores P1 a P8)](#5-reto-2b-mi-perfil)
6. [Reto extra: carrito → pedido → historial (errores C1 a C7)](#6-reto-extra-carrito--confirmar-pedido--historial)
7. [Resumen de archivos modificados](#7-resumen-de-archivos-modificados)
8. [Hipótesis vs. hallazgos (para la sustentación)](#8-hipótesis-vs-hallazgos)
9. [Totales de correcciones](#9-totales-de-correcciones)
10. [Paso a paso de ramas y commits en Git](#10-paso-a-paso-de-ramas-y-commits-en-git)

---

## 1. Diagnóstico general: la causa común

El **backend casi nunca devuelve la lista o el objeto "solo"**: lo envuelve dentro de otro objeto.
La app Android (Retrofit + Gson) esperaba la forma "sola". El código **compilaba**, pero al recibir
la respuesta Gson no podía convertir el JSON.

| Endpoint | Lo que envía el backend | Lo que esperaba la app |
|---|---|---|
| `GET /api/locations/countries` | `{"count": 5, "countries": [ ... ]}` | `[ ... ]` |
| `GET /api/locations/states` | `{"count": n, "states": [ ... ]}` | `[ ... ]` |
| `GET /api/locations/cities` | `{"count": n, "cities": [ ... ]}` | `[ ... ]` |
| `GET /api/auth/roles` | `{"roles": [ ... ]}` | `[ ... ]` |
| `GET/PUT /api/users/profile` | `{"user": { ... }}` | `{ ... }` |
| `GET /api/sales/{id}` | `{"sale": { ... }}` | `{ ... }` |

Error que aparece en Logcat:

```
com.google.gson.JsonSyntaxException: java.lang.IllegalStateException:
Expected BEGIN_ARRAY but was BEGIN_OBJECT at line 1 column 2 path $
```

**Por qué el compilador no lo detecta:** Kotlin solo sabe lo que *declaramos* que llegará
(`Response<List<Country>>`). La forma real del JSON se conoce **en tiempo de ejecución**, cuando
responde el servidor. Por eso "el código compila" no significa "el código funciona".

**Cómo se comprobó:** la terminal del backend mostraba `200` en todas las peticiones, mientras
la app mostraba listas vacías. Eso **descartó** las fallas de servidor, token y conexión, y
**confirmó** que la falla estaba en cómo la app interpretaba el JSON.

---

## 2. Puesta en marcha: backend y conexión del celular

### 2.1 Levantar el backend (Windows, PowerShell)

```powershell
cd C:\Users\panes\OneDrive\Desktop\Ecommerce_svp\Ecommerce-Api-Python
python -m venv venv                  # solo la primera vez
venv\Scripts\activate
pip install Flask==2.3.3 Flask-SQLAlchemy==3.0.5 Flask-CORS==4.0.0 python-dotenv==1.0.0 PyJWT==2.8.0 Werkzeug==2.3.7   # solo la primera vez
python app.py
```

Diferencias con el `README.MD` original:
- **Carpeta:** el README dice `cd ecommerce-api`, pero la carpeta es `Ecommerce-Api-Python`.
- **Archivo de dependencias:** el README dice `requirements.txt`, pero el archivo se llama `requirement.txt`.
- **`psycopg2-binary` no se instala:** es para PostgreSQL y falla en Python 3.14; aquí se usa SQLite.
- **Firewall de Windows:** se creó una regla para permitir el puerto 5050 desde el celular. Se ejecuta una sola vez, como administrador:
  ```powershell
  New-NetFirewallRule -DisplayName "Flask Ecommerce 5050" -Direction Inbound -Protocol TCP -LocalPort 5050 -Action Allow -Profile Any
  ```

Usuarios de prueba:

| Usuario | Contraseña | Rol |
|---|---|---|
| `admin@ecommerce.com` | `admin123` | Administrador |
| `juan@email.com` | `password123` | Cliente |

### 2.2 Conexión de la app al backend local (celular físico)

**Carpeta:** `Ecommerce-Frontend-Kotlin/app/src/main/java/com/ecommerce/ecommerceapp/network/`

#### `ApiClient.kt`: líneas 13-15 → 13-18

```kotlin
// ANTES
private const val BASE_URL = "https://ecommerce-api-python.onrender.com/"
//private const val BASE_URL = "http://192.168.1.44:5050/"
```

```kotlin
// AHORA
// Celular físico: IP del PC en la red Wi-Fi (ipconfig)
private const val BASE_URL = "http://192.168.0.107:5050/"
// Emulador: 10.0.2.2 es el "localhost" del PC visto desde el emulador
//private const val BASE_URL = "http://10.0.2.2:5050/"
//private const val BASE_URL = "https://ecommerce-api-python.onrender.com/"
```

- **Por qué:** la app apuntaba al servidor desplegado en Render, no al backend local. Así no se
  podían ver las peticiones en la terminal ni probar los cambios.
- **Cómo:** desde un celular físico, `localhost` es el propio celular. Hay que usar la IP del PC
  en la red, que se obtiene con `ipconfig`. **Si cambias de red Wi-Fi, cambia la IP.**

#### `app/src/main/res/xml/network_security_config.xml`: línea 6

```xml
<!-- ANTES -->
<domain includeSubdomains="true">192.168.1.0/24</domain>
```

```xml
<!-- AHORA -->
<domain includeSubdomains="false">192.168.0.107</domain>
```

- **Por qué:** Android bloquea HTTP sin cifrar (`http://`) salvo para los dominios listados.
  **Este archivo no acepta rangos** como `/24`, así que esa línea no permitía ninguna IP.
- **Cómo:** se puso la IP exacta del PC. Debe coincidir con la de `ApiClient.kt`.

---

## 3. Reto 1: Gestión de Ubicaciones

**Síntoma:** la pantalla "Gestión de Ubicaciones" no mostraba países, y los desplegables de país y
estado salían vacíos.

**Carpetas y archivos modificados**

| Carpeta | Archivo | Errores |
|---|---|---|
| `models/` | `Locations.kt` | 1, 3 |
| `network/` | `ApiService.kt` | 1 |
| `viewmodels/` | `LocationManagementViewModel.kt` | 2, 4, 5 |
| `viewmodels/` | `ProfileViewModel.kt` | 2 (usa `getCities`) |
| `ui/screens/` | `LocationManagementScreen.kt` | 3, 4, 5, 6, 7, 8 |

Ruta base: `Ecommerce-Frontend-Kotlin/app/src/main/java/com/ecommerce/ecommerceapp/`

### Error 1: la app esperaba una lista y el backend envía un objeto (causa raíz)

**Archivo 1:** `models/Locations.kt`, líneas nuevas 52-76, al final del archivo.

```kotlin
// ANTES: no existían estas clases
```

```kotlin
// AHORA
data class CountriesResponse(
    @SerializedName("countries") val countries: List<Country> = emptyList(),
    @SerializedName("count") val count: Int = 0
)
data class StatesResponse(
    @SerializedName("states") val states: List<State> = emptyList(),
    @SerializedName("count") val count: Int = 0
)
data class CitiesResponse(
    @SerializedName("cities") val cities: List<City> = emptyList(),
    @SerializedName("count") val count: Int = 0
)
```

**Archivo 2:** `network/ApiService.kt`, líneas 62, 68 y 74 → 71, 80 y 89.

```kotlin
// ANTES
suspend fun getCountries(): Response<List<Country>>
suspend fun getStates(@Query("country_id") countryId: Int? = null): Response<List<State>>
suspend fun getCities(@Query("state_id") stateId: Int? = null): Response<List<City>>
```

```kotlin
// AHORA
suspend fun getCountries(): Response<CountriesResponse>
suspend fun getStates(@Query("country_id") countryId: Int? = null): Response<StatesResponse>
suspend fun getCities(@Query("state_id") stateId: Int? = null): Response<CitiesResponse>
```

- **Por qué:** el backend responde `{"countries": [...]}` (ver `location_routes.py`, `get_countries`).
  Gson esperaba `[` y encontraba `{`, así que lanzaba `Expected BEGIN_ARRAY but was BEGIN_OBJECT`.
- **Cómo:** se crearon clases con **la misma forma que el JSON** y se cambió el tipo de respuesta.

### Error 2: el ViewModel asignaba el cuerpo completo como si fuera la lista

**Archivo:** `viewmodels/LocationManagementViewModel.kt`, líneas 47, 66 y 85 → 63, 85 y 107.

```kotlin
// ANTES
countries = response.body()!!
states = response.body()!!
cities = response.body()!!
```

```kotlin
// AHORA
countries = response.body()!!.countries
states = response.body()!!.states
cities = response.body()!!.cities
```

El mismo cambio se hizo en `viewmodels/ProfileViewModel.kt`, línea 173, porque también usaba
`getCities()`. Sin ese cambio el proyecto no compilaba. En la versión final esa función fue
reemplazada por `fetchCities()` (ver P4).

- **Por qué:** al fallar la lectura del JSON, la excepción iba al `catch` y `countries` se quedaba
  en `emptyList()`. La pantalla dibujaba fielmente una lista vacía.
- **Cómo:** ahora se toma la lista que viene **dentro** del objeto.

### Error 3: `State.country` era un objeto, pero llega texto

**Archivo 1:** `models/Locations.kt`, líneas 23-24 → 29-30.

```kotlin
// ANTES
@SerializedName("country")
val country: Country? = null
```

```kotlin
// AHORA
@SerializedName("country")
val country: String? = null
```

**Archivo 2:** `ui/screens/LocationManagementScreen.kt`, líneas 296 y 365 → 373 y 537.

```kotlin
// ANTES
text = state.country?.CountryName ?: "",
text = { Text("${state.StatesName} - ${state.country?.CountryName}") },
```

```kotlin
// AHORA
text = state.country ?: "",
text = { Text("${state.StatesName} - ${state.country ?: ""}") },
```

- **Por qué:** el backend construye cada estado con `'country': self.country.CountryName`
  (`models.py`), es decir, **el nombre como texto**. Gson esperaba `{` y encontraba `"`, así que
  lanzaba `Expected BEGIN_OBJECT but was STRING`. Este error **estaba escondido detrás del error 1**:
  solo aparece cuando la lista empieza a leerse.

### Error 4: seleccionar un país no cargaba sus estados

**Archivo:** `viewmodels/LocationManagementViewModel.kt`, líneas 211-214 → 244-251.

```kotlin
// ANTES
fun updateSelectedCountryForState(country: Country?) {
    selectedCountryForState = country
    clearMessages()
}
```

```kotlin
// AHORA
fun updateSelectedCountryForState(country: Country?) {
    selectedCountryForState = country
    clearMessages()
    loadStates(country?.iD_Country)   // pide solo los estados de ese país
}
```

- **Por qué:** solo se guardaba el país; nunca se pedían sus estados.
- **Cómo:** se llama `loadStates(idPaís)`, que envía `GET /api/locations/states?country_id=N`.

### Error 5: la cascada país → estado en la pestaña Ciudades y la selección inconsistente

**Archivo:** `viewmodels/LocationManagementViewModel.kt`.

**5a. Variables:** línea 31 → 31 y 40.

```kotlin
// ANTES
var selectedStateForCity by mutableStateOf<State?>(null)
```

```kotlin
// AHORA
var selectedCountryForCity by mutableStateOf<Country?>(null)   // nueva: país elegido en Ciudades
var selectedStateForCity by mutableStateOf<State?>(null)       // se conserva: estado elegido
```

Durante la corrección esta línea se **reemplazó por error** en lugar de agregarse, y aparecieron
8 errores `Unresolved reference 'selectedStateForCity'`. Se corrigió dejando **las dos variables**,
porque la cascada tiene dos niveles.

**5b. Nueva función:** líneas 260-266.

```kotlin
// ANTES: no existía
```

```kotlin
// AHORA
fun updateSelectedCountryForCity(country: Country?) {
    selectedCountryForCity = country
    selectedStateForCity = null   // el estado anterior podía ser de otro país
    states = emptyList()
    clearMessages()
    if (country != null) loadStates(country.iD_Country)
}
```

**5c. Al elegir estado se cargan sus ciudades:** líneas 221-224 → 275-282.

```kotlin
// ANTES
fun updateSelectedStateForCity(state: State?) {
    selectedStateForCity = state
    clearMessages()
}
```

```kotlin
// AHORA
fun updateSelectedStateForCity(state: State?) {
    selectedStateForCity = state
    clearMessages()
    loadCities(state?.iD_States)
}
```

**5d. Después de crear un estado o una ciudad:** líneas 149-150 → 178 y 186-187 → 220.

```kotlin
// ANTES (createState)
selectedCountryForState = null
loadStates() // Recargar lista
// ANTES (createCity)
selectedStateForCity = null
loadCities() // Recargar lista
```

```kotlin
// AHORA
loadStates(selectedCountryForState?.iD_Country)
loadCities(selectedStateForCity?.iD_States)
```

- **Por qué:**
  - Si el usuario elegía un estado y luego cambiaba de país, el estado anterior seguía
    seleccionado aunque fuera de otro país. Es la "selección inconsistente" que menciona la guía.
  - Al crear un estado se perdía el país elegido y se recargaban los estados de **todos** los países.
- **Cómo:**
  - Al cambiar de país se limpia el estado.
  - Cada lista se pide filtrada por el nivel anterior.
  - Después de crear algo se conserva la selección.

**5e. Pantalla, selector de país en Ciudades.** Archivo `ui/screens/LocationManagementScreen.kt`.

```kotlin
// ANTES (líneas 317-320): cargaba los estados de TODOS los países
LaunchedEffect(Unit) {
    viewModel.loadStates()
}
```

```kotlin
// AHORA (línea 404): solo los del país elegido en esta pestaña
LaunchedEffect(Unit) {
    viewModel.selectedCountryForCity?.let { viewModel.loadStates(it.iD_Country) }
}
```

Además se agregó:
- **Un desplegable de País nuevo** (líneas 426-482) antes del de Estado. Llama a
  `updateSelectedCountryForCity(country)`.
- **Cambios en el desplegable de Estado** (líneas 483-537):
  - Se abre solo si hay país: `onExpandedChange = { if (hayPaisElegido) ... }`.
  - Aparece deshabilitado con `enabled = hayPaisElegido`.
  - El texto guía al usuario: `"Primero elige un país"`, `"Cargando estados..."`, `"Seleccionar estado"`.
  - Si el país no tiene estados muestra `"Este país no tiene estados registrados"`.

### Error 6: sin indicador de carga ni forma de reintentar

**Archivo:** `ui/screens/LocationManagementScreen.kt`, desplegable de país de la pestaña Estados.
Las líneas 212 y 216 pasan a 251-270, y se agrega el bloque de las líneas 284-300.

```kotlin
// ANTES
value = viewModel.selectedCountryForState?.CountryName ?: "Seleccionar país",
trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = expandedCountries) },
// (si la lista estaba vacía el menú se abría sin nada)
```

```kotlin
// AHORA
value = when {
    viewModel.isLoadingCountries -> "Cargando países..."
    viewModel.selectedCountryForState != null -> viewModel.selectedCountryForState!!.CountryName
    else -> "Seleccionar país"
},
trailingIcon = {
    if (viewModel.isLoadingCountries) CircularProgressIndicator(modifier = Modifier.size(20.dp), strokeWidth = 2.dp)
    else ExposedDropdownMenuDefaults.TrailingIcon(expanded = expandedCountries)
},
// ...dentro del menú:
if (viewModel.countries.isEmpty() && !viewModel.isLoadingCountries) {
    DropdownMenuItem(
        text = { Text("No hay países. Toca para reintentar") },
        onClick = { expandedCountries = false; viewModel.loadCountries() }
    )
}
```

- **Por qué:** `loadCountries()` solo se ejecuta una vez, en el `init` del ViewModel. En el log del
  backend se vio que la app **nunca pidió `/countries`**, porque la pantalla se abrió cuando el
  servidor estaba apagado. El desplegable quedaba vacío para siempre.
- **Cómo:**
  - Se muestra el estado **cargando** con texto y un spinner.
  - Si la lista está vacía, se ofrece reintentar.

### Error 7: la pestaña activa se perdía al rotar la pantalla

**Archivo:** `ui/screens/LocationManagementScreen.kt`, línea 25 → 39, más el import nuevo de la línea 13.

```kotlin
// ANTES
var selectedTab by remember { mutableStateOf(0) }
```

```kotlin
// AHORA
import androidx.compose.runtime.saveable.rememberSaveable
var selectedTab by rememberSaveable { mutableStateOf(0) }
```

- **Por qué:** al rotar, Android **destruye y recrea la Activity**. `remember` pierde su valor;
  `rememberSaveable` lo guarda en el `Bundle` y lo restaura.
- **Nota:** el país y el estado seleccionados no se pierden porque viven en el **ViewModel**, que
  sobrevive a la rotación.

### Error 8: las pestañas y el botón ⋮ ignoraban el filtro

**Archivo:** `ui/screens/LocationManagementScreen.kt`.

**Pestañas:** líneas 71 → 90 y 79 → 102.

```kotlin
// ANTES
viewModel.loadStates()
viewModel.loadCities()
```

```kotlin
// AHORA
viewModel.loadStates(viewModel.selectedCountryForState?.iD_Country)
viewModel.loadCities(viewModel.selectedStateForCity?.iD_States)
```

**Botón ⋮ de cada país:** líneas 87 → 115, 96 → 131 y 172 → 211, más el import nuevo de la línea 22.

```kotlin
// ANTES
0 -> CountriesTab(viewModel)
fun CountriesTab(viewModel: LocationManagementViewModel) {
    onClick = { viewModel.loadStates(country.iD_Country) }
```

```kotlin
// AHORA
import com.ecommerce.ecommerceapp.models.Country
0 -> CountriesTab(viewModel) { country ->
    viewModel.updateSelectedCountryForState(country)
    selectedTab = 1
}
fun CountriesTab(viewModel: LocationManagementViewModel, onVerEstados: (Country) -> Unit) {
    onClick = { onVerEstados(country) }
```

- **Por qué:**
  - Al tocar una pestaña se pedía todo sin filtro, y se perdía el país elegido.
  - El botón ⋮ cargaba estados en segundo plano, pero la pantalla se quedaba en Países.
- **Cómo:**
  - Las pestañas envían el id seleccionado.
  - El botón selecciona el país y **cambia a la pestaña Estados**. Solo la pantalla principal puede
    cambiar `selectedTab`, por eso la acción se pasa como parámetro.

---

## 4. Reto 2A: Gestión de Usuarios y cambio de rol

**Síntoma:** la lista de usuarios cargaba, pero el desplegable "Asignar rol" salía **vacío** y no
se podía cambiar ningún rol. No aparecía ningún mensaje de error.

**Carpetas y archivos modificados**

| Carpeta | Archivo | Errores |
|---|---|---|
| `models/` | `UserManagement.kt` | U1, U3 |
| `network/` | `ApiService.kt` | U1 |
| `viewmodels/` | `UserManagementViewModel.kt` | U1, U2, U4, U5, U6 |
| `ui/screens/` | `UserManagementScreen.kt` | U6 |

### U1: roles, lista vs. objeto (causa raíz)

**Archivo 1:** `models/UserManagement.kt`, líneas nuevas 49-52.

```kotlin
// AHORA (antes no existía)
data class RolesResponse(
    @SerializedName("roles") val roles: List<Role> = emptyList()
)
```

**Archivo 2:** `network/ApiService.kt`, línea 80 → 99.

```kotlin
// ANTES
suspend fun getRoles(): Response<List<Role>>
```

```kotlin
// AHORA
suspend fun getRoles(): Response<RolesResponse>
```

**Archivo 3:** `viewmodels/UserManagementViewModel.kt`, línea 76 → 98.

```kotlin
// ANTES
availableRoles = response.body()!!
```

```kotlin
// AHORA
availableRoles = response.body()!!.roles
```

- **Por qué:** `GET /api/auth/roles` responde `{"roles": [...]}` (`auth_routes.py`, `get_roles`).
  Es el mismo problema del Reto 1.
- **Contraste útil:** `getUsers()` sí funcionaba porque `UsersResponse` ya tenía el campo `users`.
  Mismo backend y mismo token: una petición funcionaba y la otra no.

### U2: el `catch` escondía el error

**Archivo:** `viewmodels/UserManagementViewModel.kt`, líneas 75-79 → 95-112.

```kotlin
// ANTES
if (response.isSuccessful && response.body() != null) {
    availableRoles = response.body()!!
}
} catch (e: Exception) {
    Log.e("USER_MGMT_DEBUG", "Error loading roles", e)
```

```kotlin
// AHORA
if (response.isSuccessful && response.body() != null) {
    availableRoles = response.body()!!.roles
} else {
    errorMessage = "Error al cargar roles (HTTP ${response.code()})"
}
} catch (e: Exception) {
    errorMessage = "No se pudieron cargar los roles: ${e.message}"
    Log.e("USER_MGMT_DEBUG", "Error loading roles", e)
```

- **Por qué:** es el **"catch vacío que esconde la causa real"** que menciona la guía. El error
  solo se veía en Logcat; el usuario solo veía un desplegable vacío.

### U3: paginación con un nombre de campo distinto

**Archivo:** `models/UserManagement.kt`, líneas 23-24 → 29-30.

```kotlin
// ANTES
@SerializedName("total_pages")
val total_pages: Int
```

```kotlin
// AHORA
@SerializedName("pages")
val total_pages: Int
```

- **Por qué:** el backend envía `"pagination": {"page", "pages", "per_page", "total"}`. Gson dejaba
  `0` y `totalPages > 1` nunca se cumplía, así que con más de 10 usuarios no aparecerían los
  botones Anterior y Siguiente.
- **Cómo:** `@SerializedName` indica **qué clave del JSON** se lee; el nombre en Kotlin puede
  seguir igual, así no se tocan las demás clases.

### U4: mensajes claros para 401 y 403, y el caso sin token

**Archivo:** `viewmodels/UserManagementViewModel.kt`.

```kotlin
// ANTES (líneas 58, 116 y 140)
errorMessage = "Error al cargar usuarios"
errorMessage = "Error al actualizar el rol del usuario"
errorMessage = "Error al eliminar el usuario"
// y si el token era null no pasaba nada (no había else)
```

```kotlin
// AHORA (líneas 72, 164 y 203, y la función nueva de la línea 222)
errorMessage = mensajeDeError(response.code(), "cargar usuarios")
errorMessage = mensajeDeError(response.code(), "actualizar el rol")
errorMessage = mensajeDeError(response.code(), "eliminar el usuario")

private fun mensajeDeError(code: Int, accion: String): String = when (code) {
    401 -> "Tu sesión expiró o no es válida. Cierra sesión e inicia de nuevo"
    403 -> "No tienes permisos de administrador para $accion"
    404 -> "El usuario ya no existe"
    else -> "Error al $accion (HTTP $code)"
}
// + else { errorMessage = "No hay sesión activa. Inicia sesión de nuevo" }  (líneas 78, 169 y 207)
```

- **Por qué:** la guía pide verificar qué pasa si el token expiró. El token dura **24 horas**
  (`auth_routes.py`, `generate_token`); vencido, el backend responde **401**.

### U5: validar el rol de Administrador en la app

**Archivo:** `viewmodels/UserManagementViewModel.kt`, líneas nuevas 143-146 (`updateUserRole`) y 189-192 (`deleteUser`).

```kotlin
// AHORA (antes no existía)
if (!sessionManager.isAdmin.first()) {
    errorMessage = "Solo un Administrador puede cambiar roles"
    return@launch
}
```

- **Por qué:** la guía pide comprobar que solo un Administrador pueda ejecutar la acción. La app
  solo ocultaba el botón del menú.
- **Idea clave para la sustentación:**
  - **La seguridad real está en el backend.** Se comprobó que Juan recibe **403**
    `"No tienes permisos de administrador"`.
  - **La verificación en la app es solo experiencia de usuario:** avisa al instante sin enviar la
    petición. La app se puede modificar; el servidor no.

### U6: el administrador podía quitarse su propio rol

**Archivo 1:** `viewmodels/UserManagementViewModel.kt`, líneas nuevas 28 y 44.

```kotlin
var currentUserEmail by mutableStateOf<String?>(null)
// en init:
viewModelScope.launch { currentUserEmail = sessionManager.userEmail.first() }
```

**Archivo 2:** `ui/screens/UserManagementScreen.kt`.

```kotlin
// ANTES (líneas 207 y 210)
onExpandedChange = { expandedRoles = !expandedRoles }
value = "Cambiar rol",
```

```kotlin
// AHORA (líneas 96, 150, 219, 225 y 231)
esUsuarioActual = user.Email == viewModel.currentUserEmail,     // al llamar a UserCard
esUsuarioActual: Boolean,                                       // parámetro nuevo de UserCard
onExpandedChange = { if (!esUsuarioActual) expandedRoles = !expandedRoles }
value = if (esUsuarioActual) "No puedes cambiar tu propio rol" else "Cambiar rol",
enabled = !esUsuarioActual,
```

- **Por qué:** `PUT /api/users/{id}/roles` **reemplaza todos** los roles del usuario. Si el admin
  se asignaba "Cliente", perdía el rol de Administrador y nadie podía devolvérselo.

---

## 5. Reto 2B: Mi Perfil

**Síntoma:**
- El perfil cargaba **en blanco**: sin nombre, email ni roles.
- El desplegable de ciudad salía vacío.
- Al guardar se enviaba `iD_City = 0`. En la base de datos el usuario 9 quedó con `iD_City = 0`,
  una ciudad inexistente.

**Carpetas y archivos modificados**

| Carpeta | Archivo | Errores |
|---|---|---|
| `models/` | `UserProfile.kt` | P1, P2, P5 |
| `network/` | `ApiService.kt` | P1 |
| `viewmodels/` | `ProfileViewModel.kt` | P1 a P8 |
| `ui/screens/` | `ProfileScreen.kt` | P4, P5, P7, P8 |

### P1: el perfil llega envuelto en `{"user": {...}}` (causa raíz)

**Archivo 1:** `models/UserProfile.kt`, líneas nuevas 36-42.

```kotlin
// AHORA (antes no existía)
data class UserResponse(
    @SerializedName("message") val message: String? = null,
    @SerializedName("user") val user: UserProfile? = null
)
```

**Archivo 2:** `network/ApiService.kt`, líneas 18 → 21 y 24 → 30.

```kotlin
// ANTES
suspend fun getProfile(...): Response<UserProfile>
suspend fun updateProfile(...): Response<UserProfile>
```

```kotlin
// AHORA
suspend fun getProfile(...): Response<UserResponse>
suspend fun updateProfile(...): Response<UserResponse>
```

**Archivo 3:** `viewmodels/ProfileViewModel.kt`, líneas 51 → 100 y 86 → 193.

```kotlin
// ANTES
userProfile = response.body()
```

```kotlin
// AHORA
val profile = response.body()?.user
val updated = response.body()?.user
```

- **Por qué:** a diferencia del Reto 1, aquí **Gson no lanzaba error**. Simplemente no encontraba
  `UserName` ni `Email` en el primer nivel y los dejaba vacíos. El servidor respondía `200` con los
  datos, pero la app los leía en el nivel equivocado.

### P2: ciudad inventada (1 o 0)

**Archivo 1:** `models/UserProfile.kt`, línea 16 → 20.

```kotlin
// ANTES
val iD_City: Int,
```

```kotlin
// AHORA
val iD_City: Int? = null,
```

**Archivo 2:** `viewmodels/ProfileViewModel.kt`, líneas 19 → 31 y 53 → 109.

```kotlin
// ANTES
var selectedCityId by mutableStateOf(1)
selectedCityId = userProfile?.iD_City ?: 1
```

```kotlin
// AHORA
var selectedCityId by mutableStateOf<Int?>(null)
preseleccionarUbicacion(profile.city)
```

- **Por qué:** `?: 1` inventaba una ciudad, y como el perfil llegaba vacío (P1) el valor era `0`.
  Con `null` se puede exigir al usuario que elija una ciudad antes de guardar.

### P3: el administrador perdía su rol al guardar el perfil

**Archivo:** `viewmodels/ProfileViewModel.kt`, líneas 89-95 → 205-212.

```kotlin
// ANTES
isAdmin = userProfile?.roles?.any { it.TypeRole == "Administrador" } ?: false,
roles = userProfile?.roles?.map { it.TypeRole } ?: emptyList()
```

```kotlin
// AHORA
isAdmin = roles?.any { it.TypeRole == "Administrador" } ?: sessionManager.isAdmin.first(),
roles = roles?.map { it.TypeRole } ?: sessionManager.userRoles.first()
```

- **Por qué:** con P1, `roles` era `null`, y `?: false` guardaba `isAdmin = false` en la sesión.
  El admin que editaba su perfil perdía el "Panel de Administración" hasta volver a iniciar sesión.
- **Cómo:** si no llegan roles, se conservan los de la sesión.

### P4: la cascada País → Estado → Ciudad en el perfil

**Archivo 1:** `viewmodels/ProfileViewModel.kt`.

```kotlin
// ANTES (líneas 37-40 y 167-181)
init {
    loadProfile()
    loadCities()        // TODAS las ciudades; si fallaba solo se escribía en Logcat
}
private fun loadCities() { ... catch { Log.e(...) } }
```

```kotlin
// AHORA
init {                                   // línea 77: primero países, luego perfil (en orden)
    viewModelScope.launch {
        countries = fetchCountries()
        loadProfileInternal()
    }
}
private suspend fun preseleccionarUbicacion(city: City?)   // línea 133
private suspend fun fetchCountries(): List<Country>          // línea 315
private suspend fun fetchStates(countryId: Int): List<State> // línea 334
private suspend fun fetchCities(stateId: Int): List<City>    // línea 353
fun updateSelectedCountry(country: Country)   // línea 393: limpia estado y ciudad y carga estados
fun updateSelectedState(state: State)         // línea 406: limpia ciudad y carga ciudades
```

**Archivo 2:** `ui/screens/ProfileScreen.kt`, líneas 92-127 → 117-160, más el componente nuevo de la línea 324.

```kotlin
// ANTES: un único desplegable "Ciudad" con todas las ciudades mezcladas
```

```kotlin
// AHORA: tres desplegables en cascada con el componente reutilizable PerfilDropdown
PerfilDropdown(label = "País", ...)
PerfilDropdown(label = "Estado / Departamento", habilitado = viewModel.selectedCountry != null, ...)
PerfilDropdown(label = "Ciudad", habilitado = viewModel.selectedState != null, ...)
```

- **Por qué:** la guía pide que *"el país y el estado seleccionados (reto 1) se envíen dentro de
  la actualización"*. El backend guarda `iD_City`; la cascada garantiza que esa ciudad pertenezca
  al estado y al país elegidos.
- **Cómo preselecciona:** la ciudad del usuario llega como
  `{"iD_City":1, "iD_States":1, "state":"Antioquia", "country":"Colombia"}`.
  1. Se busca `"Colombia"` en la lista de países para obtener su id.
  2. Se cargan los estados de Colombia y se selecciona Antioquia.
  3. Se cargan las ciudades de Antioquia y se selecciona Medellín.

### P5: el email no se podía editar ni validar

**Archivo 1:** `models/UserProfile.kt`, línea nueva 53.

```kotlin
// ANTES
data class UpdateProfileRequest(val UserName: String, val iD_City: Int)
```

```kotlin
// AHORA
data class UpdateProfileRequest(val UserName: String, val Email: String, val iD_City: Int)
```

**Archivo 2:** `ui/screens/ProfileScreen.kt`, líneas 129-139 → 161-186, más los imports de las líneas 7 y 17.

```kotlin
// ANTES
value = viewModel.userProfile?.Email ?: "",
onValueChange = { },
enabled = false
```

```kotlin
// AHORA
value = viewModel.email,
onValueChange = viewModel::updateEmail,
keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
isError = emailInvalido,
supportingText = { if (emailInvalido) Text("Formato de correo no válido") },
```

- **Por qué:** la guía pide editar los datos y validar el formato del correo. El backend sí acepta
  cambiar `Email` y responde **409** si ya lo usa otro usuario.

### P6: validación de campos obligatorios

**Archivo:** `viewmodels/ProfileViewModel.kt`, líneas 68-71 → 150-172.

```kotlin
// ANTES
if (userName.isBlank()) { errorMessage = "El nombre de usuario es requerido"; return }
```

```kotlin
// AHORA
if (nombre.isEmpty())   -> "El nombre de usuario es requerido"
if (nombre.length < 3)  -> "El nombre de usuario debe tener al menos 3 caracteres"
if (correo.isEmpty())   -> "El correo es requerido"
if (!Patterns.EMAIL_ADDRESS.matcher(correo).matches()) -> "El formato del correo no es válido"
if (cityId == null)     -> "Selecciona país, estado y ciudad"
```

- **Por qué:** *"una app que guarda datos vacíos o inválidos no está terminada"* (guía).

### P7: mensajes de sesión, 409 e indicador de carga

**Archivos:** `viewmodels/ProfileViewModel.kt` (líneas 219-224 y función `mensajeDeError` en la
línea 437) y `ui/screens/ProfileScreen.kt` (línea 102).

```kotlin
// ANTES
errorMessage = "Error al actualizar el perfil"
```

```kotlin
// AHORA
errorMessage = if (response.code() == 409) "Ese correo ya está en uso por otro usuario"
               else mensajeDeError(response.code(), "actualizar el perfil")
// + else { "No hay sesión activa. Inicia sesión de nuevo" }
// + LinearProgressIndicator mientras isLoadingProfile
```

### P8: cambio de contraseña, código incorrecto y diálogo que no se cerraba

**Archivo 1:** `viewmodels/ProfileViewModel.kt`, líneas 152-155 → 290-295, y línea nueva 282.

```kotlin
// ANTES
401 -> "Contraseña actual incorrecta"
else -> "Error al cambiar la contraseña"
```

```kotlin
// AHORA
400 -> "La contraseña actual es incorrecta"
401 -> "Tu sesión expiró o no es válida. Cierra sesión e inicia de nuevo"
else -> "Error al cambiar la contraseña (HTTP ${response.code()})"
passwordChanged = true   // al confirmarse el cambio
```

**Archivo 2:** `ui/screens/ProfileScreen.kt`, líneas 247-251 → 303, más el bloque nuevo de la línea 55.

```kotlin
// ANTES
onClick = {
    viewModel.changePassword()
    if (viewModel.successMessage != null) { showChangePasswordDialog = false }
}
```

```kotlin
// AHORA
onClick = { viewModel.changePassword() }
LaunchedEffect(viewModel.passwordChanged) {
    if (viewModel.passwordChanged) { showChangePasswordDialog = false; viewModel.passwordChanged = false }
}
```

- **Por qué:**
  - El backend responde **400** "Password actual incorrecto" y reserva 401 para el token.
  - `successMessage` siempre era `null` justo después de llamar a `changePassword()`, porque la
    petición corre en una **corrutina** que todavía no había respondido.

---

## 6. Reto extra: carrito → confirmar pedido → historial

**Síntoma, visto en la terminal del backend:**

```
POST /api/sales/checkout 201   ← el pedido SÍ se creó
POST /api/sales/checkout 400   ← reintento: "Carrito vacío"
POST /api/sales/checkout 400
```

- **Al confirmar:** la app mostraba error aunque el pedido se había creado.
- **Al reintentar:** decía "Datos de pedido inválidos".
- **El historial fallaba** en cuanto existía al menos un pedido.

**Carpetas y archivos modificados**

| Carpeta | Archivo | Errores |
|---|---|---|
| `models/` | `Cart.kt` | C1, C5, C6, C7 |
| `models/` | `UserManagement.kt` (clase `Pagination`) | C7 |
| `network/` | `ApiService.kt` | C6 |
| `viewmodels/` | `CartViewModel.kt` | C2, C3, C4 |
| `viewmodels/` | `SalesHistoryViewModel.kt` | C6, C7 |

### C1: `Sale.user` era un objeto, pero llega texto (causa raíz)

**Archivo:** `models/Cart.kt`, líneas 80-81 → 94-95.

```kotlin
// ANTES
@SerializedName("user")
val user: User? = null,
```

```kotlin
// AHORA
@SerializedName("user")
val user: String? = null,
```

- **Por qué:** el backend envía `"user": "juan_perez"` (`models.py`, `Sales.to_dict`:
  `self.user.UserName`). Gson lanzaba `Expected BEGIN_OBJECT but was STRING at $.sale.user`.
- **Cómo se encadena el conflicto:**
  1. El backend crea la venta, descuenta el stock y vincula los ítems del carrito a la venta.
     Responde **201**.
  2. La app falla al leer `sale.user` y muestra "Error de conexión". No limpia el carrito ni abre
     "Pedido exitoso".
  3. El usuario reintenta y el backend ya no tiene ítems sin venta, así que responde
     **400 "Carrito vacío"**.
  4. El historial usa el mismo modelo `Sale`: **confirmar un pedido es justo lo que rompe el historial**.
- **¿El pedido se creaba una sola vez?** **Sí.** El backend no duplica pedidos; el problema estaba
  solo en el frontend.

### C2: el checkout, éxito por código HTTP y mensajes reales

**Archivo:** `viewmodels/CartViewModel.kt`, líneas 277-298 → 296-334, más la línea 346 y la función nueva de la línea 357.

```kotlin
// ANTES
if (response.isSuccessful && response.body() != null) {
    val checkoutResponse = response.body()!!
    if (checkoutResponse.isSuccess()) { ...; lastCompletedSale = checkoutResponse.sale }
    else { errorMessage = checkoutResponse.message ?: "Error al procesar el pedido" }
} else {
    errorMessage = when (response.code()) {
        400 -> "Datos de pedido inválidos"
        ...
    }
}
```

```kotlin
// AHORA
if (response.isSuccessful) {                       // 201 = pedido creado
    cart = Cart(items = emptyList(), total_items = 0, total_amount = 0.0)
    cartItemsCount = 0
    lastCompletedSale = checkoutResponse?.sale ?: Sale(DescripcionSale = request.DescripcionSale)
    ...
} else {
    val mensajeServidor = leerErrorDelServidor(response.errorBody()?.string())
    errorMessage = when (response.code()) {
        400 -> mensajeServidor ?: "No se pudo procesar el pedido"   // "Carrito vacío", "Stock insuficiente para X"
        401 -> "Tu sesión expiró. Inicia sesión de nuevo"
        else -> "Error al procesar el pedido (HTTP ${response.code()})"
    }
    loadCart()                                     // mostrar el estado real del servidor
}
// y en el catch también: loadCart()
```

- **Por qué:**
  - `CartScreen` navega a "Pedido exitoso" cuando `lastCompletedSale` deja de ser `null`; si quedaba
    `null`, el usuario nunca veía la confirmación.
  - El backend usa 400 tanto para "Carrito vacío" como para "Stock insuficiente". Mostrar su mensaje
    real explica lo que pasó.
  - El log usaba `checkoutResponse.sale_id`, que el backend no envía (siempre salía `null`).

### C3: al agregar un producto el carrito se "vaciaba"

**Archivo:** `viewmodels/CartViewModel.kt`, líneas 114-127 → 126-129.

```kotlin
// ANTES
if (response.isSuccessful && response.body() != null) {
    val responseBody = response.body()!!
    cart = responseBody
    cartItemsCount = cart?.total_items ?: 0
```

```kotlin
// AHORA
if (response.isSuccessful) {
    successMessage = "Producto agregado al carrito"
    loadCart()
```

- **Por qué:** `POST /cart/add` solo responde `{"message": "Producto agregado al carrito"}`.
  - Como `Cart` tiene valores por defecto en todos sus campos, Kotlin genera un constructor vacío
    y Gson lo usa. El resultado era un `Cart(items = [], total_items = 0)` que **reemplazaba el
    carrito real**, **sin lanzar ningún error**.
  - El contador del catálogo volvía a 0 y "ya en el carrito" dejaba de marcarse. Así se podía
    agregar el producto de nuevo y recibir **400 por stock**, como se vio en el log a las 21:09:20.
- **Cómo:** se pide el carrito real con `GET /api/sales/cart`.

### C4: al cambiar la cantidad desaparecía la lista

**Archivo:** `viewmodels/CartViewModel.kt`, líneas 172-176 → 182-185.

```kotlin
// ANTES
if (response.isSuccessful && response.body() != null) {
    cart = response.body()
    cartItemsCount = cart?.total_items ?: 0
```

```kotlin
// AHORA
if (response.isSuccessful) {
    successMessage = "Cantidad actualizada"
    loadCart()
```

- **Por qué:** `PUT /cart/update/{id}` responde `{"message", "item"}`, no el carrito. Es el mismo
  problema que C3.

### C5: nombres de campo distintos en `Sale` y `SaleDetail`

**Archivo:** `models/Cart.kt`.

| Campo en Kotlin | Antes: `@SerializedName` | Ahora: `@SerializedName` | Línea antes → ahora |
|---|---|---|---|
| `TotalSale` | `"TotalSale"` | `"total"` | 71 → 75 |
| `DateSale` | `"DateSale"` | `"DateCreated"` | 74 → 81 |
| `sale_details` | `"sale_details"` | `"details"` | 83 → 100 |
| `id_SaleDetail` | `"id_SaleDetail"` | `"id_SalesDetails"` | 89 → 108 |
| `Quantity` | `"Quantity"` | `"amount"` | 92 → 114 |
| `Subtotal` | `"Subtotal"` | `"ValueSale"` | 98 → 120 |
| `UnitPrice` | `"UnitPrice"` (no existe en el JSON) | **se calcula** | 95-96 → 135-136 |

```kotlin
// ANTES
@SerializedName("UnitPrice")
val UnitPrice: Double = 0.0,
```

```kotlin
// AHORA
val UnitPrice: Double
    get() = if (Quantity > 0) Subtotal / Quantity else 0.0
```

- **Por qué:** Gson no encontraba las claves y usaba los valores por defecto. El historial mostraba
  `$0.00`, sin fecha, "0 productos" y `$0.0 x 0`.
- **Cómo:** solo se cambió la clave de `@SerializedName`; los nombres en Kotlin se mantienen, así
  que `SalesHistoryScreen.kt` **no necesitó cambios**.

### C6: el detalle de un pedido llega envuelto en `{"sale": {...}}`

**Archivo 1:** `models/Cart.kt`, línea nueva 142.

```kotlin
data class SaleResponse(@SerializedName("sale") val sale: Sale? = null)
```

**Archivo 2:** `network/ApiService.kt`, línea 224 → 246.

```kotlin
// ANTES
): Response<Sale>
```

```kotlin
// AHORA
): Response<SaleResponse>
```

**Archivo 3:** `viewmodels/SalesHistoryViewModel.kt`, líneas 90-91 → 100-102.

```kotlin
// ANTES
selectedSale = response.body()
```

```kotlin
// AHORA
val sale = response.body()?.sale
selectedSale = sale
```

### C7: el contador de pedidos mostraba 0

**Archivo 1:** `models/UserManagement.kt` (clase `Pagination`, compartida con Usuarios), línea nueva 36.

```kotlin
@SerializedName("total")
val total: Int = 0
```

**Archivo 2:** `viewmodels/SalesHistoryViewModel.kt`, línea 57 → 61.

```kotlin
// ANTES
totalSales = salesResponse.total
```

```kotlin
// AHORA
totalSales = salesResponse.pagination?.total ?: sales.size
```

- **Por qué:** `GET /api/sales/` envía el total **dentro** de `"pagination"`, no en el primer nivel.

---

## 7. Resumen de archivos modificados

Ruta base: `Ecommerce-Frontend-Kotlin/app/src/main/`

| Archivo | Ubicaciones | Roles | Perfil | Carrito | Conexión |
|---|:-:|:-:|:-:|:-:|:-:|
| `java/.../models/Locations.kt` | 1, 3 | | | | |
| `java/.../models/UserManagement.kt` | | U1, U3 | | C7 | |
| `java/.../models/UserProfile.kt` | | | P1, P2, P5 | | |
| `java/.../models/Cart.kt` | | | | C1, C5, C6, C7 | |
| `java/.../network/ApiService.kt` | 1 | U1 | P1 | C6 | |
| `java/.../network/ApiClient.kt` | | | | | ✔ |
| `java/.../viewmodels/LocationManagementViewModel.kt` | 2, 4, 5 | | | | |
| `java/.../viewmodels/UserManagementViewModel.kt` | | U1, U2, U4, U5, U6 | | | |
| `java/.../viewmodels/ProfileViewModel.kt` | 2 | | P1 a P8 | | |
| `java/.../viewmodels/CartViewModel.kt` | | | | C2, C3, C4 | |
| `java/.../viewmodels/SalesHistoryViewModel.kt` | | | | C6, C7 | |
| `java/.../ui/screens/LocationManagementScreen.kt` | 3 a 8 | | | | |
| `java/.../ui/screens/UserManagementScreen.kt` | | U6 | | | |
| `java/.../ui/screens/ProfileScreen.kt` | | | P4, P5, P7, P8 | | |
| `res/xml/network_security_config.xml` | | | | | ✔ |

**Backend (Python):** no se modificó el código. Solo se creó el entorno `venv`, se instalaron las
dependencias y se abrió el puerto 5050 en el firewall.

---

## 8. Hipótesis vs. hallazgos

| Hipótesis inicial (EV01) | Resultado | Cómo se comprobó |
|---|---|---|
| El endpoint no existe o devuelve error (404/500) | ❌ Descartada | La terminal del backend mostraba `200` y con `curl` se veían los 5 países |
| La petición nunca se dispara | ⚠️ Parcial | Se dispara en el `init`; solo faltó cuando la pantalla se abrió con el servidor apagado (se corrigió con reintento, error 6) |
| El modelo no coincide con el JSON | ✅ **Confirmada (causa raíz)** | Logcat: `200` con `{"countries": [...]}` seguido de `Expected BEGIN_ARRAY but was BEGIN_OBJECT` |
| La pantalla no observa el estado | ❌ Descartada | `mutableStateOf` estaba bien; el estado nunca recibía datos |
| Falta el token (401) | ❌ Descartada | Las rutas de ubicaciones no piden token, y `/users/profile` respondía `200` |

**Frase para la sustentación:**
> "El error no estaba en la conexión ni en el backend: estaba en el **contrato de datos**. El
> servidor enviaba la lista dentro de un objeto y el modelo de Kotlin esperaba la lista sola. El
> compilador no lo detecta porque la forma del JSON solo se conoce en tiempo de ejecución."

---

## 9. Totales de correcciones

| Reto | Errores | Cantidad |
|---|---|---|
| Reto 1: Gestión de Ubicaciones | 1 a 8 | **8** |
| Reto 2A: Usuarios y roles | U1 a U6 | **6** |
| Reto 2B: Mi Perfil | P1 a P8 | **8** |
| Reto extra: carrito → pedido → historial | C1 a C7 | **7** |
| **Total de errores corregidos** | | **29** |

Cómo se refleja en el código (medido con `git diff`):

| Medida | Cantidad |
|---|---|
| Archivos modificados | 15 |
| Bloques de código modificados | 143 |
| Líneas de código nuevas (sin comentarios ni líneas en blanco) | 534 |
| Líneas originales eliminadas o reemplazadas | 174 |
| Comentarios de corrección (`[ERROR …]` / `[PASO …]`) | 118 |

- **La conexión no cuenta entre los 29 errores:** el ajuste de `ApiClient.kt` y
  `network_security_config.xml` es de configuración, pero sí está incluido en los 143 bloques.
- **Una sola causa raíz** (el backend envuelve la respuesta o usa otros nombres de campo) explica
  **10 de las 29** correcciones:

  | Grupo | Correcciones |
  |---|---|
  | Reto 1 | 1, 2 y 3 |
  | Roles | U1 y U3 |
  | Perfil | P1 |
  | Carrito | C1, C5, C6 y C7 |

  Las otras 19 son de lógica y experiencia de usuario.

---

## 10. Paso a paso de ramas y commits en Git

La guía pide trabajar el Reto 1 en la rama **`fix/desplegable-ubicaciones`** con **commits pequeños
y descriptivos**, y aclara que **el historial de commits también se evalúa**. Hoy todas las
correcciones están juntas en `main`, sin commits. Esta sección explica cómo repartirlas en ramas y
commits.

### 10.1 Estructura de ramas

Cada reto usa código del anterior; por ejemplo, el Perfil usa los modelos de Ubicaciones. Por eso las
ramas se crean **una encima de la otra** ("apiladas"):

```
main (código original del instructor)
 └── fix/desplegable-ubicaciones        ← Reto 1     (8 commits)
      └── fix/roles-usuarios             ← Reto 2A    (5 commits)
           └── fix/perfil-usuario         ← Reto 2B    (1 commit)
                └── fix/flujo-compra-historial ← Reto extra (4 commits)
```

**Truco clave:** `git switch -c <rama>` crea la rama **llevándose los cambios sin commit**. Así se
trabaja siempre sobre la misma carpeta: se hacen los commits de un reto, se crea la rama siguiente
y los cambios pendientes pasan a ella.

### 10.2 Preparación (una sola vez)

Abre la **Terminal de Android Studio**, en la carpeta `Ecommerce-Frontend-Kotlin`, y ejecuta:

```bash
# 1. Respaldo de TODAS las correcciones (por si algo sale mal)
git diff > ../respaldo_correcciones.patch
copy app\src\main\java\com\ecommerce\ecommerceapp\viewmodels\ProfileViewModel.kt ..\ProfileViewModel_final.kt

# 2. Sacar de los commits los archivos de Android Studio (no son parte de las correcciones)
git restore --staged .idea/markdown.xml
git restore .idea/deploymentTargetSelector.xml

# 3. Borrar la rama y el stash viejos que se crearon al inicio (no se usan)
git branch -D fix/roles-y-perfil
git stash drop

# 4. Crear la rama del Reto 1 (se lleva todos los cambios)
git switch -c fix/desplegable-ubicaciones
```

Si algo falla, recuperas todo con `git apply ../respaldo_correcciones.patch`.

### 10.3 Cómo elegir qué partes van en cada commit

Un mismo archivo tiene cambios de varios retos (por ejemplo, `ApiService.kt` tiene cambios de
Ubicaciones, Roles, Perfil y Carrito). Para meter en cada commit **solo los bloques que corresponden**
se usa:

```bash
git add -p <archivo>
```

Git muestra cada bloque de cambios (hunk) y pregunta qué hacer:

| Tecla | Acción |
|---|---|
| `y` | Incluir este bloque en el commit |
| `n` | No incluirlo; queda pendiente para otro commit |
| `s` | Dividir el bloque en partes más pequeñas |
| `q` | Terminar |

**Cómo saber a qué commit pertenece un bloque:** cada bloque tiene su comentario `[ERROR X]` o
`[PASO X]`. Si el bloque dice `[ERROR 1]`, va en el commit del error 1.

**Alternativa visual en Android Studio:**
1. Abre la ventana **Commit** (`Ctrl + K`).
2. Expande el archivo y **marca solo las casillas** de los cambios que quieres incluir.
3. Escribe el mensaje y presiona **Commit**.

**Después de cada commit, compila** (`Ctrl + F9`) para confirmar que ese commit funciona solo.

#### Bloques que traen dos correcciones juntas

`git add -p` junta los cambios que están cerca (a menos de 3 líneas). Se revisó el diff y estos son
los bloques que aparecen **mezclados**:

| Archivo | El bloque trae | Qué hacer |
|---|---|---|
| `ApiService.kt` (ubicaciones y roles) | `[ERROR 1]` ×3 + `[ERROR U1]` | Presiona `s` para dividirlo: `y` en las 3 partes `[ERROR 1]` y `n` en `[ERROR U1]` |
| `LocationManagementViewModel.kt` (funciones de selección) | `[ERROR 4]` + `[ERROR 5]` | `s`: `y` en `[ERROR 4]` para el commit 4 y `n` en `[ERROR 5]`, que va en el commit 5 |
| `LocationManagementScreen.kt` (imports) | `[ERROR 7]` + `[PASO 6c]` | `s`: el import `rememberSaveable` va en el commit 7 y el import `Country` en el commit 8 |
| `LocationManagementScreen.kt` (final de `CitiesTab`) | `[PASO 6e]` + `[ERROR 3]` | `s`: `[ERROR 3]` va en el commit 3 y `[PASO 6e]` en el commit 5 |
| `UserManagement.kt` (`Pagination` y `RolesResponse`) | `[ERROR U3]` + `[ERROR C7]` + `[ERROR U1]` | `s`: `[ERROR U1]` va en el commit U1. `[ERROR U3]` y `[ERROR C7]` quedan juntos (líneas seguidas) y van en el commit U3 |

Si después de presionar `s` un bloque **sigue** teniendo dos etiquetas, inclúyelo en el commit que
va **primero** en el orden de esta guía. El proyecto compila igual, porque los commits siguientes
solo agregan cosas.

### 10.4 Rama `fix/desplegable-ubicaciones` (Reto 1): 8 commits

| # | Mensaje del commit | Qué incluir |
|---|---|---|
| 1 | `chore(red): apuntar la app al backend local para celular fisico` | `network/ApiClient.kt` y `res/xml/network_security_config.xml`, completos |
| 2 | `fix(ubicaciones): deserializar listas envueltas de la API (errores 1 y 2)` | `Locations.kt`: bloque `[ERROR 1]` (clases `...Response`); `ApiService.kt`: los 3 bloques `[ERROR 1]`; `LocationManagementViewModel.kt`: los 3 bloques `[ERROR 2]`; `ProfileViewModel.kt`: **ver nota abajo** |
| 3 | `fix(ubicaciones): country de State es texto, no objeto (error 3)` | `Locations.kt`: bloque `[ERROR 3]`; `LocationManagementScreen.kt`: los 2 bloques `[ERROR 3]` |
| 4 | `feat(ubicaciones): cargar estados al seleccionar un pais (error 4)` | `LocationManagementViewModel.kt`: bloque `[ERROR 4]` |
| 5 | `feat(ubicaciones): cascada pais-estado en Ciudades y limpiar seleccion (error 5)` | `LocationManagementViewModel.kt`: bloques `[ERROR 5]`; `LocationManagementScreen.kt`: bloques `[PASO 6e]` |
| 6 | `feat(ubicaciones): indicador de carga y reintento en desplegables (error 6)` | `LocationManagementScreen.kt`: bloques `[ERROR 6]` |
| 7 | `fix(ubicaciones): conservar pestaña al rotar con rememberSaveable (error 7)` | `LocationManagementScreen.kt`: bloques `[ERROR 7]` |
| 8 | `feat(ubicaciones): pestañas y boton de pais respetan el filtro (error 8)` | `LocationManagementScreen.kt`: bloques `[PASO 6b]` y `[PASO 6c]` |

**Nota del commit 2 (`ProfileViewModel.kt`).** En el Reto 1 solo se cambió **una línea** de este
archivo (`cities = response.body()!!.cities`). Luego el Perfil (P4) lo reescribió completo, así que
esa línea ya no se puede separar con `git add -p`. Sin esa línea, el commit 2 **no compila**. Pasos:

```bash
# a) Volver temporalmente ProfileViewModel.kt al original
git restore app/src/main/java/com/ecommerce/ecommerceapp/viewmodels/ProfileViewModel.kt
# b) En Android Studio, línea 173, cambiar:
#       cities = response.body()!!
#    por:
#       cities = response.body()!!.cities
# c) Incluirlo en el commit 2
git add app/src/main/java/com/ecommerce/ecommerceapp/viewmodels/ProfileViewModel.kt
# d) Después del commit 2, devolver la versión final del perfil:
copy ..\ProfileViewModel_final.kt app\src\main\java\com\ecommerce\ecommerceapp\viewmodels\ProfileViewModel.kt
```

Ejemplo completo del commit 2:

```bash
git add -p app/src/main/java/com/ecommerce/ecommerceapp/models/Locations.kt          # "y" solo en [ERROR 1]
git add -p app/src/main/java/com/ecommerce/ecommerceapp/network/ApiService.kt        # "y" solo en los 3 [ERROR 1]
git add -p app/src/main/java/com/ecommerce/ecommerceapp/viewmodels/LocationManagementViewModel.kt  # "y" solo en [ERROR 2]
# + ProfileViewModel.kt con los pasos a), b) y c) de la nota
git commit -m "fix(ubicaciones): deserializar listas envueltas de la API (errores 1 y 2)"
```

Al terminar los 8 commits, publica la rama en **tu repositorio personal**. La guía pide el
repositorio del aprendiz, no el del instructor.

```bash
git remote add personal https://github.com/<TU-USUARIO>/Ecommerce-Frontend-Kotlin.git   # solo la primera vez
git push -u personal fix/desplegable-ubicaciones
```

### 10.5 Rama `fix/roles-usuarios` (Reto 2A): 5 commits

```bash
git switch -c fix/roles-usuarios
```

| # | Mensaje del commit | Qué incluir |
|---|---|---|
| 1 | `fix(usuarios): deserializar roles y mostrar el error al usuario (U1 y U2)` | `UserManagement.kt`: bloque `[ERROR U1]` (`RolesResponse`); `ApiService.kt`: bloque `[ERROR U1]`; `UserManagementViewModel.kt`: bloques `[ERROR U1]` y `[ERROR U2]` |
| 2 | `fix(usuarios): leer "pages" en la paginacion (U3)` | `UserManagement.kt`: bloque `[ERROR U3]` |
| 3 | `fix(usuarios): mensajes claros para 401/403 y sesion ausente (U4)` | `UserManagementViewModel.kt`: bloques `[ERROR U4]` (incluye la función `mensajeDeError`) |
| 4 | `fix(usuarios): validar rol Administrador antes de cambiar roles (U5)` | `UserManagementViewModel.kt`: bloques `[ERROR U5]` |
| 5 | `feat(usuarios): impedir que el administrador cambie su propio rol (U6)` | `UserManagementViewModel.kt` y `UserManagementScreen.kt`: bloques `[ERROR U6]` |

En el commit 2, el bloque de `Pagination` trae juntos `[ERROR U3]` y `[ERROR C7]`, porque son
líneas seguidas y `git add -p` no los puede separar. Inclúyelo completo aquí: el campo `total` de C7
no afecta a Usuarios y en el Reto extra ya estará disponible.

```bash
git push -u personal fix/roles-usuarios
```

### 10.6 Rama `fix/perfil-usuario` (Reto 2B): 1 commit

```bash
git switch -c fix/perfil-usuario
git add app/src/main/java/com/ecommerce/ecommerceapp/models/UserProfile.kt
git add -p app/src/main/java/com/ecommerce/ecommerceapp/network/ApiService.kt     # "y" solo en los 2 [ERROR P1]
git add app/src/main/java/com/ecommerce/ecommerceapp/viewmodels/ProfileViewModel.kt
git add app/src/main/java/com/ecommerce/ecommerceapp/ui/screens/ProfileScreen.kt
git commit -m "fix(perfil): leer usuario envuelto, cascada pais-estado-ciudad y validaciones (P1-P8)" -m "P1 respuesta {user}; P2 iD_City nulo; P3 conservar rol admin; P4 cascada de ubicacion; P5 email editable; P6 validaciones; P7 mensajes 401/409; P8 cambio de contrasena"
git push -u personal fix/perfil-usuario
```

**¿Por qué un solo commit?** `ProfileViewModel.kt` se reescribió casi completo. Si P1 se separa de
los demás, el commit no compila: el ViewModel usa `response.body()?.user`, y eso necesita
`UserResponse`, el tipo de `ApiService` y los otros cambios. Por eso el mensaje lista las 8
correcciones.

### 10.7 Rama `fix/flujo-compra-historial` (Reto extra): 4 commits

```bash
git switch -c fix/flujo-compra-historial
```

| # | Mensaje del commit | Qué incluir |
|---|---|---|
| 1 | `fix(pedidos): alinear modelos de venta con la API (C1, C5, C6, C7)` | `Cart.kt`: completo; `ApiService.kt`: bloque `[ERROR C6]`; `SalesHistoryViewModel.kt`: completo |
| 2 | `fix(carrito): confirmar pedido por codigo 201 y mostrar el error real (C2)` | `CartViewModel.kt`: bloques `[ERROR C2]` |
| 3 | `fix(carrito): recargar carrito real al agregar producto (C3)` | `CartViewModel.kt`: bloque `[ERROR C3]` |
| 4 | `fix(carrito): recargar carrito real al cambiar cantidad (C4)` | `CartViewModel.kt`: bloque `[ERROR C4]` |

```bash
git push -u personal fix/flujo-compra-historial
```

### 10.8 Verificación final

```bash
git status                     # debe decir "nothing to commit" (salvo archivos de .idea)
git log --oneline --graph --all
```

Deberías ver **18 commits** en total, repartidos así:

| Rama | Commits |
|---|---|
| `fix/desplegable-ubicaciones` | 8 |
| `fix/roles-usuarios` | 5 |
| `fix/perfil-usuario` | 1 |
| `fix/flujo-compra-historial` | 4 |

Para dejar todo en `main` de tu repositorio personal, puedes:
- **Opción A (recomendada):** crear en GitHub un *Pull Request* por rama, en orden, y aprobarlos.
- **Opción B, desde la terminal:**
  ```bash
  git switch main
  git merge fix/flujo-compra-historial   # trae los 4 retos, porque las ramas están apiladas
  git push personal main
  ```

**Recordatorios:**
- No subas `database/ecommerce.db` del backend si no quieres compartir los datos de las pruebas.
- Este `README_CORRECCIONES.md` **no se sube**: está excluido en `.git/info/exclude` del backend.
