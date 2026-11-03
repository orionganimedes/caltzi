# Caltzi

App de gestión residencial **local-first** en Flutter (Android, iOS; Web/Desktop como demo), con asistente conversacional integrado **Caltzibot**.

> Plan completo: [`../PLAN-CALTZI.md`](../PLAN-CALTZI.md)

- `appId`: `com.caltzi.app`
- Sin backend en la v1: todo persiste en SQLite cifrado (drift + SQLCipher).
- Capa de datos abstracta para conectar **PocketBase** en la fase 8 sin tocar la UI.
- Sin permiso `INTERNET` en release.

---

## Estructura de primer nivel

```
caltzi/
├── android/ ios/ web/ linux/ macos/ windows/   # generados por `flutter create .`
├── assets/            fuentes, iconos, imágenes, Lottie, datos demo y del bot
├── docs/              arquitectura, diseño, producto, QA, PocketBase, operación
├── lib/               código Dart (core + features + shared)
├── test/              unit, widget, golden, fixtures, helpers
├── integration_test/  flujos E2E con Patrol
├── tool/              scripts de build, l10n, seed y utilidades
└── .github/           CI y plantillas de issues
```

## `lib/`

```
lib/
├── main.dart            # entrada mínima: bootstrap().then(runApp)
├── app.dart             # MaterialApp.router, tema, localizaciones
├── bootstrap.dart       # DI, BD, seguridad, notificaciones
├── core/                # infraestructura transversal
│   ├── config/          # flavors, env, rutas de almacenamiento
│   ├── constants/       # valores estables (duraciones, límites)
│   ├── db/              # drift: schema, daos, migrations, seed
│   ├── di/              # providers globales (Riverpod)
│   ├── error/           # Result, Failure, mapeo de excepciones
│   ├── l10n/            # ARB es/en y delegates
│   ├── logging/         # logs locales con niveles
│   ├── notify/          # notificaciones locales + programación
│   ├── permissions/     # cámara, fotos, contactos, biometría
│   ├── router/          # go_router + guards de sesión
│   ├── security/        # Argon2id, Keystore, HMAC, bloqueo
│   ├── storage/         # archivos adjuntos, compresión de imágenes
│   ├── sync/            # contratos y outbox (fase PocketBase)
│   ├── theme/           # tokens de Caltzi, tipografía, componentes
│   ├── utils/           # fechas UTC, moneda en centavos, IDs UUID v7
│   ├── validation/      # validadores de formularios reutilizables
│   └── widgets/         # design system compartido
├── features/            # un módulo por dominio de negocio
└── shared/              # modelos, extensiones, mixins y formateadores
```

### Anatomía de cada `feature`

```
features/<modulo>/
├── data/
│   ├── datasources/     # LocalXDataSource (drift), futuras PocketBase
│   ├── models/          # DTOs con freezed + json_serializable
│   └── repositories/    # XRepositoryImpl
├── domain/
│   ├── entities/        # entidades puras (Visitante, Cargo, Reservacion…)
│   ├── repositories/    # interfaces (contrato de migración)
│   ├── services/        # reglas de negocio (QR, solapes, saldos)
│   └── usecases/        # casos de uso por acción
└── presentation/
    ├── pages/           # pantallas
    ├── widgets/         # widgets del módulo
    ├── controllers/     # Notifier / AsyncNotifier
    └── providers/       # providers Riverpod
```

### Módulos

| Módulo | Contenido |
|---|---|
| `auth` | PIN, biometría, sesión, roles, bloqueo por intentos |
| `onboarding` | Solicitud de acceso, aprobación local, datos de casa |
| `home` | Panel con saldo, visitas activas, próxima reservación, avisos |
| `access` | Visitantes, tarjetas de acceso, empleados, bitácora, switch de visitas |
| `qr` | Generación/validación HMAC del QR de visitante |
| `payments` | Cargos, estado de cuenta, comprobantes, saldo |
| `bookings` | Áreas comunes, calendario, solapes, aprobación |
| `announcements` | Comunicados con adjuntos y enlaces |
| `incidents` | Reportes con categoría y fotos |
| `suggestions` | Quejas y sugerencias |
| `polls` | Encuestas con voto único |
| `classifieds` | Clasificados con vigencia |
| `documents` | Documentos descargables/abribles |
| `kupones` | Beneficios y cupones |
| `profile` | Datos del residente y de la casa |
| `pets` / `vehicles` | Registro de mascotas y vehículos |
| `notifications` | Bandeja local y recordatorios programados |
| `settings` | Idioma, tema, cámara, respaldo |
| `backup` | Export/import `.caltzi` cifrado |
| **`caltzibot`** | **Asistente conversacional con IA** (ver su README) |

## Convenciones

- Dominio en **español** (`Visitante`, `Cargo`, `Reservacion`); infraestructura en **inglés** (`Repository`, `DataSource`).
- Nada de lógica en widgets: toda regla vive en `domain/services` o `domain/usecases`.
- Montos siempre en **centavos enteros**; fechas en **UTC ISO-8601**; IDs **UUID v7** generados en cliente.
- Toda escritura deja `updated_at` y `rev` para la futura sincronización con PocketBase.
- Cada módulo nace con tests en `test/unit/<modulo>/` y, si tiene pantalla, en `test/widget/<modulo>/`.

## Puesta en marcha (siguiente paso)

```bash
cd caltzi
flutter create . --org com.caltzi --project-name caltzi \
  --platforms=android,ios,web,windows,macos,linux
flutter pub get
```

`flutter create .` añade las carpetas de plataforma y no sobrescribe `lib/`, `assets/`, `docs/` ni `test/`.
