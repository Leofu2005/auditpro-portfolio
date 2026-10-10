# Técnicas de análisis de datos aplicadas en AUDITPRO

Este documento no repite qué hace cada función — eso ya está en el [README](README.md). Aquí
explico **qué técnica de análisis de datos hay detrás de cada decisión y por qué esa y no
otra**. Cada sección sigue el mismo esquema: el problema de negocio, la técnica aplicada, por
qué se eligió frente a las alternativas, y en qué parte del código vive (el código es
privado; se indica el módulo y la función).

**Filosofía de fondo, antes de entrar en detalle**: en casi todas las decisiones de este
proyecto se priorizan **reglas deterministas y explicables sobre modelos estadísticos o de
machine learning**, aunque en varios casos (agrupar movimientos por similitud, por ejemplo)
un modelo de clustering habría sido la opción "más sofisticada". La razón es el contexto: en
una herramienta de auditoría, el auditor tiene que poder justificar *por qué* dos movimientos
se consideraron relacionados. "El modelo los agrupó" no es una respuesta aceptable en un
informe de auditoría; "comparten concepto y son consecutivos en el tiempo" sí lo es. Esta
idea reaparece en el bloque 7c.

**Índice**

1. Normalización de texto (Unicode)
2. Fuzzy matching de columnas
3. Validación de partida doble con conjuntos
4. Fingerprinting estructural de asientos
5. Detección de duplicados vía fingerprint + importe
6. Ley de Benford
7. Conciliación bancaria (7a–7e)
8. Reshaping de datos
9. Mapeo de columnas con estado y bloqueo de ejecución
10. Límites actuales y qué haría distinto

---

## 1. Normalización de texto (Unicode)

**Problema**: comparar cabeceras de Excel o texto libre contra una lista de alias conocidos
falla si no se normaliza antes — "Débito", "DEBITO" y "debito" deberían tratarse como lo
mismo, y no lo son si se comparan tal cual.

**Técnica**: `unicodedata.normalize('NFD', texto)` descompone cada carácter acentuado en
su forma base + marca diacrítica, y se descartan las marcas (`category(c) != 'Mn'`) — así
"é" se convierte en "e" de forma robusta, sin tener que mantener una tabla de sustituciones
á→a, é→e, í→i... a mano.

**Por qué esta y no otra**: la alternativa ingenua (`texto.replace('á', 'a').replace('é', 'e')...`)
exige enumerar cada vocal acentuada posible y se rompe con cualquier carácter que no se
haya previsto. La normalización Unicode NFD es la forma estándar de resolver esto para
cualquier alfabeto latino, sin mantenimiento.

**En el código**: módulo `mapeo_columnas`, función `normalizar_texto()`. Es la copia que se
considera "fuente de verdad"; el mismo patrón aparece repetido en otros módulos (duplicación
conocida, ver el bloque 10).

```python
def normalizar_texto(texto):
    texto = ''.join(c for c in unicodedata.normalize('NFD', str(texto))
                   if unicodedata.category(c) != 'Mn')
    return texto.lower().strip().replace(' ', '_').replace('.', '').replace('-', '_')
```

---

## 2. Fuzzy matching de columnas

**Problema**: cada ERP exporta las mismas columnas con nombres distintos ("Cargo", "Débito",
"Importe Debe" significan todas lo mismo). Un Excel real nunca tiene exactamente las
cabeceras que el código espera.

**Técnica**: dos fases. Primero, coincidencia **exacta** contra un diccionario de alias por
rol contable (más de 50 variantes conocidas repartidas en 9 roles). Si no hay coincidencia
exacta, `difflib.get_close_matches()` con un umbral de similitud del **60%** (`cutoff=0.6`),
que compara secuencias de caracteres (algoritmo de Ratcliff/Obershelp), no palabras.

**Por qué esta y no otra**: un modelo de embeddings semánticos sería más "inteligente" para
casos raros, pero es excesivo para este problema — el diccionario de alias cubre la inmensa
mayoría de los casos reales de forma instantánea y sin dependencias nuevas, y el fuzzy
matching de respaldo solo entra cuando el diccionario no basta. Importante — **límite real
medido**: el fuzzy matching compara por similitud de caracteres, no de significado; variantes
con las palabras en otro orden (p. ej. "Cuota IVA" vs. "IVA Cuota") pueden no superar el
60% de similitud aunque signifiquen lo mismo para un humano. Por eso el proyecto combina esto
con una segunda capa — mapeo asistido con confirmación manual del usuario — para los casos
que el automatismo no resuelve (ver el bloque 9).

**En el código**: módulo `mapeo_columnas` (diccionario de alias y función de coincidencia
aproximada).

```python
similares = get_close_matches(alias, restantes, n=1, cutoff=0.6)
```

---

## 3. Validación de partida doble con conjuntos

**Problema**: antes de calcular nada sobre un asiento, hay que descartar los que no tienen
movimiento tanto en Debe como en Haber (asientos de apertura, o líneas sueltas sin su
contrapartida) — si se cuelan, distorsionan cualquier cálculo posterior.

**Técnica**: construir dos **conjuntos** (`set`) de números de asiento — los que tienen algún
`debe != 0` y los que tienen algún `haber != 0` — y quedarse con su **intersección** (`&`).

**Por qué esta y no otra**: la alternativa habría sido un `merge`/`join` de un DataFrame
consigo mismo filtrado por Debe y por Haber, que es más caro computacionalmente (construye
tablas intermedias) para resolver un problema que es, en esencia, una operación de teoría de
conjuntos pura. Con `set` la intersección es una operación O(n) sobre identificadores, sin
necesidad de pandas para esta parte concreta.

**En el código**: módulo `tipologias.detalle_tipologias`, función `generar_archivo_tipologias()`.

```python
asientos_debe = set(df[df['debe'] != 0]['asiento'])
asientos_haber = set(df[df['haber'] != 0]['asiento'])
asientos_validos = list(asientos_debe & asientos_haber)
```

---

## 4. Fingerprinting estructural de asientos

**Problema**: para comparar asientos entre sí (¿son del mismo "tipo"?), comparar importes
exactos no sirve — dos asientos de importes distintos pueden tener la misma naturaleza
contable, y dos de importe idéntico pueden ser de naturalezas totalmente distintas.

**Técnica**: construir una "huella" (fingerprint) que resume la **forma** del asiento, no su
valor — las clases de cuenta (3 primeros dígitos del PGC) que tienen movimiento en Debe y en
Haber, ordenadas y unidas en un string: `"430,472(D)/770(H)"`. Dos asientos con la misma
huella tienen la misma estructura contable, independientemente del importe.

**Por qué esta y no otra**: es el mismo principio que un hash estructural — comparar por
"forma" en vez de por "contenido" — pero expresado como un string legible por un humano en
vez de un hash binario, a propósito: el auditor necesita poder leer `"430,472(D)/770(H)"`
directamente en el informe, no un hash opaco.

**En el código**: módulo `tipologias.detalle_tipologias`, función `generar_tipologia()`.

```python
def generar_tipologia(grupo):
    debe = sorted(set(grupo.loc[grupo['debe'] != 0, 'cuenta_clase']))
    haber = sorted(set(grupo.loc[grupo['haber'] != 0, 'cuenta_clase']))
    return f"{','.join(debe)}(D)/{','.join(haber)}(H)"
```

---

## 5. Detección de duplicados vía fingerprint + importe

**Problema**: detectar posibles facturas duplicadas o errores de imputación repetidos, sin
comparar asientos uno a uno (coste cuadrático si se hiciera por fuerza bruta).

**Técnica**: reutilizar el fingerprint del bloque 4, combinarlo con el importe del asiento, y
agrupar (`groupby(['tipologia', 'importe_asiento'])`) — cualquier grupo con 2 o más asientos
es un duplicado candidato. Es una agrupación O(n log n), no una comparación par a par.

**Por qué esta y no otra**: comparar cada asiento contra todos los demás sería O(n²) e
inviable con miles de asientos. Agrupar por una clave compuesta (tipología + importe) delega
en pandas el trabajo de encontrar coincidencias, con la misma garantía de resultado.

**En el código**: módulo `criterios.importes_duplicados`, función `generar_importes_duplicados()`.

```python
grupo_duplicados = df.groupby(['tipologia', 'importe_asiento'])['asiento'].nunique().reset_index()
```

---

## 6. Ley de Benford

**Problema**: detectar manipulación de cifras sin tener ningún dato de referencia "correcto"
contra el que comparar — hace falta una propiedad estadística universal de los números
naturales, no específica de este cliente.

**Técnica**: la Ley de Benford predice que, en conjuntos de datos que surgen de procesos
naturales (no inventados), el primer dígito sigue una distribución logarítmica:
`P(d) = log₁₀(1 + 1/d)`. Se calcula la distribución real de primeros dígitos de todos los
importes y se compara contra la esperada — desviaciones grandes son indicio de cifras
manipuladas o inventadas (las personas son malas generando números "aleatorios" a mano, y
tienden a sobrerrepresentar ciertos dígitos).

**Por qué esta y no otra**: es una prueba de auditoría forense reconocida internacionalmente,
no una técnica inventada para este proyecto — es la razón por la que este criterio concreto
es el que más valor tiene de cara a un auditor profesional: no hay que justificar la
metodología desde cero.

**En el código**: módulo `criterios.Ley_Benford`, función `detectar_Ley_Benford()`.

```python
benford_esperado = {d: 100 * np.log10(1 + 1/d) for d in range(1, 10)}
```

---

## 7. Conciliación bancaria (la más extensa, con diferencia)

Es el motor más elaborado del proyecto — cruza un extracto bancario contra uno contable sin
ningún identificador común entre ambos lados, solo importe y fecha. Genera un Excel de 8
hojas: resumen, los dos extractos completos, parejas 1:1, agrupaciones a revisar, lo que
solo está en el banco, lo que solo está en la contabilidad, y los grupos de partidas sin
explicar.

### 7a. Matching 1:1 tipo bipartito, voraz (greedy) con ventana temporal

**Problema**: emparejar cada movimiento del banco con su equivalente en la contabilidad,
cuando no hay ningún ID compartido entre los dos lados.

**Técnica**: para cada línea del banco, busca candidatos con el mismo importe (tolerancia
configurable) dentro de una ventana de fechas (±N días), y si hay varios, elige el de fecha
más cercana. Es un **algoritmo voraz** — resuelve de una en una, en el orden en que procesa
las líneas, sin volver atrás.

**Por qué esta y no otra**: este es un problema de **asignación óptima en un grafo
bipartito** en su forma más pura (banco a un lado, contabilidad al otro, aristas = pares con
mismo importe/fecha compatible) — existen algoritmos exactos para esto (p. ej. el algoritmo
húngaro), que garantizan la asignación óptima global. Se descartó a propósito: para el
volumen real de un extracto bancario (cientos-miles de líneas, no millones), un algoritmo
voraz con desempate por fecha más cercana da un resultado equivalente en la práctica, es
mucho más simple de leer, depurar y explicar a un auditor no técnico, y no necesita ninguna
librería de optimización adicional. Es una decisión consciente de "suficientemente bueno y
simple" sobre "óptimo y complejo", no un desconocimiento del problema.

**En el código**: módulo `conciliacion.conciliacion_bancaria`, función `_pasada_1a1()`.

### 7b. Subset-sum con early-stopping para distinguir match único de ambiguo

**Problema**: cuando varias líneas sueltas de un lado suman exactamente el importe de una
línea del otro (p. ej. 3 recibos de 7,26 € = un cargo de 21,78 €), hay que encontrarlas sin
probar combinaciones sin límite (el número de subconjuntos posibles crece exponencialmente).

**Técnica**: búsqueda de subconjuntos acotada por tamaño máximo de grupo (`max_grupo`) y
número máximo de candidatos a probar (`max_candidatos`), con **parada temprana**: en cuanto
se encuentran 2 combinaciones válidas, se deja de buscar — no hace falta contarlas todas,
solo saber si hay "una" (se propone) o "más de una" (ambiguo, no se decide).

**Por qué esta y no otra**: fuerza bruta sin límites es inviable (combinatoria exponencial);
limitar el tamaño de grupo y el nº de candidatos es una poda de coste, no una poda de
exhaustividad en el sentido en que importa: si hay una solución **única**, se encuentra con
2 intentos como máximo gracias a la parada temprana, sin necesidad de enumerar todas las
combinaciones posibles.

**En el código**: módulo `conciliacion.conciliacion_bancaria`, funciones
`_subconjuntos_que_suman()` y `_agrupar()`.

### 7c. Agrupación por reglas deterministas en vez de clustering estadístico

**Problema**: cuando ni el cruce 1:1 ni la combinatoria acotada encuentran nada (p. ej. una
nómina repartida en 18 transferencias, muy por encima de cualquier límite razonable de
combinatoria), hace falta otra forma de agrupar movimientos relacionados.

**Técnica**: agrupar primero las líneas pendientes **de un mismo lado** por "raíz de
concepto" (texto normalizado, sin los números variables) + proximidad de fechas consecutivas,
en "lotes" — y comparar la **suma** de cada lote contra el otro lado, sin combinatoria.

**Por qué esta y no otra**: la alternativa obvia sería un clustering no supervisado (k-means,
DBSCAN) sobre embeddings del texto del concepto. Se descartó a propósito, no por
desconocimiento: en una herramienta de auditoría, cada agrupación tiene que poder justificarse
ante un cliente o en un informe. "El modelo los consideró similares" no es una respuesta
aceptable; "comparten raíz de concepto y son fechas consecutivas" sí lo es. Es la misma idea
de la filosofía de apertura del documento, aplicada aquí de forma muy concreta.

**Una aclaración de nombres**: la hoja del Excel de resultados que agrupa las partidas sin
explicar se llama "Clusters" en el sentido coloquial de "grupos". Detrás **no hay
aprendizaje automático**, sino la agrupación por reglas que se acaba de describir.

**En el código**: módulo `conciliacion.conciliacion_bancaria`, funciones `_concepto_raiz()` y
`_agrupar_por_concepto()`.

### 7d. Reconstrucción de saldos con fallback en cascada

**Problema**: calcular el saldo inicial de un extracto cuando distintos bancos/ERPs lo
representan de formas distintas — a veces hay una columna dedicada, a veces una fila marcada
como "Apertura", a veces solo el saldo corrido.

**Técnica**: tres métodos probados en cascada por orden de fiabilidad — **dato aportado a
mano** > **fila/columna de Apertura** > **primer valor del saldo corrido del extracto**. Si
ninguno existe, no se inventa nada: se devuelve explícitamente "pendiente de dato externo".

**Por qué esta y no otra**: en vez de exigir un formato único de entrada (que fallaría con
la mitad de los extractos reales), el fallback en cascada se adapta al dato disponible sin
intervención del usuario en el caso común, y solo pide el dato a mano cuando de verdad no hay
forma de deducirlo.

**En el código**: módulo `conciliacion.conciliacion_bancaria`, funciones `extraer_apertura()`,
`saldos_corridos()` y `_lado_saldos()` (donde vive la cascada de prioridad).

### 7e. Robustez ante datos de entrada reales — deduplicación de cabeceras

**Problema real de producción** (no hipotético: ocurrió con un extracto bancario real): un
Excel con dos columnas con el mismo nombre de cabecera (p. ej. celdas combinadas en el
origen) hacía que `df[nombre_columna]` devolviera un DataFrame de 2 columnas en vez de una
Serie, y el código reventaba más adelante con `ValueError: cannot reindex on an axis with duplicate labels` al intentar combinar esa columna "doble" con otra en una operación lógica.

**Técnica**: tras construir la cabecera, detectar nombres repetidos y renombrarlos con un
sufijo numérico (`Concepto`, `Concepto.1`...) — exactamente el mismo comportamiento que
pandas aplica automáticamente cuando lee un Excel de la forma estándar (`header=0`).

**Por qué esta y no otra**: el motivo de que el bug existiera en primer lugar es instructivo:
este módulo construye la cabecera **a mano** (para detectarla de forma robusta en extractos
con filas de título variables), y al hacerlo se salta sin querer esa protección automática
que pandas ya incorpora. La solución correcta no es inventar un mecanismo propio, sino
**replicar exactamente el comportamiento estándar** que se perdió al tomar el control manual
del proceso — así el resto del código (que ya maneja columnas con sufijo `.1` por diseño del
propio pandas en otros contextos) no necesita ningún cambio adicional.

**En el código**: módulo `conciliacion.conciliacion_bancaria`, función `_deduplicar_cabeceras()`.

```python
def _deduplicar_cabeceras(nombres):
    vistos = {}
    resultado = []
    for nombre in nombres:
        if nombre not in vistos:
            vistos[nombre] = 0
            resultado.append(nombre)
        else:
            vistos[nombre] += 1
            resultado.append(f"{nombre}.{vistos[nombre]}")
    return resultado
```

---

## 8. Reshaping de datos

**Problema**: un asiento puede tener líneas de varios usuarios distintos, y hay que poder
analizar "qué tipología genera cada usuario" — una relación de uno (asiento) a muchos
(usuarios) que no encaja en una fila por asiento.

**Técnica**: `DataFrame.explode()` convierte una columna que contiene listas en varias filas,
una por elemento de la lista — un asiento con 3 usuarios se convierte en 3 filas, una por
usuario (decisión de diseño explícita: el asiento cuenta para cada usuario que participó en
él, no se trata como una categoría compuesta). Después, `pd.pivot_table()` construye la
matriz Usuario × Tipología con el conteo de asientos en cada celda, para el análisis visual
de segregación de funciones.

**Por qué esta y no otra**: son las herramientas estándar de pandas para exactamente este
patrón (relación 1-a-N y matriz de doble entrada) — no hay motivo para reinventarlas con
bucles manuales, que serían más lentos y más propensos a errores.

**En el código**: módulo `tipologias.tipologias_por_usuario`, función
`generar_tipologias_por_usuario()`.

```python
expandido = asientos_df.explode('usuarios').rename(columns={'usuarios': 'usuario'})
matriz_df = pd.pivot_table(expandido, index='usuario', columns='tipologia',
                            values='asiento', aggfunc='count', fill_value=0)
```

---

## 9. Mapeo de columnas con estado y bloqueo de ejecución

**Problema**: el bloque 2 resuelve la mayoría de los casos, pero un mapeo automático que
acierta mal una columna es el peor fallo posible en esta herramienta: el análisis sigue
adelante con datos equivocados y **nadie se entera**. En auditoría, un resultado limpio
sobre una columna errónea es peor que un error visible.

**Técnica**: tras subir el archivo, siempre se muestra una tabla con un desplegable por dato
y su estado (detectada, por similitud, elegida a mano, falta, repetida). Si falta una
columna obligatoria o dos datos apuntan a la misma columna, se avisa y se **bloquea el botón
de ejecutar** hasta que el usuario lo resuelva. Hay seis columnas obligatorias (asiento,
fecha, cuenta, debe, haber, concepto) y una opcional (usuario). Una vez confirmado el mapeo,
el archivo se reescribe con los nombres canónicos antes de pasar a los análisis, porque estos
leen el Excel de disco y no deben tener su propia lógica de mapeo.

**Por qué esta y no otra**: dejar que cada análisis hiciera su propio fuzzy matching habría
multiplicado los puntos de fallo (y de mantenimiento). Un único punto de entrada que valida,
muestra y bloquea concentra la decisión de "qué columna es cuál" en un solo sitio, visible
para el usuario, y deja a los análisis con una suposición simple: las columnas ya llegan
normalizadas.

**En el código**: módulo `mapeo_columnas` (`mostrar_mapeo()`, `renombrar_para_motor()`) y
módulo `requisitos`, que lista las columnas necesarias por opción.

---

## 10. Límites actuales y qué haría distinto

Un documento técnico que solo cuenta lo bueno no es fiable. Estos son los límites que conozco
del proyecto hoy:

* **Cobertura de pruebas mínima.** Solo hay una prueba automática (el criterio de múltiplos
de 1.000). El resto se comprueba a mano con Excel de ejemplo. Es la deuda técnica más
importante.
* **Sin esquema de datos declarado.** Los nombres de columna están repartidos entre el
diccionario de alias, la validación y el acceso directo dentro de cada análisis. Declararlo
en un solo sitio (con una librería como `pandera`) daría errores claros en el punto de
entrada.
* **Código duplicado.** La normalización de texto está repetida en varios módulos. El bloque
1 señala cuál es la fuente de verdad, pero la duplicación sigue ahí.
* **Transformaciones sin vectorizar** en algunos puntos (por ejemplo, la generación de
tipologías usa `groupby().apply()`), aceptable con los volúmenes actuales pero no con
millones de filas.
* **Estado compartido en disco.** La aplicación guarda el archivo subido en una ruta fija
antes de analizarlo. Con un solo usuario por equipo no hay problema, pero sería una
condición de carrera si dos personas usaran la misma instancia a la vez.
* **Errores y límites.** Se capturan con una excepción genérica y no hay límite de tamaño de
archivo ni registros (logs) estructurados.
* **El calendario de festivos de la prueba de fechas inusuales es un archivo fijo.** La prueba
  señala los asientos registrados en fin de semana o en festivo, y los festivos salen de un
  Excel de un solo año y una sola localidad. Para otros años o ciudades hay que aportar otro
  calendario. Declarar el calendario como dato de entrada, en vez de dejarlo fijo, es una
  mejora pendiente.
* **El algoritmo voraz de la conciliación** (7a) no garantiza la asignación óptima global.
Es una decisión consciente, pero conviene medirla con extractos más grandes.

Las mejoras previstas salen directamente de esta lista: separar lectura, transformación y
escritura en cada módulo; declarar el esquema una vez; ampliar las pruebas; y añadir logs y
excepciones específicas.

