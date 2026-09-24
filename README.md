# Marketing Content Agent — generación y publicación de contenido con aprobación humana

Sistema de dos escenarios en [Make.com](https://make.com) que escribe, diseña (con una persona humana) y publica el contenido de Instagram y Facebook de [@adantonlabs](https://instagram.com/adantonlabs), con un humano en el loop antes de que nada salga a producción.

No es "IA publicando sola": es un flujo donde la IA hace el trabajo repetitivo (escribir, no repetirse, mantener el tono de marca) y una persona sigue decidiendo qué diseño se usa y cuándo se aprueba la publicación, todo coordinado por Telegram.

## Los dos escenarios

### A. Generar borrador diario (corre todos los días a las 09:00)

1. Revisa si ya hay un post generado esperando diseño o aprobación.
   - Si hay uno **vencido** (más de 20 horas sin moverse), lo descarta automáticamente, libera el slot y avisa por Telegram — así el sistema nunca se traba esperando algo que quedó colgado.
   - Si hay uno **vigente**, no genera uno nuevo y avisa que hay que terminar el anterior primero.
   - Si no hay nada pendiente, genera un post nuevo.
2. Para generar: junta los pilares y copys de los últimos posts ya *publicados* (no los descartados) y se los pasa a Claude con una regla explícita de no repetir ángulo, ejemplo o frase de apertura.
3. Claude devuelve un JSON estructurado con: si hay contenido genuinamente nuevo o no (puede decidir que no, y el sistema lo respeta en vez de forzar un post reciclado), el pilar elegido, el copy de Instagram, el copy de Facebook, y un brief visual en español para quien va a diseñar la pieza.
4. Guarda el borrador en Google Sheets, lo deja en un Data Store como "pendiente" y manda el brief de diseño por Telegram.

### A2. Recibir diseño manual (se dispara al instante, vía webhook de Telegram)

1. Cuando llega una foto por Telegram (el diseño ya armado), la reenvía junto con los dos copys y un botón inline de "✅ Aprobar y publicar".
2. Cuando se aprieta ese botón: descarga la foto, la sube a la Página de Facebook, recupera la URL pública de la foto recién subida en Facebook, y la usa para publicar en Instagram Business con el copy correspondiente.
3. Actualiza la fila en Google Sheets a "publicado" con los links reales, y borra el registro "pendiente" del Data Store.
4. Si falla cualquier paso (descarga, subida a Facebook, publicación en Instagram, actualización de la hoja), el escenario **no reintenta a ciegas**: marca la fila como error con el motivo puntual y avisa por Telegram exactamente en qué paso falló — incluyendo el caso de "se publicó en Facebook pero falló en Instagram", para que quede clarísimo qué sí salió y qué no.

## Por qué está diseñado así (y no 100% automático)

El cuello de botella real no es escribir el copy — es el diseño de la imagen, que todavía lo hace una persona. En vez de forzar generación de imágenes con IA (que en 2026 sigue sin igualar un diseño hecho a mano para una marca chica), el sistema automatiza todo lo que rodea a esa tarea humana: no repetirse, no dejar un post encajado en la cola, no publicar sin aprobación explícita, y no fallar en silencio si algo del lado de las APIs de Meta se rompe a mitad de camino.

## El prompt de generación de contenido

Ver [`docs/ai-prompt-content-generation.md`](docs/ai-prompt-content-generation.md) — el prompt real usado en el escenario A, con la definición de tono de marca, los 3 pilares de contenido y la estrategia de hashtags para Instagram en 2026.

## Arquitectura

```
A. Generar borrador diario (09:00, cron)
  └─ ¿Hay un post pendiente? (Data Store)
       ├─ Sí, vencido (>20h)   → descartar fila, liberar slot, avisar por Telegram
       ├─ Sí, vigente          → avisar que hay que terminarlo, no generar nada nuevo
       └─ No                   → generar:
            └─ Traer pilares + copys de posts YA publicados (Google Sheets)
            └─ Claude: generar copy IG + FB + brief visual (o decidir que no hay nada nuevo)
            └─ Guardar fila "esperando_diseno" en Sheets + registro "pending" en Data Store
            └─ Enviar brief de diseño por Telegram

A2. Recibir diseño manual (webhook Telegram, instantáneo)
  └─ Llega una foto → reenviarla + copys + botón "Aprobar y publicar"
  └─ Se aprueba →
       └─ Descargar foto de Telegram
       └─ Subir a Facebook Page (UploadPhoto)
       └─ Recuperar URL pública de la foto (GetPhoto)
       └─ Publicar en Instagram Business (CreatePostPhoto) usando esa URL
       └─ Marcar fila como "publicado" con los links reales
     (en cada paso: si falla, marcar error puntual + avisar por Telegram, sin reintentar a ciegas)
```

## Stack

- **Make.com**: orquestación, Data Store para estado "pendiente" de un solo post en vuelo, routing por condición y por tiempo transcurrido.
- **Claude (Anthropic API)**: redacción de copy con salida JSON estructurada y control de no-repetición contra publicaciones anteriores.
- **Telegram Bot API**: interfaz humana — recibir el diseño, aprobar con un botón, avisos de error.
- **Meta Graph API** (vía módulos nativos de Make): `facebook-pages` para subir la foto y publicar, `instagram-business` para publicar en Instagram usando la foto ya alojada en Facebook.
- **Google Sheets**: registro histórico de posts (pilar, copys, estado, links de publicación).

## Notas de auditoría honesta

- Los IDs reales (spreadsheet, chat de Telegram, Página de Facebook, cuenta de Instagram Business) fueron reemplazados por placeholders en la documentación de este repo. La lógica es la del escenario en producción.
- No hay generación de imágenes por IA todavía — es la pieza manual del flujo a propósito, no una limitación técnica sin resolver.
- El segundo escenario (A2) no tiene scheduling propio: se dispara por webhook de Telegram, por eso no aparece como "activo" en un listado que solo mire triggers programados.

## Por Andrés Serrano

Parte de una fábrica de agentes de IA para pymes ([@adantonlabs](https://instagram.com/adantonlabs)). Ver también: [crm-agent-inmobiliaria-portfolio](https://github.com/andresjsa25/crm-agent-inmobiliaria-portfolio) (agente de calificación de leads para inmobiliarias) y [lead-prospecting-agent-portfolio](https://github.com/andresjsa25/lead-prospecting-agent-portfolio) (agente de prospección de leads multi-fuente).
