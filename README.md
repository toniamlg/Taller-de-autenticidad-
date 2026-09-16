# Taller de Autenticación conFastAPI

### Arquitectura y Estructura del Proyecto

**1.** ¿Cuál es la responsabilidad específica de cada archivo Python (`main.py`, `rutas.py`, `seguridad.py`, `usuarios.py`) y por qué se considera una buena práctica separarlos de esta manera?

**main.py:** es el punto de entrada. Solo crea la instancia de FastAPI() y le "engancha" el router con app.include_router(router). No define ninguna lógica propia.

**rutas.py** define las páginas/endpoints (/login, /logout, /, /perfil, /objetos). Cada función decide qué plantilla mostrar o a dónde redirigir.

**seguridad.py:** contiene toda la lógica de sesiones — crear token, guardarlo, comprobar credenciales y la dependencia que protege rutas privadas.

**usuario.py:** contiene los datos (el "modelo" Usuario con Pydantic y los diccionarios usuarios_db / items_db).

---

**2.** ¿Qué significa "separación de responsabilidades" en el contexto de este proyecto y cómo se manifiesta en la estructura de archivos?

Es el principio de que cada módulo debe encargarse de una sola capa del problema: entrada de la app (main.py), presentación/rutas (rutas.py), seguridad/autenticación (seguridad.py) y datos (usuarios.py). Se manifiesta literalmente en la organización de archivos: no hay lógica de sesión mezclada dentro de las rutas, ni datos de usuario mezclados dentro de la seguridad. Cada import (from seguridad import ..., from usuarios import ...) hace explícita esa frontera entre capas.

---
**3.** ¿Por qué el archivo `main.py` es tan breve (apenas 29 líneas) y qué indica esto sobre el diseño de la aplicación?

Es corto porque solo coordina y ensambla los componentes. Indica un diseño modular y limpio, delegando la lógica específica a otros módulos.

---
## Mecanismo de Autenticación y Sesiones

**4.** Explica el concepto de "sesión" tal como se implementa en este proyecto. ¿Por qué HTTP necesita sesiones si cada petición es independiente?

Es un mecanismo para mantener el estado de un usuario entre diferentes peticiones. Como HTTP es un protocolo "sin estado" (stateless), cada petición es independiente; la sesión permite que el servidor reconozca al usuario sin pedirle la contraseña en cada clic.

**5.** ¿Qué es el token de sesión, cómo se genera específicamente en el código (`secrets.token_hex(16)`) y por qué es importante que sea aleatorio e impredecible?

Es una cadena única que identifica la sesión activa de un usuario. Se genera con secrets.token_hex(16) creando un valor hexadecimal aleatorio de 16 bytes (32 caracteres). Es vital que sea aleatorio e impredecible para evitar que un atacante adivine un token válido e suplante a un usuario.

**6.** Describe el flujo completo cuando un usuario ingresa credenciales válidas: desde el POST al `/login` hasta que se redirige a la página home (menciona los "PASOS" del diagrama).

**Paso 1:** El usuario envía sus datos mediante `POST /login`.

**Paso 2:** Se verifica el usuario y contraseña en `usuarios.py`.

**Paso 3:** Se genera un token de sesión único con `secrets.token_hex(16)`.

**Paso 4:** Se guarda el token y el ID del usuario en el archivo de sesiones `(sesiones.json)`.

**Paso 5:** Se responde con una cookie conteniendo el token y una redirección `303` hacia `/`.

**7.** ¿Qué diferencia hay entre las rutas públicas (`/login`, `/logout`) y las privadas (`/`, `/perfil`, `/objetos`)? ¿Por qué el login debe ser necesariamente público?

Las rutas públicas (/login, /logout) son accesibles por cualquiera sin autenticación. Las privadas (/, /perfil, etc.) exigen una sesión válida. El /login debe ser público porque de lo contrario un usuario no autenticado no podría ingresar para identificarse.

**8.** ¿Cómo funciona la dependencia `UsuarioDep` en FastAPI y qué hace exactamente la función `get_current_user` cuando se ejecuta en cada ruta protegida?

UsuarioDep inyecta la función get_current_user usando Depends() de FastAPI. Al ejecutarse en una ruta protegida, extrae la cookie de la petición, consulta el token en la persistencia de sesiones y devuelve los datos del usuario; si el token no existe o expiró, redirige al /login.

---

## Cookies y Gestión de Estado

**9.** ¿Cuál es el propósito de la cookie `COOKIE_SESION` y qué parámetros se configuran cuando se establece (`max_age=VIDA_SESION_SEGUNDOS`)? ¿Qué pasaría si no se configurara `max_age`?

Almacenar el token de sesión en el navegador del cliente.

Parámetros: `max_age=VIDA_SESION_SEGUNDOS` establece el tiempo de vida en segundos.

Sin `max_age`: La cookie se convierte en una cookie de sesión del navegador y se elimina automáticamente al cerrar la ventana/navegador.

**10.** El proyecto guarda las sesiones en un archivo `sesiones.json` en disco. Explica el flujo de lectura y escritura de este archivo: ¿cuándo se lee, cuándo se escribe y qué sucede si el disco es de solo lectura (como en Vercel)?

- **Lectura:** Se lee del disco al verificar una sesión existente.

- **Escritura:** Se escribe al crear una nueva sesión (login) o destruirla (logout).

- En entorno de solo lectura (Vercel): Intentar escribir el archivo lanzará un error de sistema de archivos (OSError / Read-only file system) o fallará en tiempo de ejecución al no poder persistir cambios.

**11.** Compara el comportamiento de las sesiones en desarrollo local vs. despliegue en Vercel. ¿Por qué en Vercel las sesiones "se pierden" cuando la función se enfría (cold start)?

En local, el servidor corre continuo y escribe en el disco local. En Vercel (arquitectura serverless), las funciones son efímeras y el disco es de solo lectura/temporal. En un cold start (reinicio de la función), el entorno se destruye y recrea, perdiendo las sesiones guardadas en memoria o archivos locales.

**Curiosidad.**: Por qué se les llama cookies en el mundo de la informática a esos fragmentos de texto?

por la analogía con los magic cookies de Unix (paquetes de datos que un programa recibe y devuelve intactos), inspirados en las galletas de la suerte que contienen un mensaje corto en su interior.

---

## Seguridad y Mejores Prácticas

**12.** El README advierte que este proyecto **deliberadamente no usa** OAuth2, JWT ni hashes de contraseña. Explica qué son estos tres conceptos y por qué NO se usan en este proyecto didáctico.

- OAuth2: Estándar de autorización para delegar acceso (ej. "Iniciar sesión con Google").

- JWT (JSON Web Tokens): Tokens firmados criptográficamente que llevan información codificada sin requerir consultar una base de datos central en cada petición.

- Hashes de contraseña: Algoritmos unidireccionales (ej. bcrypt) para almacenar contraseñas transformadas sin guardarlas en texto plano.

- Por qué no se usan: Para no añadir complejidad conceptual y enfocar la lección en la lógica básica de sesiones y cookies HTTP.

**13.** Las contraseñas en `usuarios.py` están en **texto plano** (`"password": "1234"`). ¿Qué riesgo de seguridad representa esto en producción y qué solución se usaría en un proyecto real?

**Riesgo:** Si la base de datos o el código se filtran, los atacantes obtienen inmediatamente todas las contraseñas.

**Solución real:** Aplicar un algoritmo de hashing seguro con salt (como bcrypt, Argon2 o pbkdf2) antes de almacenar la contraseña. 

**14.** ¿Qué es el "sesion hijacking" (secuestro de sesión) y qué medidas adicionales podrían implementarse para prevenirlo que este proyecto no incluye?
Es la interceptación o robo de una cookie de sesión por parte de un atacante para suplantar a la víctima.

Medidas preventivas no incluidas: Usar atributos Secure (solo HTTPS) y HttpOnly (previene acceso vía JS) en la cookie, habilitar SameSite=Strict, regenerar el ID de sesión al cambiar de privilegio y vincular la sesión a la dirección IP/User-Agent del cliente.

---

## Flujo HTTP y Códigos de Estado

**15.** El código usa `status_code=303` en las redirecciones después del login y logout. ¿Por qué se usa específicamente 303 (See Other) en lugar de 302 (Found) o 301 (Moved Permanently)?

Se utiliza deliberadamente tras procesar una petición `POST` para forzar al navegador a realizar la redirección mediante un método `GET`. Evita el problema del reenvío de formularios al actualizar la página (el patrón PRG: Post/Redirect/Get).

**16.** Explica la diferencia semántica entre usar `GET /login` (mostrar formulario) y `POST /login` (enviar credenciales). ¿Por qué no se envían las credenciales por GET?

