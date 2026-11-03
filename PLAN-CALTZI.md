# Plan de implementación — **Caltzi**

> App de gestión residencial (paridad funcional con la app analizada `com.omasol.ocirix` v4.3.1) en **Flutter multiplataforma**, **100 % local**, sin backend ni Firebase. Toda la persistencia es local y la capa de datos queda abstraída para conectar **PocketBase** más adelante sin reescribir la app.

- **Proyecto:** Caltzi
- **Plataformas objetivo:** Android, iOS (principales); Web y Desktop (soporte limitado/demo)
- **Estado:** planificación
- **Stack:** Flutter 3.x + Dart 3, Riverpod, go_router, drift + SQLCipher

---

## 1. Decisiones base

| Decisión | Elección | Consecuencia |
|---|---|---|
| Nombre | **Caltzi** | `appId: com.caltzi.app`, título "Caltzi", tema Material 3 con `#ffae3c` |
| Backend | **Ninguno (fase 1)** | Sin permiso `INTERNET`, sin cuentas remotas, sin telemetría |
| Persistencia | **SQLite con drift + SQLCipher** | Cifrado en reposo, migraciones versionadas |
| Archivos | Directorio privado de la app | Fotos y comprobantes con ruta relativa en BD |
| Auth | **Local**: perfil + PIN + biometría | Sin servidor de identidad; roles en BD |
| Sincronización | **PocketBase en fase 8** | Repositorios ya diseñados con interfaces |
| Notificaciones | **Locales** (`flutter_local_notifications`) | Recordatorios programados, sin FCM |
| QR de visitante | Firma **HMAC local** con secreto del dispositivo | Migrable a firma del servidor después |

---

## 2. Alcance funcional (paridad, todo local)

| Módulo | Comportamiento local |
|---|---|
| Identidad | Alta de perfil, PIN, biometría, recuperación por pregunta de seguridad, roles (residente/guardia/admin) |
| Onboarding "solicitar acceso" | Crea un registro `solicitudes` que el rol admin aprueba en el mismo dispositivo |
| Home | Resumen: saldo, visitas activas, próxima reservación, últimos comunicados |
| Control de acceso | Visitantes con foto/placas/vigencia, **QR generado y validado en el dispositivo**, tarjetas, empleados, bitácora, switch "permitir visitas" |
| Pagos | Cargos, estado de cuenta, CLABE/cuenta, comprobante fotográfico, marcar pagado (admin), historial, saldo |
| Reservaciones | Calendario de áreas comunes, validación de solapes, estados pendiente/autorizada |
| Comunicados | CRUD con adjuntos y enlaces |
| Incidentes | Alta con categoría y fotos, estados, seguimiento |
| Sugerencias | Formulario + historial |
| Encuestas | Voto único por perfil, cierre por fecha |
| Clasificados | Alta con vigencia, expiración automática al abrir la app |
| Documentos | Adjuntos y apertura con app externa |
| Kupones | Listado con vigencia |
| Perfil | Datos, casa, teléfono/conmutador |
| Mascotas / Vehículos | CRUD con foto (mascota) y placas (vehículo) |
| Notificaciones | Recordatorios de pago, reservaciones, avisos nuevos |
| Ajustes | Idioma es/en, calidad de cámara, tema claro/oscuro, respaldo y exportación |

**Modo demo (semilla):** al primer arranque se puede cargar un residencial de ejemplo con casas, cargos, comunicados y visitantes para que todo se vea funcionando sin capturar datos.

---

## 3. Arquitectura local-first

```
lib/
├── main.dart
├── app.dart
├── core/
│   ├── db/            drift: esquema, DAOs, migraciones, cifrado
│   ├── storage/       archivos, rutas, compresión de imágenes
│   ├── security/      PIN (Argon2id), Keystore, biometría, HMAC
│   ├── notify/        notificaciones locales + programación
│   ├── router/        go_router + guard de sesión
│   ├── theme/         tokens de Caltzi
│   ├── l10n/          ARB es/en
│   └── result/        Result/Failure
├── features/
│   └── <feature>/{data,domain,presentation}
│       data/    → LocalXRepository (implementa XRepository)
│       domain/  → entidades, casos de uso, interfaz XRepository
└── shared/widgets/
```

**Contrato de migración:** cada feature expone una interfaz en `domain` (`VisitantesRepository`, `PagosRepository`…). Hoy se implementa `Local*`; en fase 8 se añade `PocketBase*` y se cambia el binding de Riverpod. **La UI no cambia.**

Sin `dio`, sin OpenAPI, sin Firebase: el estado global es Riverpod y la fuente de verdad es SQLite.

---

## 4. Modelo de datos local (espejo del futuro PocketBase)

| Tabla | Campos principales |
|---|---|
| `residenciales` | id, nombre, direccion, logo, moneda, zona_horaria |
| `casas` | id, residencial_id, numero, calle, saldo_centavos |
| `perfiles` | id, residencial_id, casa_id, nombre, email, telefono, rol, pin_hash, salt, biometrico, idioma |
| `sesiones` | perfil_id, ultima_actividad, token_local |
| `solicitudes` | id, nombre, email, telefono, casa_id, estado |
| `visitantes` | id, casa_id, nombre, identificacion, placas, foto_path, valido_desde, valido_hasta, estado, qr_jti |
| `accesos` | id, visitante_id, casa_id, registrado_por, tipo, metodo, ts |
| `tarjetas` | id, casa_id, alias, numero_enmascarado, estado |
| `empleados` | id, nombre, puesto, foto_path, activo |
| `cargos` | id, casa_id, concepto, monto_centavos, vencimiento, estado |
| `pagos` | id, casa_id, monto_centavos, comprobante_path, estado, revisado_por, ts |
| `areas` / `reservaciones` | área, capacidad, horario / fecha, inicio, fin, invitados, estado |
| `comunicados`, `incidentes`, `sugerencias`, `polls`, `poll_opciones`, `poll_votos`, `clasificados`, `documentos`, `kupones`, `mascotas`, `vehiculos` | análogos |
| `notificaciones` | perfil_id, titulo, cuerpo, ruta, leida, ts |
| `adjuntos` | entidad, entidad_id, path, mime, bytes |
| `ajustes` | clave/valor |
| **`outbox`** | op, entidad, entidad_id, payload_json, creado_en, intentos ← **reservada para PocketBase** |

Reglas de integridad en la capa de servicios (no en la UI):

- montos en **centavos enteros**;
- fechas en **UTC ISO-8601**;
- borrado lógico (`deleted_at`) para no romper la futura sincronización;
- `updated_at`/`rev` en cada fila para resolución de conflictos.

---

## 5. Servicios locales que sustituyen a las Cloud Functions

| Servicio | Responsabilidad |
|---|---|
| `AuthService` | Alta, PIN (Argon2id), biometría, bloqueo tras N intentos, sesión |
| `QrService` | Genera `base64(payload).hmac` con secreto de 32 bytes en Keystore; valida firma, vigencia y estado; marca usado en transacción |
| `ReservaService` | Transacción: verifica solape y horario del área, crea `pendiente` |
| `PagoService` | Cargos, abonos, comprobante, recálculo de saldo, historial |
| `EncuestaService` | Un voto por perfil (índice único `perfil_id + poll_id`) |
| `ExpiracionService` | Al abrir la app / al volver de background: vence clasificados, cargos y visitantes |
| `NotificacionService` | Programa locales (recordatorios, reservaciones) y bandeja interna |
| `BackupService` | Export/import `.caltzi` (zip cifrado con contraseña): BD + adjuntos |
| `SeedService` | Datos demo del residencial de ejemplo |
| `AuditService` | Bitácora interna de accesos y aprobaciones |

---

## 6. Seguridad local

- **SQLCipher** con clave aleatoria guardada en Keystore/Keychain (`flutter_secure_storage`); la clave nunca se escribe en el código.
- **PIN** con Argon2id + salt por perfil; bloqueo exponencial; biometría como desbloqueo del Keystore, no como contraseña.
- Archivos en directorio privado; sin `allowBackup` (o backup cifrado propio) para evitar fugas.
- Capturas bloqueadas en pantallas sensibles (`FLAG_SECURE` en Android, protección equivalente en iOS) — opcional en fase 7.
- **Sin permiso `INTERNET`** en la build release: la app no puede exfiltrar nada. Declaración de Play Data Safety trivial.
- Exportación solo bajo confirmación explícita y con contraseña.

---

## 7. Estructura y dependencias

```yaml
dependencies:
  flutter_riverpod, go_router, freezed_annotation, json_annotation
  drift, sqlcipher_flutter_libs, sqlite3_flutter_libs, path_provider, path
  flutter_secure_storage, local_auth
  cryptography            # HMAC + Argon2id
  qr_flutter, mobile_scanner, share_plus, url_launcher, open_filex
  image_picker, image, flutter_image_compress
  flutter_local_notifications, timezone
  table_calendar, intl, flutter_localizations
  cached_network_image   # solo para assets locales/adjuntos
  connectivity_plus, package_info_plus, in_app_update, upgrader

dev_dependencies:
  build_runner, drift_dev, freezed, json_serializable, mocktail,
  patrol, golden_toolkit
```

Configuración por plataforma:

```
android/app/src/main/AndroidManifest.xml   → sin INTERNET (release)
ios/Runner/Info.plist                      → NSCameraUsageDescription,
                                             NSPhotoLibraryUsageDescription,
                                             NSFaceIDUsageDescription,
                                             NSContactsUsageDescription
```

---

## 8. Multiplataforma

| Tema | Android | iOS | Web | Desktop |
|---|---|---|---|---|
| BD | SQLCipher nativo | SQLCipher nativo | drift web (IndexedDB, sin cifrado) | SQLite + cifrado |
| Biometría | huella/rostro | Face ID | WebAuthn (opcional) | opcional |
| Archivos | SAF | Document picker | descarga/blobs | nativo |
| Notificaciones | locales | locales | limitadas (ocultar) | bandeja interna |
| Cámara/QR | sí | sí | sí | webcam opcional |
| Ajuste de texto | clamp a 1.15× (paridad con el original) | igual | CSS | igual |

Web se declara **soporte de lectura/demo**; el uso real (QR, archivos, BD cifrada) es Android/iOS.

---

## 9. Fases

| Fase | Sem | Entregable | Criterio de aceptación |
|---|---|---|---|
| **0. Base Caltzi** | 1,5 | Repo, flavors (dev/prod), tema, i18n es/en, router, drift + SQLCipher, CI (`analyze`, `test`), `SeedService` | App arranca en Android/iOS/Web, BD cifrada creada, demo cargable |
| **1. Identidad** | 2 | Perfil, PIN, biometría, roles, onboarding + aprobación local, ajustes de idioma | Bloqueo por PIN/biometría; roles restringen módulos; sesión persiste |
| **2. Acceso + QR** | 2,5 | Visitantes (foto, placas, vigencia), QR firmado y validación, tarjetas, empleados, bitácora, switch de visitas | QR expirado se rechaza; QR usado no reutilizable; bitácora paginada |
| **3. Pagos** | 1,5 | Cargos, estado de cuenta, CLABE, comprobante, aprobación admin, saldo | Saldo cuadra tras abonos; comprobante visible y reemplazable |
| **4. Reservaciones** | 1,5 | Áreas, calendario, horarios, invitados, aprobación | Sin solapes; estados correctos; notificación local al aprobar |
| **5. Comunidad** | 2,5 | Comunicados, incidentes, sugerencias, encuestas, clasificados, documentos, kupones | 7 módulos con lista/detalle/alta y adjuntos |
| **6. Notificaciones y datos** | 1,5 | Recordatorios programados, bandeja, expiraciones, respaldo/exportación, tema oscuro | Recordatorio llega a la hora; import restaura todo |
| **7. Hardening y tiendas** | 1,5 | Tests E2E, golden, accesibilidad, rendimiento, firmas, fichas de tienda | Cobertura >60 %, 0 crashes en 3 dispositivos, build en internal track |

**Total ≈ 14 semanas.** **MVP usable** (identidad + acceso/QR + pagos + comunicados) ≈ **8 semanas**.

---

## 10. Fase 8 (futura): PocketBase, sin reescribir

1. **Esquema espejo:** una colección por tabla; `perfiles` como colección de auth; roles en campo `rol` + reglas de API; archivos como campos `file` (reemplazan `*_path`).
2. **Interfaces ya listas:** se implementa `PocketBase*Repository` y se cambia el provider; cero cambios en UI.
3. **Outbox:** toda escritura encola `{op, entidad, payload}`; al recuperar conexión se envía y se marca `synced`. Los IDs se generan en cliente (UUID v7) para evitar remapeos.
4. **Conflictos:** `updated_at` + `rev` por fila; gana el más reciente salvo entidades autoritativas (QR, votos, aprobaciones de pago), que pasan a validarse en servidor.
5. **Migración de datos:** script de export local → import PocketBase, con verificación de conteos y hashes.
6. **Estimación:** 3–4 semanas adicionales (servidor + sync + reglas + pruebas de conflicto).

---

## 11. Pruebas y entrega

- Unit: servicios (QR, reservas, saldos, votos) con BD en memoria.
- Widget/golden por pantalla crítica; Patrol para flujos E2E (alta de visita → QR → validación).
- Migraciones de drift probadas con BD v1→vN (importante para no perder datos).
- CI: `flutter analyze`, `flutter test`, build debug por plataforma; releases con Fastlane/Codemagic.
- Definición de Done por módulo: paridad de campos, estados vacío/error, i18n es/en, test, y pantalla verificada en Android + iOS.

---

## 12. Riesgos

| Riesgo | Mitigación |
|---|---|
| Pérdida de datos (todo es local) | Export/import cifrado + recordatorio de respaldo; opción de copia automática al directorio elegido |
| Dispositivo compartido entre residentes | Perfiles múltiples con PIN; datos aislados por `casa_id` y `perfil_id` |
| Sin aprobación real de administración | Flujo admin local explícito, marcado como "modo local" hasta PocketBase |
| Fuga entre residenciales al migrar | `residencial_id` en todas las tablas + pruebas de aislamiento |
| Funciones que requieren servidor (push real, pagos en línea) | Sustituidas por notificaciones locales y registro manual; documentadas como pendientes |

---

## 13. Siguientes pasos

1. Confirmar 3 detalles de producto:
   - ¿varios perfiles por dispositivo?
   - ¿incluimos modo **guardia** (escaneo de QR) en la v1?
   - ¿idioma por defecto es-MX con inglés secundario?
2. Arrancar **Fase 0**: repo `caltzi`, drift con cifrado, tema, i18n, router, `SeedService`, CI.
3. Definir los 24 namespaces de traducción de Caltzi (equivalentes a los del original) en `lib/core/l10n`.

---

## Anexo A — Trazabilidad con la app analizada

Funciones identificadas en `kolonus.apk` 4.3.1 (paquete `com.omasol.ocirix`) que este plan reimplementa localmente:

- **Módulos i18n (24):** access-control, access-tags, announce, booking, classified, complaint, documents, employee, entry-record, guest, home, incident, kupones, login, maintenance, payment, pets, poll, profile-edit, profile-info, push-diagnostic, setup, vehicle, common.
- **Rutas del cliente (34):** `/`, `/access-control`, `/access-tags`, `/add`, `/announce`, `/bank`, `/booking`, `/classified`, `/complaint`, `/detail`, `/detail/:id`, `/detail/eventGuest`, `/documents`, `/employee`, `/entry-record`, `/event`, `/guest`, `/home`, `/incident`, `/kupones`, `/list`, `/maintenance`, `/method`, `/payment`, `/pet`, `/poll`, `/profile`, `/profile-edit`, `/push-diagnostic`, `/setup`, `/summary`, `/type`, `/vehicle`, `/zone`.
- **Permisos Android relevantes:** `INTERNET`, `READ/WRITE_CONTACTS`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`, `USE_BIOMETRIC`/`USE_FINGERPRINT`, `FOREGROUND_SERVICE`.
- **Diferencias deliberadas de Caltzi:** sin backend ni red, sin SDKs de terceros (Firebase/OneSignal), sin permisos de contactos salvo que se habilite "invitar desde agenda", y cifrado real de la base local.

## Anexo B — Convenciones de código

- Nombres de dominio en **español** (`Visitante`, `Cargo`, `Reservacion`) y de infraestructura en inglés (`Repository`, `DataSource`).
- Sin lógica en widgets: toda regla en `domain/services`.
- Un archivo por entidad, `freezed` para inmutabilidad, `drift` para consultas.
- IDs **UUID v7** generados en cliente (compatibilidad futura con PocketBase).
- Fechas siempre UTC en BD; formateo con `intl` en la capa de presentación.
