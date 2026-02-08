# **📑 DOCUMENTO 2: ESPECIFICACIÓN TÉCNICA M1 (EL NOTARIO)**

Módulo: Auditoría de Topología, Ciclo de Vida e Integridad

Responsabilidad: Garantizar que el Grafo refleje fielmente la realidad del servidor local y la nube.

## 1. Misión del Módulo

M1 actúa como el sensor de bajo nivel. Su función es mapear la estructura de carpetas de "Obra-Mayor", identificar el estado de los proyectos y registrar cada archivo con su "DNI Digital" (Hash) antes de que cualquier proceso de IA intervenga.

## 2. El Proceso de Auditoría Híbrida (Windows ➔ Cloud)

Dado que el flujo de datos nace en Windows y se sincroniza a Drive, M1 ejecutará un proceso en dos capas:

### A. Capa Local (Extracción NTFS) - Responsabilidad: Node.js/Python

- Identificación del Autor Real: Uso de pywin32 (o similar en Node) para leer el SID del propietario original en Windows, evitando el problema de "Autor Único" que genera GoogleSync.
- Huella Digital (Hash SHA-256): Generación obligatoria del Hash para cada archivo. Es la clave primaria para la de-duplicación.
- Detección de Cambios: Si el Hash actual != Hash en ArangoDB, se marca el archivo para Re-indexación (M2).

### B. Capa Cloud (Vinculación Drive) - Responsabilidad: API Google Drive

- Sincronización de IDs: Obtener el id_drive y el mimeType oficial de Google.
- Gestión de Accesos Directos: Identificar los application/vnd.google-apps.shortcut creados por humanos para registrarlos como relaciones de navegación, no como archivos nuevos.

## 3. Topología y Ciclo de Vida

M1 traduce la jerarquía de carpetas en lógica de negocio dentro de ArangoDB:

- Identificación de Obra: Cada carpeta de primer nivel en "Obra-Mayor" se convierte en un Nodo Proyecto.
- Detección de Estado: El nombre de la carpeta "Padre" define el atributo estado:
  - PRESUPUESTANDO, ACEPTADA, PROXIMA, EJECUCION, FINALIZADA, ARCHIVADA.
- Clasificación de Roles (Subcarpetas):
  - A[N]: Etiquetado automático como Documentación Administrativa.
  - O[N]: Etiquetado automático como Documentación de Obra.
  - T[N]: Etiquetado automático como Documentación Técnica.

## 4. Gestión de la Integridad y Duplicidad

Este proceso se ejecuta tras el escaneo y antes de la fase de IA:

1. Deduplicación Cross-Project: Si dos archivos en distintas obras tienen el mismo Hash, ArangoDB crea dos nodos Archivo pero un solo nodo Contenido.
2. Detección de Basura: Exclusión automática de archivos de sistema (desktop.ini, .DS_Store, .tmp).
3. Alerta de Limpieza: Notificación al M5 si existen archivos con contenido idéntico dentro de un mismo proyecto pero con nombres diferentes.

## 5. Contrato de Datos (Esquema del Nodo Archivo)

Cada registro en ArangoDB para M1 debe seguir esta estructura para ser compatible con el Dashboard (SvelteKit) y el Traductor (Python):

```TypeScript

interface ArchivoNodo {
  _key: string;               // Hash SHA-256**
  nombre: string;
  path_local: string;
  drive_id: string;
  mimeType: string;
  size_bytes: number;
  autor_real: string;         // Extraído de NTFS (SID)**
  fechas: {
    creacion: Date;
    modificacion: Date;       // Última modificación en disco**
    auditoria: Date;          // Fecha del último escaneo M1**
  };
  contexto: {
    obra_id: string;
    ciclo_vida: string;       // Ej: "EJECUCION"**
    rol_carpeta: string;      // Ej: "T1"**
  };
  stats: {
    paginas?: number;         // Extraído en lectura rápida (M1.5)**
    to_scan: boolean;    // Flag para necesidad de OCR**
  };
  estado_ia: "PENDIENTE" | "PROCESANDO" | "INDEXADO" | "ERROR";
}
```

## 6. Integración Técnica (Comunicación SvelteKit ➔ Python)

Para los tests y ejecuciones desde el Dashboard:

- Motor de Ejecución: FastAPI (Python) actuando como servidor de funciones remotas.
- Endpoint de Control: POST /m1/audit-path
- Payload: { path: string, recursive: boolean, force_hash: boolean }
- Seguimiento: El Dashboard (M5) realizará *polling* a ArangoDB para mostrar el progreso del conteo de archivos y detección de duplicados en tiempo real.

---

### Directrices de Mantenimiento para el Desarrollador:

- Lazy Metadata: M1 solo debe abrir archivos para extraer el Hash y el Índice (TOC). No debe intentar leer el texto completo para no saturar el bus de datos.
- Resiliencia: Si el acceso a un archivo está bloqueado por otro proceso (ej: un usuario con el Excel abierto), M1 debe saltarlo, loguear el aviso y reintentarlo en el próximo ciclo nocturno.
192