# IrantoAI

App general de CGR para gestión de proyectos

## **M115** Certificaciones de trabajos ejecutados

Modulo IrantoAI 115, certificaciones de trabajos ejecutados por contratas externas o personal interno.

### Finalidad

Datos de ejecución y base de documentación, para:

- Facturación de subcontratas.
- Incentivos de producción a personal propio.
- Estadísticas:
  - Seguimientos de productividad de personal ajeno/propio.
  - Velocidad y estado de ejecución de proyectos.
  - Repercusión de costes por unidad.
  - Consumo de materiales.
  - Otras a definir.
- Control de costes:
  - Comprobar desvío de costes por Ud.
  - Alinear precios partidas a repercusión real, para presupuestos futuros.
  - Identificar tareas ejecutadas no contempladas en presupuesto.
  - Otras a definir.

### Stack usado por empresa

- Arangodb
- Redis
- Ollama
- Nginx
- Docker
- OpenWebAi
- GitHub
- Sveltekit
- Python

### Metodología

Buscamos explorar las capacidades AI agentica, LLM y coordinador de procesos.

Como implementamos:

- Persistencia de datos, Graph-RAG ?
- Entrenamiento AI, que especialidades y cuantas se necesitan?
- Procesos auxiliares, que especialidades y cuantas se necesitan?
- Agente coordinador?

### Procesos auxiliares

Usaremos como documento fuente, el último presupuesto aceptado (pdf) y crearemos procesos auxiliares
que permitan la creación de datos necesarios.

- Plantillas
- formulas de cálculo y aplicación.
- Estructura de datos y relaciones.
- Impresión y firma de documento contractual.
- etc...

#### Proceso de comparación

Comparación de Capítulos, Partidas y mediciones del presupuesto con lo reflejado en Proyecto
del Arquitecto en busca de incongruencias.

#### Estructurar datos de origen

A partir del presupuesto deberemos extraer y estructurar los datos de manera que sirvan de base
para generar datos de certificaciones en el proceso de ejecución de la obra.

- Forma de presentación de presupuestos:
  - Único documento con todos los Capítulos.
  - Varios documentos por capítulos.
- Estructura esperada de presupuesto:
  - Capítulos: Incluye todas las partidas que se contempla ejecutar para la realización propuesta.
  - Partidas: Incluye todos los trabajos y características condicionales que deberían realizarse.
    para la correcta ejecución de la partida.
  - Unidades: tipo de unidad de referencia para medición [m, m², m³, Kg, Ud., ...].
  - Cantidad: cantidad de unidades contempladas.
  - Precio: Precio de cada unidad, relevante sólo para estadísticas de control de costes. El precio ce certificaciones aplicable al pago de contratas e incentivos de personal propio se definirá en las plantillas

#### Datos de destino

Se refiere a estructura de datos a la que se debe "traducir" los datos de la estructura de origen. También, resto de estructura de datos
que permitan organizar los datos para analizar y visualizar otros aspectos de la gestión de proyectos asociada.

1.Correspondencia de la organización de ejecución

_La nomenclatura de origen, como vista normalizada entre interlocutores: Empresa, Arquitecto, Cliente,
difiere de la interna al ejecución de obra, así:_

- Vista original: [Capítulo -> Partida -> Trabajos]
- Vista interna a obra: [Actuaciones -> Familia -> Grupo -> Partida -> Tareas]

  2.Trabajos mas habituales en vista interna:

- Fachada Ventilada grapa vista (efivento)
- Fachada Ventilada grapa oculta (efivento+)
- SATE
- Papel (efimeta)
- Cubierta
- Carpinterías
- Fontanería
- Otros (Fotovoltaica, equitación, etc.)

#### Ejemplo datos de origen y su 'traducción'

Utilizaremos el documento "Pto. Tirso de Molina. 10 - v04.pdf, Ejemplo de presentación en único documento.

> CAPÍTULO 1 FACHADA PRINCIPAL - FACHADA VENTILADA, CERÁMICO Y COMPOSITE

_El capítulo 1 indica que es actuación en facha principal del tipo FACHADA VENTILADA (efivento o efivento+); no sabemos
todavía que variedad es, pero leyendo las partidas lo descubriremos para definirlo exactamente._

---

> Partidas: 01.01 **ud Acometidas Agua y Electricidad** hasta **01.08 ud Retirada de ganchos de mantenimiento**

_Las partidas de la 1 a la 8 las clasificaríamos pertenecientes a la familia de "Trabajos previos", normalmente realizadas por personal propio._

> **01.09 ud Pre-instalación de tubos acometidas adap. nuevo sist. Fachada / 20,00 ud** ->
> **01.10 ud Modificación tuberias gas i/suministro de nuevas llaves de corte** ->
> **01.11 ud Adaptación de rejillas de ventilación** ->
> **01.12 Ud Adaptación Conductos Calderas y al nuevo sistema de fachada**

_El texto de las partidas definen trabajos de instalaciones, la asignamos como muestra objeto_.
**Sujeto a revisión** se debe optimizar desde contexto global del presente plan!

```javascript
[
  {
    familia: "instalaciones",
    grupo: "acometidas", // el nombre es facilmente deducible por contexto
    name: "conductos", // quedaria definido mediante plantillas para nomralizar nombre.
    tareas: [
      {
        name: "instalcion",
        qty: 20,
        price: 130, // el aplicado por nivel.
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
    ],
  },
  {
    familia: "instalaciones",
    grupo: "acometidas",
    name: "gas",
    tareas: [
      {
        name: "modificacion",
        qty: 20,
        price: 260, // el aplicado por nivel
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
    ],
  },
  {
    familia: "instalaciones",
    grupo: "ventilacion",
    name: "rejillas",
    tareas: [
      {
        name: "modificacion",
        qty: 10,
        price: 12, // el aplicado por nivel
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
    ],
  },
  {
    familia: "instalaciones",
    grupo: "ventilacion",
    name: "calderas",
    tareas: [
      {
        name: "modificacion",
        qty: 10,
        price: 110, // el aplicado por nivel
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
    ],
  },
];
```

---

> **01.13 m2 Levantado carpintería exterior de terrazas** -> **01.14 m2 Carp. PVC con vidrio 4-16-8 sin compacto según planos**

_El texto de las partidas definen trabajos de carpinterías_.

Los trabajos de carpinterías requieren tratamiento especial.
Normalmente obedecen a un proyecto especifico que detallan los tipos y características. Por, ej.: se detallan 3 tipos de ventanas
a sustituir: v1 de dimensiones alto x ancho (características de vidrio ...), tipo 2, tipo 3.... El presupuesto unifica todo en superficie por
tipo, dando el importe. Como hacer una traducción de certificación ???:

- Una opción es encontrar en las mediciones del proyecto el desglose de carpinterías.
- Otra, definir manualmente los tipos a certificar.
- Las dos opciones deberán tener una clasificación de familia, Grupo, Partida, tareas, que sean coherentes, **propuesta**:
  "carpinterías", ["ventanas", "balcones", "claraboyas", "cierres", "barandillas", ...], ["v1", "v2", ...], ["retirada", "instalacion"]

**Sujeto a revisión** se debe optimizar desde contexto global del presente plan!

---

> **01.15 m2 Sub-estructura de aluminio para Fachada Ventilada** -> **01.16 m2 Aislamiento cámara ventilada con panel tipo Kooltherm K15 80mm** ->
> **01.17 m2 Revest. placas cerámicas alveolares, p.p.composite- F. Ventilada** -> **01.23 m2 Revestimiento vuelo horizontal P. 1a con aislamiento**

- _El texto define partidas hacen referencia con la instalación de estructura, aislamiento y revestimiento de una fachada ventilada._
- _En la descripción de la partida 01.17, nos aclara que las piezas cerámicas se instalan con sujeción oculta, por lo tanto, podemos inferir el grupo como **Ventilada Grapa Oculta** y la actuación como: Fachada Ventilada Grapa Vista (efivento+)_

```javascript
[
  {
    familia: "Fachada",
    grupo: "Ventilada grapa oculta", // "Ventilada grapa vista", "SATE", "Muro cortina", "Panel"
    name: "aislamiento",
    uds: "m2",
    tareas: [
      {
        name: "instalacion",
        price: 5,
        qty: 133.36, // +(13.23 de la partida 01.23)
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m2"
      },
      {
        name: "encintado",
        price: 2.5,
        qty: 133.36, // +(13.23 de la partida 01.23)
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
    ],
  },
  {
    familia: "fachada",
    grupo: "Ventilada grapa oculta",
    name: "subestructura",
     uds: "m2",
    tareas: [
      {
        name: "mensulas",
        price: 8,
        qty: 133.36, // +(13.23 de la partida 01.23)
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
      {
        name: "montantes",
        price: 8,
        qty: 133.36, // +(13.23 de la partida 01.23)
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
      {
        name: "travesanos",
        price: 4,
        qty: 133.36, // Aqui la 01.23 no aplica
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
    ],
  },
  {
    familia: "fachada",
    grupo: "Ventilada grapa oculta",
    name: "revestimiento",
    /* Las tareas de revestimiento, normalmente, las componen "ceramica" y " composite",
    pero también pueden ser "SATE" o "panel" en pequeñas zonas de la fachada.
    Se definen como tareas que realiza la misma contrata y se certifican sin desglosar.
    No deben ser confundidas con actuaciones que en el grupo se definan "SATE" o "panel",
    que se refieren a la instalación de zona entera y asignada a otra contrata. Estas sí
    llevan desglosados de partidas y tareas.
    */
    uds: "m2",
    tareas: [
      {
        name: "cerámica",
        price: 8,
        qty: 133.36,
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
      {
        name: "composite",
        price: 8,
        qty: 13.68, // viene de partida 01.23
        percentPrevCert: 0,
        percentThisCert: 0.5,
         uds: "m2"
      },
      {
        name: "SATE",
        price: 38,
        qty: 20.26, // viene de partida 01.24
        percentPrevCert: 0,
        percentThisCert: 0,
         uds: "m2"
      },
      {
        name: "panel",
        price: 18,
        qty: 0,
        percentPrevCert: 0,
        percentThisCert: 0,
         uds: "m2"
      },
    ],
  },
];
```

> **01.23 m2 Revestimiento vuelo horizontal P. 1a con aislamiento** -> **01.24 m2 Revestimiento horizontal y vertical alero aislamiento S.A.T.E**

***Observamos detalladamente 01.23 y 01.24***. Hablan de revestimiento en zonas especiales del edificio,
Aleros y Vuelos, debemos incorporarlas en las certificaciones. El sistema SATE no tiene sub-estructura y como es una zona concreta y de pequeño tamaño, por eficiencia la unificamos en una tarea, sumando al objeto anterior. La 01.23 pide usar composite y por lo tanto, sub-estructura. Lo descomponemos en las tareas comunes de fachada ventilada.

---

> **01.18 ml Aislamiento en recercados ventanas y picado previo** -> **01.19 ml Remate recercados en huecos de ventanas** -> **01.20 ml Vierteaguas de aluminio composite** -> **01.22 ml Albardilla Composite con Aislamiento térmico, Impermeabilización** -> **01.25 ml Remate de esquinas con chapa de aluminio**

_Esta partidas podemos inferir claramente tareas de remates del sistema._

```javascript
[
  {
    familia: "remates",
    grupo: "recercados",
    name: "jambas",
    uds: "m",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 77.6,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "epingles",
        price: 4,
        qty: 77.6,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 77.6,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 77.6,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },
  {
    familia: "remates",
    grupo: "recercados",
    name: "dinteles",
    uds: "m",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 60.3,
        /* en la descripción de la partida **01.19** unan en una medición jambas y dinteles.
      Debido a la diferencia de producción del remate, es necesario separar las mediciones.
      El método es pensar que la longitud de los vierteaguas (inferior de ventana) es igual y paralelo
      al dintel, por lo tanto, usamos medicion de vierteaguas y restamos esta de **01.19** */
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "epingles",
        price: 4,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },
  {
    familia: "remates",
    grupo: "recercados",
    name: "vierteaguas",
    uds: "m",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "epingles",
        price: 4,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 60.3,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },
  {
    familia: "remates",
    grupo: "horizontales",
    name: "superiores",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 14.47,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "epingles",
        price: 4,
        qty: 14.47,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 14.47,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 14.47,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },
  {
    familia: "remates",
    grupo: "horizontales", // "Ventilada grapa vista", "SATE", "Muro cortina", "Panel"
    name: "inferiores",
    uds: "m",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 133.36,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "epingles",
        price: 4,
        qty: 133.36,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 133.36,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 133.36,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },
  {
    familia: "remates",
    grupo: "verticales", // "Ventilada grapa vista", "SATE", "Muro cortina", "Panel"
    name: "esquinas",
    uds: "m",
    tareas: [
      {
        name: "instalacion",
        price: 10,
        qty: 56.0,
        percentPrevCert: 0,
        percentThisCert: 0.5,
      },
      {
        name: "epingles",
        price: 4,
        qty: 56.0,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "aislamiento",
        price: 4,
        qty: 56.0,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
      {
        name: "inpermebilizado",
        price: 2.5,
        qty: 56.0,
        percentPrevCert: 0,
        percentThisCert: 0.5,
        uds: "m",
      },
    ],
  },º
];
```

#### Retos de diseño de datos

Lo anterior es una estructura básica, que puede ser útil si solo pretendemos gestionar certificaciones con subcontratas y operarios (descendente), pero hay otra información y procesos que no contemplan per necesitarían de ellos, retos:

- Seguimientos de productividad de personal ajeno/propio.
- Velocidad y estado de ejecución de proyectos.
- Emisión de certificaciones a clientes: Los dos procesos están relacionados, pero el Arquitecto espera que le enviemos el grado de avance con formato de sus mediciones y descomposición de partidas y no, nuestra "traducción descendente", se necesita aplicar el avance con "traducción ascendente" al presupuesto de partida.
- Control de costes: Lo anterior no indica granularidad de materiales; podemos controlar solo la productividad , y no es necesario para el control de costes. Hay partidas que claramente indican el material a usar, otras no y deben ser indicados, por ej.; en 01.23 sabemos que necesitamos Composite, pero en la 01.17, aunque diga cerámica en general, por diseño elegido por cliente, puede tener zonas de composite y otras de cerámica. Así,
  - Como especificamos el material a usar, podemos controlar el coste de materiales?
  - Como reflejamos las perdidas de sobrantes y desperdicios por cortes?
  - Como enlazamos con el control de residuos?
  - Como generamos conocimiento para saber repercusión de costes por Ud., para futuros presupuestos?
- Como definimos que tareas especificas van a desarrollar los operarios y subcontratas?, si, por necesidades de ejecución de obra, mezclamos recursos humanos en una misma obra.
- Como implementamos control de calidad de los trabajos?, por ej.: se instalan las ménsulas con sus fijaciones, el encargado hace una inspección aleatoria de control de par de apriete con herramienta digital, inspecciones visuales, etc.
- otros a definir.
