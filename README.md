# Administrador de Usuarios (WMS Latam)

> Análisis del plugin a partir del código fuente (43 clases PHP, 18 módulos JS,
> 7 hojas CSS, 3 vistas de pestaña y 13 modales).
> Versión analizada: **1.49.0** — 16 de octubre de 2026.
> Ubicación: `wp-content/plugins/administrador-usuarios-wms/`

---

## 1. Ficha técnica

| Dato | Valor |
|---|---|
| **Nombre** | Administrador de Usuarios (WMS Latam) |
| **Versión** | 1.49.0 |
| **Text domain** | `au-wms` |
| **Requisitos** | WordPress 5.8+, PHP 7.4+ |
| **Dependencia blanda** | WooCommerce (para la sincronización de cupones; si no está, ese paso se omite sin error) |
| **Menú** | "Usuarios WMS" (posición 26) + submenú "Configuración" |
| **Capacidad requerida** | `manage_options` |
| **Tamaño** | ~32.000 líneas entre PHP, JS y CSS |
| **Archivo raíz** | `administrador-usuarios-wms.php` |

## 2. Qué resuelve

Es el panel central de operación del ecosistema WMS Latam. En este ecosistema
una "empresa" no es un CPT ni una tabla propia: **es un usuario de WordPress con
rol `company`**, sus "áreas" son cupones de WooCommerce registrados en
`wp_paquetes`, y la pertenencia usuario ↔ área vive en
`wp_correos_configurados`.

El plugin existe para administrar esa estructura —que de otro modo exigiría
tocar tres tablas a mano— desde una sola pantalla, y para mantenerla
sincronizada con las plataformas hermanas (Inspira y Octágono del Sentido).

No crea tablas propias: trabaja sobre las que ya existen.

## 3. Arquitectura

Arranca en `plugins_loaded` → `AU_Loader::init()`, que carga 43 clases en orden
explícito (repositorios y servicios antes que sus consumidores; no hay
autoloader PSR-4) y arranca 16 controladores AJAX como singletons.

Hay una separación de capas consistente en todo el código:

- **Repositorios** (`AU_Repository_*`) — único lugar donde hay SQL. Empresas,
  usuarios, usuarios externos, licencias y usermeta.
- **Servicios** (`AU_*_Service`) — reglas de negocio: borrado, impersonación,
  áreas, cupones, importación, correos, contraseñas, sincronización, creación de
  empresas, webhooks.
- **Controladores AJAX** (`AU_Ajax*`) — validan, sanean y delegan. Nunca
  contienen SQL ni lógica de negocio.
- **Vistas** (`views/`) — HTML puro con `<template>` para las filas; el JS clona
  nodos en lugar de construir HTML por concatenación.
- **Utilidades transversales** — `AU_Security`, `AU_Settings`,
  `AU_Password_Generator`, `AU_Cupos_Lock`, `AU_File_Parser`,
  `AU_Column_Mapper`.

**49 endpoints AJAX** registrados (`wp_ajax_au_wms_*`), todos solo para usuarios
autenticados.

**Frontend**: jQuery + 4 módulos núcleo encadenados por dependencias
(`admin-core.js` → `acciones-usuario.js` → `borrado-masivo.js` → `admin.js`),
más un archivo por pestaña/modal. Los módulos se comunican por
`window.AU_WMS_APP` y por funciones globales con nombre estable
(`AU_WMS_confirmarLoginComoUsuario`, `AU_WMS_abrirUsermeta`,
`AU_WMS_refrescarTabla`…). Librerías externas por CDN: SweetAlert2 11,
TomSelect 2.3.1 y `xlsx-js-style` 1.2.0.

## 4. Modelo de datos

| Tabla | Uso | Nº de referencias |
|---|---|---|
| `wp_usermeta` | Perfil, rol, licencia, programas, banderas del plugin | 91 |
| `wp_correos_configurados` | Puente usuario ↔ área ↔ empresa | 23 |
| `wp_users` | Cuentas | 21 |
| `wp_paquetes` | Áreas/grupos (incluye `cupon_id` y `licencia_id`) | 18 |
| `wp_licencias` | Catálogo de licencias y sus 9 accesos `incluir_*` | 11 |

**Metadatos clave que el plugin lee o escribe**: `nombre_empresa`,
`nit_empresa`, `nombre_encargado`, `empresa`, `empresa_id`, `licencia`,
`fecha_final_licencia`, `licencia_activa`, `inspira`, `octagono`,
`question_test_ids`, `user_code_company`, `created_by_api`,
`au_wms_login_disabled`.

Toda la configuración vive en **una sola opción** (`au_wms_settings`), y los
flujos por lotes guardan su estado en transients con prefijo `au_wms_` (que el
hook de desactivación limpia).

## 5. Seguridad

Es el área con más trabajo deliberado del plugin (endurecida en la 1.39.0):

- **`AU_Security::guard()`** en los 49 endpoints: capacidad `manage_options` +
  nonce `au_wms_nonce`.
- **Roles protegidos** — `puede_gestionar_usuario()` bloquea cualquier operación
  sobre super admins y sobre quien tenga `manage_options`, `promote_users`,
  `edit_users`, `delete_users` o `manage_woocommerce`, sin importar cómo se
  llame su rol. Solo opera sobre `subscriber`, `group_leader`, `external_user`,
  `company` y usuarios sin rol. Se revalida en servidor aunque el ID llegue
  manipulado desde el navegador.
- **Transacciones reales** en los borrados, con `ROLLBACK` ante cualquier fallo y
  recálculo de cupones solo si la operación se confirmó. Requiere InnoDB.
- **`$wpdb->prepare()`** en todas las consultas; los IDs de área se revalidan
  dentro del propio SQL (`p.empresa_id = %d`), de modo que un ID ajeno nunca
  devuelve datos de otra empresa.
- **Claves de usermeta reservadas** (`wp_capabilities`, `wp_user_level`,
  `session_tokens`…) visibles pero no editables desde el panel.
- **Tokens fuera del ZIP** — los cuatro tokens Bearer (Inspira/Octágono, sync y
  creación) salen vacíos de fábrica; mientras falten, las acciones que los usan
  quedan bloqueadas con aviso.
- **Sincronización a prueba de tokens inválidos** — las respuestas HTTP 0, 401,
  403, 429 y 5xx se tratan como error y nunca borran los metadatos de programa
  del usuario.
- **Cookie de impersonación firmada** con `wp_hash()`, httponly, caducidad de 12
  horas.

## 6. Las tres pestañas

### Empresas (rol `company`)

Columnas: Empresa (ID + nombre + correo + encargado + fecha), Áreas, Licencia
adquirida, Programas, Acciones. Filtros de licencia, programa, buscador con
debounce y paginación 10/25/50/100.

**Acciones de la barra**: Crear Empresa · Licencias · Refrescar programas.

**Acciones por fila**, agrupadas en tres submenús:

- *Datos de la empresa*: Ver Áreas, Editar empresa, Cambiar Licencia,
  Ver WP_USERMETA
- *Cuenta y credenciales*: Cambiar correo, Cambiar contraseña,
  Resetear contraseñas
- *Sincronización y reportes*: Refrescar programas, Actualizar id empresa
  usuario, Exportar usuarios, Comparar usuarios
- Y, fuera de los submenús, Iniciar sesión como usuario.

### Integrantes (`subscriber` / `group_leader`)

Columnas: Usuario, Asignación, Test, Programas, Acciones, con checkbox de
selección múltiple. Filtros de empresa → área (dependiente), test y programa.

**Acciones de la barra**: Importar usuarios · Actualizar correos ·
Borrar seleccionados (borrado masivo).

**Por fila**: Editar perfil, Áreas del usuario, Refrescar programas,
Ver WP_USERMETA, Cambiar correo, Cambiar contraseña, Habilitar/Deshabilitar
acceso, Borrar usuario, Iniciar sesión como usuario. La columna Test incluye
descarga del PDF de resultados vía microservicio externo.

### Usuarios Externos (`external_user`)

Es la vista más simple: Usuario, Programas, Acciones. Un botón global
"Refrescar programas" y, por fila, Iniciar sesión, Ver WP_USERMETA, Refrescar
programas, Cambiar contraseña y Borrar.

## 7. Módulos principales

**Crear Empresa** — Asistente por pasos que da de alta la misma identidad en los
tres sitios. Orden fijo: WMS primero (rol `company` con áreas si se eligió
Liderazgo, o `external_user` sin licencia ni áreas si no), después Inspira y
Octágono por API REST desde el servidor, y por último el webhook de
credenciales. Cada plataforma genera su propia contraseña —ninguna de las dos
APIs acepta recibirla—, así que el resultado muestra una credencial por
plataforma. El estado queda en un transient de 1 hora, de modo que si falla una
API el panel ofrece "Reintentar" solo para esa operación, sin volver a crear
nada de lo que ya salió bien.

**Importación masiva** — 5 pasos: empresa/área con resumen de cupos → subir
archivo → mapeo de columnas con sugerencia automática → previsualización sin
escribir nada (clasifica cada correo en nuevo/actualizar/omitido, detecta
duplicados y bloquea si no alcanzan los cupos) → procesamiento en lotes de 20.
Parser de CSV y XLSX nativo, sin Composer (XLSX se lee como ZIP+XML). El consumo
de cupos se serializa con `GET_LOCK()` de MySQL para evitar que dos
importaciones simultáneas ocupen el mismo cupo.

**Actualizar correos** — Mismo patrón de asistente, pero cada fila se procesa en
su propia transacción: si una falla se revierte sola y el proceso continúa. Las
validaciones se repiten dentro de la transacción, no se confía en la foto tomada
durante la previsualización.

**Refrescar programas** — Consulta las APIs de Inspira y Octágono por cada
usuario (2 llamadas HTTP independientes) y actualiza o borra los metadatos
`inspira`/`octagono`. Tiene tres alcances: un solo usuario, una empresa con
todos sus integrantes, o globalmente todas las empresas / todos los externos.
Siempre por lotes de 5, con modal de progreso bloqueante y resumen final de
actualizados/eliminados/errores.

**Borrar usuario** — Flujo de dos caminos. Primero refresca los programas del
usuario sin preguntar, y según el resultado ofrece *"Borrar definitivamente"*
(solo si no tiene Inspira ni Octágono, revalidado en servidor) o *"Dejar de ser
usuario WMS"*, que lo degrada a `external_user` conservando el acceso a los
otros programas. Ambos corren en transacción, recalculan cupones y limpian
LearnDash.

**Impersonación con retorno** — Además de `wp_set_auth_cookie()`, deja una
cookie firmada que pinta un enlace "⇦ Volver a {admin}" en la admin bar y una
franja al pie como respaldo. Dispara `au_wms_user_impersonated` y
`au_wms_volvio_a_administrador`.

**Licencias** — CRUD completo sobre `wp_licencias` replicando las pantallas del
tema, con los 9 atributos `incluir_*` y exportación de matriz comparativa.

**Resetear contraseñas / Exportar usuarios / Comparar usuarios** — Las tres
generan `.xlsx` con estilo en el navegador (colores de marca, autofiltros,
bandas por área). "Comparar usuarios" cruza una lista pegada de correos contra
*todas* las empresas y los clasifica en tres estados, con exportación por grupo.

**Ver WP_USERMETA** — CRUD paginado con buscador sobre los metadatos de
cualquier usuario, con claves reservadas en solo lectura.

## 8. Integraciones externas

| Destino | Para qué |
|---|---|
| `inspira.wmslatam.com` | Verificar usuario (sync) y crear empresa |
| `octagono.wmslatam.com` | Actualizar suscriptor (sync) y crear empresa |
| `api.efrenmartinezortiz-ia.com` | 4 webhooks: usuario nuevo, usuario actualizado, recuperación de contraseña y reset masivo |
| `pdf.wmslatam.com` | Microservicio del PDF de resultados del test |

Las 11 URLs y los 4 tokens son editables en *Usuarios WMS → Configuración*;
nada está fijo en el código.

## 9. Puntos de extensión

```php
// Ampliar los roles que el panel puede gestionar
add_filter( 'au_wms_roles_gestionables', fn( $roles ) => [ ...$roles, 'mi_rol' ] );

// Ampliar las capacidades consideradas privilegiadas (intocables)
add_filter( 'au_wms_capacidades_privilegiadas', $callback );

// Auditoría de impersonación
add_action( 'au_wms_user_impersonated', $callback );
add_action( 'au_wms_volvio_a_administrador', $callback );
```

## 10. Limitaciones conocidas

Documentadas por el propio plugin:

- **`.xls` legado (Excel 97-2003) no se parsea** — se pide convertirlo a `.xlsx`
  o `.csv`. Soportarlo exigiría PhpSpreadsheet vía Composer.
- **Escala de la importación** — el dataset validado se guarda entero en un
  transient; para decenas de miles de filas convendría paginar también la fase
  de validación.
- **Limpieza de LearnDash por patrones de `meta_key`** — no existe una API
  pública de LearnDash para "borrar todo el progreso", así que si tu instalación
  guarda algo con una clave que no calce con los patrones, queda sin borrar.
- **InnoDB obligatorio** en `wp_users`, `wp_usermeta` y
  `wp_correos_configurados`: en MyISAM los `ROLLBACK` se ignoran silenciosamente.
- **Una empresa/área representativa por fila** en el listado general: un usuario
  en varias áreas se agrupa con `MAX()` para no duplicar filas.

## 11. Nota sobre el README del plugin

El `README.md` incluido en el plugin está escrito como bitácora de cambios
acumulada, y su sección "Estructura del plugin" se quedó corta: lista 8 clases
de las 43 que existen hoy. Este documento sale del código, no de ese README.
