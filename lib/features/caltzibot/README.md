# Caltzibot — asistente conversacional de Caltzi

Chatbot integrado en la app que responde en lenguaje natural sobre la información del residente y ejecuta acciones del propio sistema (consultar saldo, crear una visita, revisar comunicados…).

## Principios

1. **Local-first.** Funciona sin red. El motor por defecto es determinista (intenciones + plantillas + datos de SQLite); si se configura un proveedor LLM, se usa como mejora, nunca como requisito.
2. **Privacidad.** Ningún dato sale del dispositivo en la v1. Cuando se conecte PocketBase, el adaptador remoto será explícito y desactivable.
3. **Sin alucinaciones sobre datos.** El bot únicamente afirma hechos que provienen de consultas tipadas a los repositorios. Si no hay dato, responde que no lo tiene.
4. **Acciones con confirmación.** Toda acción que escribe (crear visita, reservar) pasa por una tarjeta de confirmación y por los mismos servicios de dominio que usa la UI.

## Capas

```
caltzibot/
├── data/
│   ├── datasources/     local_bot_datasource (SQLite), prompt_local_datasource,
│   │                    remote_llm_datasource (opcional, futura)
│   ├── models/          mensaje_model, conversacion_model, contexto_model
│   └── repositories/    caltzibot_repository_impl
├── domain/
│   ├── entities/        Mensaje, Conversacion, Intencion, Herramienta
│   ├── repositories/    CaltzibotRepository
│   ├── services/        intent_classifier, response_composer, tool_router,
│   │                    prompt_builder, safety_guard, context_builder
│   └── usecases/        enviar_mensaje, crear_conversacion, cargar_historial,
│                        resumir_contexto, ejecutar_accion
├── presentation/
│   ├── pages/           chat_page, historial_page, ajustes_bot_page
│   ├── widgets/         burbuja_mensaje, barra_entrada, sugerencias_chips,
│   │                    indicador_escribiendo, tarjeta_accion, cita_fuente
│   ├── controllers/     chat_controller, historial_controller
│   └── providers/       caltzibot_providers
└── engine/
    ├── llm/             llm_client, llm_models
    │   └── adapters/    local_rule_adapter (default), llama_local_adapter,
    │                    openai_adapter, gemini_adapter, pocketbase_proxy_adapter
    ├── prompts/         system_prompt_es.md, system_prompt_en.md, templates/
    ├── tools/           tool_registry + tools tipadas
    ├── memory/          memoria_corta (ventana), memoria_larga (resúmenes)
    └── context/         snapshot del residente (casa, saldo, visitas, avisos)
```

## Herramientas (tools) previstas

| Tool | Lee/escribe | Fuente |
|---|---|---|
| `consultar_saldo` | lectura | `PagosRepository` |
| `listar_cargos` | lectura | `PagosRepository` |
| `listar_visitantes_activos` | lectura | `VisitantesRepository` |
| `crear_visita` | escritura (con confirmación) | `VisitantesRepository` + `QrService` |
| `consultar_reservaciones` | lectura | `ReservacionesRepository` |
| `crear_reservacion` | escritura (con confirmación) | `ReservaService` |
| `resumir_comunicados` | lectura | `ComunicadosRepository` |
| `buscar_documento` | lectura | `DocumentosRepository` |
| `estado_cuenta_casa` | lectura | `PagosRepository` + `CasasRepository` |

El registro es explícito: el bot no puede inventar herramientas ni ejecutar SQL.

## Motor por defecto (sin LLM)

1. **Normalización** del texto (minúsculas, acentos, sinónimos del residencial).
2. **Clasificador de intención** por reglas + embeddings ligeros locales (`assets/bot/intents/*.json`) con umbral de confianza.
3. **Planificación**: intención → tool → argumentos → ejecución tipada.
4. **Composición** de la respuesta con plantillas ARB (es/en) y **citas de fuente** (módulo + registro).
5. **Fallback** a "no entendí" con chips de sugerencia; nunca inventa datos.

## Memoria

- **Corta:** últimas N interacciones de la conversación activa.
- **Larga:** resúmenes guardados en `conversaciones.resumen` para no reenviar historial completo.
- **Contexto:** snapshot de solo lectura del residente (casa, saldo, visitas activas), regenerado por mensaje.

## Seguridad

- `safety_guard` bloquea temas fuera de alcance y datos de otras casas/residenciales.
- Las herramientas reciben el `perfilId` de la sesión: el filtro por casa se aplica en el repositorio, no en el prompt.
- Historial local cifrado con la misma BD; opción de borrar conversaciones desde ajustes.
- Si se habilita un LLM remoto, se envían solo los campos necesarios, con aviso explícito y opción de desactivar.

## Recursos

- `assets/bot/prompts/` — prompts de sistema por idioma.
- `assets/bot/intents/` — catálogo de intenciones, sinónimos y ejemplos.
- `assets/bot/faq/` — respuestas canónicas de preguntas frecuentes.
- `assets/bot/examples/` — conversaciones de referencia para pruebas (golden del bot).
