# Arquitectura de AUDITPRO

Cómo está organizada la herramienta y cómo llega a los usuarios. El código es privado; este
documento explica el diseño con tres diagramas. Las decisiones técnicas detrás de cada pieza
están en [TECNICAS.md](TECNICAS.md).

## 1. Flujo de un análisis

Lo que ocurre desde que el usuario sube su libro diario en Excel hasta que descarga el
resultado. El diagrama muestra que **todo análisis pasa por el mismo punto de entrada**, el
mapeo de columnas, y que este puede detener la ejecución.

```mermaid
flowchart TD
    A["Usuario sube el Excel del libro diario"] --> B["Lectura del archivo<br/>(mapeo_columnas)"]
    B --> C["Mapeo de columnas con estado<br/>detectada · por similitud · a mano · falta · repetida"]
    C -->|"falta una obligatoria<br/>o hay una repetida"| D["Aviso y botón bloqueado"]
    D --> C
    C -->|"mapeo correcto"| E["Archivo con nombres canónicos<br/>(las columnas ya llegan normalizadas)"]
    E --> F["Tipologías"]
    E --> G["Importes duplicados"]
    E --> H["Partes vinculadas"]
    E --> I["Pruebas de revisión<br/>(carga dinámica según selección)"]
    F --> J["Excel de resultados<br/>para descargar"]
    G --> J
    H --> J
    I --> J
```

## 2. Páginas independientes

La conciliación bancaria y las tipologías por usuario son páginas propias del menú lateral.
Tienen su propio flujo de subida y mapeo para **no poner en riesgo los análisis ya en uso**
cada vez que se añade una funcionalidad nueva.

```mermaid
flowchart LR
    subgraph CB["Conciliación bancaria"]
        direction TB
        b1["Dos extractos<br/>(banco y contabilidad)"] --> b2["Detección de hoja y cabecera<br/>+ mapeo de columnas"]
        b2 --> b3["Cruce 1:1<br/>importe + ventana de fechas"]
        b3 --> b4["Agrupaciones propuestas<br/>(subconjuntos acotados)"]
        b4 --> b5["Lotes por concepto<br/>(reglas, no ML)"]
        b5 --> b6["Saldos en cascada<br/>a mano › apertura › saldo corrido"]
        b6 --> b7["Excel de 8 hojas"]
    end
    subgraph TU["Tipologías por usuario"]
        direction TB
        t1["Libro diario<br/>(columna usuario obligatoria)"] --> t2["Mapeo de columnas"]
        t2 --> t3["Una fila por usuario y asiento"]
        t3 --> t4["Matriz usuario × tipología"]
        t4 --> t5["Excel: resumen, matriz<br/>y detalle por usuario"]
    end
```

## 3. Cómo llega la herramienta a las oficinas

La aplicación se ejecuta en el equipo de cada usuario, sin servidor. Para poder añadir
mejoras sin que nadie instale nada, el reparto funciona así. El diagrama muestra que una
versión **no llega a la oficina hasta que yo la doy por probada**.

```mermaid
flowchart LR
    dev["Desarrollo"] --> main["Rama de pruebas"]
    main --> test["Pruebas en un PC de desarrollo"]
    test -->|"aprobada"| prod["Rama de producción"]
    prod --> pc["Lanzador en cada PC<br/>(doble clic)"]
    pc --> step1["Descarga la última<br/>versión aprobada"]
    step1 --> step2["Instala dependencias"]
    step2 --> step3["Arranca la aplicación"]
```

Qué consigue este diseño:

* **Los datos no salen de cada equipo.** No hay servidor central que procese los Excel.
* **Los usuarios no hacen nada distinto.** Siguen abriendo la aplicación con doble clic.
* **Un error no llega a todos a la vez.** Lo que se prueba y lo que usa la oficina son
ramas separadas.
* **Funciona en equipos sin Python ni Git**, porque el paquete de instalación los incluye de
forma portátil.

