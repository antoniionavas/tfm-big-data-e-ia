# Entrega 2 - Selección de Idea y Datos Necesarios

## 1. Idea seleccionada

Antes de detallar la propuesta elegida definitiva, conviene contextualizar la evolución del proyecto. Las tres ideas iniciales planteadas en la primera fase del curso (centradas en predicción clínica, monitorización hospitalaria y analítica deportiva) cumplían un propósito exploratorio, pero tras un análisis más profundo de viabilidad y accesibilidad de fuentes, se barajó inicialmente un enfoque orientado a desarrollar soluciones analíticas y de IA a medida para el entorno empresarial real, para lo cual se solicitó la colaboración y el acceso a datos corporativos a diversas entidades. 

Sin embargo, ante la imposibilidad de obtener la aprobación y los permisos necesarios sobre dichos datos confidenciales en los plazos requeridos para el desarrollo académico, se replanteó el rumbo estratégico del proyecto. Buscando un equilibrio óptimo entre viabilidad técnica, alto impacto social y la aplicación rigurosa de tecnologías avanzadas, se descartaron las líneas anteriores y se optó de manera firme por el desarrollo de un **asistente inteligente basado en una arquitectura RAG (Retrieval-Augmented Generation) y agentes de Inteligencia Artificial, orientado a la búsqueda, filtrado y recomendación automatizada de subvenciones, ayudas y convocatorias públicas en la Comunidad Autónoma de Andalucía**.

A continuación, se desarrolla en detalle la propuesta seleccionada:

#### Párrafo 1: Problema que resuelve

En Andalucía existe una alta dificultad y desconocimiento a la hora de localizar, comprender y solicitar ayudas y subvenciones públicas. Las bases reguladoras son documentos extensos, densos y repletos de jerga administrativa, lo que provoca que muchos potenciales beneficiarios pierdan oportunidades de financiación por desconocimiento o por la dificultad de interpretar los requisitos de elegibilidad, plazos y obligaciones. Este proyecto aporta un valor crítico al democratizar y simplificar el acceso a la información pública, reduciendo el tiempo de búsqueda y minimizando el riesgo de errores en la solicitud.

#### Párrafo 2: Solución planteada

La solución planteada consiste en desarrollar una app / plataforma inteligente capaz de recopilar, procesar y estructurar información procedente de fuentes oficiales sobre subvenciones, ayudas y convocatorias públicas de Andalucía, para posteriormente ofrecer al usuario un sistema de consulta y recomendación mediante lenguaje natural. A través de una arquitectura RAG y el uso de agentes de Inteligencia Artificial, el sistema permitirá expresar las necesidades y características del usuario, localizar las ayudas potencialmente relevantes, filtrarlas según diferentes criterios y generar respuestas contextualizadas basadas en la información oficial disponible. De esta forma, se pretende simplificar y automatizar el proceso de su búsqueda de ayudas públicas, reduciendo el tiempo necesario para localizar oportunidades de financiación y facilitando al usuario información relevante, actualizada y trazable mediante el acceso a las fuentes oficiales.

#### Párrafo 3: MVP del proyecto final 
El Producto Mínimo Viable (MVP) consistirá en una aplicación web interactiva con interfaz de chat (desarrollada en Streamlit o una interfaz web moderna en React) conectada a un pipeline backend de RAG. El usuario podrá interactuar en lenguaje natural planteando su perfil y sus necesidades (ej. "Soy un autónomo en Córdoba del sector tecnológico y busco ayudas para digitalización"). El sistema devolverá un listado de convocatorias vigentes que encajen con su perfil, los plazos límite exactos, un resumen claro de los requisitos imprescindibles y enlaces directos a los textos oficiales, permitiendo además realizar preguntas de seguimiento sobre los detalles específicos de cada base reguladora.

## 2. Datos necesarios
Para que el proyecto tenga rigor técnico y cumpla con los objetivos de la maestría, los datos requeridos se estructuran de la siguiente manera:

**Variables o campos necesarios**: El corpus de datos se compone de un esquema tabular normalizado que incluye las siguientes variables clave para alimentar el sistema inteligente:

 - Identificación y metadatos institucionales: id, titulo, organismo, tipo_ayuda, codigo_procedimiento, bdns, url_oficial, fuente, fecha_publicacion.

 - Segmentación y público objetivo: sector, subsector, beneficiarios, perfil_beneficiario, edad_min, edad_max.

 - Ámbito territorial: ambito_geografico, provincias.

 - Condiciones económicas y cuantías: importe_min, importe_max, porcentaje_subvencion.

 - Cronograma y estado: fecha_inicio, fecha_fin, estado_convocatoria, fecha_verificacion, condicion_cierre.

 - Contenido normativo y operativo (clave para la recuperación semántica y RAG): finalidad, actuaciones_subvencionables, requisitos, gastos_subvencionables, documentacion_necesaria, palabras_clave.

**Nivel de granularidad**: Granularidad a nivel de convocatoria oficial individual, donde cada fila del dataset representa una línea de ayuda o subvención perfectamente acotada con sus respectivos metadatos estructurados y campos descriptivos en texto libre.

**Profundidad histórica**: Datos actuales correspondientes a las convocatorias vigentes y abiertas del ejercicio en curso, complementados con un histórico reciente que permite evaluar el comportamiento del sistema ante diferentes estados de tramitación (estado_convocatoria) y fechas de cierre.

**Volumen aproximado de datos**: Un volumen inicial estructurado en formato tabular de entre 100 y 300 registros/convocatorias, dimensión óptima para realizar pruebas robustas de filtrado estructurado y validación de las respuestas generadas por los agentes de IA.

**Imprescindibles vs. Deseables**:

- **Imprescindibles**: Campos descriptivos y normativos esenciales para la comprensión del usuario y la vectorización (titulo, organismo, finalidad, requisitos, actuaciones_subvencionables, beneficiarios) junto con las fechas de validez (fecha_inicio, fecha_fin).

- **Deseables**: Variables de acotación económica y geográfica precisa (importe_min, importe_max, porcentaje_subvencion, provincias) que permiten aplicar filtros estructurados previos a la búsqueda semántica mediante el asistente.

## 3. Fuentes de datos previstas 

**Fuente o fuentes concretas previstas**:
 - BOJA (Boletín Oficial de la Junta de Andalucía) y portales institucionales de la administración autonómica para la recopilación y extracción de las convocatorias.

 - Base de Datos Nacional de Subvenciones (BDNS) y catálogos de procedimientos oficiales como fuentes complementarias de validación y contraste normativo.

**Carácter de las fuentes**: Son fuentes completamente abiertas, públicas e institucionales, accesibles sin restricciones comerciales ni barreras de pago relevantes.

**Enlace a las fuentes**:
- Boletín Oficial de la Junta de Andalucía (BOJA)
- Base de Datos Nacional de Subvenciones (BDNS)

**Formato esperado de los datos**: Los datos primarios se obtienen mediante el procesamiento y estructuración de documentos normativos en formato PDF y páginas HTML institucionales, los cuales son posteriormente transformados, depurados y consolidados en un dataset estructurado en formato CSV (con los campos normalizados como id, titulo, organismo, requisitos, fecha_inicio, fecha_fin, url_oficial, etc.). Seguramente termine pasando todos los CSV a Markdown debido a  facilitar la lectura del agente de IA al analizar los datos pedidos por el usuario. 

**Histórico disponible**: Las fuentes oficiales disponen de un archivo histórico amplio, estructurado y consultable desde hace décadas, lo que permite asegurar la trazabilidad temporal de las convocatorias.

**Estabilidad de la fuente**: Alta estabilidad. Al tratarse de canales institucionales oficiales de la administración pública, su mantenimiento y disponibilidad están garantizados por ley.

**Riesgos detectados**:
 - Presencia de datos incompletos o descripciones ambiguas en los extractos iniciales de algunas convocatorias, requiriendo validación cruzada con las bases reguladoras completas.

 - Variabilidad en las estructuras y diseño de las páginas o documentos originales, lo que puede dificultar los procesos automatizados de extracción (parsing).

 - Posibles cambios en la nomenclatura de los campos institucionales o en las URLs de redirección oficial (url_oficial).

## 4. Consideraciones de privacidad y protección de datos  
  - **Información personal identificable (PII)**: El proyecto procesa exclusivamente información de carácter público, normativo y administrativo (bases reguladoras, requisitos de elegibilidad por sector o tamaño de empresa, importes y plazos). No se recopilan, almacenan ni tratan datos de carácter personal identificable de ciudadanos o solicitantes reales.

 - **Necesidad de anonimización**: Al tratarse estrictamente de datos públicos institucionales y normativos, no es necesario aplicar procesos de anonimización, enmascaramiento o agregación de información.

 - **Seguridad en el ámbito académico**: Al no contener información confidencial ni datos de usuarios finales, el dataset estructurado en CSV y los repositorios asociados pueden utilizarse de forma totalmente segura en un entorno académico y de desarrollo.

 - **Riesgos éticos o legales**: No se identifican riesgos éticos ni legales. La reutilización y estructuración de información pública procedente de boletines oficiales para facilitar su acceso y comprensión ciudadana está plenamente amparada por las normativas de transparencia y reutilización de datos abiertos.

 - **Decisiones sobre privacidad**: Se ha decidido excluir de forma deliberada cualquier registro o interacción que pudiera incorporar datos personales de solicitantes reales, limitando el ámbito del sistema al asesoramiento técnico sobre las bases de las ayudas.


## 5. Viabilidad inicial del proyecto

**¿Parece viable obtener los datos necesarios?**: 
Sí, completamente. Al basarse en convocatorias públicas oficiales y contar ya con una estructura normalizada en formato tabular (CSV), el flujo de datos está asegurado.

**¿La información disponible tiene suficiente calidad, granularidad y profundidad histórica?**: 
Sí. El nivel de detalle aportado por los campos del dataset (que desglosan desde la finalidad y los requisitos hasta las actuaciones subvencionables y los gastos elegibles) ofrece una granularidad excelente para alimentar tanto modelos de filtrado estructurado como motores de recuperación semántica (RAG).

**¿La idea puede desarrollarse de forma realista durante el curso?**: 
Sí, el volumen de datos manejado y la complejidad técnica se ajustan de manera realista a los plazos y objetivos de la maestría.

**¿Qué parte del proyecto veis más arriesgada en este momento?**: 
El principal reto técnico se encuentra en conseguir total las ayudas existentes para todos los ciudadanos teniendo en cuenta posibilidades de situaciones, empresas, trabajos y diversos problemas. Además tambien es dificil asegurar la consistencia y limpieza del texto libre en campos complejos (como requisitos, actuaciones_subvencionables y gastos_subvencionables) para que el sistema RAG recupere con total precisión los fragmentos clave sin generar alucinaciones en las respuestas del agente conversacional.

**¿Qué alternativa tendríais si la fuente principal de datos no funciona?**: 
No tengo alternativas pero tampoco dudas de que consiga la fuente principal de datos, la única diferencia es que quizás no consigo obtener un dataset con muchas ayudas o subvenciones que sean reales y existan en documentos oficiales. 