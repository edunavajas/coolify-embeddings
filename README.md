# Modelo de embeddings (TEI) para Coolify

Repositorio listo para desplegar en Coolify un **servidor de embeddings** con
[TEI](https://github.com/huggingface/text-embeddings-inference) (Text Embeddings
Inference). Es la pieza que convierte texto en vectores numéricos para el RAG del
episodio 2.

- Imagen: `ghcr.io/huggingface/text-embeddings-inference:cpu-1.9.4` (versión fijada)
- Modelo por defecto: `intfloat/multilingual-e5-small` (384 dimensiones, multilingüe)
- Un único servicio `embeddings` con volumen para cachear el modelo
- Coste de ejecución: **cero**. Lo pagas con RAM y CPU de tu propio VPS.

---

## 1. Qué es un servidor de embeddings y por qué se despliega aparte

Un **embedding** es una representación numérica de un texto: una lista de números
(el *vector*) que captura su significado. Textos que significan cosas parecidas
quedan cerca en ese espacio de números. Eso es lo que permite buscar "por
significado" y no por coincidencia exacta de palabras.

El modelo que genera esos vectores tiene que ejecutarse **en algún sitio**. Tenemos
dos opciones:

1. **Un servicio en la nube** (OpenAI, Cohere, etc.) al que le mandas el texto por
   HTTP y te devuelve el vector. Cómodo, pero pagas por uso y tu documentación sale
   de tu servidor.
2. **Un servicio self-hosted** (este repositorio): el modelo corre en tu propio VPS.
   Coste cero, nada sale de tu máquina.

En esta serie elegimos **self-hosted**. TEI está hecho específicamente para servir
modelos de embeddings: es rápido en CPU, ligero y expone una API muy simple.

---

## 2. Qué modelo y por qué

Elegimos **`intfloat/multilingual-e5-small`**:

- **Multilingüe**: funciona bien en español, que es el idioma de la documentación.
  Muchos modelos de embeddings pequeños solo van bien en inglés.
- **384 dimensiones**: vectores pequeños. Ocupan poco en PostgreSQL y el índice
  vectorial es más rápido de construir. (La columna `vector(384)` del repositorio
  `coolify-pgvector` está preparada exactamente para esto.)
- **Tamaño del modelo**: ~120 MB. Cabe de sobra en el mismo VPS que Outline y n8n.
- **Los prefijos `passage:` y `query:`**: los modelos E5 están entrenados para
  distinguir el texto que se indexa (documento → `passage:`) del texto que se busca
  (pregunta → `query:`). Los workflows de n8n añaden ese prefijo; el modelo no lo
  hace solo. Si te lo saltas, la calidad de la búsqueda baja.

### Coste de la alternativa en la nube (referencia)

Si en algún momento prefieres no self-hostear, la opción más directa es
`text-embedding-3-small` de OpenAI:

- Precio orientativo: **~0,02 $ por millón de tokens**.
- Indexar un corpus de **1.000 documentos** (~700 tokens cada uno ≈ 700.000 tokens)
  costaría del orden de **1 o 2 céntimos** (una sola vez, más las reindexaciones).
- **Contra**: esa API produce vectores de **1536 dimensiones**, no de 384. Habría que
  cambiar el tipo de la columna en `coolify-pgvector` y reindexar todo.

Traducción honesta: self-hosted es más barato y privado, pero consume RAM de tu VPS y
te obliga a mantener un servicio más. La nube es más simple, pero cuesta céntimos y
saca tu contenido de casa.

---

## 3. Arquitectura (importante)

TEI publica la imagen CPU **solo para `linux/amd64` (x86_64)**. Es decir:

- **Funciona** en un VPS con procesador Intel/AMD (Hostinger KVM, Hetzner, DigitalOcean,
  Oracle x86, etc.).
- **NO arranca** en servidores con procesador ARM (por ejemplo Oracle Ampere A1).
  Si tu VPS es ARM, o usas otro servidor de embeddings self-hosted (Ollama, por
  ejemplo) o tiras de una API en la nube.

Comprueba la arquitectura de tu servidor con:

```bash
uname -m
```

Si devuelve `x86_64`, perfecto. Si devuelve `aarch64` o `arm64`, esta imagen no te
sirve.

---

## 4. Despliegue en Coolify

Igual que en los episodios anteriores: **Docker Compose desde un repositorio de Git**.

1. En Coolify: **+ New → Resource → Docker Compose from a Git repository**.
2. Elige el proveedor de Git y este repositorio.
3. Coolify detecta `/docker-compose.yaml`.
4. Variables de entorno (pestaña *Environment Variables*):
   - `EMBEDDING_MODEL` → déjalo por defecto (`intfloat/multilingual-e5-small`) salvo
     que quieras otro modelo, en cuyo caso **cambia también la dimensión** en el repo
     `coolify-pgvector`.
5. **Deploy**. La primera vez descarga el modelo (~120 MB) y tarda un poco más; a
   partir de ahí arranca en segundos porque el modelo queda cacheado en el volumen.

### Cómo llega n8n hasta aquí

n8n tiene que llamar a este servicio por la **red interna de Docker**. Para que dos
recursos distintos de Coolify se vean entre sí, conéctalos a la misma red (opción
*Connect to Predefined Network*) y usa como host el nombre interno del servicio
(`embeddings`). El README del repositorio `knowledge-hub-n8n` lo explica paso a paso
al configurar el nodo de embeddings.

---

## 5. Cómo verificar que funciona

Con el servicio levantado, desde dentro del contenedor o desde otro contenedor de la
misma red:

```bash
# ¿Está vivo?
curl -s http://localhost:80/health

# ¿Devuelve un vector de 384 dimensiones?
curl -s -X POST http://localhost:80/embed \
  -H 'Content-Type: application/json' \
  -d '{"inputs":"passage: la documentación se guarda en Outline"}'
```

La segunda llamada devuelve un array `[[...384 números...]]`. Ese es el vector del
texto. Con `"query: ..."` obtienes el vector de una pregunta.

> Nota: la API de TEI (`/embed`) recibe un campo `inputs` que puede ser un texto o una
> lista de textos. Devuelve una lista de vectores, uno por cada entrada.

---

## 6. Uso de recursos (referencia)

Números para `intfloat/multilingual-e5-small` en un VPS pequeño:

- **En reposo (idle, modelo cargado)**: ~600 MB – 1,2 GB de RAM.
- **Indexando**: sube de forma transitoria según el tamaño del lote.
- **Disco**: ~120 MB el modelo (en el volumen `embeddings-data`).

Es, con diferencia, el servicio más "gordo" del episodio en RAM, pero sigue cabiendo
con holgura en un VPS de 4–8 GB junto a Outline, PostgreSQL y n8n.

---

## Estructura del repositorio

```
.
├── docker-compose.yaml   # Servicio embeddings (TEI), volumen, healthcheck
├── .env.example          # Modelo de embeddings
├── LICENSE               # MIT
└── README.md
```

Licencia MIT © Eduardo Navajas.
