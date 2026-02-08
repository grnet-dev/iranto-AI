
# IRANTO AI Project

## 1. Visión General y Contexto

Sistema de inteligencia local y auditoría documental para la gestión técnica de obras, diseñado para capturar la inmediatez del campo y estructurarla en un grafo de conocimiento estratégico.

* **Infraestructura:** Servidor Local Windows con Docker (WSL2).
* **Hardware Base:** Evolución RTX 2060 (Dev) -> RTX 5070 Ti 16GB (Prod).
* **Stack Tecnológico:**
* **Cerebro:** Ollama / vLLM (Llama 3.1 / Mistral).
* **Base de Datos:** ArangoDB (Multi-modelo: Document + Graph).
* **Orquestación Agentica:** Agent Zero / LlamaIndex.
* **Interfaz y Control:** **SvelteKit 5 (Node.js, TypeScript, Tailwind, Lucia Auth).**
* **Privacidad:** 100% On-Premise. Ningún dato sensible sale de la red local.

---

## 2. Módulos de Desarrollo (Hoja de Ruta)

Utilizamos este cuadro como índice guía de desarrollo vivo.

| Módulo | Rol del Sistema | Descripción | Estado | Prioridad |
| --- | --- | --- | --- | --- |
| **M1: Auditoría Drive** | *El Notario Digital* | Sincronización API Google Drive -> ArangoDB. Control de versiones y cambios. | `[DEV]` | Alta |
| **M2: Ingesta Técnica** | *El Traductor Técnico* | Splitting de PDFs por capítulos, OCR y parsing de tablas Excel/BC3. | `[PND]` | Alta |
| **M3: Graph-RAG Core** | *La Fábrica de Conocimiento* | Generación de Embeddings, extracción de entidades y relaciones AQL. | `[PND]` | Media |
| **M4: Agente de Campo** | *El Corresponsal de Obra* | Integración de audios (Whisper) y procesamiento de mensajes de WhatsApp. | `[PND]` | Media |
| **M5: Dashboard Svelte** | *La Cabina de Mando* | Interfaz de configuración, métricas de hardware y visualización del grafo. | `[PND]` | Alta |

---

## 3. Directrices Técnicas (Módulo 5: Dashboard)

Para garantizar la **privacidad de datos** en el desarrollo con SvelteKit, establecemos las siguientes normas:

* **Aislamiento de Lógica:** Todo acceso a ArangoDB o Google Drive debe ocurrir exclusivamente en archivos `+page.server.ts` o `server.ts`.
* **Gestión de Sesiones:** Se utiliza **Lucia Auth** para asegurar que solo personal autorizado acceda a la configuración de los modelos de IA y logs de auditoría.
* **Visualización Segura:** El cliente (navegador) nunca recibe el documento completo, solo "snippets" o metadatos necesarios para la visualización de métricas.
* **Interfaz Lucide:** Uso sistemático de iconos Lucide para una navegación intuitiva para personal técnico no familiarizado con IA.

---

### Módulo 5: Arquitectura de Control

1. **Vista de Métricas:** Monitorización del progreso de la indexación nocturna (M1 y M2).
2. **Configuración de Modelos:** Selector de modelo LLM activo en Ollama (para pruebas con la 2060 vs la futura 5070).
3. **Explorador de Grafos:** Visualización de cómo los documentos se conectan entre sí y con los reportes de campo.

---
