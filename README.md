# backend-llm-agent
Backend LLM Agent

# Plan de desarrollo — Backend Knowledge Base + Agente/Modelo LLM Pequeño
Fecha: 2026-02-07

Este plan está pensado para:
1) **Construir una base de conocimiento sólida de backend** (fundamentos + complementos modernos) a partir de tus transcripciones (`BackendAgent.txt/.srt/.vtt`) y fuentes externas.
2) **Alimentar un LLM pequeño** (fine-tuning o RAG) y
3) **Crear un agente** que responda con alta precisión, cite fuentes internas y sea útil para arquitectura/diseño/implementación backend.

---

## 0) Objetivos del proyecto

### Objetivos funcionales
- Responder preguntas backend sobre:
  - APIs (REST/GraphQL/WebSockets), contratos, versionado, errores, idempotencia.
  - Infra: reverse proxy vs load balancer, caching, rate-limit, TLS, redes.
  - Seguridad: AuthN/AuthZ, JWT/OAuth2, sesiones, CSRF, CORS.
  - Datos: SQL/NoSQL, ACID, índices, CAP, replicación, particionado.
  - Sistemas distribuidos: colas, eventos, consistency, sagas, retries.
  - Observabilidad: logs/métricas/trazas, SLO/SLI, alertas.
  - DevOps: Docker/Kubernetes/serverless, CI/CD, despliegues, rollbacks.
  - Testing: unit/integration/e2e, mocks, contract tests.

### Objetivos no funcionales
- **Trazabilidad**: cada respuesta debe poder mapearse a:
  - (a) un chunk de knowledge base local (tu material) y/o
  - (b) una fuente externa (si usas RAG con docs externas).
- **Consistencia**: estructura de respuestas (definición → cuándo usar → pitfalls → ejemplo).
- **Seguridad**: no filtrar secretos, no inventar credenciales, recomendar prácticas seguras.
- **Mantenibilidad**: pipeline reproducible para actualizar dataset y re-entrenar/re-indexar.

---

## 1) Arquitectura recomendada del “Agente Backend”

### Opción A — RAG (recomendado como base)
- **Retriever**: índice vectorial (FAISS/Chroma/pgvector).
- **Re-ranker opcional**: modelo pequeño o BM25+heurísticas.
- **Generator**: tu LLM pequeño genera respuesta usando contexto recuperado.
- **Citations**: almacenar `source_id` + rangos en el KB para citar.

**Ventajas**: actualizable sin re-entrenar; fácil añadir más docs.  
**Desventajas**: depende de calidad del retrieval; necesita buena chunking y eval.

### Opción B — Fine-tuning (complemento)
- Dataset curado (JSONL) con:
  - definiciones, comparaciones, reglas, pitfalls, mini-casos
  - ejemplos de código (cortos y correctos)
- Entrenar para:
  - estilo consistente
  - reducir alucinaciones en temas base
- Mantener RAG para conocimiento cambiante (frameworks, cloud, versiones, etc.).

**Recomendación**: empezar con **RAG**, luego FT para “estilo + fundamentos”.

---

## 2) Estructura de repositorio (sugerida)

```
backend-llm-agent/
  README.md
  LICENSE
  data/
    raw/
      BackendAgent.txt
      BackendAgent.srt
      BackendAgent.vtt
    processed/
      kb_chunks.jsonl
      qa_pairs.jsonl
      code_snippets.jsonl
  src/
    ingest/
      chunker.py
      normalize.py
      dedupe.py
    rag/
      embed.py
      index_build.py
      retrieve.py
      rerank.py
    agent/
      prompt_templates/
        system.md
        answer.md
      agent.py
      cite.py
    eval/
      goldset.jsonl
      eval_runner.py
      metrics.py
  configs/
    chunking.yaml
    embedding.yaml
    agent.yaml
  notebooks/
  scripts/
    make_kb.sh
    make_index.sh
    run_agent.sh
  ci/
    github-actions.yml
```

---

## 3) Pipeline de datos (KB → dataset)

### 3.1 Ingesta y normalización
**Entrada**:
- `data/raw/BackendAgent.txt` (principal), `.srt`, `.vtt` (opcional, para timestamps)

**Salida**:
- `kb_chunks.jsonl` con campos recomendados:
  - `chunk_id`, `topic`, `text`, `source_file`, `source_span` (líneas o timestamps), `tags`

**Reglas**
- Normalizar espacios, quitar repeticiones, corregir puntuación ligera (sin “inventar” contenido).
- Separar por temas detectados (API, auth, idempotencia, etc.).
- Guardar metadatos de origen (líneas Lx-Ly o timestamps si usas srt/vtt).

### 3.2 Generación de QA + reglas
De cada chunk:
- **Definition cards** (Q/A)
- **Comparisons** (X vs Y)
- **Checklists** (si pasa X → haz Y)
- **Pitfalls** (error común → mitigación)
- **Mini-casos** (pagos, posts, inventario, etc.)
- **Code snippets**: mínimo, correcto, y con contexto (lenguaje/framework)

Salida:
- `qa_pairs.jsonl`
- `code_snippets.jsonl`

### 3.3 Dedupe + calidad
- Eliminar duplicados semánticos (minhash/embeddings).
- Validar:
  - longitudes (respuestas 1–6 frases)
  - no contradicciones obvias
  - no incluir secretos
  - no incluir URLs con tokens

---

## 4) Diseño del agente (prompts + formato de respuesta)

### 4.1 Estilo de respuesta (plantilla)
1. **Definición breve**
2. **Cuándo usar**
3. **Errores comunes**
4. **Ejemplo** (código o pseudo)
5. **Checklist** (si aplica)
6. **Citas** (source_id/rango)

### 4.2 Políticas internas (guardrails)
- Si no hay evidencia en KB: decirlo y sugerir “búsqueda externa” o “añadir doc”.
- Si el usuario pide acciones peligrosas (secrets/hacking): rechazar y redirigir.
- En seguridad: preferir prácticas estándar (OAuth2, OWASP), sin inventar.

---

## 5) Evaluación (indispensable)

### 5.1 Goldset
Crear `src/eval/goldset.jsonl` con ~150 casos:
- 50 fundamentos (API, idempotencia, ACID, CAP)
- 50 infraestructura/devops (Docker/K8s, reverse proxy, CI/CD)
- 50 datos/mensajería/testing (SQL/NoSQL, Kafka/Rabbit, integration tests)

Cada caso:
- `question`
- `expected_points` (lista de bullets)
- `must_cite` (bool)
- `topics`
- `difficulty`

### 5.2 Métricas
- **Answer coverage**: % de expected_points cubiertos.
- **Hallucination rate**: claims no soportados por KB.
- **Citation rate**: cuando debe citar, ¿cita?
- **Retrieval quality**: MRR/Recall@k (si tienes labels por chunk).

---

## 6) Plan por fases (entregables concretos)

### Fase 1 — Base KB (Semana 1)
- [ ] Importar archivos raw a `data/raw/`
- [ ] `chunker.py` + `normalize.py`
- [ ] Generar `kb_chunks.jsonl` (mínimo 200 chunks)
- [ ] Script `make_kb.sh`

**Criterio de salida**: chunks con topics y spans de origen.

### Fase 2 — RAG MVP (Semana 2)
- [ ] Embeddings + índice vectorial
- [ ] Retrieval top-k + prompt template
- [ ] Respuestas con citas (chunk_id + spans)

**Criterio**: responde 30 preguntas con citas correctas.

### Fase 3 — Dataset de entrenamiento (Semana 3)
- [ ] `qa_pairs.jsonl` (mínimo 500)
- [ ] `code_snippets.jsonl` (mínimo 150)
- [ ] Dedupe + validaciones

**Criterio**: dataset sin duplicados obvios, con tags y temas.

### Fase 4 — Evaluación + regresión (Semana 4)
- [ ] goldset + eval_runner
- [ ] métricas base + reporte en Markdown
- [ ] CI (GitHub Actions) ejecuta tests + eval mínima

### Fase 5 — Fine-tuning opcional (Semana 5–6)
- [ ] entrenar modelo pequeño con subset de QA
- [ ] comparar FT vs RAG-only en goldset
- [ ] decidir estrategia híbrida final

---

## 7) Definición del formato final para “cargar al modelo/agente”

### KB (para RAG)
- `kb_chunks.jsonl` con:
  - `chunk_id`, `topic`, `text`, `tags`, `source_file`, `source_span`

### QA (para fine-tuning o para retrieval adicional)
- `qa_pairs.jsonl` con:
  - `id`, `q`, `a`, `topic`, `tags`, `sources` (lista)

### Snippets
- `code_snippets.jsonl` con:
  - `id`, `language`, `framework`, `title`, `code`, `explanation`, `sources`

---

## 8) “Done criteria” final
- El agente:
  - contesta con estructura consistente
  - cita fuentes internas para claims importantes
  - maneja dudas de arquitectura con ejemplos
  - pasa el goldset con:
    - hallucination rate bajo
    - coverage alto
    - citation compliance alta

---

## 9) Próximos pasos inmediatos (para empezar hoy)
1. Crear el repo con la estructura sugerida.
2. Copiar tus archivos raw a `data/raw/`.
3. Implementar `chunker.py` (heurístico por títulos/temas + longitud máxima por chunk).
4. Generar `kb_chunks.jsonl`.
5. Levantar un MVP de RAG y validar con 20 preguntas reales tuyas.
