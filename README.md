# Sistema Backend de Triaje y Resolución Inteligente con RAG + LangChain (v5)

**Trabajo Práctico 5 – Seminario de LLMs: Integración del Framework LangChain sobre el Pipeline RAG**

---

## Descripción del Proyecto

Este proyecto constituye la **versión 5 (v5)** del motor de triaje y resolución inteligente de reclamos de E-Commerce. Mantiene exactamente la misma lógica de negocio, corpus normativo, grilla experimental, métricas y casos de evaluación que el **TP4**, incorporando como **único cambio arquitectónico** el uso integral del framework **LangChain** (`langchain-core`, `langchain-openai`, `langchain-chroma`, `langchain-huggingface`, `langchain-text-splitters`).

Ante cada consulta o reclamo de un cliente:
1. Se fragmenta [`politicas_ecommerce_integral.md`](./politicas_ecommerce_integral.md) como objetos `Document` de LangChain (`RecursiveCharacterTextSplitter.from_tiktoken_encoder` y `MarkdownHeaderTextSplitter`).
2. Se vectoriza localmente mediante **`HuggingFaceEmbeddings`** (`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`) y se indexa en **`Chroma`** de `langchain-chroma` con similitud del coseno (`hnsw:space = "cosine"`).
3. Se recupera dinámicamente el conjunto Top-$K$ de cláusulas relevantes mediante el **Vector Store / Retriever de LangChain** (`similarity_search_with_score` y `as_retriever`).
4. Se construyen los prompts con **`ChatPromptTemplate`** inyectando `{contexto_politicas}` delimitado con etiquetas XML `<contexto_politicas>`.
5. Se orquesta el pipeline en dos etapas (Triaje Few-Shot en `Prompt 1.md` $\rightarrow$ Resolución con *Chain-of-Thought* guiado en `Prompt 2.md`) mediante cadenas **LCEL (LangChain Expression Language)** usando **`ChatOpenAI`** (o un `BaseChatModel` determinista local de respaldo cuando no hay API Key configurada).
6. Se valida estrictamente la salida JSON mediante **Pydantic (`RespuestaSoporteRAG`)** y `validar_esquema_salida`.

---

## Estructura del Repositorio

* `TP5_Resolucion.ipynb`: **Entregable principal**. Jupyter Notebook ejecutable en español con las 7 secciones, tablas empíricas, gráficos/heatmaps y todas las salidas visibles utilizando LangChain.
* `politicas_ecommerce_integral.md`: Corpus oficial de políticas de envíos, garantías, facturación/notas de crédito y botón de arrepentimiento.
* `Prompt 1.md`: Plantilla de la Etapa 1 (Triaje y Clasificación con 10 ejemplos *few-shot* y aislamiento `<reports>`).
* `Prompt 2_TP3.md`: Plantilla original de la Etapa 2 del TP3 (conservada para comparación y cálculo de `diff` antes/después).
* `Prompt 2.md`: Plantilla evolucionada de la Etapa 2 para RAG (sin políticas hardcodeadas, con inyección de `<contexto_politicas>{contexto_politicas}</contexto_politicas>` y reglas estrictas de *grounding*).
* `requirements.txt`: Dependencias reproducibles del entorno (incluyendo `langchain`, `langchain-core`, `langchain-openai`, `langchain-chroma`, `langchain-huggingface` y `langchain-text-splitters`).
* `.env.example`: Plantilla para configuración de variables de entorno (`OPENAI_API_KEY`, `OPENAI_BASE_URL`, `MODEL_NAME`).

---

## Cambios del TP4 al TP5 (Inclusión de LangChain)

| Componente | TP4 (SDKs Directos) | TP5 (Framework LangChain) |
| :--- | :--- | :--- |
| **Documentos y Chunking** | `str` planos con `split_text` | `langchain_core.documents.Document` con `split_documents` |
| **Embeddings** | `SentenceTransformer` directo | `langchain_huggingface.HuggingFaceEmbeddings` |
| **Vector Store** | `chromadb.EphemeralClient` directo | `langchain_chroma.Chroma` (`from_documents`, `similarity_search_with_score`, `as_retriever`) |
| **Prompts** | Reemplazo manual con `.replace()` / f-strings | `langchain_core.prompts.ChatPromptTemplate` |
| **Cliente LLM** | `openai.OpenAI` directo | `langchain_openai.ChatOpenAI` + `BaseChatModel` determinista local de respaldo |
| **Orquestación** | Llamadas imperativas a funciones | Cadenas declarativas **LCEL** (`prompt \| llm \| StrOutputParser()`) |

---

## Instrucciones de Ejecución

### 1. Crear y activar el entorno virtual
```bash
python -m venv venv
# En Windows (PowerShell):
.\venv\Scripts\Activate.ps1
# En Linux / macOS:
source venv/bin/activate
```

### 2. Instalar dependencias
```bash
pip install -r requirements.txt
```

### 3. Configurar variables de entorno
```bash
cp .env.example .env
```
Podés configurar tu `OPENAI_API_KEY` (OpenAI, Groq, Google Gemini compatible con OpenAI o servidor local Ollama). Si no se configura una clave externa, el notebook utiliza el `BaseChatModel` determinista de respaldo consciente del contexto RAG (`DeterministicRAGChatModel`) para garantizar reproducibilidad offline de punta a punta.

### 4. Ejecutar el Notebook
```bash
jupyter notebook TP5_Resolucion.ipynb
```
O para ejecutar todas las celdas desde consola:
```bash
jupyter nbconvert --to notebook --execute --inplace TP5_Resolucion.ipynb
```
