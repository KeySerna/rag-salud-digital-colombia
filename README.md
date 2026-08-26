# RAG sobre Salud Digital en Colombia: Telemedicina e Historia Clínica Electrónica

Alumna: Keyla Yunuette Serna Illescas

Escuela Colombiana de Ingeniería Julio Garavito - ISIS TDSE


Aplicación RAG (Retrieval-Augmented Generation) construida con **Gemini +
LangChain + Chroma**, adaptando la arquitectura profesional vista en el
workshop del curso a un dominio propio: normativa colombiana sobre
**telemedicina** e **interoperabilidad de la historia clínica electrónica
(IHCE)**.

Elegí este ángulo de salud porque conecta directamente con mi carrera de
ingeniería de sistemas: son proyectos de transformación digital e
interoperabilidad de datos aplicados al sector salud.

## Objetivo y caso de uso

Responder preguntas sobre las categorías de telemedicina en Colombia, el
marco normativo de la historia clínica electrónica, y las ventajas que se
le atribuyen a su digitalización, citando siempre las fuentes usadas y
señalando explícitamente cuándo la evidencia disponible no alcanza para
responder (por ejemplo, cifras o presupuestos que las fuentes no reportan).

## Colección de documentos

Tres fuentes públicas en español, sobre Colombia, cargadas directamente desde
internet (no se almacena copia local en `data/`):

| # | Título | URL |
|---|--------|-----|
| 1 | Parámetros para la Telemedicina en Colombia – Resolución 2654 de 2019 (CONSULTORSALUD) | https://consultorsalud.com/parametros-para-la-telemedicina-en-colombia-resolucion-2654-de-2019/ |
| 2 | Normatividad — Interoperabilidad de la Historia Clínica Electrónica (Ministerio de Salud y Protección Social, página oficial) | https://www.minsalud.gov.co/ihce/Paginas/Normatividad.aspx |
| 3 | MinTIC y MinSalud firman resolución para Historia Clínica Electrónica (ENTER.CO) | https://www.enter.co/empresas/colombia-digital/mintic-minsalud-historia-clinica-electronica/ |

Ver la sección "1. Selección de fuentes" del notebook para la justificación
completa de cada fuente.

## Arquitectura

```
Fuentes web (3 páginas HTML públicas)
        │
        ▼
Carga como Document de LangChain (extractor HTML estándar)
        │
        ▼
Chunking: RecursiveCharacterTextSplitter
  (chunk_size=1000, chunk_overlap=150)
        │
        ▼
Embeddings: Gemini (models/gemini-embedding-001)
        │
        ▼
Chroma (base de datos vectorial local, persist_directory)
        │
        ▼
Retrieval (similarity_search, top-k=4)
        │
        ├──► RAG chain de 2 pasos (siempre retrieve → generate)
        │
        └──► RAG agent (decide cuándo usar la herramienta retrieve_context)
        │
        ▼
Respuesta de Gemini (google_genai:gemini-2.5-flash-lite) + fuentes citadas
```

## Instalación y ejecución

```bash
git clone <url-de-este-repositorio>
cd rag-salud-digital-colombia
python -m venv .venv
source .venv/bin/activate  # En Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edita .env y agrega tu GOOGLE_API_KEY
jupyter notebook notebooks/rag_application.ipynb
```

## Variables de entorno

| Variable | Requerida | Descripción |
|---|---|---|
| `GOOGLE_API_KEY` | Sí | API key de Gemini (Google AI Studio, free tier) |
| `GEMINI_API_KEY` | No | Alias alternativo de `GOOGLE_API_KEY`, también soportado |
| `GEMINI_CHAT_MODEL` | No | Por defecto `google_genai:gemini-2.5-flash-lite` |
| `GEMINI_EMBEDDING_MODEL` | No | Por defecto `models/gemini-embedding-001` |

**Nunca subas tu `.env` real al repositorio.** Ya está incluido en `.gitignore`.

## Modelos usados

- **Chat:** `google_genai:gemini-3.5-flash-lite` (Gemini Developer API, free tier)
- **Embeddings:** `models/gemini-embedding-001`

## Decisiones de diseño principales

- **Chunking:** `chunk_size=1000`, `chunk_overlap=150`. La fuente oficial de
  MinSalud mezcla texto de navegación con una lista de normas (cada una con
  una descripción corta), y la fuente sobre telemedicina tiene secciones
  temáticas medianas (categorías, consentimiento, financiación). Un tamaño
  moderado evita que un chunk mezcle dos normas o dos secciones distintas, y
  el overlap evita cortar una descripción justo en el borde.
- **top-k = 4:** suficiente para cubrir varias normas o varias fuentes a la
  vez sin diluir demasiado el contexto.
- **Prompt de seguridad:** el contexto recuperado se trata explícitamente
  como datos (no instrucciones) y se envuelve en delimitadores `<context>`,
  como defensa básica contra inyección indirecta de prompts.

## Resultados de evaluación

Ver la sección "9. Evaluación con tres preguntas" del notebook para la tabla
completa con las tres preguntas de prueba (clara / evidencia parcial / no
respondible), sus resultados y el groundedness check.
Las 3 preguntas de prueba (clara, evidencia parcial, no respondible) obtuvieron
`grounded: true` en el groundedness check — el sistema no alucinó en ningún caso:
reconoció explícitamente cuando no tenía evidencia suficiente (presupuesto del
proyecto) y cuando la evidencia era solo cualitativa, no cuantitativa (reducción
de costos). Se observó una inconsistencia en el retrieval: la misma pregunta
sobre categorías de telemedicina trajo resultados completos en una corrida
(Sección 8) e incompletos en otra (Sección 9), lo que sugiere ajustar el top-k
o el chunk_size para preguntas que requieren cubrir listas completas.

## Comparación: RAG chain vs. RAG agent

Ver la sección "8. Comparación" del notebook. Regla general: la chain es más
simple y rápida cuando casi siempre se quiere recuperar una vez por pregunta;
el agente es más apropiado si el flujo necesita decidir cuándo y cuántas
veces buscar.

Ambos usaron la herramienta/retrieval y trajeron contexto real de la fuente antes de responder.
Ambos citaron únicamente la Resolución 2654 de 2019 (ConsultorSalud), la misma URL.
Ambas respuestas listan las mismas 4 categorías de telemedicina y coinciden en cuáles permiten prescripción de medicamentos (Telemedicina interactiva y Telexperticia sincrónica). El agente además citó la fuente de forma más explícita al final de su respuesta.
 La arquitectura que me parece más simple es la RAG chain, porque la pregunta siempre necesita el mismo tipo de búsqueda (una sola consulta al vector store); el agente no aportó ventaja adicional aquí y consumió más tiempo/llamadas al modelo para llegar a un resultado prácticamente igual.
## Limitaciones y mejoras posibles

- Las fuentes 1 y 3 son artículos periodísticos/divulgativos que resumen
  resoluciones oficiales, no el texto legal completo — una mejora sería
  indexar directamente los PDF de las resoluciones (2654 de 2019 y 866 de
  2021) publicados por el Ministerio de Salud.
- La fuente 2 (página oficial de MinSalud) es una página SharePoint con
  mucho ruido de navegación; el extractor HTML simple captura parte de ese
  ruido, lo que puede degradar algunos chunks.
- Ninguna fuente reporta cifras o evidencia cuantitativa de impacto (ahorro
  de costos, tiempos), solo afirmaciones cualitativas — el sistema debería
  reconocerlo explícitamente en vez de inventar números.
- Mejora posible: agregar un reranker, o filtrar por metadata de fuente
  cuando la pregunta distinga explícitamente entre telemedicina e historia
  clínica electrónica.

## Seguridad

- Solo se usan fuentes públicas de acceso abierto.
- No se sube ninguna API key a este repositorio.
- La base de datos Chroma generada (`chroma_salud_digital_db/`) no se
  incluye en el repo porque se puede regenerar corriendo el notebook.
