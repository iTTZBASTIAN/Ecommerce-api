# Guía del estudiante: cómo encontramos y arreglamos los errores de la app

Esta guía explica, con palabras sencillas y ejemplos, **qué estaba fallando** en la app de
e-commerce, **cómo lo arreglamos** y **qué debes ver en la pantalla** cuando cada arreglo funciona.

No necesitas saberlo todo de Kotlin ni de Python para entenderla. Vamos paso a paso.

---

## Contenido

0. [Antes de empezar: ¿cómo viaja un dato?](#0-antes-de-empezar-cómo-viaja-un-dato)
1. [Levantar el backend (el servidor)](#1-levantar-el-backend-el-servidor)
2. [Conectar la app del celular con el backend](#2-conectar-la-app-del-celular-con-el-backend)
3. [Reto 1: Gestión de Ubicaciones](#3-reto-1-gestión-de-ubicaciones)
4. [Reto 2A: Cambiar el rol de un usuario](#4-reto-2a-cambiar-el-rol-de-un-usuario)
5. [Reto 2B: Editar el perfil](#5-reto-2b-editar-el-perfil)
6. [Reto extra: carrito, pedido e historial](#6-reto-extra-carrito-pedido-e-historial)
7. [Resumen final y la lección más importante](#7-resumen-final-y-la-lección-más-importante)

---

## 0. Antes de empezar: ¿cómo viaja un dato?

Imagina que pides una pizza por teléfono:

1. **Tú** haces el pedido. En la app, eres **la pantalla**.
2. **El que contesta** anota el pedido y lo pasa a la cocina. Ese es el **ViewModel**.
3. **El mensajero** lleva y trae cosas. Ese es **Retrofit / ApiService**.
4. **La cocina** prepara la pizza. Ese es el **backend en Python (Flask)**.
5. **La despensa** guarda los ingredientes. Esa es la **base de datos (SQLite)**.

En la app, el recorrido de un dato (por ejemplo, la lista de países) es este:

```
 BASE DE DATOS  ──►  BACKEND (Flask)  ──►  ApiService (Retrofit)  ──►  ViewModel  ──►  PANTALLA
 (tabla country)     GET /countries        convierte JSON a Kotlin      guarda el estado  dibuja la lista
```

**Si la pantalla sale vacía, la cadena se rompió en algún punto.** Nuestro trabajo es encontrar
**en cuál**.

### ¿Qué es JSON?

Es el "idioma" en que el backend le responde a la app. Se ve así:

```json
{ "CountryName": "Colombia", "iD_Country": 1 }
```

- Lo que va entre **llaves `{ }`** es un **objeto**: una cosa con propiedades.
- Lo que va entre **corchetes `[ ]`** es una **lista**: varias cosas seguidas.

```json
[ {"CountryName": "Colombia"}, {"CountryName": "México"} ]        ← esto es una LISTA
{ "countries": [ {"CountryName": "Colombia"} ] }                  ← esto es un OBJETO que tiene una lista adentro
```

**Esta diferencia es la causa de casi todos los errores de este proyecto.** Guárdala en la cabeza.

---

## 1. Levantar el backend (el servidor)

El backend es el programa en Python que responde a la app. Si no está encendido, la app no
tiene a quién pedirle datos.

### Paso 1: abrir una terminal en la carpeta del backend

En VS Code, ve a **Terminal → New Terminal** y escribe:

```powershell
cd C:\Users\panes\OneDrive\Desktop\Ecommerce_svp\Ecommerce-Api-Python
```

### Paso 2: crear el entorno virtual (solo la primera vez)

Un **entorno virtual** es como una caja aparte donde se instalan las librerías de este
proyecto, sin mezclarlas con las de otros.

```powershell
python -m venv venv
```

### Paso 3: activar el entorno virtual

```powershell
venv\Scripts\activate
```

✅ **Sabes que funcionó** si al inicio de la línea aparece `(venv)`.

Si aparece un error de *"la ejecución de scripts está deshabilitada"*, escribe esto una vez y
vuelve a activar:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

### Paso 4: instalar las librerías (solo la primera vez)

```powershell
pip install Flask==2.3.3 Flask-SQLAlchemy==3.0.5 Flask-CORS==4.0.0 python-dotenv==1.0.0 PyJWT==2.8.0 Werkzeug==2.3.7
```

- **El nombre del archivo:** el README original dice `requirements.txt`, pero en el proyecto el
  archivo se llama `requirement.txt`, sin la "s".
- **Por qué instalamos a mano:** ese archivo incluye `psycopg2-binary`, que sirve para otra base de
  datos (PostgreSQL) y da error en Python 3.14. Aquí usamos SQLite, así que no lo necesitamos.

### Paso 5: encender el servidor

```powershell
python app.py
```

✅ **Qué debes ver:**

```
Iniciando servidor Flask...
 * Running on http://127.0.0.1:5050
 * Running on http://192.168.0.107:5050
 * Debugger is active!
```

**No cierres esta terminal.** Mientras esté abierta, el servidor está encendido. Aquí vas a ver
cada petición que haga la app, por ejemplo:

```
192.168.1.34 - - [22/Sep/2026 20:57:34] "GET /api/auth/roles HTTP/1.1" 200 -
```

Cómo leer esa línea:
- **`192.168.1.34`** es **quién** pidió; en este caso, tu celular.
- **`GET /api/auth/roles`** es **qué** pidió.
- **`200`** es **cómo le fue**:

| Código | Significa |
|---|---|
| **200** | Todo bien |
| **201** | Se creó algo nuevo |
| **400** | Datos rechazados |
| **401** | No has iniciado sesión o tu sesión venció |
| **403** | No tienes permiso |
| **404** | No existe |

### Paso 6: probar que responde

Abre en el navegador: `http://localhost:5050/api/locations/countries`

✅ **Debes ver** un texto con 5 países (Colombia, Estados Unidos, México, Argentina, España).

**Para apagar el servidor:** presiona `Ctrl + C` en la terminal.

### Usuarios para probar

| Correo | Contraseña | Rol |
|---|---|---|
| `admin@ecommerce.com` | `admin123` | Administrador |
| `juan@email.com` | `password123` | Cliente |

---

## 2. Conectar la app del celular con el backend

### El problema

La app tenía escrita esta dirección:

```kotlin
private const val BASE_URL = "https://ecommerce-api-python.onrender.com/"
```

Esa es la dirección de un servidor en internet (Render), **no la de tu computador**. Es como
tener la dirección de otra pizzería: la app le hacía pedidos a otro lugar y tú nunca veías nada
en tu terminal.

### La solución

**Archivo:** `network/ApiClient.kt`

```kotlin
// ANTES
private const val BASE_URL = "https://ecommerce-api-python.onrender.com/"

// AHORA
private const val BASE_URL = "http://192.168.0.107:5050/"
```

**¿Por qué no `localhost`?** En el celular, `localhost` significa **"el mismo celular"**, no tu
computador. Hay que usar la **IP del computador en el Wi-Fi**, que se obtiene escribiendo
`ipconfig` en la terminal.

| Dónde corre la app | Dirección que se usa |
|---|---|
| Emulador de Android Studio | `http://10.0.2.2:5050/` |
| Celular físico | `http://<IP de tu PC>:5050/` |

**Archivo:** `res/xml/network_security_config.xml`

Android, por seguridad, **bloquea las conexiones `http://` (sin la "s")** salvo a las
direcciones que tú autorices en este archivo.

```xml
<!-- ANTES: un "rango" de IPs; Android NO entiende rangos aquí, así que no autorizaba nada -->
<domain includeSubdomains="true">192.168.1.0/24</domain>

<!-- AHORA: la IP exacta del computador -->
<domain includeSubdomains="false">192.168.0.107</domain>
```

**Firewall de Windows:** Windows bloqueaba las conexiones que llegaban desde el celular. Se
abrió el puerto 5050 una sola vez, en PowerShell **como administrador**:

```powershell
New-NetFirewallRule -DisplayName "Flask Ecommerce 5050" -Direction Inbound -Protocol TCP -LocalPort 5050 -Action Allow -Profile Any
```

✅ **Cómo comprobarlo:** con el celular en el **mismo Wi-Fi**, abre en su navegador
`http://192.168.0.107:5050/api/locations/countries`. Debes ver los 5 países.

⚠️ **Si cambias de red Wi-Fi, tu IP cambia.** Revísala con `ipconfig` y actualiza los dos archivos.

---

## 3. Reto 1: Gestión de Ubicaciones

### ¿Qué se veía mal?

Al abrir **Gestión de Ubicaciones**:
- La lista de países salía **vacía**.
- Aparecía un mensaje: *"Error de conexión: Expected BEGIN_ARRAY but was BEGIN_OBJECT…"*
- En la pestaña **Estados**, el desplegable de país no tenía opciones.
- En **Ciudades** no se podía elegir primero un país.

Pero en la terminal del backend decía **`200`** (todo bien). **El servidor sí respondía.**
Entonces el problema estaba en la app.

### Lista de errores del Reto 1

| # | Error en palabras simples |
|---|---|
| 1 | La app esperaba una **lista** y el servidor mandaba un **objeto con la lista adentro** |
| 2 | El ViewModel guardaba el paquete completo en vez de sacar la lista |
| 3 | La app esperaba el país como objeto, pero llegaba como **texto** |
| 4 | Al elegir un país **no se cargaban sus estados** |
| 5 | En Ciudades no había selector de país, y el estado anterior no se borraba al cambiar de país |
| 6 | No se veía "Cargando…" ni había forma de **reintentar** |
| 7 | Al **girar el celular** la pantalla volvía a la primera pestaña |
| 8 | Las pestañas y el botón **⋮** ignoraban el país elegido |

---

### Error 1: lista vs. objeto (la causa principal)

**Ejemplo para entenderlo:** pediste **una caja de galletas**, pero te entregan **una bolsa que
tiene la caja adentro**. Si esperas abrir directamente la caja y te dan la bolsa, no sabes qué
hacer.

Eso le pasaba a la app:

```json
// Lo que la app ESPERABA (una lista directa):
[ {"CountryName": "Colombia"}, {"CountryName": "México"} ]

// Lo que el backend MANDABA (un objeto con la lista adentro):
{ "count": 5, "countries": [ {"CountryName": "Colombia"}, {"CountryName": "México"} ] }
```

La librería que traduce el JSON (**Gson**) leía el primer símbolo. Esperaba `[` y encontraba
`{`, así que se rendía con este error:

```
Expected BEGIN_ARRAY but was BEGIN_OBJECT
(Esperaba el inicio de una LISTA pero encontré el inicio de un OBJETO)
```

**Cómo se arregló, paso a paso:**

**Paso 1.** En `models/Locations.kt`, crear una clase con **la misma forma que el paquete** que
manda el servidor (la "bolsa"):

```kotlin
data class CountriesResponse(
    @SerializedName("countries") val countries: List<Country> = emptyList(),  // la "caja" que va adentro
    @SerializedName("count") val count: Int = 0
)
// Igual con StatesResponse (clave "states") y CitiesResponse (clave "cities")
```

**Paso 2.** En `network/ApiService.kt`, decirle a Retrofit que va a recibir la bolsa, no la caja:

```kotlin
// ANTES
suspend fun getCountries(): Response<List<Country>>
// AHORA
suspend fun getCountries(): Response<CountriesResponse>
```

Haz lo mismo con `getStates` y `getCities`.

**¿Por qué el compilador no avisó del error?** Porque el compilador solo revisa que **tú**
escribiste las cosas bien. No sabe qué va a mandar el servidor; eso solo se sabe **cuando la app
corre**. Por eso **"compila" no es lo mismo que "funciona"**.

---

### Error 2: sacar la lista de la bolsa

**Archivo:** `viewmodels/LocationManagementViewModel.kt`

Ya recibimos la bolsa; ahora hay que **sacar la caja**:

```kotlin
// ANTES: guardaba la bolsa completa (y fallaba)
countries = response.body()!!

// AHORA: saca la lista que está adentro
countries = response.body()!!.countries
```

Haz lo mismo con `states = response.body()!!.states` y `cities = response.body()!!.cities`.
También en `ProfileViewModel.kt`, que usaba `getCities()`; si no, el proyecto no compila.

**¿Por qué la pantalla salía vacía y no con error?** Cuando fallaba la lectura, el programa
saltaba al `catch` (el "plan B") y la variable `countries` se quedaba como estaba al inicio: **una
lista vacía**. La pantalla dibujaba **fielmente** lo que tenía, que era nada.

✅ **Qué debes ver al corregir los errores 1 y 2:**
- Al abrir **Gestión de Ubicaciones** aparecen los **5 países**.
- En la terminal: `GET /api/locations/countries 200`.
- Ya no sale el mensaje de *BEGIN_ARRAY*.

---

### Error 3: el país llegaba como texto, no como objeto

**Ejemplo:** esperabas una **tarjeta de presentación** con nombre, teléfono y correo (un objeto),
pero te dieron un **papelito que solo dice "Colombia"** (un texto).

Así manda el backend cada estado:

```json
{ "iD_States": 1, "StatesName": "Antioquia", "country": "Colombia" }
```

Fíjate: `"country": "Colombia"` es **texto**, no `{ ... }`.

**Cómo se arregló:**

**Paso 1.** En `models/Locations.kt`:

```kotlin
// ANTES: esperaba un objeto
val country: Country? = null
// AHORA: acepta texto
val country: String? = null
```

**Paso 2.** En `ui/screens/LocationManagementScreen.kt`, mostrar el texto directamente:

```kotlin
// ANTES
text = state.country?.CountryName ?: ""
// AHORA
text = state.country ?: ""
```

**Dato curioso:** este error estaba **escondido detrás del error 1**. Mientras la lista fallaba al
principio, la app nunca llegaba a leer el país. Solo apareció después de arreglar el error 1. Es
como arreglar una llave del agua y descubrir que la tubería de más adelante también goteaba.

✅ **Qué debes ver:** en la pestaña **Estados**, cada estado muestra debajo el nombre de su país.
Por ejemplo: **Antioquia** y, debajo, *Colombia*.

---

### Error 4: elegir un país no cargaba sus estados

**Ejemplo:** en un formulario eliges "Colombia", pero en la lista de departamentos siguen
apareciendo los de todos los países, o ninguno.

**Archivo:** `viewmodels/LocationManagementViewModel.kt`

```kotlin
// ANTES: solo guardaba el país
fun updateSelectedCountryForState(country: Country?) {
    selectedCountryForState = country
    clearMessages()
}

// AHORA: guarda el país Y pide sus estados
fun updateSelectedCountryForState(country: Country?) {
    selectedCountryForState = country
    clearMessages()
    loadStates(country?.iD_Country)   // pide: GET /api/locations/states?country_id=1
}
```

✅ **Qué debes ver:**
- En **Estados**, eliges **Colombia** en el desplegable y la lista de abajo muestra **solo** los
  estados de Colombia (Antioquia, Cundinamarca, Valle del Cauca, Atlántico).
- En la terminal: `GET /api/locations/states?country_id=1 200`.

---

### Error 5: la cascada País → Estado en la pestaña Ciudades

**Ejemplo de "selección inconsistente":** eliges **Colombia → Antioquia**, luego cambias el país a
**México**… pero sigue seleccionado **Antioquia**. ¡Antioquia no está en México! Si guardas una
ciudad así, queda mal registrada.

**Cómo se arregló:**

**Paso 1.** En el ViewModel se usan **dos variables**, una por cada nivel:

```kotlin
var selectedCountryForCity by mutableStateOf<Country?>(null)  // el país elegido en Ciudades
var selectedStateForCity by mutableStateOf<State?>(null)      // el estado elegido en Ciudades
```

⚠️ **Error típico:** si reemplazas una variable por la otra en vez de dejar las dos, aparecen 8
errores `Unresolved reference 'selectedStateForCity'`. Se necesitan **las dos**.

**Paso 2.** Nueva función: al cambiar de país, **se borra el estado anterior**:

```kotlin
fun updateSelectedCountryForCity(country: Country?) {
    selectedCountryForCity = country
    selectedStateForCity = null      // ← borra el estado viejo (era de otro país)
    states = emptyList()
    clearMessages()
    if (country != null) loadStates(country.iD_Country)
}
```

**Paso 3.** Al elegir un estado, se cargan **solo sus ciudades**:

```kotlin
fun updateSelectedStateForCity(state: State?) {
    selectedStateForCity = state
    clearMessages()
    loadCities(state?.iD_States)     // GET /api/locations/cities?state_id=1
}
```

**Paso 4.** En la pantalla (`LocationManagementScreen.kt`):
- **Agregar un desplegable de País** antes del de Estado, en la pestaña Ciudades.
- **El de Estado queda gris** y dice *"Primero elige un país"* hasta que elijas uno.

**Paso 5.** Después de crear un estado o una ciudad, **ya no se borra lo que elegiste**:

```kotlin
// ANTES: borraba el país y recargaba los estados de TODOS los países
selectedCountryForState = null
loadStates()
// AHORA: conserva el país y recarga solo los suyos
loadStates(selectedCountryForState?.iD_Country)
```

**Cómo funciona el flujo ahora:**

```
Eliges "Colombia"  →  se borra el estado anterior  →  GET /states?country_id=1  →  aparecen sus estados
Eliges "Antioquia" →  GET /cities?state_id=1  →  aparecen Medellín, Bello, Envigado, Itagüí
Cambias a "México" →  "Antioquia" se borra solo  →  se cargan los estados de México
```

✅ **Qué debes ver en la pestaña Ciudades:**
1. Al entrar, el campo **Estado** está **gris** y dice *"Primero elige un país"*.
2. Eliges **Colombia** y el campo Estado se activa con los estados de Colombia.
3. Eliges **Antioquia** y abajo aparecen **solo** sus ciudades.
4. Cambias a **México** y **Antioquia desaparece** del campo Estado.
5. Creas una ciudad y **el estado sigue seleccionado** para crear otra.

---

### Error 6: sin "Cargando…" y sin forma de reintentar

**Ejemplo:** abres una app y ves una lista vacía. ¿Está cargando? ¿Falló? ¿No hay datos? No sabes.

Además, la lista de países **solo se pedía una vez**, al abrir la pantalla. Si en ese momento el
servidor estaba apagado, **quedaba vacía para siempre**. En el log del backend se vio que la app
nunca pidió `/countries` justamente por eso.

**Archivo:** `ui/screens/LocationManagementScreen.kt`

```kotlin
// AHORA: el texto del campo cambia según lo que pasa
value = when {
    viewModel.isLoadingCountries -> "Cargando países..."   // mientras llega la respuesta
    viewModel.selectedCountryForState != null -> viewModel.selectedCountryForState!!.CountryName
    else -> "Seleccionar país"
}

// Y si la lista está vacía, se ofrece reintentar:
if (viewModel.countries.isEmpty() && !viewModel.isLoadingCountries) {
    DropdownMenuItem(
        text = { Text("No hay países. Toca para reintentar") },
        onClick = { viewModel.loadCountries() }
    )
}
```

✅ **Qué debes ver:**
- **Mientras carga:** el campo dice *"Cargando países..."* con un **círculo girando** a la derecha.
- **Prueba de reintento:**
  1. Apaga el backend (`Ctrl + C`), sal de la pantalla y vuelve a entrar.
  2. Al abrir el desplegable sale *"No hay países. Toca para reintentar"*.
  3. Enciende el backend y toca esa opción: aparecen los 5 países.

---

### Error 7: al girar el celular se perdía la pestaña

**Ejemplo:** estás en la pestaña **Estados**, giras el celular y la pantalla vuelve sola a
**Países**.

**¿Por qué pasa?** Cuando giras el celular, Android **destruye la pantalla y la vuelve a
crear**. `remember` es como una **nota en un papel que se bota** al rehacer la pantalla.
`rememberSaveable` es como una nota que **se guarda en un cajón** y se recupera después.

```kotlin
// ANTES
var selectedTab by remember { mutableStateOf(0) }
// AHORA
var selectedTab by rememberSaveable { mutableStateOf(0) }
```

**¿Y el país elegido?** No se pierde, porque vive en el **ViewModel**, que **sobrevive** al giro.

✅ **Qué debes ver:** activa la rotación automática, ve a **Estados**, gira el celular y **sigues
en Estados**, con el mismo país elegido.

---

### Error 8: las pestañas y el botón ⋮ ignoraban el filtro

- **Las pestañas:** al tocar **Estados**, se pedían los estados de **todos** los países, y se perdía
  el filtro que habías elegido.
- **El botón ⋮ de cada país:** cargaba sus estados "por detrás", pero la pantalla **se quedaba en
  Países**. Parecía que el botón no hacía nada.

```kotlin
// ANTES (al tocar la pestaña)
viewModel.loadStates()
// AHORA: respeta el país elegido
viewModel.loadStates(viewModel.selectedCountryForState?.iD_Country)

// ANTES (botón ⋮)
onClick = { viewModel.loadStates(country.iD_Country) }
// AHORA: elige el país y CAMBIA a la pestaña Estados
0 -> CountriesTab(viewModel) { country ->
    viewModel.updateSelectedCountryForState(country)
    selectedTab = 1
}
```

✅ **Qué debes ver:** en **Países**, tocas **⋮** junto a **Colombia** y la app **salta a la
pestaña Estados**, con Colombia elegido y solo sus estados en la lista.

---

## 4. Reto 2A: Cambiar el rol de un usuario

### ¿Qué se veía mal?

Entrando como **admin** a **Gestión de Usuarios**:
- **Sí** aparecía la lista de usuarios.
- El desplegable **"Asignar rol"** salía **vacío**.
- **No aparecía ningún mensaje de error.** Todo parecía normal, pero no se podía cambiar ningún rol.

### Lista de errores

| # | Error en palabras simples |
|---|---|
| U1 | Otra vez lista vs. objeto: los roles llegan dentro de `{"roles": [...]}` |
| U2 | El error quedaba **escondido**: solo se veía en Logcat, nunca en pantalla |
| U3 | La paginación buscaba un dato con un nombre equivocado |
| U4 | Mensajes de error que no explicaban nada (sesión vencida, sin permisos) |
| U5 | La app no revisaba si eras **Administrador** antes de cambiar un rol |
| U6 | El admin podía **quitarse su propio rol** y quedarse sin acceso |

---

### U1: lista vs. objeto, versión roles

Es **el mismo problema del Reto 1**:

```json
// El backend manda:
{ "roles": [ {"iDRole": 1, "TypeRole": "Administrador"}, {"iDRole": 2, "TypeRole": "Usuario"} ] }
```

**Cómo se arregló:** con los mismos pasos que en el Reto 1.

```kotlin
// 1) models/UserManagement.kt: la "bolsa"
data class RolesResponse(@SerializedName("roles") val roles: List<Role> = emptyList())

// 2) network/ApiService.kt
suspend fun getRoles(): Response<RolesResponse>      // antes: Response<List<Role>>

// 3) viewmodels/UserManagementViewModel.kt: sacar la lista
availableRoles = response.body()!!.roles             // antes: response.body()!!
```

**Dato para la sustentación:** ¿por qué los **usuarios** sí cargaban? Porque su modelo
(`UsersResponse`) **ya tenía la forma correcta** `{"users": [...]}`. Mismo servidor, mismo
token: una petición funcionaba y la otra no. Eso prueba que el problema era **el modelo**, no la
conexión.

---

### U2: el error escondido

**Ejemplo:** tu carro tiene una falla, pero el testigo del tablero está **apagado**. Sigues
manejando sin saber que algo anda mal.

```kotlin
// ANTES: el error solo iba a Logcat (el usuario nunca lo veía)
} catch (e: Exception) {
    Log.e("USER_MGMT_DEBUG", "Error loading roles", e)
}

// AHORA: además se muestra en pantalla
} catch (e: Exception) {
    errorMessage = "No se pudieron cargar los roles: ${e.message}"
    Log.e("USER_MGMT_DEBUG", "Error loading roles", e)
}
```

La guía del SENA lo llama *"un catch vacío que esconde la causa real"*.

✅ **Qué debes ver al corregir U1 y U2:**
- El desplegable **"Asignar rol"** muestra los **4 roles**: Administrador, Usuario, Vendedor y Cliente.
- En Logcat: `USER_MGMT_DEBUG: Roles cargados: 4`.

---

### U3: la paginación con el nombre equivocado

El servidor manda `"pages": 1`, pero la app buscaba `"total_pages"`. Como no lo encontraba, ponía
**0**, y los botones **Anterior / Siguiente** nunca aparecían.

```kotlin
// ANTES
@SerializedName("total_pages")
// AHORA: @SerializedName dice qué nombre buscar en el JSON
@SerializedName("pages")
val total_pages: Int
```

**Ejemplo:** es como buscar a alguien por "Juanito" en una lista donde está como "Juan". Existe,
pero no lo encuentras porque buscas con otro nombre.

✅ **Qué debes ver:** hoy hay exactamente 10 usuarios, que caben en una página, así que no cambia
nada visible. **Cuando haya más de 10**, aparecerán los botones *Anterior · Página 1 de 2 · Siguiente*.

---

### U4: mensajes claros

Antes, cualquier error decía *"Error al cargar usuarios"*. Ahora el mensaje depende del código
que manda el servidor:

```kotlin
private fun mensajeDeError(code: Int, accion: String): String = when (code) {
    401 -> "Tu sesión expiró o no es válida. Cierra sesión e inicia de nuevo"
    403 -> "No tienes permisos de administrador para $accion"
    404 -> "El usuario ya no existe"
    else -> "Error al $accion (HTTP $code)"
}
```

**Dato:** la sesión (el **token**) dura **24 horas**. Después, el servidor responde **401**.

✅ **Qué debes ver:** si la sesión venció, en vez de un error raro sale *"Tu sesión expiró o no es
válida. Cierra sesión e inicia de nuevo"*.

---

### U5: solo el Administrador puede cambiar roles

```kotlin
// AHORA: antes de pedirle nada al servidor
if (!sessionManager.isAdmin.first()) {
    errorMessage = "Solo un Administrador puede cambiar roles"
    return@launch
}
```

**Idea importante: dos candados.**
- **Candado 1, en la app:** avisa rápido. Pero una app se puede modificar, así que no es seguro por sí solo.
- **Candado 2, en el servidor:** es **la seguridad real**. Lo probamos: si Juan (Cliente) intenta
  cambiar un rol, el servidor responde **403 "No tienes permisos de administrador"**.

✅ **Qué debes ver:**
- Entrando como **Juan**, **no aparece** el "Panel de Administración" en el menú.
- **En Postman**, con el token de Juan, `PUT /api/users/2/roles` responde **403**.

---

### U6: el admin no puede quitarse su propio rol

**Ejemplo:** es como cerrar la puerta de tu casa con las llaves adentro.

Cambiar el rol **reemplaza todos** los roles del usuario. Si el admin se ponía "Cliente" a sí
mismo, perdía el acceso de administrador y nadie podía devolvérselo.

```kotlin
// En la tarjeta del propio usuario:
value = if (esUsuarioActual) "No puedes cambiar tu propio rol" else "Cambiar rol",
enabled = !esUsuarioActual,
```

✅ **Qué debes ver:** en la tarjeta del **admin**, el campo sale **gris** y dice *"No puedes cambiar
tu propio rol"*. En las demás tarjetas sí puedes elegir un rol.

### Flujo completo de cambiar un rol

```
1. Tocas "Asignar rol" en la tarjeta de Juan y eliges "Vendedor"
2. ViewModel: ¿eres Administrador?  ── no ──► "Solo un Administrador puede cambiar roles"
                  │ sí
3. PUT /api/users/2/roles  {"role_ids": [3]}  con el token en la cabecera
4. Backend: ¿el token es válido?        ── no ──► 401 ► "Tu sesión expiró…"
            ¿el usuario es admin?       ── no ──► 403 ► "No tienes permisos…"
                  │ sí
5. Backend guarda el rol ► 200
6. ViewModel muestra "Rol de usuario actualizado exitosamente" y recarga la lista
7. La tarjeta de Juan ahora dice "Roles: Vendedor"
```

✅ **Qué debes ver:** un mensaje verde de éxito abajo, y la tarjeta de Juan con **"Roles:
Vendedor"**. En la terminal: `PUT /api/users/2/roles 200` seguido de `GET /api/users/ 200`.

---

## 5. Reto 2B: Editar el perfil

### ¿Qué se veía mal?

Al abrir **Mi Perfil**:
- El nombre, el correo y los roles salían **en blanco**.
- El desplegable de ciudad estaba **vacío**.
- Si guardabas, se mandaba **ciudad = 0**, que no existe. En la base de datos un usuario ya quedó así.
- Si el admin guardaba su perfil, **perdía el menú de administración**.

### Lista de errores

| # | Error en palabras simples |
|---|---|
| P1 | Los datos del usuario llegan dentro de `{"user": {...}}` y la app no los sacaba |
| P2 | Si faltaba la ciudad, la app **inventaba** la ciudad 1 (o 0) |
| P3 | Al guardar, el admin perdía su rol en la sesión |
| P4 | No existía la cascada **País → Estado → Ciudad** |
| P5 | El correo **no se podía editar** ni se revisaba si estaba bien escrito |
| P6 | Faltaban validaciones: se podía guardar sin ciudad o con datos inválidos |
| P7 | Mensajes de error poco claros |
| P8 | Cambiar contraseña: mensaje equivocado y la ventana no se cerraba |

---

### P1: los datos venían dentro de `"user"`

```json
{ "user": { "UserName": "juan_perez", "Email": "juan@email.com", "roles": [...] } }
```

**Ejemplo:** te mandan una carta **dentro de un sobre**. La app buscaba el texto de la carta en el
sobre y, como no estaba ahí, **no leía nada**.

Diferencia con el Reto 1: aquí **no salía error**. Gson simplemente no encontraba `UserName` ni
`Email` en el primer nivel y **los dejaba vacíos**. Por eso el perfil se veía en blanco y "sin
problemas".

```kotlin
// models/UserProfile.kt: el "sobre"
data class UserResponse(
    @SerializedName("message") val message: String? = null,
    @SerializedName("user") val user: UserProfile? = null
)

// ApiService.kt
suspend fun getProfile(...): Response<UserResponse>     // antes: Response<UserProfile>

// ProfileViewModel.kt: sacar la carta del sobre
val profile = response.body()?.user                     // antes: userProfile = response.body()
```

✅ **Qué debes ver:** el perfil muestra **tu nombre**, **tu correo** y, abajo, tus **roles**.

---

### P2: no inventar la ciudad

```kotlin
// ANTES: si no había ciudad, inventaba la 1
selectedCityId = userProfile?.iD_City ?: 1

// AHORA: empieza vacía (null) y se exige elegirla
var selectedCityId by mutableStateOf<Int?>(null)
```

**Ejemplo:** es como si un formulario marcara solo "Medellín" en tu dirección sin preguntarte.

---

### P3: el admin ya no pierde su rol al guardar

Al guardar el perfil se actualiza la sesión. Como los roles llegaban vacíos (por P1), la app
guardaba **"no es admin"**.

```kotlin
// ANTES: si no había roles → isAdmin = false
isAdmin = userProfile?.roles?.any { it.TypeRole == "Administrador" } ?: false

// AHORA: si no llegan roles, conserva lo que ya estaba en la sesión
isAdmin = roles?.any { it.TypeRole == "Administrador" } ?: sessionManager.isAdmin.first()
```

✅ **Qué debes ver:** como **admin**, guardas tu perfil, vuelves al inicio y **sigue apareciendo el
"Panel de Administración"**.

---

### P4: la cascada País → Estado → Ciudad

La guía pide que el país y el estado del Reto 1 **se usen al editar el perfil**. Antes había un
solo desplegable de ciudad, y además vacío.

**Ahora hay tres desplegables, uno después del otro:**

```
[ País: Colombia ▼ ]  →  [ Estado: Antioquia ▼ ]  →  [ Ciudad: Medellín ▼ ]
```

**Cómo funciona:**
- Al abrir el perfil se **preselecciona tu ubicación actual**. El servidor manda tu ciudad así:
  `{"CityName": "Medellín", "state": "Antioquia", "country": "Colombia"}`.
  1. Se busca "Colombia" en la lista de países.
  2. Se cargan los estados de Colombia y se marca **Antioquia**.
  3. Se cargan las ciudades de Antioquia y se marca **Medellín**.
- **Si cambias el país,** se borran el estado y la ciudad. **Si cambias el estado,** se borra la ciudad.

✅ **Qué debes ver:**
1. Al abrir el perfil de Juan: **Colombia / Antioquia / Medellín** ya elegidos.
2. Cambias el Estado a **Atlántico**: la ciudad se borra y solo aparece **Barranquilla**.
3. Mientras carga, cada campo dice *"Cargando..."* con un círculo girando.

---

### P5 y P6: correo editable y validaciones

**Ejemplo:** un formulario bien hecho **no te deja enviarlo** si falta tu nombre o si escribiste
el correo como `juan@correo` (sin el ".com").

```kotlin
if (nombre.isEmpty())   → "El nombre de usuario es requerido"
if (nombre.length < 3)  → "El nombre de usuario debe tener al menos 3 caracteres"
if (correo.isEmpty())   → "El correo es requerido"
if (correo no tiene formato válido) → "El formato del correo no es válido (ejemplo: nombre@dominio.com)"
if (no hay ciudad)      → "Selecciona país, estado y ciudad"
```

El campo de correo ahora **se puede escribir**, abre el **teclado con @** y se pone **rojo**
mientras el formato esté mal.

✅ **Qué debes ver:**

| Si haces esto… | …debe pasar esto |
|---|---|
| Borras el nombre y guardas | *"El nombre de usuario es requerido"* |
| Escribes `juan@correo` | El campo se pone **rojo** y dice *"Formato de correo no válido"* |
| Pones el correo de otro usuario (`admin@ecommerce.com`) | *"Ese correo ya está en uso por otro usuario"*, porque el servidor respondió **409** |
| Todo está bien | *"Perfil actualizado exitosamente"* |

---

### P7: mensajes claros

Igual que en Roles, ahora hay mensajes para cada caso: **401** (sesión vencida), **409** (correo
repetido) y cuando no hay sesión. Además, mientras carga el perfil aparece una **barra de progreso**
arriba.

---

### P8: cambiar contraseña

**Problema 1, el mensaje equivocado.**
- **Lo que hace el servidor:** si escribes mal tu contraseña actual, responde **400**. Deja el 401
  para "sesión vencida".
- **Lo que hacía la app:** esperaba 401, así que con una contraseña equivocada mostraba un error
  genérico.

```kotlin
// AHORA
400 -> "La contraseña actual es incorrecta"
401 -> "Tu sesión expiró o no es válida…"
```

**Problema 2, la ventana no se cerraba.**
- **Qué pasaba:** la app revisaba si todo salió bien **justo después** de enviar la petición. Pero
  la petición **tarda**: todavía no había respuesta, así que nunca cerraba la ventana.
- **Ejemplo:** es como preguntar *"¿ya llegó la pizza?"* un segundo después de pedirla.
- **Cómo se arregló:** ahora la ventana se cierra **cuando llega la respuesta**, con una "bandera"
  `passwordChanged`.

✅ **Qué debes ver:**
- Con la contraseña actual equivocada: *"La contraseña actual es incorrecta"*.
- Con todo correcto: *"Contraseña cambiada exitosamente"* y **la ventana se cierra sola**.

### Flujo completo de editar el perfil

```
1. Abres Mi Perfil ──► GET /countries ──► GET /users/profile ──► se leen los datos de "user"
2. Se preselecciona: GET /states?country_id=1 ──► GET /cities?state_id=1
3. Cambias datos ──► pulsas "Actualizar Perfil"
4. ViewModel valida: nombre, correo y ciudad   ── falla ──► mensaje en pantalla (no se envía nada)
                  │ bien
5. PUT /users/profile {"UserName", "Email", "iD_City"}
6. Servidor: ¿correo repetido? ── sí ──► 409 ► "Ese correo ya está en uso…"
                  │ no
7. 200 ► se leen los datos de "user" ► se actualiza la sesión (sin perder el rol) ► "Perfil actualizado"
```

---

## 6. Reto extra: carrito, pedido e historial

### ¿Qué se veía mal?

Esta era la secuencia en la terminal del backend:

```
POST /api/sales/checkout 201   ← ¡el pedido SÍ se creó!
POST /api/sales/checkout 400   ← el usuario reintenta: "Carrito vacío"
POST /api/sales/checkout 400   ← reintenta otra vez
```

Y en la app:
- **Al confirmar el pedido** salía *"Error de conexión"*, aunque **el pedido sí se había creado**.
- **Al reintentar** salía *"Datos de pedido inválidos"*.
- **El historial de pedidos** daba error.
- **Al agregar productos,** el contador del carrito volvía a **0**.
- **Al tocar + en el carrito,** la lista **desaparecía**.

### Lista de errores

| # | Error en palabras simples |
|---|---|
| C1 | El nombre del usuario del pedido llega como **texto** y la app esperaba un **objeto** |
| C2 | Por el C1, la app creía que el pedido falló, aunque se había creado |
| C3 | Al agregar un producto, la app **borraba** el carrito que tenía guardado |
| C4 | Lo mismo al cambiar la cantidad |
| C5 | El historial buscaba los datos con **nombres equivocados** |
| C6 | El detalle de un pedido llega dentro de `{"sale": {...}}` |
| C7 | El total de pedidos se buscaba en el lugar equivocado |

---

### C1: el usuario del pedido llega como texto (la causa principal)

Es **el mismo tipo de error que el Error 3 del Reto 1**:

```json
"sale": { "id_Sale": 2, "total": 999.99, "user": "juan_perez" }
```

La app esperaba `"user": { ... }` (un objeto) y llegaba `"user": "juan_perez"` (texto). Gson
fallaba con `Expected BEGIN_OBJECT but was STRING`.

```kotlin
// models/Cart.kt
// ANTES
val user: User? = null
// AHORA
val user: String? = null
```

### C2: el pedido fantasma

**Así se encadenaba el problema:**

```
1. Confirmas el pedido
2. Servidor: crea el pedido, descuenta el stock y marca el carrito como comprado ► responde 201
3. App: intenta leer la respuesta ► falla en "user" (C1) ► muestra "Error de conexión"
4. Tú crees que falló y reintentas
5. Servidor: ya no hay nada en tu carrito ► 400 "Carrito vacío"
6. App: muestra "Datos de pedido inválidos"
7. Abres el historial ► usa el mismo modelo con "user" ► también falla
```

**Ejemplo:** pagas en una tienda, la caja registra la venta, pero **se traba la impresora del
recibo**. Te dicen "error". Vuelves a pagar y te dicen "no tienes productos". Y cuando pides ver
tus compras, el sistema también se traba.

**Pregunta de la guía:** *¿el pedido se crea una sola vez?* **Sí.** El servidor **nunca duplicó**
pedidos. El problema estaba **solo en la app**.

**Cómo se arregló** (`viewmodels/CartViewModel.kt`):

```kotlin
// AHORA: si el servidor dijo 201, el pedido existe. Punto.
if (response.isSuccessful) {
    // limpiar el carrito en pantalla y abrir "Pedido exitoso"
    lastCompletedSale = checkoutResponse?.sale ?: Sale(DescripcionSale = request.DescripcionSale)
} else {
    // mostrar el mensaje REAL del servidor: "Carrito vacío", "Stock insuficiente para X"
    errorMessage = when (response.code()) {
        400 -> mensajeServidor ?: "No se pudo procesar el pedido"
        ...
    }
    loadCart()   // mostrar el carrito real que tiene el servidor
}
```

✅ **Qué debes ver al confirmar el pedido:**
- Se abre la pantalla **"Pedido exitoso"**.
- El carrito queda **vacío**.
- En la terminal aparece **un solo** `POST /api/sales/checkout 201`, sin los 400 de antes.

---

### C3 y C4: el carrito que se vaciaba solo

Al agregar un producto, el servidor solo responde:

```json
{ "message": "Producto agregado al carrito" }
```

**No manda el carrito.** Pero la app tomaba esa respuesta **como si fuera el carrito completo**.
Como no traía productos, **reemplazaba tu carrito real por uno vacío**, y **sin dar ningún error**.

**Ejemplo:** le preguntas a alguien *"¿qué hay en mi carrito?"* y te responde *"listo, agregué
tu producto"*. Si anotas esa respuesta como la lista del carrito… anotaste una lista vacía.

```kotlin
// ANTES
cart = responseBody          // guardaba {"message": "..."} como si fuera el carrito

// AHORA: después de agregar, pide el carrito real
if (response.isSuccessful) {
    successMessage = "Producto agregado al carrito"
    loadCart()               // GET /api/sales/cart → este SÍ devuelve el carrito completo
}
```

Lo mismo se hizo al **cambiar la cantidad** (C4), porque el servidor responde
`{"message": "...", "item": {...}}`, que tampoco es el carrito.

✅ **Qué debes ver:**
- Agregas un producto en el catálogo y el **número del carrito sube** (1, 2, 3…) y **se queda así**.
- En el carrito tocas **+** y la cantidad cambia, pero **la lista no desaparece**.
- En la terminal: `POST /cart/add 200` seguido de `GET /cart 200`.

---

### C5: el historial buscaba con otros nombres

El servidor usa unos nombres y la app buscaba otros. Como no los encontraba, mostraba ceros:

| La app buscaba | El servidor manda |
|---|---|
| `TotalSale` | `total` |
| `DateSale` | `DateCreated` |
| `sale_details` | `details` |
| `Quantity` | `amount` |
| `Subtotal` | `ValueSale` |
| `UnitPrice` | **no existe** |

```kotlin
// Ejemplo del arreglo: se cambia solo el nombre que se busca en el JSON
@SerializedName("total")        // antes: "TotalSale"
val TotalSale: Double = 0.0

// El precio unitario no viene: se CALCULA
val UnitPrice: Double
    get() = if (Quantity > 0) Subtotal / Quantity else 0.0
```

**Ejemplo del cálculo:** si compraste **2 unidades** y pagaste **$100.000** por esa línea, el
precio unitario es 100.000 ÷ 2 = **$50.000**.

---

### C6 y C7: el detalle del pedido y el total de pedidos

- **C6:** el detalle llega dentro de `{"sale": {...}}`. Es el mismo arreglo de "bolsa" de siempre:
  se creó `SaleResponse` y se lee `response.body()?.sale`.
- **C7:** el total de pedidos viene **dentro** de `"pagination"`, no afuera:
  ```json
  { "sales": [...], "pagination": { "page": 1, "pages": 1, "total": 2 } }
  ```
  ```kotlin
  totalSales = salesResponse.pagination?.total ?: sales.size
  ```

✅ **Qué debes ver en el Historial de pedidos:**
- Cada pedido muestra su **número**, **fecha**, **total real** (por ejemplo, *$999.99*) y la
  **cantidad de productos**.
- Al tocar un pedido se abre el **detalle** con cada producto: *"$50000.0 x 2"* y su subtotal.
- El contador de pedidos muestra el **número correcto**, no 0.

### Flujo completo de una compra

```
1. Catálogo ► "Agregar" ► POST /cart/add (200) ► GET /cart ► el contador sube
2. Carrito ► "+" ► PUT /cart/update/{id} (200) ► GET /cart ► la lista se actualiza
3. Carrito ► "Confirmar pedido" ► POST /checkout
      ├── 201 ► carrito vacío + pantalla "Pedido exitoso"
      └── 400 ► mensaje real del servidor ("Stock insuficiente para X") + recargar carrito
4. Historial ► GET /sales/ ► lista de pedidos con total, fecha y productos
5. Tocar un pedido ► GET /sales/{id} ► detalle con precio unitario × cantidad
```

---

## 7. Resumen final y la lección más importante

### ¿Cuántos errores se corrigieron?

| Reto | Errores |
|---|---|
| Reto 1: Ubicaciones | 8 |
| Reto 2A: Roles | 6 |
| Reto 2B: Perfil | 8 |
| Reto extra: Carrito, pedido e historial | 7 |
| **Total** | **29** |

### Los tres tipos de error que encontramos

| Tipo | Ejemplo | Dónde apareció |
|---|---|---|
| **La "bolsa" y la "caja":** el servidor envuelve los datos en un objeto | `{"countries": [...]}`, `{"user": {...}}` | Países, estados, ciudades, roles, perfil, detalle del pedido |
| **Texto donde se esperaba un objeto** | `"country": "Colombia"`, `"user": "juan_perez"` | Estados y pedidos |
| **Nombres distintos** | `pages` vs. `total_pages`, `total` vs. `TotalSale` | Paginación e historial |

Además se arreglaron cosas de **lógica y experiencia de usuario**:
- Las cascadas país → estado → ciudad.
- Los mensajes de "Cargando…" y reintentar.
- Las validaciones del formulario.
- Los permisos de administrador.
- La recarga del carrito.

### La lección más importante

> **"El código compila" no significa "el código funciona".**
>
> El compilador solo revisa que tú escribiste bien. **No sabe qué va a responder el servidor.**
> Para encontrar estos errores hay que **mirar las dos puntas**:
> - La **terminal del backend**, que dice si el servidor respondió bien (`200`).
> - **Logcat** en Android Studio, que dice si la app pudo leer la respuesta.
>
> Si el servidor dice `200` pero la pantalla está vacía, **el problema está en la app**: casi
> siempre el modelo de Kotlin no tiene la misma forma que el JSON.

### Cómo buscar los cambios en el código

Cada corrección tiene un comentario que empieza con `[ERROR …]` o `[PASO …]`. En Android Studio,
presiona `Ctrl + Shift + F` y busca, por ejemplo, `[ERROR U1]`. Te lleva directo a ese cambio,
con la explicación de cómo era antes y por qué se cambió.
