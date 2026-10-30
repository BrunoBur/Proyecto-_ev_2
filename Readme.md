# 🏥 Agente Autónomo de Gestión Farmacéutica y Auditoría Transaccional

Sistema multi-herramienta basado en la arquitectura **Plan-and-Execute** u orquestado mediante **LangGraph**, diseñado para la automatización operativa de farmacias, consulta semántica de inventario (RAG), lectura multimodal de recetas médicas (Visión OCR) y detección adaptativa de fraudes o riesgo financiero mediante Machine Learning y bases de datos relacionales SQL.

---

## 📐 1. Arquitectura del Sistema y Diagrama de Orquestación (IE6, IE8)

El agente utiliza un flujo cíclico guiado por grafos de estado (**LangGraph**), donde un modelo razona la secuencia de pasos (**Plan-and-Execute**) antes de ejecutar condicionalmente las herramientas especializadas.

### Diagrama de Flujo y Orquestación de Nodos

```
                    +-----------------------------------+
                    |      Entrada del Usuario          |
                    +-----------------------------------+
                                      |
                                      v
                    +-----------------------------------+
                    |   Planificador (Planner Chain)    |
                    |   Descomposición en N Pasos       |
                    +-----------------------------------+
                                      |
                                      v
                    +-----------------------------------+
                    |    Nodo Agente (LLM Principal)    | <-----+
                    |       openai/gpt-oss-120b         |       |
                    +-----------------------------------+       |
                                      |                         |
                           ¿Requiere herramienta?               |
                                /           \                   |
                             SÍ              NO                 |
                            /                  \                |
                           v                    v               |
             +--------------------+      +---------------+      |
             |   Nodo "Tools"     |      |  Respuesta    |      |
             |  (ToolNode Exec)   |      | Final Usuario |      |
             +--------------------+      +---------------+      |
             /         |          \                             |
            /          |           \                            |
           v           v            v                           |
    +-----------+ +----------+ +-------------+                  |
    | Motor RAG | | Modelo ML| | Vision OCR  |                  |
    | Inventario| | Random   | | llama-3.2-  | -----------------+
    |  (Cosine) | |  Forest  | | 11b-vision  |
    +-----------+ +----------+ +-------------+
           |           |              |
           v           v              v
      Embeddings    SQLite DB    Imagen Base64
```

---

## 🛠️ 2. Justificación de Componentes y Frameworks (IE2, IE9)

La selección de la pila tecnológica responde directamente a los requerimientos organizacionales de escalabilidad, compatibilidad y robustez técnica:

| Componente / Framework | Función en el Sistema | Justificación Técnica |
| :--- | :--- | :--- |
| **LangGraph** | Orquestador de Agentes | Proporciona control de ciclo estructurado, soporte para nodos condicionales y compatibilidad nativa con persistencia de memoria episódica. |
| **LangGroq (`ChatGroq`)** | Motor de Razonamiento LLM | Acceso a alta velocidad y baja latencia para los modelos `openai/gpt-oss-120b` (razonamiento) y `llama-3.2-11b-vision-preview` (multimodal). |
| **SentenceTransformers** | Embeddings Semánticos | El modelo `all-MiniLM-L6-v2` permite convertir textos de inventario en vectores de alta densidad para búsquedas por similitud de coseno rápida en CPU. |
| **Scikit-Learn (`RandomForestClassifier`)** | Clasificador de Riesgo ML | Modelo supervisado ensemble entrenado con balance de clases para detectar patrones anómalos de horario y monto en transacciones. |
| **SQLite (`sqlite3`)** | Base de Datos Relacional | Almacenamiento transaccional en memoria para auditar compras aprobadas vs. puestas en cuarentena con sanitización de datos (hashing de usuarios). |

---

## 🧩 3. Características Principales y Módulos (IE1, IE3, IE4, IE5, IE7)

### 🤖 Configuración de Herramientas Autónomas (IE1)
El agente dispone de 3 herramientas decoradas con `@tool` que pueden ser invocadas de forma autónoma según las necesidades de la consulta:
1. **`consultar_inventario`**: Ejecuta búsqueda semántica RAG sobre el catálogo de productos.
2. **`evaluar_y_registrar_transaccion`**: Evalúa una compra con ML y registra el resultado en tablas SQL sanitizadas.
3. **`analizar_receta`**: Analiza imágenes de recetas médicas mediante visión artificial para extraer prescripciones.

### 🧠 Memoria Conversacional y Continuidad Contextual (IE3)
* Utiliza **`MemorySaver`** como motor de *checkpointing* en LangGraph.
* Mantiene el historial acumulativo de la conversación ligado a un identificador único de hilo (`thread_id`), garantizando coherencia en sesiones interactivas prolongadas.

### 🔎 Motor RAG para Búsqueda Semántica (IE4)
* **Chunking**: Utiliza `RecursiveCharacterTextSplitter` para fragmentar el inventario en bloques procesables.
* **Recuperación**: Calcula la similitud coseno entre el vector de la consulta y la base de datos vectorial de inventario:
  $$\text{Similitud}(\mathbf{u}, \mathbf{v}) = \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\| \|\mathbf{v}\|}$$

### 📋 Módulo de Planificación Plan-and-Execute (IE5)
Antes de interactuar con el entorno, la cadena `planner_chain` toma la meta del usuario y genera una secuencia ordenada de pasos lógicos (máximo 3), evitando llamadas redundantes o ambiguas a las herramientas.

### ⚡ Toma de Decisiones Adaptativas y Detección de Riesgo (IE7)
El sistema evalúa el contexto en tiempo real. Si detecta un pago fuera de horario habitual (ej. madrugada) o por montos anómalos:
1. El modelo Random Forest califica la transacción como **Riesgo (1)**.
2. La transacción se reenruta automáticamente a la tabla SQL `transacciones_cuarentena`.
3. El agente informa al usuario del bloqueo sin interrumpir el flujo del sistema.

---

## 💻 4. Instalación y Configuración

### Prerrequisitos
* Python 3.10 o superior.
* API Key de Groq (configurada en el entorno o en los Secretos de Google Colab como `LLM_API_KEY`).

### Instalación de Dependencias
```bash
pip install -q "numpy<2.3.0" "pandas<3.0.0" langchain-groq langgraph sentence-transformers python-dotenv pydantic scikit-learn groq langchain-text-splitters
```

---

## 🚀 5. Guía de Ejecución

1. **Apertura e Inicialización**: Ejecuta todas las celdas del notebook en secuencia en Google Colab o Jupyter Notebook.
2. **Configuración de la API Key**:
   Asegúrate de registrar tu clave API de Groq en los secretos de Google Colab con la clave `LLM_API_KEY`.
3. **Interfaz Interactiva**:
   Ejecuta la celda final para iniciar el menú en consola. Dispones de dos modos de operación:
   * **Modo 1: Formulario de Compra**: Ingrese el medicamento, la cantidad y el estado de pago. El agente calculará subtotales y registrará la transacción de manera automatizada.
   * **Modo 2: Consulta Libre**: Chat en vivo para resolver dudas sobre inventario, stock o subir recetas médicas.
   * **Auditoría SQL**: Al finalizar cada operación, el sistema imprimirá una tabla comparativa con los registros aprobados y en cuarentena.

---

## 📚 6. Referencias Bibliográficas (Formato APA) (IE9)

* Groq. (2024). *Groq Cloud Documentation and API Reference*. https://console.groq.com/docs
* LangChain. (2024). *LangGraph: Building Language Agents as Graphs*. LangChain AI. https://python.langchain.com/docs/langgraph
* Pedregosa, F., Varoquaux, G., Gramfort, A., Michel, V., Thirion, B., Grisel, O., Blondel, M., Prettenhofer, P., Weiss, R., Dubourg, V., Vanderplas, J., Passos, A., Cournapeau, D., Brucher, M., Perrot, M., & Duchesnay, E. (2011). Scikit-learn: Machine Learning in Python. *Journal of Machine Learning Research*, 12, 2825-2830.
* Reimers, N., & Gurevych, I. (2019). Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks. *Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing (EMNLP)*. https://arxiv.org/abs/1908.10084