# Entrega 4 - Diseño del análisis y estrategia de modelado

## 1. Problema que se busca resolver

**Qué ocurre hoy.** Quien busca una ayuda pública se enfrenta a información dispersa entre varios organismos, redactada en lenguaje administrativo y con plazos que cambian cada año. El resultado habitual es que la persona no sabe qué existe, no sabe si le aplica o descubre demasiado tarde que la convocatoria ya cerró. Por eso hay ayudas a las que se tiene derecho y que no se piden.

**Quién usará el resultado y para qué.** Cualquier persona de Córdoba (autónomos, personas en desempleo, jóvenes que buscan alquiler, pensionistas, pymes) que quiera saber **qué ayuda le conviene solicitar ahora y dónde hacerlo**. La decisión que mejora es "¿solicito esta ayuda o no, y con qué plazo?".

**Qué resultado hace útil al proyecto.** Que, dada una consulta en lenguaje natural, el sistema:

1. devuelva las ayudas realmente relevantes para el perfil descrito;
2. no presente como disponible una ayuda que ya cerró;
3. dé el importe y el plazo correctos, con su significado (presupuesto total o máximo por persona) y el enlace a la fuente oficial;
4. reconozca con honestidad cuando no hay ninguna ayuda que encaje, en lugar de inventarla.

El proyecto **no necesita un modelo predictivo clásico**: no hay una variable que predecir. Lo que hay que resolver es un problema de **recuperación de información y recomendación** sobre un catálogo pequeño y verificado, que explico en la sección 3.

## 2. Análisis de datos planteado y utilidad esperada

El análisis tiene tres momentos: entender el catálogo antes de construir nada, observar cómo se comporta la búsqueda durante el desarrollo y medir los resultados al final.

| Momento | Pregunta que quiero responder | Análisis | Utilidad |
|---|---|---|---|
| **Antes** | ¿Qué contiene realmente el catálogo? | Distribución por `ambito_geografico`, `tipo_ayuda` y `sector`; cobertura de `edad_min`/`edad_max`; porcentaje de "No especificado" por columna | Saber qué perfiles están bien cubiertos y cuáles no, y qué campos son fiables para filtrar |
| **Antes** | ¿Qué está abierto y cuándo cierra? | Recuento por `estado_calculado`; calendario de cierres a 30, 60 y 90 días | Indicador útil para el usuario y para explicar cuándo conviene actuar |
| **Antes** | ¿Puedo comparar importes? | Distribución de `importe_max` separada por `tipo_importe` | Justificar por qué el sistema nunca ordena ni compara importes sin distinguir su tipo |
| **Durante** | ¿La búsqueda semántica mezcla ámbitos por parecido de palabras? | Revisión de las ayudas recuperadas con y sin filtros, y de la distribución de similitudes | Decidir si hace falta un filtro estructurado previo y con qué campos |
| **Durante** | ¿El agente extrae bien los filtros de la pregunta? | Comparación manual entre filtros extraídos y filtros esperados en el conjunto de desarrollo | Ajustar el prompt de extracción y detectar casos que se le escapan |
| **Después** | ¿Dónde falla el sistema? | Errores por segmento: ámbito, tipo de ayuda y perfil del usuario | Priorizar mejoras y redactar las limitaciones con datos, no con impresiones |

**Hipótesis que quiero comprobar**

- **H1.** La búsqueda semántica sin filtros mezcla ámbitos por solapamiento léxico. Ya lo observé en una prueba preliminar: la consulta "estoy desempleado en Córdoba" devolvía ayudas municipales de emprendimiento por coincidir en la palabra "Córdoba" y dejaba fuera la prestación estatal por desempleo. Quiero medir cuánto ocurre.
- **H2.** Extraer filtros estructurados (ámbito, tipo de ayuda) de la pregunta antes de buscar mejora la precisión sin perder cobertura.
- **H3.** Ordenar por importe sin tener en cuenta `tipo_importe` produciría recomendaciones engañosas.
- **H4.** Los huecos ("No especificado") se concentran en campos opcionales y no afectan a los campos que se usan para filtrar.

**Qué llega al MVP.** Un indicador del número de ayudas indexadas y su estado, la insignia de estado (Abierta, Cerrada, Permanente) en cada recomendación y, para la memoria, las figuras del análisis exploratorio (barras por ámbito y tipo, calendario de cierres, nulos por columna).

## 3. Tipo de modelos que se van a plantear

**Tipo de tarea:** recuperación de información y recomendación basada en contenido, con una capa de NLP (extracción de filtros y generación de la respuesta con un modelo de lenguaje) y una regla determinista para la vigencia.

**Por qué no un modelo supervisado.**

- No existe una variable objetivo histórica: no tengo datos de qué ayudas solicitó o consiguió cada persona.
- No hay interacciones usuario-ayuda, así que un filtrado colaborativo no tiene de dónde aprender.
- Con 130 registros no tiene sentido entrenar nada. El valor está en recuperar bien y en redactar sin inventar.

Lo que sí comparo son **tres estrategias de recuperación**, y evalúo por separado la fidelidad de la respuesta generada.

| Alternativa | Tipo | Por qué se plantea | Limitación principal |
|---|---|---|---|
| **Baseline** | Búsqueda léxica (TF-IDF o BM25) sobre título, finalidad, beneficiarios y palabras clave, sin filtros | Es la referencia mínima y honesta: lo que haría un buscador de texto. Si el sistema no la supera, la complejidad no se justifica | No entiende sinónimos ni paráfrasis, y sufre el solapamiento léxico (la palabra "Córdoba" pesa demasiado) |
| **Candidato 1** | Búsqueda semántica pura: embeddings `multilingual-e5-small` + FAISS, sin filtros | Captura el significado y cubre bien preguntas formuladas de forma coloquial | Puede recuperar ayudas del ámbito equivocado o ya cerradas, porque no distingue datos objetivos |
| **Candidato 2** | Búsqueda híbrida: filtros estructurados extraídos de la pregunta + semántica dentro del subconjunto filtrado + diversificación por tipo de ayuda | Combina exactitud (datos objetivos) y significado. Es el diseño con el que estoy construyendo el agente | Depende de un modelo de lenguaje para extraer los filtros: puede fallar, es más lento y consume cuota |

**Decisiones técnicas ya tomadas**

- **Embeddings locales.** Uso `multilingual-e5-small` en lugar de los embeddings de la API de Gemini porque la cuota gratuita se agota con facilidad y no quería que la demo fallara el día de la defensa. Tiene un beneficio añadido: es reproducible y no cuesta nada.
- **Modelo de lenguaje para generar.** `gemini-3.5-flash-lite`. No compito entre modelos de lenguaje: la generación es el último paso, controlado por un prompt y evaluado por fidelidad.
- **Vigencia por regla, no por modelo.** El estado de una ayuda lo calcula el código comparando fechas; el modelo de lenguaje nunca decide si una ayuda está abierta.

**Alternativas que dejo fuera del alcance** (y que aplicaría si el candidato 2 no rinde): un *reranker* de tipo cross-encoder y la fusión de rankings léxico y semántico (Reciprocal Rank Fusion).

## 4. Datos de entrada del análisis y los modelos

**Dataset gold:** `gold/docs/*.md` (documentos con frontmatter) y `gold/index/` (índice FAISS y `metadatos.json`), definidos en la Entrega 3.

- **Granularidad:** una fila (un documento) por ayuda, convocatoria o prestación.
- **Identificador:** `id`.
- **Fecha de referencia:** la fecha de ejecución. Para evaluar de forma reproducible la fijo a un valor concreto, porque `estado_calculado` cambia con el calendario.

| Entrada | Descripción | Granularidad / tipo | Uso en el análisis o modelo |
|---|---|---|---|
| Documento Markdown (título + cuerpo) | Texto en lenguaje natural de cada ayuda | Texto, un documento por ayuda | Base de los embeddings y del baseline léxico |
| `ambito_geografico`, `tipo_ayuda`, `sector`, `provincias` | Datos objetivos de clasificación | Categóricas | Filtros exactos previos a la búsqueda semántica |
| `edad_min`, `edad_max` | Límites de edad | Numérica (con "No especificado") | Filtro opcional y contexto para la respuesta |
| `estado_calculado` | Vigencia recalculada con la fecha de referencia | Categórica derivada | Regla que impide presentar una ayuda cerrada como disponible |
| `importe_min`, `importe_max`, `tipo_importe` | Cuantía y su significado | Numérica y categórica | Solo para presentar; nunca para ordenar sin considerar el tipo |
| `url_oficial`, `id` | Trazabilidad | Texto | Cita de la fuente en cada recomendación |
| Pregunta e historial de la conversación | Lo que el usuario cuenta de sí mismo | Texto | Extracción de filtros y perfil, con memoria entre turnos |

**Variables que no se usan y por qué**

- `estado_convocatoria_original`: envejece, por eso se sustituye por `estado_calculado`.
- `fecha_verificacion`: solo sirve para trazabilidad y explicación, no para buscar.
- `bdns`, `codigo_procedimiento`: muy escasos, no aportan al filtrado.
- Datos personales del usuario: no se guardan en el servidor. El historial vive en el navegador y la memoria de la conversación, en el proceso del backend.

**Qué información existe cuando se genera la respuesta.** Todo el gold, la pregunta y el historial de la conversación. No se conoce si la persona cumple realmente los requisitos: eso solo lo puede decidir el organismo, y el sistema lo indica.

## 5. Datos de salida y forma de consumo

La salida es una **recomendación en lenguaje natural** apoyada en una lista corta de ayudas recuperadas (5 como máximo). En el MVP la API devuelve la respuesta redactada; en la fase de diseño devolvería, además, la lista estructurada de ayudas para pintarlas como tarjetas.

| Campo de salida | Descripción | Tipo | Uso posterior |
|---|---|---|---|
| `respuesta` | Texto redactado (Markdown) con las ayudas recomendadas, su estado, importe, plazo y enlace | string | Se muestra en el chat |
| `thread_id` | Identificador de la conversación | string | Mantiene la memoria entre turnos |
| `id` (por ayuda) | Ayuda recuperada | string | Trazabilidad y evaluación |
| `score` (por ayuda) | Similitud coseno con la pregunta | float | Ordenación interna y análisis; no se muestra como "probabilidad" |
| `estado_calculado` (por ayuda) | Vigencia en la fecha de ejecución | categoría | Insignia de estado y advertencia si está cerrada |
| `importe_max` + `tipo_importe` | Cuantía con su significado | numérico + categoría | Presentar "por beneficiario" o "presupuesto total" |
| `url_oficial` | Fuente oficial | URL | Enlace clicable en la respuesta |
| `filtros_aplicados` | Filtros que el agente ha extraído de la pregunta | objeto | Explicación ("por qué te lo recomiendo") y análisis de errores |
| `fecha_ejecucion` | Momento en que se genera el resultado | datetime | Reproducibilidad y control de vigencia |

**Granularidad:** por ayuda (dentro de una respuesta por consulta). **Formato:** endpoint `POST /chat` de una API FastAPI, consumido por la interfaz web de React.

**Qué decisión permite tomar.** Ir a la fuente oficial de la ayuda adecuada, o descartar las que no aplican. **Qué contexto necesita ver:** el estado, el tipo de importe y un aviso de que la información debe confirmarse en la fuente oficial. Cuando no hay ayuda relevante, el sistema lo dice de forma explícita.

## 6. Estrategia para diseñar y seleccionar el modelo

1. **Preparación del corpus.** Parto de la capa gold ya validada. Los documentos se embeben con el prefijo `passage:` y las consultas con `query:`, como pide el modelo, y los vectores se normalizan para que el producto interno equivalga al coseno. Los nulos no se imputan: el texto lleva "No especificado".
2. **Definición de la "salida correcta".** Como no hay una variable objetivo, la construyo yo: para cada consulta de evaluación, la lista de ayudas relevantes que un lector humano recomendaría (véase la sección 7).
3. **Baseline.** TF-IDF o BM25 sobre título, finalidad, beneficiarios y palabras clave, sin filtros.
4. **Candidatos.** La búsqueda semántica pura y la híbrida, y solo esas dos. Compruebo con ellas si cada capa extra (semántica, después filtros) aporta algo medible.
5. **Comparación con los mismos datos y las mismas consultas.** Los tres sistemas se ejecutan sobre el mismo conjunto de evaluación, con la misma fecha de referencia.
6. **Criterios de comparación:**
   - **calidad de recuperación** (Recall@5, MRR y Precision@5);
   - **seguridad**: cero ayudas cerradas presentadas como disponibles;
   - **estabilidad**: que reformular la misma pregunta no cambie las ayudas recomendadas;
   - **interpretabilidad**: se pueden mostrar los filtros aplicados;
   - **coste**: latencia y llamadas a la API (el candidato 2 hace en torno a 2 por consulta: extracción de filtros y generación, con una cuota gratuita limitada);
   - **utilidad para el MVP** y complejidad de mantenimiento.
7. **Regla de decisión final.** Elijo el candidato 2 solo si:
   - supera al baseline y al candidato 1 en Recall@5 con una mejora clara;
   - no presenta ninguna ayuda cerrada como disponible;
   - responde con "no hay ayuda" en las consultas sin resultado relevante;
   - tiene una latencia razonable para el uso real.

   Si el candidato 1 empata en calidad, me quedo con el más simple, porque es más rápido, barato y fácil de explicar. Un sistema un poco menos preciso pero más estable y explicable puede ser mejor que uno con una métrica ligeramente más alta.

## 7. Estrategia de validación y evaluación

Aquí no hay conjunto de entrenamiento, así que la validación consiste en **medir la recuperación sobre un conjunto de consultas etiquetadas de antemano**, lo más parecidas posible a cómo escribiría un ciudadano.

**Conjunto de evaluación (unas 40 consultas)**

- **10 de desarrollo**, para ajustar el prompt de extracción de filtros, el número de resultados y los umbrales.
- **30 de prueba**, congeladas: se ejecutan una sola vez al final para comparar los sistemas. Si ajusto algo después de verlas, dejan de ser de prueba.
- Se reparten por perfil (autónomo o pyme, desempleo y prestaciones, vivienda y jóvenes, mayores y pensionistas), por ámbito y por tipo de ayuda.
- Incluyen **consultas sin ayuda relevante** (por ejemplo, una ayuda para comprar un yate) y **conversaciones de varios turnos** para comprobar la memoria.

**Etiquetado.** Antes de ejecutar ningún sistema, marco a mano para cada consulta qué ayudas son relevantes según las fichas y con un criterio escrito. Así evito que el resultado influya en la etiqueta.

**Cómo evito la contaminación**

- Las consultas se redactan como un ciudadano, sin copiar títulos ni frases de las fichas.
- Las de desarrollo y las de prueba no comparten consultas ni reformulaciones.
- La fecha de referencia se fija para que `estado_calculado` sea reproducible.
- No uso `estado_convocatoria_original`, que ya no refleja la realidad.

| Elemento | Decisión prevista | Justificación |
|---|---|---|
| Separación de datos | 10 consultas de desarrollo y 30 de prueba congeladas, estratificadas por perfil y ámbito | No hay entrenamiento, pero sí ajuste de prompts y parámetros que no debe contaminar la prueba |
| Métrica principal | Recall@5 | Lo que importa es que la ayuda adecuada aparezca entre las que se muestran al usuario |
| Métricas secundarias | Precision@5, MRR, tasa de vigencia correcta, tasa de rechazo correcto en consultas sin ayuda relevante y fidelidad de los datos citados | Miden la calidad del ranking, la seguridad y que el modelo no invente importes, plazos ni enlaces |
| Baseline | Búsqueda léxica sin filtros | Permite medir la mejora real de cada capa |
| Criterio de aceptación | Recall@5 de al menos 0,80 en el candidato elegido, con una mejora de al menos 10 puntos sobre el baseline; cero ayudas cerradas presentadas como disponibles; rechazo correcto en al menos el 90 % de las consultas sin ayuda relevante; datos citados que coinciden con la ficha en al menos el 95 % de los casos revisados | Umbrales de partida que puedo ajustar si los datos lo justifican, dejando constancia |

**Cómo compararé con el baseline.** Con una tabla por sistema y, además, por segmento (ámbito, tipo de ayuda y perfil), y contando en cuántas consultas mejora o empeora cada sistema respecto al baseline. Con solo 30 consultas de prueba las diferencias serán **indicativas y no estadísticamente concluyentes**, y lo diré así en la memoria.

**Análisis de errores.** Reviso uno a uno los fallos del sistema elegido y los clasifico: filtro mal extraído, solapamiento léxico, ayuda multifase, dato "No especificado" o generación que se aparta de la ficha. La fidelidad de la respuesta la compruebo a mano sobre una muestra: importe, plazo, estado y URL deben coincidir con la ficha.

**Si ningún sistema alcanza el mínimo.** Ajustaría los filtros y el prompt, probaría el *reranker* o la fusión de rankings, y revisaría si el problema está en las fichas (datos vacíos o ambiguos) antes que en el modelo. Si aun así no se alcanza, presento los resultados tal cual y detallo las causas en las limitaciones.

## 8. Riesgos y alternativas

- **¿La variable objetivo está disponible y representa el fenómeno?** No hay variable objetivo: la "relevancia" la etiqueto yo. Es un riesgo de subjetividad, que reduzco escribiendo el criterio antes y etiquetando antes de ejecutar los sistemas. Lo asumo como limitación.
- **¿Hay riesgo de fuga de datos?** No en sentido clásico, porque no se entrena. Los riesgos equivalentes son que las consultas de evaluación se parezcan demasiado a los títulos de las fichas, que se ajuste algo mirando el conjunto de prueba y que se use el estado original en vez del calculado. Lo controlo con las reglas de la sección 7.
- **¿Son suficientes el volumen, el histórico y la calidad?** Para recuperar y recomendar, sí: 130 fichas verificadas. Para sacar conclusiones estadísticas, no: el conjunto de evaluación es pequeño. Tampoco hay histórico, así que no se estudian tendencias.
- **¿Hay desbalance, cambios temporales o sesgos de cobertura?** Sí. 110 de las 130 ayudas son subvenciones y solo 15 son prestaciones, y predomina el ámbito andaluz (69). La cobertura municipal es limitada (13). Además las ayudas caducan: el catálogo envejece y necesita revisarse periódicamente. El sistema lo mitiga con `estado_calculado` y declarando que no es exhaustivo.
- **¿Qué genera más incertidumbre?** La dependencia del modelo de lenguaje: la extracción de filtros y la generación no son deterministas, se ven afectadas por la cuota de la API y pueden variar entre ejecuciones. También los datos enviados a la API externa: las preguntas del usuario se procesan por un tercero, y se informa de ello.
- **¿Qué haría si el modelo no supera al baseline o no puedo validarlo con rigor?** Me quedaría con la solución más simple que funcione: búsqueda semántica con filtros por reglas (por ejemplo, palabras clave de ámbito y tipo en lugar de un modelo de lenguaje). Si la validación cuantitativa no fuera posible por falta de tiempo, la sustituiría por una revisión cualitativa de los casos representativos ya probados (una subvención con plazo, una prestación permanente, una consulta sin ayuda relevante y una conversación de varios turnos), dejando claro que no es una evaluación estadística.
