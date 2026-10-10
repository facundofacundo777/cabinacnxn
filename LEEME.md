# Cabina CNXN

App de reunión en vivo: checklist en tres fases, programa con timers, chat de áreas con flash y links compartibles.
Un solo archivo, `cabina.html`. Vanilla JS + Firebase Realtime Database.

---

## Probarlo ahora

Abrí `cabina.html` en el navegador. Sin configurar nada arranca en **modo demo**: los datos quedan
guardados en ese navegador y se sincronizan entre pestañas de la misma computadora.

Para entrar como productor, el PIN del modo demo es **1234** (se cambia en `CFG.pinDemo`).

---

## Ponerlo online

Es una página estática: sirve cualquier hosting. Subí `cabina.html` como `index.html` de la carpeta
pública, idealmente en un subdominio corto tipo `cabina.tudominio.com`.

### GitHub Pages

Anda perfecto. La limitación que tenía GitHub —que una página HTTPS no puede hablar con equipos de la
red local— era solo para las integraciones con ProPresenter y OBS, que quedaron fuera de alcance.

Pasos: repo nuevo → subir el archivo como `index.html` → Settings → Pages → Source: rama `main`,
carpeta raíz → queda en `usuario.github.io/repo/`. Para usar tu dominio, agregá un archivo `CNAME` con
`cabina.tudominio.com` adentro y apuntá el DNS a GitHub.

**Lo único a tener en cuenta:** con cuenta gratuita, GitHub Pages exige que el repo sea **público**. Eso
deja tu `firebaseConfig`, tu mail de admin y el alias de los links permanentes visibles y —esto es lo que
importa— **buscables**: hay gente que escanea GitHub buscando configs de Firebase. Cualquier página web
expone su config a quien abra el código fuente, así que el problema no es la exposición en sí: es que un
repo público es indexable.

Con las reglas de seguridad de arriba puestas, el daño posible es acotado: alguien que junte todo podría
entrar a la reunión activa y marcar ítems o escribir en el chat. No puede listar tus reuniones, ni
cambiar cuál está activa, ni borrar nada.

Tres formas de bajar el riesgo, de menor a mayor esfuerzo:

1. **Cambiá el alias** (`CFG.alias`) por algo que no sea el nombre de la iglesia. Que no se adivine.
2. **Activá App Check** en Firebase con reCAPTCHA: rechaza los pedidos que no vengan de tu dominio. Es el
   arreglo real y son unos 15 minutos.
3. **Repo privado** —en GitHub Pages requiere plan pago— o tu propio hosting, que no es más seguro pero
   tampoco es indexable.

Para un tablero de producción de una iglesia, la opción 1 sola ya es razonable. Si en algún momento
metés datos de personas, hacé la 2.

---

## Conectarlo a Firebase (necesario para que funcione entre celulares)

1. Entrá a `console.firebase.google.com` y creá un proyecto.
2. Activá **Realtime Database** (no Firestore) y, en **Authentication**, dos proveedores:
   **Google** (para el admin) y **Anónimo** (para todos los demás — es invisible, nadie ve un login).
3. En Configuración del proyecto → Tus apps → Web, copiá el objeto `firebaseConfig`.
4. Pegalo en el bloque `CFG.firebase` de `cabina.html`.
5. Poné tu mail en `CFG.admins`.
6. En Authentication → Settings → Dominios autorizados, agregá tu dominio.

### Reglas de seguridad

No las escribas a mano: en el panel de admin, el botón **Reglas de Firebase** te muestra las reglas
con los mails de `CFG.admins` ya adentro, y un botón para copiarlas. Pegalas en
Realtime Database → Reglas. Quedan así (con tu mail):

```json
{
  "rules": {
    "salas": {
      ".read": "auth != null && auth.token.email_verified == true && (auth.token.email == 'facundo.chiniecc@gmail.com')",
      ".indexOn": "inicioMs",
      "$sala": {
        ".read": "auth != null",
        ".write": "auth != null && $sala.length >= 12"
      }
    },
    "activa": {
      ".read": "auth != null",
      "$alias": {
        ".write": "auth != null && auth.token.email_verified == true && (auth.token.email == 'facundo.chiniecc@gmail.com')"
      }
    }
  }
}
```

Qué hacen, en tres niveles:

- **Todo requiere un token.** La app entra sola con ingreso anónimo, así que el voluntario no ve ningún
  login — pero nadie puede leer ni escribir la base desde afuera de la app con un `curl`.
- **Dentro de la app, el link es la llave.** Quien conozca el id de una sala la puede leer y escribir,
  igual que en Stagetimer: es lo que permite que un voluntario entre sin cuenta. Los ids tienen 12
  caracteres al azar, así que no se adivinan.
- **La lista de reuniones y el botón Activar son solo para los mails de `CFG.admins`.** No alcanza con
  cualquier cuenta de Google: la regla compara el mail. Para sumar un productor, agregá su mail en
  `CFG.admins`, volvé a copiar las reglas desde la app y pegalas de nuevo.

Consecuencia práctica: no pongas nada sensible en los ítems del checklist.

**Sobre la `apiKey` de Firebase:** no es un secreto. Va en el código de todas las apps web de Firebase y
se ve desde el navegador de cualquiera que abra la página. Lo que protege los datos son las reglas.

---

## Las vistas

Cada reunión genera seis links, desde el botón **Links** del panel de admin:

| Vista | Ruta | Para quién |
|---|---|---|
| Consola | `#/c/SALA` | **Todo el equipo.** Tres columnas para monitor grande. Los controles aparecen solo para el productor. |
| Productor | `#/r/SALA` | Lo mismo en una columna, para la mano. |
| Área | `#/a/SALA` | Versión simple para el voluntario: su checklist + chat. |
| Cabina | `#/v/SALA/cabina` | TV de producción. Solo lectura. |
| Escenario | `#/v/SALA/escenario` | Monitor de confianza. Timer gigante. |
| Púlpito | `#/v/SALA/pulpito` | El que predica. Timer + semáforo + avisos. |
| Seguimiento | `#/v/SALA/live` | Para mirar el progreso desde afuera. |
| Reporte | `#/reporte/SALA` | Después de cerrar: planificado vs real por sección, ítems sin cerrar y cerrados tarde. Imprimible. |

Si un voluntario abre la raíz del sitio (sin `#/...`), no ve el panel del productor: ve una pantalla con
dos botones, "Entrar a la reunión de hoy" y "Vista simple". El panel solo aparece para los mails de
`CFG.admins`.

### Links permanentes — esto es lo que hace que el equipo lo use

Los links de arriba llevan `SALA`, que cambia con cada reunión. Repartirlos todos los domingos es la
forma más rápida de que nadie los use.

Por eso existe el **alias**: `CFG.alias` (viene como `"cnxn"`). Los links con el alias en lugar de la
sala apuntan siempre a la reunión que el admin marcó como **Activa**:

```
tudominio.com/#/c/cnxn              ← consola del equipo
tudominio.com/#/a/cnxn              ← vista simple
tudominio.com/#/v/cnxn/cabina       ← TV de cabina
tudominio.com/#/v/cnxn/escenario    ← monitor de confianza
tudominio.com/#/v/cnxn/pulpito      ← celular del predicador
tudominio.com/#/v/cnxn/live         ← seguimiento desde afuera
```

Estos se reparten **una sola vez**. Cada domingo el productor toca **Activar** en la reunión del día y
todas las pantallas que ya estaban abiertas se enganchan solas, sin recargar nada.

En el panel de admin, el botón **Cartel de links para el equipo** abre una hoja lista para imprimir y
pegar en la cabina.

Las tres pantallas de sala (cabina, escenario, púlpito) no piden identidad: se abren y listo.
Además piden *wake lock*, así que la pantalla no se apaga sola.

**La consola sirve para todo el equipo, no solo para el productor.** Es el mismo link para todos
(`#/c/SALA`) y la vista se adapta a quién entra:

- **Productor**: los botones de control (iniciar, avanzar, ajustar, cerrar), los avisos al púlpito y
  las notas editables. Ve el checklist completo.
- **Cualquier otra área**: la misma pantalla sin los controles, con una tarjeta de su propia área y su
  progreso, las notas del productor en modo lectura, y el checklist filtrado a lo suyo. El botón de la
  derecha de las pestañas cambia entre **su área** y **TODO** cuando quiere ver el panorama completo.

Nadie puede arrancar ni avanzar la reunión salvo el productor, aunque abra el link de la consola.

**Consola y vista simple son la misma información con dos layouts.** La consola está pensada para un
monitor de 24" o más; la vista simple (`#/r/SALA` para el productor, `#/a/SALA` para el resto) es la
misma reunión en una columna, con botones grandes, para el celular. Se salta de una a la otra con el
botón ▦ / ▤ de la barra superior.

---

## Configurar

Todo está en el bloque `CFG` arriba del archivo:

- `CFG.areas` — para sumar un área, agregá una línea con `id`, `nombre` y `color`.
- `CFG.rapidos` — los botones de mensaje rápido del chat.
- `CFG.avisosPulpito` — los avisos que el productor le manda al predicador.
- `CFG.semaforo` — a partir de cuántos minutos de desvío pasa a ámbar y a rojo.

## El chat

Tiene una lista de canales a la izquierda: **Todos** más un canal por área. El canal elegido define a
quién le estás hablando — no hace falta elegir destinatario aparte. Cada canal muestra un contador de
mensajes sin leer.

Un mensaje dirigido a un área hace **flash** solo en las pantallas de esa área; uno a Todos flashea en
todas menos en el púlpito, que nunca ve el chat. El botón **✦** de la barra superior dispara un flash de
prueba para que veas cómo se ve antes del domingo.

Detalle importante: la identidad de sesión es **por pestaña**, no por navegador. Si fuera por navegador,
dos pestañas de la misma máquina se verían como la misma persona y el flash se silenciaría solo.

## Notas del productor

Cada reunión tiene un bloc de notas libre ("el bajo retumbaba el domingo pasado, ojo con el bombo") y una
lista de recordatorios sueltos con checkbox. Es aparte del checklist a propósito: el checklist es el
proceso estandarizado, las notas son lo que pasó esta semana.

## Bloques del programa

Las secciones se agrupan en bloques (APERTURA, WORSHIP, ANUNCIOS, PALABRA, CIERRE). El bloque es solo un
texto: se edita por sección en el panel de admin, y las secciones seguidas con el mismo nombre se agrupan
bajo un mismo encabezado. Poné el nombre que uses vos.

## Plantillas

Están en `PLANTILLAS`, abajo de `CFG`. Vienen tres: domingo normal, ensayo y evento especial.
**Los ítems de ejemplo están inventados — reemplazalos por los reales de CNXN.**

- `secciones`: `{ bloque, titulo, min, area, esPredica }`
- `pre` y `post`: `{ texto, off, area, critico }` — `off` en minutos, negativo antes del inicio, positivo después del cierre
- `vivo`: `{ texto, sec, area, critico }` — `sec` es el índice de la sección (0 es la primera)

---

## Cómo funciona el tiempo

- **Los timers no viajan por la red.** Se transmite cuándo arrancó cada sección y cuánto dura;
  cada pantalla calcula el número con su propio reloj, corregido contra la hora del servidor
  (`.info/serverTimeOffset`). Por eso el que mira desde su casa ve exactamente lo mismo que la cabina,
  y si se corta internet los timers siguen corriendo.
- **Desvío** = a qué hora arrancó realmente la sección actual, menos la hora que le tocaba según el plan.
- **Fin proyectado** = ahora + lo que falta de las secciones restantes.

---

## Lo que falta / lo que no hace

- No se conecta con ProPresenter, OBS ni Ableton. Eso queda para más adelante y necesita
  un programa corriendo en la máquina de cabina.
- El chat es por reunión: se archiva con ella, no hay historial entre semana.
- El "quién marcó" es declarado, no verificado. Para un equipo de iglesia alcanza.
- **La vibración no funciona en iPhone** — Safari no la permite desde una web. Por eso el aviso
  principal es el flash de pantalla, no la vibración.


---

## Puesta en marcha, paso a paso

### 1. Firebase (una vez, ~10 minutos)

Sin esto cada dispositivo ve sus propios datos, así que es el primer paso obligatorio.
Está detallado más arriba: crear el proyecto, activar Realtime Database y Google Auth, pegar el
`firebaseConfig` en `CFG.firebase`, poner tu mail en `CFG.admins`, y aplicar las reglas de seguridad.

### 2. Subirlo (una vez)

Tu hosting o GitHub Pages, los dos andan — mirá **Ponerlo online** para lo que hay que tener en cuenta
con un repo público. Idealmente como `index.html` de un subdominio corto. Después, en Firebase →
Authentication → Settings → Dominios autorizados, agregá ese dominio (o `usuario.github.io`) o el login
de admin no va a andar.

### 3. Cargar el domingo real (una vez, después se duplica)

En `PLANTILLAS`, reemplazá los ítems de ejemplo por los de CNXN: los horarios reales del armado, las
secciones del programa con sus duraciones, y las tareas de cada área. **Este es el paso que decide si la
app sirve o no.** Los ítems inventados hacen que el equipo la ignore.

### 4. Repartir los links (una vez)

Creá la reunión del domingo, tocá **Activar**, abrí **Cartel de links para el equipo** e imprimilo para
la cabina. Mandá por WhatsApp el link de la consola a cada área. Cada uno lo abre, pone su nombre, elige
su área, y no se lo vuelve a pedir en ese teléfono.

### 5. El primer domingo

- Dejá abierto el link de cabina en la TV y el de escenario en el monitor de confianza.
- **Corré la app en paralelo con como lo hacen hoy.** No saques el método viejo el primer domingo.
- Que una sola persona la maneje: el productor. El resto solo marca lo suyo.
- Al terminar, mirá qué quedó sin cerrar. Eso te dice qué ítems están mal escritos o de más.

### 6. Cada semana

Duplicar el domingo anterior, ajustar el programa, tocar **Activar**. Dos minutos.

Y antes de duplicar, abrí el **Reporte** del domingo pasado (botón en la lista de reuniones): te dice
qué secciones se fueron de tiempo, qué ítems nadie cerró y cuáles se cerraron tarde. Eso es lo que
hay que corregir en la plantilla — un ítem que tres domingos seguidos queda sin cerrar está mal
escrito, mal asignado, o sobra.

---

## Errores frecuentes al ponerlo en marcha

- **"No me deja entrar como admin"** — falta agregar tu dominio en Authentication → Settings →
  Dominios autorizados, o tu mail no está en `CFG.admins`.
- **"Falta activar el ingreso anónimo en Firebase"** — en Authentication → Sign-in method habilitá el
  proveedor **Anónimo**. Sin eso nadie puede leer ni escribir nada.
- **"Cada uno ve cosas distintas"** — está corriendo en modo demo: falta el `firebaseConfig`.
- **"Permission denied" en la consola del navegador** — faltan las reglas de seguridad.
- **"El link permanente dice que no hay reunión activa"** — falta tocar **Activar** en la reunión.
- **"No flashea"** — el mensaje fue dirigido a otra área, o lo mandaste desde la misma pestaña.
  Probá con el botón ✦.
- **Aparece "Sin conexión" arriba** — se cortó el wifi o la conexión con Firebase. La pantalla sigue
  mostrando lo último que recibió y los timers siguen corriendo; se reconecta sola. Si queda más de un
  minuto, revisá el wifi de la cabina.
- **"Sin permiso en la base"** — las reglas no están puestas, o tu mail no está en `CFG.admins` y
  estás intentando ver la lista de reuniones.
