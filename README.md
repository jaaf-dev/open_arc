# OpenArc + Open WebUI — IA local en Intel

Guía rápida para montar inferencia local con [OpenArc](https://searchsavior.github.io/OpenArc/) (OpenVINO, API compatible con OpenAI) y [Open WebUI](https://github.com/open-webui/open-webui) en entornos **Intel** (Arc, iGPU o CPU).

**Modelo recomendado y probado:** [Qwen3-4B-Instruct-2507](https://huggingface.co/Echo9Zulu/Qwen3-4B-Instruct-2507-int4_asym-awq-ov) (~2.5 GB INT4, cómodo en Arc A750 8 GB).

Los pesos del modelo **no se versionan en git**: se descargan dentro del contenedor y persisten en `./openarc_models/`.

## Requisitos

- Docker y Docker Compose
- GPU Intel con OpenVINO (p. ej. **Arc A750**) **o** CPU
- `/dev/dri` accesible en el host (montado en el compose)
- ~5 GB libres en disco para Qwen3-4B INT4 (~10 GB si añades más modelos)

## Estructura del proyecto

```
open_arc/
├── docker-compose.yml      # OpenArc + Open WebUI
├── .env.example            # Plantilla de variables
├── .env                    # Secretos locales (no en git)
├── knowledge/              # Docs para RAG (Open WebUI)
├── openarc_models/         # Modelos → /models en el contenedor
├── openarc_data/           # Config OpenArc → /persist
└── open_webui_data/        # Datos de Open WebUI
```

---

## Inicio rápido (Qwen3-4B)

### 1. Clonar y entorno

```bash
git clone https://github.com/jaaf-dev/open_arc.git
cd open_arc

cp .env.example .env
# Editar .env: OPENARC_API_KEY, OPENAI_API_KEY (misma clave), WEBUI_SECRET_KEY
```

### 2. Levantar el stack

```bash
docker compose up -d
```

| Servicio   | URL                      |
|------------|--------------------------|
| OpenArc    | http://localhost:8000/v1 |
| Open WebUI | http://localhost:3000    |

### 3. Descargar y registrar Qwen3-4B (dentro del contenedor)

```bash
docker exec -it openarc_server bash
```

```bash
huggingface-cli download Echo9Zulu/Qwen3-4B-Instruct-2507-int4_asym-awq-ov \
  --local-dir /models/qwen3-4b

openarc add \
  --model-name qwen3-4b \
  --model-path /models/qwen3-4b \
  --engine ovgenai \
  --model-type llm \
  --device GPU

openarc load qwen3-4b
openarc status
```

Sal del contenedor: `exit`

> Con `OPENARC_AUTOLOAD_MODEL=qwen3-4b` en `.env` (valor por defecto en `.env.example`), el modelo se carga solo al reiniciar el contenedor **después** de haberlo registrado una vez.

### 4. Verificar la API

```bash
curl http://localhost:8000/v1/models

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TU_OPENARC_API_KEY" \
  -d '{
    "model": "qwen3-4b",
    "messages": [{"role": "user", "content": "Hola"}],
    "max_tokens": 50
  }'
```

### 5. Open WebUI

1. Abre http://localhost:3000
2. Crea tu cuenta (primera visita)
3. Selecciona el modelo **`qwen3-4b`**

---

## Por qué Qwen3-4B-Instruct-2507

| | Qwen3-4B-Instruct-2507 |
|--|------------------------|
| **VRAM (GPU)** | ~3–4 GB cargado + margen para chat |
| **Peso en disco** | ~2.5 GB |
| **Tipo** | Instruct (sin bloques `` de thinking) |
| **Uso ideal** | Chat, código, agentes con tools (OpenCode) |
| **Arc A750 8 GB** | ✅ Cómodo — un modelo cargado a la vez |

Variantes **Qwen3 base** (p. ej. 8B sin sufijo Instruct-2507) pueden activar modo *thinking* y mostrar etiquetas ``; esta variante **Instruct-2507** evita eso.

---

## Gestión de modelos

Documentación: [modelos](https://searchsavior.github.io/OpenArc/models/) · [comandos](https://searchsavior.github.io/OpenArc/commands/)

```bash
docker exec -it openarc_server bash
openarc tool device-detect    # GPU, CPU o HETERO:GPU,CPU
openarc list
openarc load qwen3-4b
openarc unload <otro>
openarc status
openarc bench qwen3-4b
openarc list --rm <nombre>    # quitar del config (no borra archivos)
```

### Autocarga al arrancar

En `.env`:

```env
OPENARC_AUTOLOAD_MODEL=qwen3-4b
```

### Registrar en GPU

```bash
openarc add \
  --model-name qwen3-4b \
  --model-path /models/qwen3-4b \
  --engine ovgenai \
  --model-type llm \
  --device GPU
```

### Modelos opcionales (secundarios)

Solo si necesitas otro perfil; en 8 GB VRAM **descarga uno antes de cargar otro**.

| Modelo | Uso | Tamaño INT4 | Notas |
|--------|-----|-------------|-------|
| [Phi-4-mini-instruct](https://huggingface.co/Echo9Zulu/Phi-4-mini-instruct-OpenVINO) | Chat rápido | ~2.5 GB | Subcarpeta en el repo HF |
| [NextCoder 11B](https://huggingface.co/Echo9Zulu/Qwen2.5-Microsoft-NextCoder-Soar-Instruct-FUSED-CODER-Fast-11B-int4_asym-awq-ov) | Código en WebUI | ~5.8 GB | Justo en 8 GB; mal tool-calling en OpenCode |
| [Qwen2.5-7B-Instruct (OpenVINO)](https://huggingface.co/OpenVINO/Qwen2.5-7B-Instruct-int4-ov) | Instruct 7B | ~4.5 GB | Alternativa más grande; probar aparte |

Ejemplo Phi-4-mini (ruta con subcarpeta):

```bash
huggingface-cli download Echo9Zulu/Phi-4-mini-instruct-OpenVINO \
  --include "Phi-4-mini-instruct-int4_asym-awq-se-ov/*" \
  --local-dir /models/phi4-mini

openarc add \
  --model-name phi4-mini \
  --model-path /models/phi4-mini/Phi-4-mini-instruct-int4_asym-awq-se-ov \
  --engine ovgenai \
  --model-type llm \
  --device GPU
```

Modelos grandes (14B+): usa `--device HETERO:GPU,CPU` y `PIPELINE_PARALLEL` — ver [comandos OpenArc](https://searchsavior.github.io/OpenArc/commands/).

---

## RAG (Open WebUI + Knowledge)

El modelo no memoriza tus docs. Usa **Knowledge** para inyectar solo trozos relevantes.

```
knowledge/
├── docs/       # Documentación del stack
├── snippets/   # Plantillas de código
└── projects/   # Notas por proyecto
```

1. **Workspace** → **Knowledge** → crear colección
2. Subir archivos de `./knowledge/`
3. En el chat: activa la colección (`#` o botón Knowledge)
4. Modelo: **`qwen3-4b`**, chats cortos por tarea

---

## OpenCode (opcional, configuración local)

Para usar [OpenCode](https://opencode.ai/) como agente en terminal contra OpenArc:

1. Instala OpenCode y configura el proveedor en `~/.config/opencode/opencode.json`
2. Modelo local: `openarc/qwen3-4b` con `tool_call: true`
3. API: `http://localhost:8000/v1`, clave = `OPENARC_API_KEY`

```bash
export OPENARC_API_KEY=$(grep OPENARC_API_KEY .env | cut -d= -f2)
cd /ruta/a/tu/proyecto
opencode
```

Prompts avanzados (`opencode.json`, `.opencode/`) son **configuración local** y no forman parte de esta guía — quedan en `.gitignore`.

---

## Comandos útiles

```bash
docker compose logs -f openarc
docker compose logs -f open-webui
docker compose restart
docker compose down
docker exec -it openarc_server bash
```

---

## Notas VRAM (Intel Arc ~8 GB)

- Un **solo** LLM cargado a la vez con Qwen3-4B.
- Prompts e historial largos **aumentan** KV cache y VRAM.
- Chats nuevos en WebUI; RAG en lugar de pegar repos enteros.
- Los archivos del modelo están en disco (`openarc_models/`); la VRAM se usa solo al inferir.

---

## Enlaces

- [OpenArc — documentación](https://searchsavior.github.io/OpenArc/)
- [OpenArc — modelos](https://searchsavior.github.io/OpenArc/models/)
- [OpenArc — comandos](https://searchsavior.github.io/OpenArc/commands/)
- [Colección OpenVINO LLM (HF)](https://huggingface.co/collections/OpenVINO/llm)
- [Echo9Zulu — modelos preconvertidos](https://huggingface.co/Echo9Zulu)
- [Repositorio OpenArc](https://github.com/searchsavior/OpenArc)
