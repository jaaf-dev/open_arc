# OpenArc + Open WebUI

Stack local de inferencia con [OpenArc](https://searchsavior.github.io/OpenArc/) (OpenVINO, API compatible con OpenAI) y [Open WebUI](https://github.com/open-webui/open-webui) como interfaz de chat.

Los modelos **no se versionan en git**: se descargan dentro del contenedor y persisten en `./openarc_models/`.

## Requisitos

- Docker y Docker Compose
- GPU Intel con soporte OpenVINO (p. ej. Arc A750) **o** CPU
- Acceso a `/dev/dri` en el host (ya montado en el compose)
- ~10 GB libres en disco por modelo INT4 pequeño/mediano

## Estructura del proyecto

```
open_arc/
├── docker-compose.yml      # Stack OpenArc + Open WebUI
├── .env.example            # Plantilla de variables
├── .env                    # Secretos locales (no se sube a git)
├── openarc_models/         # Modelos descargados → /models en el contenedor
├── openarc_data/           # Config OpenArc → /persist (openarc_config.json)
└── open_webui_data/        # Datos de Open WebUI (usuarios, chats, RAG)
```

## Puesta en marcha

### 1. Clonar y configurar entorno

```bash
git clone https://github.com/jaaf-dev/open_arc.git
cd open_arc

cp .env.example .env
# Editar .env con claves seguras (OPENARC_API_KEY, OPENAI_API_KEY, WEBUI_SECRET_KEY)
```

> `OPENAI_API_KEY` en `.env` debe coincidir con `OPENARC_API_KEY` para que Open WebUI se conecte a OpenArc.

### 2. Levantar el stack

```bash
docker compose up -d
```

Servicios:

| Servicio   | URL                         |
|------------|-----------------------------|
| OpenArc    | http://localhost:8000/v1    |
| Open WebUI | http://localhost:3000       |

### 3. Verificar que OpenArc responde

```bash
curl http://localhost:8000/v1/models
```

---

## Gestión de modelos

Todo se hace **dentro del contenedor** (no hace falta instalar nada en el host).

```bash
docker exec -it openarc_server bash
```

Documentación oficial:

- [Modelos recomendados](https://searchsavior.github.io/OpenArc/models/)
- [Comandos OpenArc](https://searchsavior.github.io/OpenArc/commands/)

### Detectar dispositivos disponibles

```bash
openarc tool device-detect
```

Usa `GPU`, `CPU` o `HETERO:GPU,CPU` según el resultado.

---

### Descargar un modelo (Hugging Face)

OpenArc requiere modelos **convertidos a OpenVINO** (archivos `openvino_model.bin`, `openvino_model.xml`, etc.).

Fuentes habituales:

- [Echo9Zulu en Hugging Face](https://huggingface.co/Echo9Zulu)
- [Modelos OpenVINO en HF](https://huggingface.co/models?library=openvino)

#### Ejemplo: Phi-4-mini-instruct (chat general, ~2.5 GB)

```bash
huggingface-cli download Echo9Zulu/Phi-4-mini-instruct-OpenVINO \
  --include "Phi-4-mini-instruct-int4_asym-awq-se-ov/*" \
  --local-dir /models/phi4-mini
```

#### Ejemplo: Qwen3-4B-Instruct (~2.5 GB)

```bash
huggingface-cli download Echo9Zulu/Qwen3-4B-Instruct-2507-int4_asym-awq-ov \
  --local-dir /models/qwen3-4b
```

#### Ejemplo: NextCoder 11B (código, ~5.8 GB — requiere ~8 GB VRAM)

```bash
huggingface-cli download Echo9Zulu/Qwen2.5-Microsoft-NextCoder-Soar-Instruct-FUSED-CODER-Fast-11B-int4_asym-awq-ov \
  --local-dir /models/nextcoder-11b
```

Si el repositorio es privado o pide login:

```bash
huggingface-cli login
```

Alternativa con Python:

```bash
python - <<'EOF'
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="Echo9Zulu/Phi-4-mini-instruct-OpenVINO",
    allow_patterns=["Phi-4-mini-instruct-int4_asym-awq-se-ov/*"],
    local_dir="/models/phi4-mini",
)
print("Descarga completa")
EOF
```

---

### Registrar un modelo (`openarc add`)

Guarda la configuración en `openarc_config.json` (persiste en `./openarc_data/`).

#### LLM en GPU

```bash
openarc add \
  --model-name phi4-mini \
  --model-path /models/phi4-mini/Phi-4-mini-instruct-int4_asym-awq-se-ov \
  --engine ovgenai \
  --model-type llm \
  --device GPU
```

#### LLM en CPU

```bash
openarc add \
  --model-name phi4-mini \
  --model-path /models/phi4-mini/Phi-4-mini-instruct-int4_asym-awq-se-ov \
  --engine ovgenai \
  --model-type llm \
  --device CPU
```

#### Modelo grande (GPU + RAM del sistema)

Para modelos que no caben enteros en VRAM (p. ej. 14B en GPU de 8 GB):

```bash
openarc add \
  --model-name nouscoder-14b \
  --model-path /models/nouscoder-14b \
  --engine ovgenai \
  --model-type llm \
  --device HETERO:GPU,CPU \
  --runtime-config '{"MODEL_DISTRIBUTION_POLICY": "PIPELINE_PARALLEL"}'
```

#### Otros tipos de modelo

| Tipo        | `--model-type`   | `--engine`  |
|-------------|------------------|-------------|
| LLM         | `llm`            | `ovgenai`   |
| VLM (visión)| `vlm`            | `ovgenai`   |
| Whisper     | `whisper`        | `ovgenai`   |
| Kokoro TTS  | `kokoro`         | `openvino`  |

Consulta la [documentación de comandos](https://searchsavior.github.io/OpenArc/commands/) para TTS, ASR, embeddings, etc.

---

### Cargar y gestionar modelos

```bash
# Cargar uno o varios modelos en memoria
openarc load phi4-mini
openarc load phi4-mini nextcoder-11b

# Listar modelos registrados
openarc list
openarc list phi4-mini -v

# Ver modelos cargados
openarc status

# Benchmark (LLM)
openarc bench phi4-mini

# Eliminar registro de un modelo
openarc list --remove phi4-mini
```

> Cargar varios LLM grandes a la vez consume mucha VRAM. En GPUs de 8 GB conviene tener **solo uno cargado**.

---

### Autocarga al arrancar

En `.env`, define el nombre usado en `openarc add`:

```env
OPENARC_AUTOLOAD_MODEL=phi4-mini
```

Tras reiniciar el contenedor, OpenArc intentará cargar ese modelo automáticamente.

---

## Probar la API

```bash
curl http://localhost:8000/v1/models

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TU_OPENARC_API_KEY" \
  -d '{
    "model": "phi4-mini",
    "messages": [{"role": "user", "content": "Hola"}],
    "max_tokens": 50
  }'
```

---

## Open WebUI

1. Abre http://localhost:3000
2. Crea tu cuenta (primera visita)
3. Selecciona el modelo cargado en OpenArc (p. ej. `phi4-mini`)

Si no aparece el modelo:

```bash
docker restart open_webui
docker exec openarc_server openarc status
```

---

## Comandos útiles del día a día

```bash
# Logs
docker compose logs -f openarc
docker compose logs -f open-webui

# Reiniciar stack
docker compose restart

# Parar stack
docker compose down

# Entrar al contenedor OpenArc
docker exec -it openarc_server bash
```

---

## Referencia rápida: flujo completo de un modelo nuevo

Sustituye `NOMBRE`, `REPO_HF`, `RUTA_LOCAL` y `RUTA_DENTRO` según el modelo elegido.

```bash
docker exec -it openarc_server bash

# 1. Descargar
huggingface-cli download REPO_HF --local-dir /models/RUTA_LOCAL

# 2. Registrar
openarc add \
  --model-name NOMBRE \
  --model-path /models/RUTA_DENTRO \
  --engine ovgenai \
  --model-type llm \
  --device GPU

# 3. Cargar
openarc load NOMBRE

# 4. Comprobar
openarc status
```

---

## Notas sobre VRAM

- Al **cargar** el modelo se ocupa una base fija de VRAM (pesos del modelo).
- Cada **prompt largo** o **historial de chat** añade KV cache y aumenta el uso de VRAM.
- Los modelos descargados viven en **disco** (`./openarc_models/`); la VRAM solo se usa durante la inferencia.
- Para chats largos sin llenar VRAM: usa chats nuevos, limita el contexto en Open WebUI o RAG con base de conocimiento.

---

## Enlaces

- [OpenArc — documentación](https://searchsavior.github.io/OpenArc/)
- [OpenArc — modelos](https://searchsavior.github.io/OpenArc/models/)
- [OpenArc — comandos](https://searchsavior.github.io/OpenArc/commands/)
- [OpenArc — Docker](https://searchsavior.github.io/OpenArc/install/#docker)
- [Repositorio OpenArc](https://github.com/searchsavior/OpenArc)
