# Trazo de Cargue TOPTEC

Herramienta web para planear el cargue de vehículos: plano, vista 3D, modo muelle, fotos y hoja LS-RG-09.
Se publica en **GitHub Pages** y guarda usuarios y datos en **Firebase** (Authentication + Firestore), con la **misma lógica de acceso del tablero de Gastos**.

## Archivos

| Archivo | Para qué sirve |
|---|---|
| `index.html` | La aplicación completa. La configuración de Firebase y del sitio (correo de contacto del admin, minutos de inactividad) está **dentro**, en el bloque `CONFIGURACIÓN` (busca `firebaseConfig`), igual que en Gastos |
| `firestore.rules` | Reglas de seguridad: qué puede hacer cada rol |

`firebase-config.js` ya no se usa (la configuración quedó dentro de `index.html`); se puede borrar.

Plan gratuito (Spark): no se usan Cloud Functions ni Storage.

## Cómo funciona el acceso (igual que Gastos)

- Antes de ver cualquier cosa aparece la **pantalla de ingreso** (correo empresarial + contraseña).
- Solo entran cuentas **creadas por el administrador** que existan y estén **activas** en la colección `users` de Firestore. Si no, aparece "Sin acceso".
- La sesión se **cierra sola tras 30 minutos sin actividad** (avisa 2 minutos antes).
- **Panel admin** (botón ⚙ arriba, solo lo ve el administrador): maestro de ítems y parámetros, restricciones por ítem, estibas y usuarios.
- **Contraseñas**: solo las maneja el administrador.
  - Al crear un usuario le pone una contraseña (o pulsa **Generar**) y se la entrega por un canal privado.
  - Para cambiarla: **Editar** el usuario → escribir su **contraseña actual** (la que le asignó) y la **nueva** → Guardar.
  - Si no recuerda la actual: borrar la cuenta en Firebase → Authentication → Usuarios, y en la herramienta guardar el usuario con la contraseña nueva y la actual vacía (se crea de nuevo).
  - El administrador cambia la suya con **Cambiar contraseña** en el encabezado. Los operarios ven a quién escribirle.

| Rol | Puede |
|---|---|
| **Administrador** | Todo: maestro de ítems, restricciones (máximos por arrume), tipos de estiba, parámetros, usuarios; aprobar y eliminar trazos |
| **Operario** | Cargar órdenes, calcular y guardar trazos en borrador, ver todos los trazos, hacer el modo muelle y subir fotos |
| **Inactivo** (casilla "Activo" desmarcada) | No puede entrar |

Flujo de un trazo: **Borrador** → **Aprobado** (solo admin) → **Cargado** (al terminar el modo muelle). Si un operario cambia un trazo aprobado, se guarda como uno **nuevo** en borrador.

---

## Montaje paso a paso (una sola vez)

### 1. Firebase
1. Proyecto `trazo-cargue` en <https://console.firebase.google.com>.
2. **Authentication** → Método de acceso → habilita **Correo electrónico/contraseña**.
3. **Firestore** → la base de datos del proyecto se llama **`trazo-cargue`** (no es la `(default)`). Si algún día se cambia, ajusta `FIRESTORE_DB` en el bloque `CONFIGURACIÓN` de `index.html`. Las reglas y los datos se manejan dentro de esa base de datos.
4. `index.html` ya tiene la configuración del proyecto (bloque `CONFIGURACIÓN`). **Cuidado si se vuelve a copiar la `apiKey`**: tiene `FLlQ` (L mayúscula y luego l minúscula); si se cambia una letra, nadie puede entrar ("API key not valid").

### 2. Reglas de seguridad
Firestore Database → **Reglas** → borra todo, pega el contenido de `firestore.rules` → **Publicar**.

### 3. Primer administrador (a mano, solo esta vez)
1. **Authentication → Usuarios → Agregar usuario**: tu correo y una contraseña.
2. **Firestore Database → Datos → Iniciar colección** `users`
   - **ID del documento**: tu correo **en minúsculas** (ej. `epatino@toptec.com.co`). No uses "ID automático".
   - Campos:

     | Campo | Tipo | Valor |
     |---|---|---|
     | `email` | string | tu correo en minúsculas (igual al ID) |
     | `name` | string | tu nombre |
     | `role` | string | `admin` |
     | `active` | **boolean** | `true` |

> Si antes creaste la colección `usuarios` (con UID y `rol`), ya no se usa: puedes borrarla.

Desde ahí, los demás usuarios se crean en la herramienta: **⚙ Panel admin → Usuarios**.

### 4. GitHub Pages
1. Sube `index.html`, `firestore.rules` y este `README.md` a la raíz del repositorio (reemplaza los anteriores; si había `firebase-config.js`, una carpeta `functions/`, `firebase.json` o `.firebaserc`, bórralos: ya no se usan).
2. **Settings → Pages** → rama `main`, carpeta `/ (root)` → Save.
3. Queda en `https://epatinos18.github.io/<repositorio>/`.

### 5. Dominio autorizado
Authentication → Configuración → **Dominios autorizados**: debe estar `epatinos18.github.io` (ya está).

### 6. Primer ingreso
Entra con tu usuario admin. En ese primer ingreso la herramienta sube a Firestore el maestro de ítems (1.042 ítems) y la configuración. Luego, en **⚙ Panel admin**: crea los tipos de estiba, carga las restricciones y crea los operarios.

---

## Uso diario
- **Nuevo** → cargar la orden (Excel de SAP o a mano) → revisar el plano → **Guardar**.
- **Trazos guardados**: búsqueda por placa, pedido, cliente o conductor; filtro por estado.
- El admin abre el borrador, lo ajusta si hace falta y **Aprueba** (se guarda solo).
- En el muelle, el operario abre el trazo en el celular (enlace directo `…/#t=<id>`), usa **Modo muelle** y sube las fotos. El avance se guarda solo.

## Si algo falla
| Mensaje | Causa |
|---|---|
| "Correo o contraseña incorrectos" | Datos mal escritos, o la cuenta no existe en Authentication |
| "No se pudo verificar tu acceso · the client is offline" | `FIRESTORE_DB` no coincide con el nombre de la base de datos de Firestore, o la red bloquea Firestore |
| "Sin acceso" | No existe `users/{correo}` en Firestore, el ID no está en minúsculas, `active` no es `true` (boolean) o `role` no es `admin`/`operario` |
| "No tienes permiso…" / "Sin permiso para guardar" | Las reglas de `firestore.rules` no están publicadas |
| La página no carga el ingreso o dice "API key not valid" (consola del navegador) | `apiKey` mal copiada en el bloque `CONFIGURACIÓN` de `index.html` |
| "Activa Correo electrónico/contraseña…" | Falta habilitar el método de acceso en Authentication |
