# n8n RAG – Asistente interno (PROINGES S.A.S.)

Este repositorio contiene un flujo n8n que implementa un sistema RAG (Retrieval-Augmented Generation) para responder preguntas en Telegram utilizando documentos almacenados en Google Drive, vectorizados en Postgres (Neon) con PGVector y consultados por un agente de IA (OpenAI).

- Ingesta: Google Drive → n8n → Embeddings OpenAI (`text-embedding-3-large`) → Postgres PGVector
- Consulta: Telegram → Agente IA (`gpt-4.1-mini`) + Tool de búsqueda en vector store → Respuesta al usuario
- Trazabilidad: Tabla `ingested_files` para saber qué documentos ya se cargaron y si cambiaron
- Aclaración importante: n8n no crea el campo `file_id` en la tabla de vectores. Este campo debe agregarse manualmente porque el flujo lo utiliza para relacionar chunks con su archivo origen.

> Diagrama general del flujo (referencia visual): ver imagen adjunta en la conversación.

---

## 1) Arquitectura del flujo

1. Selección de documentos (azul)
   - Triggers: Manual + Schedule
   - Google Drive: lista archivos de una carpeta (excluye directorios, ignora papelera)
   - Verificación en Postgres: se consulta `ingested_files` usando `file_id` + `md5_checksum` para saber si el archivo ya fue procesado
   - Si ya existe y no cambió → se descarta
   - Si cambió → se borran sus vectores previos y se reingesta

2. Alimentar base de datos (amarillo)
   - Descarga el archivo de Google Drive
   - Carga y troceo del texto: `Recursive Character Text Splitter` con `chunkSize: 1000` y `chunkOverlap: 200–250` para mantener contexto entre bloques
   - Embeddings: `text-embedding-3-large` (óptimo para documentos grandes)
   - Almacenamiento: `Postgres PGVector Store` con metadatos enriquecidos
   - Relación `file_id`: posterior `UPDATE` para asociar cada chunk con su archivo original

3. Consumo de base de datos (verde)
   - Agente IA (`AI Agent`): modelo `gpt-4.1-mini`
   - Memoria: `Simple Memory` (ventana de contexto = 10)
   - Tool: “Answer questions with a vector store” → consulta semántica sobre la misma base Postgres PGVector
   - El modelo del Tool también es `gpt-4.1-mini` y utiliza embeddings `text-embedding-3-large` para una lectura/recuperación consistente

4. Conexión y consultas (rojo)
   - Bot de Telegram: `Telegram Trigger` da la bienvenida en `/start`
   - Normalización de texto: se limpia y estandariza la consulta del usuario (acentos, ñ, etc.)
   - Se envía la consulta al Agente IA del punto 3 y se formatea la respuesta para Telegram (HTML seguro)
   - Si no hay respuesta relevante: el bot devuelve un mensaje claro de “no encontré información”

---

## 2) ¿Por qué una base de datos vectorial?

- Búsqueda semántica: Los embeddings convierten el texto en vectores. Consultar “por aproximación” (similaridad) permite encontrar fragmentos relevantes aunque el usuario no escriba las palabras exactas.
- Escalabilidad: Cuando crece el volumen de documentos, la búsqueda tradicional por palabras clave se vuelve limitada. La búsqueda vectorial mantiene buena precisión y rendimiento, especialmente con índices adecuados.
- Contexto para LLMs: La recuperación de los chunks más similares permite construir prompts con evidencia, reduciendo alucinaciones y mejorando la precisión de las respuestas.

En este flujo, PGVector almacena:
- `text`: contenido del chunk
- `metadata`: detalles como `loc` (rango de líneas, nombre de archivo, etc.)
- `embedding`: vector numérico del chunk
- `file_id`: relación directa con el archivo fuente en Google Drive

---

## 3) Esquema de base de datos (Neon + PGVector)

Asegura que las extensiones estén disponibles (en Neon suelen estar listas):
```sql
-- Extensiones recomendadas
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pgcrypto; -- para gen_random_uuid()
```

Tabla de vectores (con `file_id`):
```sql
-- Tabla para almacenar chunks vectorizados
CREATE TABLE IF NOT EXISTS public.n8n_vectors (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  text text,
  metadata jsonb,
  embedding vector,
  file_id text
);

-- Índices recomendados (opcionales)
-- Si usas pgvector >= 0.5 puedes crear un índice ivfflat para acelerar búsquedas aproximadas
-- Nota: Ajusta el 'lists' según el tamaño de tu dataset y setea 'SET ivfflat.probes' en consultas críticas
-- CREATE INDEX IF NOT EXISTS n8n_vectors_embedding_idx
--   ON public.n8n_vectors
--   USING ivfflat (embedding vector_cosine_ops);
```

Tabla de control de documentos:
```sql
-- Tabla para controlar qué documentos ya se ingirieron y si cambiaron
CREATE TABLE IF NOT EXISTS public.ingested_files (
  file_id text PRIMARY KEY,
  name text,
  mime_type text,
  md5_checksum text,
  modified_time timestamptz,
  last_ingested timestamptz DEFAULT now()
);

-- Índices adicionales (opcionales)
-- CREATE INDEX IF NOT EXISTS ingested_files_modified_time_idx ON public.ingested_files (modified_time DESC);
```

Consultas útiles:
```sql
-- Verificar si un archivo específico ya fue ingestado sin cambios
SELECT count(*) AS ingested
FROM public.ingested_files
WHERE file_id = '{{ $json.id }}' AND md5_checksum = '{{ $json.md5Checksum }}';

-- Borrar los vectores de un archivo antes de reingestar (en caso de cambios)
DELETE FROM public.n8n_vectors
WHERE file_id = '{{ $json.id }}';

-- Asociar cada chunk con su file_id después de insertar embeddings
UPDATE public.n8n_vectors
SET file_id = '{{ $('Extraer variables').item.json.id }}'
WHERE metadata->'loc'->'lines' = '{{ $json.metadata.loc.lines.toJsonString() }}'
  AND file_id IS NULL;
```

---

## 4) Nodos con JavaScript (ejemplos usados en el flujo)

1) Normalización de entrada de Telegram
```ts
// n8n Code node (TypeScript válido en JS node)
// - Normaliza el texto del usuario: elimina tildes, maneja "ñ", limpia espacios
// - Convierte a minúsculas para consultas robustas en la búsqueda vectorial
let text: string = $('Telegram Trigger').first().json.message.text || '';

text = text
  .normalize('NFD')
  .replace(/[\u0300-\u036f]/g, '')     // elimina tildes
  .replace(/ñ/gi, 'n')                 // reemplaza ñ por n
  .replace(/[^\x00-\x7F]/g, ' ')       // elimina caracteres no ASCII
  .replace(/\s+/g, ' ')                // colapsa espacios múltiples
  .trim()
  .toLowerCase();

return [{ json: { text } }];
```

2) Sanitización de respuesta para Telegram (HTML)
```ts
// n8n Code node (TypeScript válido en JS node)
// - Telegram con parse_mode=HTML solo admite tags específicas
// - Este nodo limpia la salida del LLM y deja un subconjunto seguro de HTML
// - Si no hay contenido, retorna un mensaje explícito

const allowedTags = new Set(['b', 'strong', 'i', 'em', 'u', 's', 'code', 'pre', 'a', 'br']);
const sanitizeHtml = (raw: string): string => {
  if (!raw) return 'No se encontró información relevante.';

  // Reemplaza Markdown básico a HTML ligero (opcional, heurístico)
  let html = raw
    .replace(/^###\s+(.*)$/gm, '<b>$1</b>')
    .replace(/^##\s+(.*)$/gm, '<b>$1</b>')
    .replace(/^#\s+(.*)$/gm, '<b>$1</b>')
    .replace(/\*\*(.+?)\*\*/g, '<b>$1</b>')
    .replace(/\*(.+?)\*/g, '<i>$1</i>')
    .replace(/`{3}([\s\S]*?)`{3}/g, '<pre><code>$1</code></pre>')
    .replace(/`([^`]+)`/g, '<code>$1</code>')
    .replace(/\n{2,}/g, '<br><br>')
    .trim();

  // Elimina cualquier etiqueta no permitida
  html = html.replace(/<\/?([a-z0-9]+)(\s[^>]*)?>/gi, (match, tag) => {
    return allowedTags.has(tag.toLowerCase()) ? match : '';
  });

  // Asegura que <a> tenga href válido (simple)
  html = html.replace(/<a\s+[^>]*href="([^"]+)"[^>]*>(.*?)<\/a>/gi, (m, href, text) => {
    try {
      const u = new URL(href);
      return `<a href="${u.toString()}">${text}</a>`;
    } catch {
      return text; // si no es URL válida, deja solo el texto
    }
  });

  return html || 'No se encontró información relevante.';
};

const items = $input.all().map((item) => {
  const output = item.json.output ?? item.json.text ?? '';
  return { json: { sanitizedText: sanitizeHtml(String(output)) } };
});

return items;
```

---

## 5) Configuración y requisitos

- Credenciales en n8n:
  - OpenAI: clave para `gpt-4.1-mini` y `text-embedding-3-large`
  - Postgres (Neon): cadena de conexión; base con extensión `vector` habilitada
  - Google Drive OAuth2: acceso de solo lectura a carpeta de documentos
  - Telegram Bot: token del bot

- Parámetros relevantes del flujo:
  - Text Splitter: `chunkSize: 100`, `chunkOverlap: 200–250` (para mantener contexto)
  - Simple Memory: `contextWindowLength: 10`
  - Carpeta de Google Drive: se filtran solo archivos (no carpetas), ignorando `trashed`
  - Dedupe: `ingested_files` con `file_id` + `md5_checksum`
  - Reingesta: si cambió `md5`, se borran vectores vinculados (`file_id`) y se reprocesa

- Variables de entorno (ejemplo):
```bash
# OpenAI
OPENAI_API_KEY=...

# Postgres (Neon)
PGHOST=...
PGUSER=...
PGPASSWORD=...
PGDATABASE=...
PGPORT=5432
PGSSL=true

# Telegram
TELEGRAM_BOT_TOKEN=...

# Google Drive (n8n maneja OAuth, no expongas secretos en el repo)
```

---

## 6) Importar y ejecutar

1. Importa el workflow n8n:
   - Archivo: [RAG - PRUEBA TECNICA.json](https://github.com/fabiofruto88/prueba-tenica-n8n-2025/blob/main/RAG%20-%20PRUEBA%20TECNICA.json)
2. Crea las tablas en Postgres (Neon) con el esquema de la sección 3.
3. Configura las credenciales de OpenAI, Postgres, Google Drive y Telegram en n8n.
4. Ajusta el ID de carpeta de Drive y cualquier regla de filtrado.
5. Ejecuta el trigger manual o deja activo el `Schedule Trigger`.
6. Chatea con el bot en Telegram:
   - Envía `/start` para la bienvenida
   - Envía preguntas en lenguaje natural; el bot responderá con contexto de los documentos.

Archivos de ejemplo (datos de muestra exportados):
- [n8n_vectors.json](https://github.com/fabiofruto88/prueba-tenica-n8n-2025/blob/main/n8n_vectors.json)
- [ingested_files.json](https://github.com/fabiofruto88/prueba-tenica-n8n-2025/blob/main/ingested_files.json)

---

## 7) Buenas prácticas y notas

- Consistencia de embeddings: usar el mismo modelo (`text-embedding-3-large`) para indexar y consultar asegura mayor precisión.
- Índices vectoriales: para grandes volúmenes, evalúa `ivfflat` con métrica cosine y ajusta `probes`.
- Seguridad: no compartas claves o tokens en el repo. Usa credenciales seguras en n8n.
- Observabilidad: registra `last_ingested` en `ingested_files` para auditar cargas y detectar cambios.

---

## 8) Solución de problemas

- “No existe la columna file_id en n8n_vectors”: agrega el campo según el esquema de la sección 3.
- “No se encuentran resultados”: verifica que:
  - El archivo fue descargado, troceado y vectorizado
  - Los embeddings se guardaron y el `file_id` fue actualizado con el `UPDATE`
  - La carpeta de Drive es correcta y `includeTrashed=false`
- Errores de Telegram al enviar HTML: la sanitización elimina tags no soportados.

---

## 9) Licencia

Uso interno para PROINGES S.A.S. Ajusta según las políticas de tu organización.
