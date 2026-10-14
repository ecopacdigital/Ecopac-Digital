# Plan de implementacion - Ecopac Digital

Como se pone el sistema en produccion, a nombre de la organizacion, y que quedo creado. Es el
documento de la issue #252, seccion 7. Se escribio **antes** de ejecutar y se actualiza con lo que
de verdad paso: la seccion 9, "Registro de ejecucion", dice que se hizo, cuando y como se comprobo.

**Ningun valor secreto se escribe aqui.** Este documento dice que se configuro, donde y de donde
sale cada valor; nunca el valor. Las credenciales se entregan por el canal de la seccion 14.

| | |
| --- | --- |
| Version | 1.0 |
| Fecha | 13 de octubre de 2026 |
| Issue | #252 (repositorio del curso, `URL-ECOPAC/Ecopac-Digital`) |

---

## 1. Objetivo y alcance

Dejar el sistema funcionando en produccion con **todo a nombre de la organizacion**, de forma que
no dependa de ninguna cuenta personal del equipo cuando el equipo se retire.

Se pone en produccion:

| Pieza | Donde vive | Como se despliega |
| --- | --- | --- |
| Web (panel de administracion) | Vercel, cuenta Hobby de la organizacion | Integracion Git de Vercel: `main` a produccion, `develop` y los PR a preview |
| Base de datos, Auth y Edge Functions | Supabase, organizacion de la cuenta de Ecopac: `ecopac-prod` y `ecopac-dev` | Workflow `supabase.yml`: `db push` y `functions deploy` en cada push a `main` / `develop` |
| Rutinas programadas (alertas de vencimiento, keep-alive, verificacion, respaldo) | GitHub Actions del repositorio nuevo | `schedule:` desde `main` |
| App movil (Android) | APK instalado a mano en los telefonos del personal | EAS Build, cuenta Expo de la organizacion |
| Respaldo de la base | Google Drive de la organizacion, cifrado con GPG | Workflow `respaldo-base.yml`, diario |
| Codigo fuente | Repositorio privado en la cuenta de GitHub de la organizacion | - |

Fuera de alcance: publicar en Play Store o App Store (cuesta dinero que la organizacion no tiene;
ver seccion 7), dominio propio (se queda en `*.vercel.app`), y migrar los datos del proyecto
`Ecopac-Digital-Prod` de la cuenta del equipo (no tiene datos reales; produccion arranca limpia).

## 2. Por que el repositorio se mueve de cuenta

El repositorio del curso vive en la **organizacion** de GitHub `URL-ECOPAC`. El plan Hobby de
Vercel no permite conectar un repositorio que pertenezca a una organizacion de GitHub
(`docs/COSTOS-Y-LIMITES.md`, seccion 6.1). En vez de pagar Vercel Pro o desplegar por CLI desde un
workflow, el repositorio se copia a la **cuenta personal de GitHub de la organizacion**, que si se
puede conectar a una cuenta Hobby de Vercel.

Eso implica que **todo lo que dependia del repositorio se crea de nuevo**: secrets, proyecto de
Vercel, proteccion de ramas, etiquetas. No hay traspaso de secrets entre repositorios; GitHub no
deja leerlos de vuelta.

## 3. Decisiones tomadas (13-10-2026)

| Decision | Eleccion | Por que |
| --- | --- | --- |
| Ambientes en la cuenta de la organizacion | `ecopac-prod` **y** `ecopac-dev` | El plan Free permite dos proyectos; el repositorio nuevo no depende de nada del equipo cuando se retire |
| Visibilidad del repositorio | **Privado** | Es un sistema clinico; 2,000 minutos de Actions al mes alcanzan para los crons y los PR |
| Correo (SMTP) | **Gmail de la organizacion**, `smtp.gmail.com:465`, contrasena de aplicacion | No hay dominio propio; Resend lo exige y Brevo con remitente `@gmail.com` cae en spam por DMARC. Limite ~500 correos/dia, el sistema manda unos pocos al mes |
| Dominio de la web | **`*.vercel.app`** | Gratis, HTTPS automatico. Cambiarlo despues es actualizar Site URL, redirects, `WEB_URL` y `ALLOWED_ORIGINS` |
| Seeds | **Fuera del repositorio.** Lo que usan las pruebas pasa a fixture (`supabase/fixtures/datos-de-prueba.sql`) | Los ambientes arrancan limpios. Las suites pgTAP y las e2e necesitan datos, y el job que las corre es la compuerta del despliegue |
| Historial de git | **Filtrado** con `git filter-repo`: se conservan los commits y se quitan de todos los seeds y los entregables | Si solo se borraran en el ultimo commit, seguirian en el historial al alcance de cualquiera con acceso |
| Donde se trabaja despues | **En el repositorio nuevo** | El del curso queda congelado con sus entregables; mantener dos repositorios con historiales distintos obligaria a copiar cada cambio a mano |
| Entregables del curso | **No van** (`docs/entregables/`) | El repositorio nuevo es de la organizacion: solo codigo y documentacion |
| Manuales en PDF (`docs/formal/`) | **Si van** | Son documentacion para la organizacion |
| Distribucion movil | **APK interno** | Play Store cobra 25 USD de registro; ver seccion 7 |

Decisiones que siguen abiertas y no bloquean la salida:

- **Identificador del paquete movil** (`org.ecopacguatemala.digital`, provisional en
  `apps/mobile/app.config.js`). Con APK interno pesa menos que en tienda, pero **cambiarlo despues
  obliga a desinstalar y reinstalar** la app en cada telefono. Conviene fijarlo antes del primer
  APK que se reparta.
- **Duracion de la sesion** (CFG-08, 3600 s provisional).
- **Plan de Supabase de produccion**: Free (sin respaldos propios de Supabase; el respaldo diario a
  Drive es el unico) o Pro (25 USD al mes, 7 dias de respaldos). Ver `docs/RESPALDO-Y-RECUPERACION.md`.

## 4. Prerrequisitos

### Cuentas de la organizacion

Todas creadas con el Gmail de la organizacion, con verificacion en dos pasos y el telefono de
recuperacion de la organizacion, no de un estudiante.

| Cuenta | Para que | Estado |
| --- | --- | --- |
| Gmail (`ecopac.digital@gmail.com`) | Registro de las demas cuentas, remitente de los correos del sistema, Drive de los respaldos, administrador inicial del sistema (migracion 00188) | Creada |
| GitHub (cuenta personal, no organizacion) | Duena del repositorio, secrets y workflows | Ver registro |
| Supabase | Organizacion duena de `ecopac-prod` y `ecopac-dev` | Ver registro |
| Vercel (Hobby) | Hosting de la web | Ver registro |
| Expo | Builds de EAS e identificador de la app | Ver registro |

### Accesos que el equipo necesita durante el aprovisionamiento

Tokens **de vida corta** (30 dias) generados desde las cuentas de la organizacion, guardados en un
archivo local fuera del repositorio y borrados al terminar. Se revocan en la entrega (seccion 14).

| Token | Donde se genera | Para que |
| --- | --- | --- |
| GitHub, token clasico con `repo` y `workflow` | Settings > Developer settings > Personal access tokens | Crear el repositorio, empujar el codigo, cargar secrets |
| Supabase, access token | Account > Access tokens | Crear los proyectos y configurarlos por la Management API |
| Vercel, token | Account Settings > Tokens | Crear el proyecto y sus variables |
| Gmail, contrasena de aplicacion | myaccount.google.com/apppasswords | SMTP de Auth y de las Edge Functions |

### Codigo

Cambios que el repositorio nuevo necesita y que entraron antes de la copia (rama
`chore/252-repo-y-cuentas-de-la-organizacion`):

- `supabase.yml` y `verificar-despliegue.yml`: **una contrasena de base por ambiente**
  (`SUPABASE_DB_PASSWORD_DEV` y `_PROD`), elegida por rama. Era el punto 2 de la #252.
- `alertas-vencimiento.yml` y `keep-alive-supabase.yml`: corren contra **los dos** ambientes en una
  matriz. Antes solo tocaban dev: produccion nunca habria recibido alertas ni pings.
- Seeds fuera; `supabase/fixtures/datos-de-prueba.sql` solo para el stack local y el CI.
- `apps/mobile/eas.json` con los perfiles `desarrollo` y `produccion`, los dos en APK.
- Documentacion: este plan, `docs/CI-CD.md`, `docs/DATOS-DE-PRUEBA.md`, `docs/COSTOS-Y-LIMITES.md`.

## 5. Pasos en orden

Cada paso dice que se hace, como se comprueba y como se vuelve atras si falla. La columna
"Estado" se llena en el registro (seccion 9).

### Paso 1. Crear los proyectos de Supabase

En la organizacion de Supabase de la cuenta de Ecopac, plan Free:

| Proyecto | Region | Contrasena de base |
| --- | --- | --- |
| `ecopac-prod` | `us-east-1` | Generada aleatoria, 32 caracteres, solo para este proyecto |
| `ecopac-dev` | `us-east-1` | Generada aleatoria, distinta de la de prod |

`us-east-1` es la region mas cercana a Guatemala por latencia de red entre las que ofrece Supabase.

- **Comprobacion:** `supabase projects list` muestra los dos en `ACTIVE_HEALTHY`.
- **Reversa:** borrar el proyecto desde Settings > General. No hay datos que perder.

### Paso 2. Cerrar el registro antes que nada

En cada proyecto nuevo, **antes de cualquier otra cosa**: el valor por defecto de Supabase deja el
registro publico abierto, y el rol por defecto de `perfiles` escribe pacientes y expedientes
(`docs/CONFIGURACION-SUPABASE.md`, "Orden de aprovisionamiento"). La migracion 00074 ya lo cierra
en la base, pero esa migracion todavia no se aplico.

- Authentication > Sign In / Providers > **Allow new users to sign up: desactivado** (CFG-03).
  Por API: `PATCH /v1/projects/{ref}/config/auth` con `disable_signup: true`.
- **Comprobacion:** `POST /auth/v1/signup` responde `422 signup_disabled`.

### Paso 3. Configurar Auth

Todo lo de `docs/CONFIGURACION-SUPABASE.md` (CFG-01 a CFG-13), con los valores de produccion de su
seccion "Lo que cambia en produccion". Se aplica por la Management API
(`PATCH /v1/projects/{ref}/config/auth`), que no deja nada a medias en un editor del Dashboard:

| ID | Ajuste | `ecopac-prod` | `ecopac-dev` |
| --- | --- | --- | --- |
| CFG-01 | Site URL | URL de produccion de Vercel | URL de la rama `develop` en Vercel |
| CFG-02 | Redirect URLs | `<web>/nueva-contrasena`, `<web>/nueva-contrasena?origen=invitacion`, `ecopac://recuperar`. **Sin comodines** | Las mismas con la URL de dev, mas `http://localhost:5173/nueva-contrasena`, su variante `?origen=invitacion` y `exp://*` |
| CFG-03 | Registro | Cerrado | Cerrado |
| CFG-04 | Proveedor de correo | Activado (no se toca) | Activado |
| CFG-05 | Confirm email | Activado | Activado |
| CFG-06 / 07 | Contrasena | Minimo 8, letras y digitos | Igual |
| CFG-08 / 09 | Sesion | JWT 3600 s, rotacion activada, reuso 10 s | Igual |
| CFG-10 | Rate limits | 30 / 30 / 150 por 5 minutos | Igual |
| CFG-11 | OTP | 3600 s | Igual |
| CFG-12 | Plantilla de recuperacion | `supabase/templates/recovery.html` desde `<!doctype html>`, asunto "Restablece tu contraseña de Ecopac Digital" | Igual |
| CFG-13 | SMTP de Auth | `smtp.gmail.com`, 465, usuario y contrasena de aplicacion del Gmail de la organizacion, remitente "Ecopac Digital" | Igual |

- **Comprobacion:** cada fila de la columna "Como se comprueba" de
  `docs/CONFIGURACION-SUPABASE.md`, y anotarla en su "Registro por ambiente".
- **Reversa:** el mismo `PATCH` con el valor anterior.

### Paso 4. Crear el repositorio en la cuenta de la organizacion

1. Repositorio **privado** `Ecopac-Digital` en la cuenta de GitHub de la organizacion, vacio.
2. Copia local del repositorio del curso, filtrada con `git filter-repo`, que quita de **todo el
   historial**:
   - `docs/entregables/`
   - `supabase/seed.sql`, `supabase/seed-demo.sql`, `supabase/seed-volumen.sql`
   - `scripts/recargar-datos-demo.sh` y `scripts/recargar-datos-demo/`
3. Se empujan `main` y `develop`. Ninguna otra rama: las de trabajo del curso se quedan alla.
4. Se copian las etiquetas (`type:*`, `module:*`, `priority:*`, `platform:*`) con
   `gh label clone`, porque los workflows abren issues con ellas.
5. **Antes** de que corra nada contra los proyectos, se cargan los secrets (paso 5). Si se empuja
   primero, los workflows avisan que faltan secrets y terminan en verde sin tocar ninguna base.

- **Comprobacion:** `git log --all -- docs/entregables supabase/seed-demo.sql` en un clon del
  repositorio nuevo no devuelve nada.
- **Reversa:** borrar el repositorio nuevo. El del curso no se toca.

### Paso 5. Secrets de GitHub

Diecisiete, en Settings > Secrets and variables > Actions del repositorio nuevo:

| Secret | De donde sale |
| --- | --- |
| `SUPABASE_ACCESS_TOKEN` | Cuenta de Supabase de la organizacion > Account > Access tokens. Uno dedicado al CI, **sin vencimiento corto** (si vence, deja de desplegar) |
| `SUPABASE_PROJECT_REF_PROD`, `SUPABASE_PROJECT_REF_DEV` | El `<ref>` de `https://<ref>.supabase.co` de cada proyecto |
| `SUPABASE_DB_PASSWORD_PROD`, `SUPABASE_DB_PASSWORD_DEV` | Las del paso 1 |
| `SUPABASE_URL_PROD`, `SUPABASE_URL_DEV` | `https://<ref>.supabase.co`, sin barra final |
| `SUPABASE_SERVICE_ROLE_KEY_PROD`, `SUPABASE_SERVICE_ROLE_KEY_DEV` | Project Settings > API Keys > la **`sb_secret_`** (no la legacy `service_role`) |
| `VITE_SUPABASE_URL_PROD`, `VITE_SUPABASE_URL_DEV` | Igual que `SUPABASE_URL_*` |
| `VITE_SUPABASE_ANON_KEY_PROD`, `VITE_SUPABASE_ANON_KEY_DEV` | Project Settings > API Keys > la publica (legacy `anon`, `eyJ...`) |
| `RESPALDO_GPG_CLAVE_PUBLICA` | Llave publica GPG de la organizacion (`docs/RESPALDO-Y-RECUPERACION.md`, seccion 7) |
| `RESPALDO_DRIVE_CLIENT_ID`, `RESPALDO_DRIVE_CLIENT_SECRET` | Cliente de OAuth del proyecto de Google Cloud de la organizacion |
| `RESPALDO_DRIVE_TOKEN` | Token de rclone de la cuenta de Drive de la organizacion |

**Por que la anonima legacy y no la `sb_publishable_`:** la web y la app todavia validan la llave
publica como JWT; migrar a `sb_publishable_` es trabajo de codigo pendiente
(`docs/CONFIGURACION-SUPABASE.md`). Por la misma razon **no se toca** "Disable JWT-based API keys".

- **Comprobacion:** `gh secret list` muestra los diecisiete.
- **Reversa:** `gh secret delete`.

### Paso 6. Primer despliegue de la base: el merge, no a mano

Con los secrets cargados, se relanza el workflow **Supabase** sobre `develop` y luego sobre `main`
(Actions > Supabase > Run workflow, o un push). El job **Aplicar migraciones**:

1. corre las 180 migraciones desde cero contra cada proyecto (`supabase db push`);
2. despliega las tres Edge Functions (`alertas-vencimiento`, `enviar-notificaciones`,
   `invitar-usuario`).

Nadie corre `supabase db push` desde su maquina (`AGENTS.md`, "Migraciones de base de datos").

- **Comprobacion:** `supabase migration list` muestra local y remoto iguales hasta la ultima; las
  tres funciones aparecen `ACTIVE` en `supabase functions list`.
- **Reversa:** si una migracion falla, `db push` es transaccional por archivo y se detiene ahi; se
  corrige con una migracion nueva hacia adelante. Si la base quedara inservible, se borra el
  proyecto y se vuelve al paso 1: todavia no hay datos reales.

### Paso 7. Configuracion que depende del esquema

Despues del paso 6, porque necesita las extensiones y las funciones:

| ID | Ajuste | Valor |
| --- | --- | --- |
| CFG-14 | Secret de funciones `ALLOWED_ORIGINS` | La URL de la web del ambiente (en dev, ademas `http://localhost:5173`) |
| CFG-15 | `WEB_URL` | La URL de la web del ambiente |
| CFG-16 | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | `smtp.gmail.com`, **465**, el Gmail, la contrasena de aplicacion, `Ecopac Digital <ecopac.digital@gmail.com>` |
| - | `AMBIENTE` | `production` / `development` (etiqueta de Sentry) |
| CFG-17 | Extensiones | `pg_net` y `supabase_vault` (las crea la 00138; se comprueba) |
| CFG-18 | Vault `notificaciones_url` | `https://<ref>.supabase.co/functions/v1/enviar-notificaciones` |
| CFG-19 | Vault `notificaciones_llave` | La `sb_secret_` del proyecto |

Los secrets de funciones con `supabase secrets set --project-ref <ref>`. Las entradas de Vault por
la Management API (`POST /v1/projects/{ref}/database/query`), que **no deja la llave en el
historial del SQL Editor**, el riesgo que advierte `docs/CONFIGURACION-SUPABASE.md`.

- **Comprobacion:** la prueba de punta a punta de CFG-19 (`net.http_post` y `200` en
  `net._http_response`); invitar a una cuenta de prueba y ver que llega el correo de **bienvenida**,
  no el de restablecimiento.

### Paso 8. Vercel

1. La cuenta de Vercel de la organizacion se conecta a la cuenta de GitHub de la organizacion
   (Account Settings > Authentication > GitHub) y se instala la app de Vercel **solo** sobre el
   repositorio `Ecopac-Digital`. Esto se hace en el navegador, con la sesion de la organizacion.
2. Proyecto `ecopac-digital`, importado de ese repositorio:

   | Ajuste | Valor |
   | --- | --- |
   | Root Directory | `apps/web` |
   | Framework | Vite (lo detecta) |
   | Production Branch | `main` |
   | Variables, Production | `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY` de `ecopac-prod`; `VITE_SENTRY_DSN` si hay |
   | Variables, Preview | Las mismas de `ecopac-dev` |

   Con `apps/web` como raiz, `apps/web/vercel.json` aplica el rewrite de SPA y las seis cabeceras
   de seguridad; la instalacion de dependencias la hace Vercel desde la raiz del monorepo.
3. Desplegar `main` y `develop`.

- **Comprobacion:** la URL de produccion abre el login; recargar `/pacientes` no da 404;
  `node scripts/verificar-cabeceras-http.mjs <url>` pasa; `http://` redirige a `https://` (RNF-8).
- **Reversa:** Vercel conserva cada despliegue; "Promote to Production" sobre el anterior.

Con las URL ya conocidas se vuelve al paso 3 (CFG-01, CFG-02) y al 7 (CFG-14, CFG-15) si se
habian dejado provisionales.

### Paso 9. Proteccion de ramas

**En una cuenta gratuita de GitHub, un repositorio privado no admite proteccion de ramas** (ni
reglas clasicas ni rulesets: los dos exigen GitHub Pro, 4 USD al mes). Es el precio de la decision
de hacerlo privado, y conviene saberlo:

- Los checks siguen corriendo en cada PR y en cada push, pero **no impiden mergear** un PR en rojo
  ni empujar directo a `main`.
- Lo que si sigue protegiendo es el propio pipeline: "Aplicar migraciones" depende de "Validar
  migraciones y funciones" y de "Migraciones no editadas", asi que una migracion rota **no llega a
  la base** aunque se mergee.
- Si la organizacion contrata GitHub Pro, se configura en Settings > Branches, para `main` y
  `develop`: PR obligatorio, checks requeridos `Lint y build`, `Migraciones no editadas` y
  `Validar migraciones y funciones`, sin force-push, conversaciones resueltas.

### Paso 10. Administrador inicial

La migracion 00188 crea `ecopac.digital@gmail.com` como `administrador`, **sin contrasena**. La
organizacion la fija con "Olvide mi contrasena" en la web de produccion; el correo llega a su
propio Gmail. Nadie del equipo conoce esa contrasena.

- **Comprobacion:** la organizacion inicia sesion y ve el panel completo.

### Paso 11. Crons y respaldo

Con el codigo en `main`, GitHub programa los cuatro crons. Se dispara cada uno a mano una vez
(Actions > Run workflow) para no esperar al horario:

| Workflow | Que se espera |
| --- | --- |
| Alertas de vencimiento | Las dos filas de la matriz en verde, con `{"alertasGeneradas":0,...}` (base limpia) |
| Mantener Supabase activo | Las dos filas en verde |
| Verificar despliegue | Los dos ambientes "Al dia" |
| Respaldo de la base | Archivo cifrado nuevo en Drive; luego, la primera restauracion real (`docs/RESPALDO-Y-RECUPERACION.md`) |

### Paso 12. App movil

Seccion 7. Se genera el APK de produccion y se instala en un telefono Android real contra
`ecopac-prod`.

### Paso 13. Pruebas posteriores a la implementacion

Seccion 8. Con datos **ficticios marcados como prueba**, que se retiran al terminar.

### Paso 14. Congelar el repositorio del curso

- Se borran de `URL-ECOPAC/Ecopac-Digital` los secrets de despliegue (`SUPABASE_ACCESS_TOKEN`,
  `SUPABASE_DB_PASSWORD`, los `_PROD`, los `RESPALDO_*`) para que no despliegue a nada.
- Los proyectos `Ecopac-Digital-Dev` y `Ecopac-Digital-Prod` de la cuenta del equipo se **pausan**
  y, cerrado el periodo de soporte, se borran.
- Se deja un aviso en el README del curso apuntando al repositorio de la organizacion.

## 6. Configuracion de Vercel y Supabase que no viaja en el repositorio

Resumen de lo que solo existe en los Dashboard, para quien tenga que reconstruirlo:

- **Supabase:** todo `docs/CONFIGURACION-SUPABASE.md` (CFG-01 a CFG-20) por proyecto, mas los
  secrets de funciones y las dos entradas de Vault.
- **Vercel:** Root Directory, Production Branch y las variables por ambiente (paso 8).
- **GitHub:** los diecisiete secrets (y la proteccion de ramas, si se contrata Pro).
- **EAS:** las variables `EXPO_PUBLIC_*` por ambiente (seccion 7).

## 7. App movil: APK sin tienda

Play Store cobra 25 USD de registro de desarrollador, y App Store 99 USD al ano. Sin presupuesto,
la app Android se reparte como **APK instalado directamente** en los telefonos del personal. iOS
queda fuera: sin cuenta de desarrollador de Apple no hay forma de instalar una build fuera de Expo
Go.

### Que esta en el repositorio

`apps/mobile/eas.json`, con dos perfiles; los dos generan un `.apk` (no un `.aab`, que solo sirve
para la tienda):

| Perfil | Ambiente de variables EAS | Base |
| --- | --- | --- |
| `desarrollo` | `preview` | `ecopac-dev` |
| `produccion` | `production` | `ecopac-prod` |

Las URL y llaves **no** estan en `eas.json`: viven en las variables de entorno de EAS de cada
ambiente, y EAS las inyecta al compilar. `appVersionSource: remote` y `autoIncrement` hacen que
cada APK de produccion lleve un `versionCode` mayor, que es lo que Android exige para instalar una
version nueva encima de la anterior sin desinstalar.

### Generar el APK

Una sola vez, desde `apps/mobile`, con la sesion de Expo de la organizacion:

```bash
npm install --global eas-cli
eas login                      # cuenta de Expo de la organizacion
eas init                       # crea el proyecto en Expo y devuelve el projectId
```

`eas init` no puede editar `app.config.js` (es JavaScript, no JSON) y pide agregar a mano
`owner: "<cuenta de expo>"` y `extra: { eas: { projectId: "<id>" } }` dentro de `expo`. Ese cambio
va en un PR.

Variables por ambiente (valores publicos por diseno, pero no se versionan):

```bash
eas env:create --environment production --name EXPO_PUBLIC_SUPABASE_URL --value https://<ref-prod>.supabase.co --visibility plaintext
eas env:create --environment production --name EXPO_PUBLIC_SUPABASE_ANON_KEY --value <anon de prod> --visibility plaintext
eas env:create --environment preview --name EXPO_PUBLIC_SUPABASE_URL --value https://<ref-dev>.supabase.co --visibility plaintext
eas env:create --environment preview --name EXPO_PUBLIC_SUPABASE_ANON_KEY --value <anon de dev> --visibility plaintext
# opcional: EXPO_PUBLIC_SENTRY_DSN en los dos
```

Cada build:

```bash
eas build --platform android --profile produccion
```

Compila en los servidores de Expo (plan gratuito: unas 30 builds al mes, en cola) y al terminar
da un enlace de descarga del `.apk` y un codigo QR. La llave de firma la genera EAS la primera vez
y la guarda en la cuenta de Expo: **todas las versiones se firman con la misma**, y por eso
actualizar funciona. Si esa llave se pierde, la app instalada no se puede actualizar: hay que
desinstalar en cada telefono. Se descarga un respaldo con `eas credentials` y se guarda con las
demas credenciales de la organizacion.

### Repartirlo

1. Descargar el `.apk` del enlace de EAS y guardarlo en una carpeta del Drive de la organizacion,
   con la version en el nombre (`ecopac-digital-1.0.0-12.apk`).
2. En cada telefono: abrir el enlace de Drive, descargar, y aceptar "Instalar apps de fuentes
   desconocidas" para el navegador o Drive cuando Android lo pida.
3. Abrir la app e iniciar sesion con la cuenta de la persona.

Para actualizar: nueva build con el perfil `produccion`, subirla a la misma carpeta y avisar al
personal. Se instala encima y conserva la sesion.

**El enlace de EAS caduca** (los artefactos del plan gratuito se conservan un tiempo limitado):
por eso la copia que manda es la de Drive.

### Alternativa sin la nube de Expo

`eas build --platform android --profile produccion --local` compila en la maquina propia (necesita
Android SDK y JDK 17). Sirve si se agota la cuota mensual; la llave de firma sigue en EAS.

## 8. Pruebas posteriores a la implementacion

Se corren **en produccion, recien desplegada**, y quedan como evidencia en `docs/evidencias/` con
fecha y commit. Con datos ficticios y marcados: nombres con el prefijo `PRUEBA`, que se eliminan al
terminar, antes de que la organizacion cargue datos reales.

- [ ] Inicio de sesion con una cuenta de cada rol, y cada rol ve solo lo suyo (`docs/PERMISOS.md`)
- [ ] Flujo completo: jornada, paciente, triaje, consulta, receta y entrega
- [ ] Un movimiento de inventario con su aprobacion
- [ ] Correo de invitacion (el de **bienvenida**) y de recuperacion llegan y abren la web de
      produccion (`scripts/verificar-redireccion-recuperacion.mjs`)
- [ ] Recarga de rutas internas y cabeceras HTTP (`scripts/verificar-cabeceras-http.mjs`)
- [ ] `http://` redirige a `https://`
- [ ] APK de produccion instalado en un telefono Android real, contra `ecopac-prod`
- [ ] En verde: alertas de vencimiento, keep-alive, verificar despliegue y el primer respaldo
- [ ] Restauracion del primer respaldo bajado de Drive

## 9. Registro de ejecucion

| Fecha | Paso | Que se hizo | Comprobado |
| --- | --- | --- | --- |
| 13-10-2026 | - | Decisiones de la seccion 3. Cambios de codigo de la seccion 4 | - |

## 10. Lo que quedo creado

Se llena al ejecutar.

| Que | Produccion | Desarrollo |
| --- | --- | --- |
| Repositorio | `<cuenta>/Ecopac-Digital`, ramas `main` / `develop` | |
| Proyecto Supabase (ref) | | |
| URL de Supabase | | |
| Web | | |
| Proyecto de Vercel | | |
| Proyecto de Expo | | |

## 11. Riesgos y plan de reversa

| Riesgo | Como se nota | Que se hace |
| --- | --- | --- |
| Una migracion falla al aplicar desde cero en Supabase (privilegios que el stack local no tiene, `docs/CI-CD.md`) | Job "Aplicar migraciones" en rojo, issue automatica | Migracion nueva que corrige hacia adelante; nunca se edita la aplicada. Sin datos reales, borrar y recrear el proyecto tambien es valido |
| `Validar migraciones y funciones` en rojo en el repositorio nuevo (por ejemplo, por el cambio del fixture) | El despliegue no corre: depende de ese job | Corregir y volver a empujar. Nada llega a las bases mientras tanto |
| La web no responde o apunta a la base equivocada | La URL no abre o el login falla | "Promote to Production" del despliegue anterior en Vercel; revisar las variables por ambiente |
| El correo no llega | No llega la invitacion ni la recuperacion | Revisar la contrasena de aplicacion y el puerto 465; mientras tanto la contrasena se fija desde el Dashboard (Authentication > Users) |
| Gmail bloquea el envio por volumen o por "actividad sospechosa" | Errores SMTP en los logs de las funciones | Revisar la alerta de seguridad en el Gmail; el limite (~500/dia) esta lejos del uso real |
| Un token del CI vence | "Aplicar migraciones" y los crons fallan con `Unauthorized` | Generar otro en la cuenta de la organizacion y actualizar el secret |
| Se pierde la llave de firma del APK | No se puede instalar una version nueva encima | Respaldo de `eas credentials` guardado con las credenciales |
| Produccion se pausa por inactividad (plan Free) | La web no carga datos | El keep-alive lo evita; si paso, "Restore project" en el Dashboard |

## 12. Criterio para declarar produccion abierta

Todas a la vez:

1. Los pasos 1 a 13 en verde en el registro.
2. Las pruebas de la seccion 8 hechas, con su evidencia, y los datos de prueba retirados.
3. Un respaldo de produccion restaurado con exito.
4. La organizacion inicio sesion con su cuenta de administradora y cambio las contrasenas de todas
   las cuentas (seccion 13).
5. El marco legal y el acta de entrega firmados (#915).

## 13. Comunicacion con la organizacion

| Cuando | Que se le avisa |
| --- | --- |
| Antes de empezar | Este plan, y que no cargue datos reales hasta el aviso de apertura |
| Al terminar los pasos 1 a 11 | URL de la web, como fijar su contrasena (paso 10) |
| Al terminar las pruebas | Que se retiraron los datos de prueba |
| Apertura | Que ya puede cargar datos reales, y el enlace al APK |

## 14. Entrega

Lo que se le entrega a la organizacion, y como:

| Que | Como |
| --- | --- |
| Repositorio | Ya es suyo: vive en su cuenta de GitHub |
| Base de datos y scripts | `supabase/migrations/` (el esquema completo), `docs/DICCIONARIO-DE-DATOS.md`, `docs/MODELO-DE-DATOS.md`, `scripts/respaldo-base.sh` y `scripts/comprobar-restauracion.sh` |
| Documentacion | `docs/` (incluidos los manuales de `docs/formal/` y `docs/manual/`) |
| Credenciales de las cuentas | En persona o por un gestor de contrasenas. **Nunca** por WhatsApp ni correo en texto plano |
| Lista de secrets | La tabla del paso 5: nombres y de donde sale cada valor, nunca los valores |
| APK | La carpeta de Drive de la seccion 7 |

Despues de entregar:

- La organizacion **cambia la contrasena de cada cuenta** en su primer ingreso y confirma la
  verificacion en dos pasos.
- Se **revocan** los tokens del aprovisionamiento (GitHub, Supabase, Vercel) y la contrasena de
  aplicacion de Gmail que conocio el equipo, y se generan nuevos para el CI.
- Se **rotan** las contrasenas de las dos bases y las `sb_secret_`, y se actualizan los secrets.
- Al cerrar el periodo de soporte (#921), el equipo se quita de todas las cuentas y se deja
  constancia en el acta (#915).
