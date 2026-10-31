# Mapa de Navegación — Especificación de Rutas y Navegación (FaceAttend EDU)

> Define la estructura de pantallas del sistema FaceAttend EDU, cómo se conectan entre sí y qué rutas existen.
> Es la referencia cuando frontend y backend discuten qué endpoints existen o cómo llegar a una función.

**Plataformas:** Web + Mobile (ambas construidas con React Navigation).  
**Roles:** Administrador, Docente/Instructor, Estudiante/Alumno.  
**Validado contra el código real** en la rama `develop`: `front-end/Web/src/navegation/` (`appNavigator.js`, `AuthenticatedNavigator.js`, `linking.ts`), `front-end/Mobile/src/navigations/AppNavigator.js`, `useDashboardViewModel.js`, `MenuScreen.js`, `useRolePermissions.js`, `NotAuthorized.js`, y `back-end/99-api-gateway/kong/kong.yml`.

---

## 1. Nota de Arquitectura (según el código)

- **Web** usa un stack de dos niveles: `AppNavigator` (público) → `AuthenticatedNavigator` (privado). La barra lateral vive **fuera** del stack interno de navegación (`PersistentSidebar`) para que su estado sobreviva a la navegación.  
  Patrón enforced en código: **Navigator → Screen → View** — las rutas registran solo Screens; las Views nunca navegan.

- **Mobile** usa un único stack raíz que intercambia ramas pública/privada según `isAuthenticated` (token en AsyncStorage + `userRole`, validado con `hasValidToken()`).

- **Deep linking en Web** (`linking.ts`): `/`, `/login`, `/register`, `/recovery`, `/dashboard`; fallback SPA vía `nginx try_files`.

---

## 2. Estructura de Rutas del Frontend

### 2.1 Rutas Web (todos los roles — acceso filtrado por rol)

```
/ ................................. Página de aterrizaje (pública, sin autenticación)
                                     [ruta "FaceAttendEDU" → LandingScreen]
├── /login ........................ Autenticación (email/contraseña, indicador de
│                                    seguridad, PasswordPolicyModal) [FaceAttendEDU-Login]
├── /register ..................... Registro [FaceAttendEDU-Register]
│
└── /dashboard .................... Área autenticada [FaceAttendEDU-Dashboard]
    │  = AuthenticatedNavigator: CollapsibleSidebar persistente + stack interno con 12 pestañas:
    ├── Dashboard ................. KPIs (MetricsSummary, DailyBarChart, WeeklyTrend,
    │                              AtRiskStudentsList, PerfectAttendanceList…)
    ├── Users ..................... etiqueta cambia por rol: "Usuarios" (admin) /
    │                              "Alumnos" (docente).
    │                              Admin: crear/editar/importar/activar-desactivar;
    │                              eliminar lógico solo Super Admin ·
    │                              Docente: ver + registrar rostro
    ├── Roles ..................... matriz rol x permiso (SOLO Super Admin, HU-IAM-003)
    ├── Alertas ................... bandeja propia: ver, marcar resuelta, eliminar (HU-JUS-003)
    ├── Courses ................... RegisterCourseModal, ImportCoursesModal, CourseDetailModal
    │                              (eliminar solo Super Admin)
    ├── Academic .................. programas, períodos, fichas (activar/desactivar +
    │                              eliminar, SOLO admin)
    ├── Environments .............. cuadrícula EnvironmentCard, EnvironmentFormModal,
    │                              ScheduleModal — gestión SOLO ADMIN;
    │                              docente ve en lectura sus ambientes
    ├── History ................... consulta por parámetros (los 3 roles + ambos admin,
    │                              HU-HIST-001 AC10/AC11/AC12)
    ├── Justifications ............ alumno envía (HU-JUS-001) · instructor/admin
    │                              revisan (HU-JUS-002) · admin gestiona tipos
    │                              (HU-CONF-008, JustificationTypesManager)
    ├── Reports ................... filtros + gráficas + exportar (admin: todo el alcance;
    │                              docente: propio; estudiante: pestaña oculta,
    │                              dashboard personal vía canViewOwnReports)
    ├── Settings .................. secciones por rol: AdminSettings / TeacherSettings /
    │                               StudentSettings (general · apariencia · facial ·
    │                               notificaciones · seguridad)
    └── (sin tab, vía "Tomar asistencia") Biometrics + AttendanceKiosk
                                     (abrir/cerrar/cancelar clase, corregir a Tarde,
                                      quitar registro — canManageSessions/canCorrectAttendance)
```

---

### 2.2 Rutas Mobile (nombres de pantalla, no rutas URL — native stack)

**Rama desautenticada:**

```
HomesScreen ....................... Login (primera pantalla)
├── ForgotPasswordScreen ......... Solicitar código de recuperación
│   └── VerifyCodeScreen ........ Ingresar código + nueva contraseña → volver a login
└── (credenciales válidas) ...... DashboardScreen
```

**Rama autenticada:**

```
DashboardScreen ................. Panel principal (rol + nombre, acciones rápidas por rol)
├── Menu ........................ Menú lateral (Profile, UpdatePhoto, FacialFail,
│                                  MenuJustify, LanguageSettings, SchoolConfiguration)
├── Profile ..................... Perfil de usuario (+ logout)
├── Novedades ................... Feed de noticias
├── RegisterFace ................ Registro facial inicial
│   └── SuccessScreen ........... Confirmación tras inscribir
├── UpdatePhoto ................. Flujo de actualización de rostro (verificar → escanear → validar)
├── FacialFail .................. Ruta de fallo de reconocimiento → reintentar / justificar manual
├── DisplayingAttendance ........ Mi asistencia (vista)
├── AttendanceReportScreen ...... Reportes de asistencia (acción rápida de admin)
├── MenuJustify ................. Menú de justificaciones
│   ├── AddJustification ........ Enviar nueva justificación (fecha, docente del día,
│   │                              ambiente, motivo, adjuntos)
│   ├── ConsultJustify .......... Consultar justificaciones enviadas
│   ├── PendingJustificationScreen .. Cola pendiente (valida el docente)
│   ├── AddValidJustification ... Aprobar/rechazar una pendiente (docente)
│   └── ValidJustifications ..... Historial aprobadas/rechazadas
├── Ajustes:
│   ├── LanguageSettingsScreen .. Selección de idioma
│   └── AppearanceSettingsScreen .. Tema / apariencia
└── Pantallas solo-admin:
    ├── ManageUsersScreen ....... Gestión de usuarios desde mobile
    ├── ManageEnviromentScreen .. Gestión de ambientes
    └── SchoolConfigurationScreen Información del colegio
```

## 3. Mapa de Pantallas

### 3.1 Pantallas Web

| Pantalla | Ruta | Componente | Rol mínimo | Servicio backend vía Kong `:8000` |
|----------|------|-----------|------------|----------------------------------|
| Landing Page | `/` | `LandingScreen` → `LandingView` | Público | — |
| Login | `/login` | `LoginScreen` → `LoginView` | Público | `01-ms-identity` (`authApi.js`) |
| Signup | `/register` | `SignupScreen` → `SignupView` | Público | `01-ms-identity` |
| Dashboard | `/dashboard` (pestaña) | `DashboardScreen` | Estudiante+ | `05-ms-attendance` (resumen), `04-ms-scheduling` |
| Users | pestaña `users` | `UsersScreen` (`AdminUsers` / `TeacherUsers`) | Admin/Docente (docente: solo lectura + registrar rostro) | `01-ms-identity`, `02-ms-authorization`, `06-ms-biometric` |
| Courses | pestaña `courses` | `CoursesScreen` | Estudiante+ (ver) | `03-ms-academic` |
| Environments | pestaña `environments` | `EnvironmentsScreen` | Admin/Docente (docente: lectura; gestión: admin) | `03-ms-academic`, `04-ms-scheduling` |
| Reports | pestaña `reports` | `ReportsScreen` | Docente+ | `05-ms-attendance`, `08-ms-notification` (notify-all) |
| Settings | pestaña `settings` | `SettingsScreen` (`AdminSettings` / `TeacherSettings` / `StudentSettings`) + sección `Sede` (datos + directorio, oculta para Super Admin) | Estudiante+ | `07-ms-configuration`, `09-ms-quality` |
| Schools | pestaña `schools` | `SchoolsScreen` → `SchoolsView` | Super Admin | `03-ms-academic` |
| Roles | pestaña `roles` | `RolesScreen` → `RolesView` (`RolesManager`) | Super Admin | `02-ms-authorization` |
| History | pestaña `history` | `HistoryScreen` → `HistoryView` | Estudiante+ (alcance por rol, HU-HIST-001 AC10/AC11/AC12) | `05-ms-attendance`, `04-ms-scheduling` |
| Justifications | pestaña `justifications` | `JustificationsScreen` (submit vs review + `JustificationTypesManager` admin) | Estudiante+ | `05-ms-attendance` |
| Academic | pestaña `academic` | `AcademicScreen` → `AcademicView` (programas/períodos/fichas + activar/eliminar) | Estudiante+ (gestiona: admin) | `03-ms-academic` |
| Alerts | pestaña `alerts` | `AlertsScreen` → `AlertsView` (descripción humana por tipo + botón Ver contextual + resolver/eliminar, alcance propio) | Todos (alcance propio) | `08-ms-notification` |
| Biometrics | sin tab (vía Tomar asistencia) | `BiometricsScreen` → `BiometricsView` (revocar rostro/todo solo admin) | Admin/Docente | `06-ms-biometric` |
| AttendanceKiosk | sin tab (vía Tomar asistencia) | `AttendanceKioskScreen` (abrir/cerrar/cancelar, corregir, quitar) | Admin/Docente | `04-ms-scheduling`, `05-ms-attendance`, `06-ms-biometric` |

---

### 3.2 Pantallas Mobile

| Pantalla | Ruta del stack | Archivo de componente | Rol | Servicio |
|----------|---------------|----------------------|-----|----------|
| Login | `HomesScreen` | `view/screens/login/Login.js` | Público | `AuthService` → `01-ms-identity` |
| Forgot Password | `ForgotPasswordScreen` | `login/Forgotpasswordscreen.js` | Público | `01-ms-identity` (código de recuperación) |
| Verify Code | `VerifyCodeScreen` | `login/Verifycodescreen.js` | Público | `01-ms-identity` (`ResetPasswordUseCase`) |
| Dashboard | `DashboardScreen` | `DashboardScreen.js` + `useDashboardViewModel.js` | Todos | `05`, `04`, `03` (estadísticas por rol) |
| Menu | `Menu` | `MenuScreen.js` | Todos | — |
| News | `Novedades` | `NewsScreen.js` | Todos | `08-ms-notification` (`AlertService`) |
| Profile | `Profile` | `ProfileScreen.js` | Todos | `01-ms-identity` (`PersonService`) |
| Register Face | `RegisterFace` | `RegisterFace.js` | Estudiante, Docente (+admin/docente inscriben a otros en web) | `06-ms-biometric` (`facial/enroll`) |
| Enrollment success | `SuccessScreen` | `components/auth/SuccessScreen.js` | — | — |
| Update Face | `UpdatePhoto` | `UpdatePhotoScreen.js` | Estudiante, Docente | `06-ms-biometric` (`update-request`) |
| Facial Fail | `FacialFail` | `FacialFailScreen.js` | Todos | `06-ms-biometric` (verify fail path) |
| My Attendance | `DisplayingAttendance` | `DisplayingAttendance.js` | Todos | `05-ms-attendance` |
| Attendance Report | `AttendanceReportScreen` | `AttendanceReportScreen.js` | Admin | `05-ms-attendance` |
| Justifications Menu | `MenuJustify` | `MenuJustifyScreen.js` | Estudiante, Docente | `05-ms-attendance` (justificaciones) |
| Submit Justification | `AddJustification` | `AddJustifyScreen.js` | Estudiante | `05-ms-attendance` |
| Consult Justifications | `ConsultJustify` | `ConsultJustifyScreen.js` | Estudiante | `05-ms-attendance` |
| Pending (validate) | `PendingJustificationScreen` | `PendingJustificationScreen.js` | Docente | `05-ms-attendance` |
| Validate One | `AddValidJustification` | `AddValidJustificationScreen.js` | Docente | `05-ms-attendance` |
| Validated History | `ValidJustifications` | `ConsultJustifyScreen.js` (`ValidJustificationsScreen`) | Estudiante, Docente | `05-ms-attendance` |
| Language Settings | `LanguageSettingsScreen` | `LanguageSettingsScreen.js` | Todos | `07-ms-configuration` (`LanguageService`) |
| Appearance Settings | `AppearanceSettingsScreen` | `AppearanceSettingsScreen.js` | Todos | `07-ms-configuration` |
| Manage Users | `ManageUsersScreen` | `ManageUsersScreen.js` | **ADMIN** | `01-ms-identity`, `02-ms-authorization` |
| Manage Environments | `ManageEnviromentScreen` | `ManageEnviromentScreen.js` | **ADMIN** | `03-ms-academic` |
| School Configuration | `SchoolConfigurationScreen` | `SchoolConfigurationScreen.js` | **ADMIN** | `03-ms-academic`, `07-ms-configuration` |

---

### 3.3 Resumen de acceso por rol

> Validado contra `useRolePermissions.js` + `getVisibleTabs`.

| Capacidad | Estudiante | Docente | Admin |
|-----------|-----------|---------|-------|
| Pestañas web visibles | `dashboard`, `history`, `justifications`, `academic` (solo su ficha), `courses`, `alerts`, `settings` | `dashboard`, `users` (solo sus alumnos), `courses` (solo suyos), `environments` (lectura), `reports`, `history`, `justifications`, `alerts`, `settings` | Admin sede: `dashboard`, `reports`, `users`, `courses`, `academic`, `environments`, `history`, `justifications`, `alerts`, `settings` · Super Admin: `dashboard`, `schools`, `reports`, `users`, `roles`, `history`, `alerts`, `settings` (sin justificaciones ni detalle académico: son de cada institución) |
| Etiqueta de Users | — (sin tab) | "Alumnos" | "Usuarios" |

| Ver estudiantes | Sí (contexto propio) | `canViewStudents` | Sí |
| Crear/editar/importar/activar-desactivar/eliminar usuarios | No | No | `canManageStudents` / `canImportStudents` / `canToggleUserStatus` / `canDeleteUser` (alcance de sede; cambio de rol incl. super: solo Super Admin) |
| Registrar rostro propio (mobile) | Sí | Sí | Sí |
| Registrar rostro a otros (web) | No | `canRegisterFace` | Sí |
| Gestionar cursos | No (solo ver) | No (solo ver + detalle) | Sí (eliminar: solo Super Admin) |
| Ambientes (ver / gestionar) | No / No | Ver sí, gestionar no | Sí / Sí |
| Reportes (exportar) | No | Propio (`canExportReports`) | Total (`canViewAllReports`) |
| Notificar a todos (`canNotifyAll`) | No | No | Sí |
| Enviar justificación | Sí | No | No |
| Aprobar/rechazar justificación | No | Sí (`PendingJustificationScreen`) | Sí |
| Cambiar tema/idioma/apariencia | `canEditAppearance` | Sí | Sí |
| Editar info del colegio / seguridad | No | Parcial | Sí |

---

## 4. Flujos Principales de Usuario

### Flujo 1 — Autenticación (Login Mobile)

1. `HomesScreen` → usuario ingresa credenciales → `AuthService` → `sessionId` opaco (Kong no tiene plugin `jwt`; valida la sesión en el edge y cada servicio la revalida por su cuenta, ver `07-api/authentication.md`).
2. **Credenciales válidas** → `DashboardScreen` (guarda `userRole` + token en AsyncStorage; muestra rol, nombre y acciones rápidas por rol).
3. **Credenciales inválidas** → Login con mensaje de error.
4. **Olvidé contraseña** → `ForgotPasswordScreen` → `VerifyCodeScreen` (código de recuperación → nueva contraseña → volver a login).

**HUs relacionadas:** `HU-IAM-001`, `HU-IAM-007`.

> **Notas verificadas en código:** la validez de sesión se comprueba en arranque en frío con `hasValidToken()`; si expira se purga el almacenamiento (`multiRemove` de `userRole`, `userEmail`, `userProfile`, `appLanguage`, `alertsConfig`). El logout llama a `AuthService.logout(sessionId)` en el servidor antes de limpiar el estado local y resetea el idioma a `es`. No hay pantalla dedicada de "éxito de autenticación".

---

### Flujo 2 — Registro Facial y Asistencia (Mobile)

1. `DashboardScreen` → tarjeta "Registrar rostro" → `RegisterFace` → captura → `POST /biometric/facial/enroll` (`06` vía Kong) → `SuccessScreen` si OK, reintento si falla.
2. **Asistencia del día a día** (dispositivo edge, no pantalla de la app): contenedor `fae-edge` → `facialIdentify` / `facialVerify` (`06`) → registro de asistencia (`05`).
   - Reconocido → registro creado (entrada/salida).
   - **NO reconocido** → `FacialFailScreen` → reintentar **O** justificar manualmente (`MenuJustify` → `AddJustification`).

**HUs:** `HU-ATT-001`, `HU-ATT-003`.

---

### Flujo 3 — Envío de Justificación (Mobile — Estudiante)

`DashboardScreen` → `Menu` → `MenuJustify` → "Agregar justificación" → `AddJustification` (llenar fecha, docente del día, ambiente, motivo, adjuntos) → submit a `05-ms-attendance` (`JustificationService`).  
- **OK** → estado Pendiente visible en `ConsultJustify`.  
- **Error** → mensaje con detalles.

**HUs:** `HU-JUS-001`, `HU-JUS-002`, `HU-JUS-003`.

---

### Flujo 4 — Evaluación de Justificaciones (Mobile — Docente)

`DashboardScreen` → tarjeta "Justificaciones pendientes" → `PendingJustificationScreen` → seleccionar → `AddValidJustification` → **Aprobar** (estado "Aprobada" + notificación al estudiante vía `08`) o **Rechazar** (estado "Rechazada" + notificación). Resultado visible en `ValidJustifications`.

**HUs:** `HU-JUS-002`, `HU-JUS-003`.

> **Nota:** la aprobación por admin ocurre en Web; en Mobile la cola de validación está cableada al rol docente (`menuAccionesDocente`).

---

### Flujo 5 — Actualización de Parámetros Faciales (Mobile)

`Menu` / tarjeta "Actualizar rostro" → `UpdatePhoto`.

| Paso | Acción | Resultado |
|------|--------|-----------|
| 1 | Verificación de identidad (datos del usuario) | Verificado → paso 2; Fallido → error/reintento |
| 2 | Capturar nuevo rostro → `POST /biometric/update-request` (`06`) | — |
| 3 | Validar previsualización | Correcta → actualización procesada + confirmación; Incorrecta → reintentar captura |

**HUs:** `HU-CONF-006`, `HU-ATT-002`.

---

### Flujo 6 — Gestión de Usuarios por Admin (Web)

`/dashboard` → pestaña "Usuarios" (`AdminUsers`):

- **Listar/buscar/filtrar:** `UsersTable` + paginación `limit/offset`.
- **Crear usuario:** formulario → validar → `POST 01` → refrescar.
- **Editar:** formulario → diálogo de confirmación → `PUT`.
- **Desactivar/eliminar:** `ConfirmModal` — siempre confirmado.
- **Importar estudiantes:** modal `ImportStudentsModal` — CSV/Excel (.xlsx), columnas requeridas `name`/`code`(`documentNumber`)/`email`; las filas sin curso se matriculan en el par **Curso destino → Ficha destino**; las filas cuyo correo o documento ya existen en el sistema se marcan "ya existe" y se omiten (re-importar no se ve como fallo); alta real por fila con pausa entre filas contra el rate-limit de Kong.
- **Registrar rostro:** modal `FaceRegistration` → `06/facial/enroll`.

**HUs:** `HU-IAM-002`, `HU-IAM-007`, `HU-IAM-008`, `HU-IAM-009`.

---

### Flujo 7 — Generación y Exportación de Reportes (Web)

`/dashboard` → pestaña "Reportes" (alcance según rol).

- **Filtros** (`UserFilters`, `ContextualFilterDropdown`, `AttendanceFilterInput`): rango de fechas, curso, ambiente, programa, grupo.
- **Ver:** gráficas (`DailyBarChart` / `WeeklyTrend`) + tablas + `AtRiskStudentsList`.
- **Exportar:** descarga de archivo (`canExportReports`: docente/admin).
- **Notificar a todos:** solo admin (`canNotifyAll` → `08`).

**HUs:** `HU-REP-001`, `HU-REP-003`, `HU-REP-004`.

---

## 5. Reglas de Navegación (validadas contra código)

| Regla | Detalle |
|-------|---------|
| **Autenticación requerida** | Todas las pantallas privadas requieren sesión activa (`sessionId` opaco validado por el edge de Kong y de nuevo por cada servicio — no hay JWT, ver `07-api/authentication.md`). Web: deep link a `/dashboard` sin sesión renderiza `NotAuthorized`. Mobile: el stack raíz solo expone la rama pública cuando `isAuthenticated === false`. |
| **Excepciones de autorización** | `NotAuthorized.js` define 5 tipos: `no_session`, `session_expired`, `no_role`, `invalid_token`, `insufficient_permissions`; renderiza un modal bloqueante sobre toda la pantalla y redirige a login al aceptar. |
| **Rol desconocido = cero acceso** | `getVisibleTabs(role)` devuelve lista vacía para cualquier rol no reconocido (*"Vacío a propósito, no queremos fugas"*). Defensa en profundidad: filtrado UI + validación de sesión en el edge de Kong + validaciones de `02-ms-authorization`. |
| **Acceso por rol** | Mismas rutas/pestañas para todos los roles en Web filtradas por `useRolePermissions`; en Mobile el dashboard muestra menús de acción rápida por rol (`menuAccionesAdmin` / `Docente` / `Estudiante`). |
| **Barra lateral persistente (Web)** | Vive fuera del stack interno; la pestaña/sub-pestaña seleccionada se sincroniza con el estado de navegación (incluida entrada por URL), evitando pérdida de estado. |
| **Diálogos de confirmación** | Acciones destructivas/críticas usan `ConfirmModal` antes de ejecutar. |
| **Paginación** | Las tablas reflejan el contrato de API `limit/offset` + cabecera `x-total-count` (máx 100). |
| **Deep linking (Web)** | 5 rutas expuestas: `/`, `/login`, `/register`, `/recovery`, `/dashboard`; las pestañas internas son rutas del stack alcanzadas desde el dashboard. `nginx try_files` da fallback SPA. |
| **Manejo 404** | No existe pantalla 404 personalizada. |
| **Permisos info del colegio** | Solo lectura para estudiantes/docentes; edición vía `SchoolConfigurationScreen` (mobile, admin) y Settings (web, admin). |
| **Permisos justificaciones** | Estudiantes envían; docentes validan vía cola móvil; admin actúa con alcance total. |

---

## 6. Referencia de Servicios Backend

> Nombres reales en `back-end/` + rutas Kong.

| Servicio | Host interno | Responsabilidades |
|----------|-------------|-------------------|
| `01-ms-identity` | `ms-identity:8081` | Login, registro, sesiones (`sessionId` opaco, no JWT), restablecimiento de contraseña con código de recuperación, CRUD de usuarios y personas |
| `02-ms-authorization` | `ms-authorization:8082` | Roles, permisos, validación de control de acceso |
| `03-ms-academic` | `ms-academic:8083` | Colegios, programas, cohortes/fichas, cursos, ambientes, actores académicos |
| `04-ms-scheduling` | `ms-scheduling:8084` | Sesiones de clase, bloques de horario, asignación de docentes por ficha/ambiente/día/curso |
| `05-ms-attendance` | `ms-attendance:8085` | Registros y resumen de asistencia, historiales, justificaciones (enviar/validar), datos de reportes |
| `06-ms-biometric` | `ms-biometric:8086` | Inscripción/verificación/identificación facial, endpoints de huella (API lista, UI pendiente), solicitudes de actualización de rostro |
| `07-ms-configuration` | `ms-configuration:8087` | Idioma, tema/apariencia, ajustes de alertas y seguridad, umbrales biométricos |
| `08-ms-notification` | `ms-notification:8088` | Alertas/feed de noticias, notificaciones de resultado de justificaciones, notificar-a-todos (admin) |
| `09-ms-quality` | `quality-service` | Instrumentos/métricas de calidad consumidos por widgets del dashboard |
| `99-api-gateway` | Kong OSS `:8000` | Punto de entrada único para ambas apps: `pre-function` de validación de sesión en el edge (sin plugin `jwt`) + rate-limit por ruta; rutas anidadas para sub-recursos |

---

## 7. Brechas Conocidas (corregir o documentar)

1. ✅ Resuelto 2026-10-15: deep link `/recovery` agregado en `linking.ts` (pantalla `FaceAttendEDU-Recovery` ya existía en el stack).
2. ✅ Desactualizada: la huella SÍ está implementada en web (`FingerprintModule.js`, kiosco, enroll/verify/identify) — entrega anticipada al roadmap (`vision.md` la ubica en Horizon 3). Pendiente decisión de producto: actualizar roadmap o mantener como anticipada.
3. ⚠️ Aclarado: `max_login_attempts=5` existe como seed pero ningún servicio lo lee (HU-IAM-001 AC4 no implementado) — pendiente de back end, no de frontend.
4. 🔤 Los nombres en Mobile mezclan inglés/español (`Novedades`, `MenuJustify`, errata `ManageEnviromentScreen`) — estandarizar.
5. ❓ La capacidad de docentes para enviar justificaciones a nombre de estudiantes necesita confirmación de producto (`canSubmitOnBehalf=false` explícito en código hasta entonces).
6. 📌 Decisión 2026-10-15: `ContactsView` queda como directorio de solo lectura sin HU (gold-plating documentado, no se retira por ser útil y sin costo); `Signup` público sigue sin endpoint backend (creación oportunista, ver `frontend-integration.md` §3).

---

## 8. Correlaciones

| Documento | Contenido |
|-----------|-----------|
| `04-requirements/user-stories.md` | Historias de usuario |
| `04-requirements/non-functional.md` | Requisitos no funcionales |
| `04-requirements/traceability-matrix.md` | Matriz de trazabilidad |
| `02-domain/entities-and-rules.md` | Entidades y reglas del dominio |
| `00-governance/security-policy.md` | Roles y permisos |
| `12-ux-ui/design-system.md` | Tokens de diseño y componentes |
| `12-ux-ui/wireframes.md` | Bocetos de estructura de pantallas |
| `07-api/contracts/` + `back-end/*/SERVICE.md` | Contratos de API consumidos por las pantallas |