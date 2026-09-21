# Entrega 3 - Diseño del modelo de datos y capa gold

## 1. Resumen de la idea y datos del proyecto

**Qué problema resuelve.** Hay mucha gente que tiene derecho a una ayuda pública (una subvención, una prestación, una beca) y no llega a pedirla porque no sabe que existe. La información está repartida entre Ayuntamiento, Diputación, Junta de Andalucía y Estado, cada organismo la publica en un sitio distinto, con un lenguaje administrativo poco amigable y con plazos que cambian de un año a otro.

**Qué solución quiero construir.** Un asistente conversacional que reciba la situación de la persona en lenguaje natural ("soy autónomo, tengo 56 años y vivo en Córdoba") y le recomiende las ayudas existentes que mejor encajan, con el importe, el plazo y el enlace a la fuente oficial. Bajo el capó es un sistema RAG con un agente de IA: primero filtra con datos objetivos (ámbito, tipo de ayuda, estado) y después busca por significado.

**Fuentes de datos.** El dataset lo he construido a mano, ayuda por ayuda, contrastando cada registro con su página oficial. No hay un único origen automatizable, así que cada fila guarda su propia `url_oficial` y la fecha en que la verifiqué.

| Fuente | Qué aporta |
|---|---|
| BOJA y sedes electrónicas de la Junta de Andalucía | Convocatorias autonómicas: bases, plazos, importes |
| BOP de Córdoba, Ayuntamiento de Córdoba (IMDEEC), Diputación | Ayudas provinciales y municipales |
| Cámara de Comercio de Córdoba | Programas para pymes cofinanciados (por ejemplo FEDER) |
| Organismos estatales (por ejemplo SEPE y Seguridad Social) | Prestaciones de solicitud continua: paro, ingreso mínimo vital, pensiones |
| BDNS (Base de Datos Nacional de Subvenciones) | Solo como referencia manual: su `robots.txt` impide la extracción automática |

El punto de partida fue una primera recopilación de 80 registros que se auditó de forma independiente (con un informe de calidad y una tabla de verificación fila a fila). Después la amplié hasta **130 registros** cubriendo los tres niveles administrativos: Estado, Andalucía y Córdoba.

## 2. Tecnología o formato de almacenamiento elegido

Con 130 registros, el criterio ha sido no complicarme la vida: elegir lo más simple que dé un resultado fiable, reproducible y que se pueda explicar.

| Formato | Uso en el proyecto | Por qué |
|---|---|---|
| **CSV** (UTF-8, separador coma) | Tabla maestra de ayudas: 130 filas × 32 columnas | Cabe en memoria, se edita y revisa fácilmente, se versiona en Git y los cambios se ven en un diff |
| **Markdown con frontmatter YAML** | Un documento por ayuda (`data/gold/docs/`) | El frontmatter guarda los campos objetivos para filtrar; el cuerpo, en lenguaje natural, es lo que se convierte en embedding. Así separo el dato exacto del texto |
| **Índice FAISS + JSON** | Vectores de los 130 documentos y sus metadatos | FAISS es una librería, no un servidor: no añade infraestructura y me obliga a implementar yo mismo el filtrado por metadatos, con lo que controlo cada paso |

**Lo que he descartado, y por qué:**

- **Base de datos relacional (SQLite/PostgreSQL):** no hay escrituras concurrentes, ni relaciones complejas, ni volumen que lo justifique. Sería infraestructura extra sin beneficio.
- **Parquet:** con 130 filas no aporta rendimiento y perdería legibilidad.
- **Excel:** no se versiona bien y facilita errores silenciosos de formato en fechas e importes.
- **ChromaDB:** la conocía de la asignatura de IA Generativa y lo comparé con FAISS. Con este tamaño de dataset ChromaDB ofrece persistencia y gestión de colecciones que no necesito, y FAISS me permite entender y documentar todo el proceso de búsqueda. Me quedo con FAISS como decisión razonada, no por inercia.

Los **embeddings** los calculo en local con `multilingual-e5-small` (vectores de 384 dimensiones) en lugar de usar una API. Lo decidí porque la cuota gratuita de la API de Gemini es limitada y no quería que la demo dependiera de ella.

## 3. Estructura de capas de datos

```
data/
├── raw/
│   ├── subvenciones_maestro_ampliado.csv   # recopilación inicial, tal cual llegó
│   ├── tabla_verificacion.csv              # verificación fila a fila contra la fuente
│   └── informe_calidad.md                  # informe de calidad de la recopilación inicial
├── processed/
│   └── subvenciones.csv                    # CSV consolidado y normalizado (130 registros)
└── gold/
    ├── docs/                               # 130 documentos .md (frontmatter + cuerpo)
    └── index/                              # índice FAISS + metadatos.json
```

| Capa | Contenido en este proyecto | Transformación que se aplica |
|---|---|---|
| **Raw** | Ficheros originales de la recopilación y de la auditoría | Ninguna. Se conservan para poder demostrar de dónde sale cada dato |
| **Processed** | CSV consolidado de 130 registros | Corrección de estados caducados, eliminación de un duplicado, normalización de `tipo_ayuda` y `ambito_geografico` a valores cerrados, y nueva columna `tipo_importe` |
| **Gold** | Documentos Markdown validados e índice vectorial | Cálculo de `estado_calculado`, generación del frontmatter y del cuerpo, validación, embeddings e indexación |

En el repositorio de trabajo estos ficheros ya existen (`data/subvenciones.csv`, `data/docs/`, `data/index/`); los ordeno en tres carpetas para que la separación entre capas sea explícita y trazable. Todo lo que hay en `gold/` se regenera a partir de `processed/` con el notebook, así que puedo borrarlo y reconstruirlo sin perder nada.

## 4. Definición de la capa gold

La capa gold está formada por dos datasets que se consumen juntos: los **documentos de ayudas** y el **índice vectorial** construido sobre ellos.

| Dataset gold | Granularidad | Campos clave | Uso posterior |
|---|---|---|---|
| `gold/docs/*.md` | Un archivo por ayuda (convocatoria, línea o prestación) | `id`, `titulo`, `tipo_ayuda`, `ambito_geografico`, `estado_calculado`, `importe_max`, `tipo_importe`, `url_oficial` | EDA, evaluación del sistema y fuente del agente RAG |
| `gold/index/` (índice FAISS + `metadatos.json`) | Un vector por documento | Vector de 384 dimensiones y el frontmatter completo de cada documento | Búsqueda semántica con filtros dentro del agente y de la API |

**Detalle de `gold/docs/*.md`**

- **Registros esperados:** 130 (previsiblemente entre 130 y 200 si amplío cobertura).
- **Identificador principal:** `id` (texto, único; por ejemplo `AND-2026-087` o `ESP-2026-007`). El nombre del archivo es el propio `id` en minúsculas (por ejemplo `esp-2026-007.md`).
- **Estructura de cada archivo:** un bloque de frontmatter YAML con 21 campos estructurados y, debajo, un cuerpo en lenguaje natural con la finalidad, los beneficiarios, los requisitos, el importe, el plazo y la fuente.
- **Campos relevantes para el análisis:** `estado_calculado` (lo que decide si una ayuda se puede recomendar como disponible), `tipo_importe` (evita comparar un presupuesto total con un máximo por persona) y `url_oficial` (trazabilidad).
- **Distribución actual:** por ámbito, 69 de Andalucía, 30 provinciales, 18 estatales y 13 municipales; por tipo, 110 subvenciones, 15 prestaciones, 3 premios y 2 becas.

**Detalle de `gold/index/`**

- **Índice:** `IndexFlatIP` de FAISS con 130 vectores normalizados (el producto interno equivale a similitud coseno).
- **Correspondencia:** la posición del vector en el índice coincide con la posición del documento en `metadatos.json`, que a su vez lleva el `id`. Esa es la clave de unión.
- **Consumidor:** la función `buscar()` filtra primero por metadatos, reconstruye los vectores de los candidatos y solo entonces aplica la similitud, de modo que un filtro exacto nunca se pierde por "no parecerse lo bastante" a la pregunta.

## 5. Relaciones entre datos

Solo hay **una entidad**: la ayuda. La misma ayuda existe en tres formas, unidas por su `id`:

```
processed/subvenciones.csv (id)  1 --- 1  gold/docs/<id>.md  1 --- 1  vector en gold/index (posición ↔ id)
```

No necesito un modelo relacional más complejo porque no hay otras entidades con vida propia. Los campos multivalor (`provincias`, `palabras_clave`) podrían ser una relación N:M en una base de datos normalizada, pero con este volumen los dejo desnormalizados: `palabras_clave` se guarda como lista en el YAML y `provincias` como texto.

Los otros datos del sistema no forman parte del dataset gold:

- **Conversaciones:** se identifican por un `thread_id` y viven aparte, en el navegador del usuario y en la memoria del proceso del backend. No se cruzan con las ayudas.
- **Código BDNS (`bdns`):** es una referencia externa opcional; no se hace ningún join con la base nacional.

El principal problema al combinar fuentes es que cada organismo describe los importes y los plazos de forma distinta. Lo resuelvo dentro de una única tabla armonizada (con `tipo_importe` y un `estado_calculado` propio), no cruzando tablas.

## 6. Diccionario de datos inicial

Estos son los campos que llegan al frontmatter (21 en total). Los campos de texto largo (`finalidad`, `beneficiarios`, `perfil_beneficiario`, `requisitos`, `actuaciones_subvencionables`, `gastos_subvencionables`, `documentacion_necesaria`, `condicion_cierre`, `fuente`) pasan al cuerpo del documento.

| Campo | Descripción | Tipo de dato | Fuente | Obligatorio | Observaciones |
|---|---|---|---|---|---|
| `id` | Identificador único de la ayuda | string | Propia | Sí | Formato `PREFIJO-2026-NNN` |
| `titulo` | Nombre oficial de la convocatoria o prestación | string | Fuente oficial | Sí | |
| `organismo` | Entidad que convoca | string | Fuente oficial | Sí | |
| `tipo_ayuda` | Naturaleza de la ayuda | categoría cerrada | Propia (normalizada) | Sí | Subvención, Prestación, Premio, Beca |
| `sector` / `subsector` | Colectivo o ámbito al que se dirige (p. ej. Empresas) | string | Propia | Sí / No | |
| `ambito_geografico` | Nivel administrativo | categoría cerrada | Propia (normalizada) | Sí | Andalucía, Provincial, Estatal, Municipal |
| `provincias` | Territorio donde aplica | string | Fuente oficial | No | |
| `edad_min` / `edad_max` | Límites de edad de los beneficiarios | entero | Fuente oficial | No | "No especificado" cuando no hay límite conocido |
| `importe_min` / `importe_max` | Cuantías mínima y máxima | numérico (€) | Fuente oficial | No | Su significado depende de `tipo_importe` |
| `tipo_importe` | Qué representa `importe_max` | categoría cerrada | Derivado de la verificación | Sí | `presupuesto_total`, `por_beneficiario`, `no_aplica`, `revisar`, `no_determinado` |
| `fecha_inicio` / `fecha_fin` | Rango del plazo de solicitud | date | Fuente oficial | No | ISO `YYYY-MM-DD`. En convocatorias multifase es el rango que envuelve todas las fases |
| `estado_convocatoria_original` | Estado en el momento de verificar | categoría | Propia | Sí | Se conserva solo por trazabilidad; envejece |
| `estado_calculado` | Estado vigente recalculado con la fecha de ejecución | categoría derivada | Calculado | Sí | Abierta, Cerrada, Prevista, Permanente. Sin fecha de fin exacta se marca "revalidar" |
| `fecha_verificacion` | Día en que comprobé el registro contra su fuente | date | Propia | Sí | |
| `bdns` | Código BDNS de la convocatoria | string | BDNS | No | Muchos vacíos |
| `url_oficial` | Página oficial de la ayuda | URL | Fuente oficial | Sí | Base de la trazabilidad |
| `palabras_clave` | Términos que describen la ayuda | lista de strings | Propia | No | Se guarda como lista YAML |

## 7. Problemas de calidad esperados

Estos son los problemas reales que me he ido encontrando o que espero encontrar, no una lista genérica:

- **`importe_max` ambiguo.** Unas veces es el presupuesto total de la convocatoria y otras el máximo por beneficiario. Ordenar o comparar importes sin distinguirlo daría respuestas engañosas. Lo resuelvo con la columna `tipo_importe`, deducida siempre del texto de las observaciones y no de la magnitud del número.
- **Contradicciones entre texto y cifra.** Detecté 3 registros (`AND-2026-038`, `043` y `056`) donde el texto de las observaciones dice una cosa y el `importe_max` guardado dice otra (en uno, el texto cita 35.000 € y el campo guarda 15.000 €). Los dejé marcados como `revisar` en vez de decidir a ojo.
- **Datos desactualizados.** El estado de una convocatoria caduca. Por ejemplo, `AND-2026-006` seguía "Abierta" con plazo hasta el 11/09 cuando ya era 14/09. Por eso no me fío del estado guardado y lo recalculo al ejecutar.
- **Convocatorias con varias fases.** Un único par `fecha_inicio`/`fecha_fin` no puede describir una convocatoria como `AND-2026-087`, con líneas que cerraron en abril y otras con un segundo periodo en septiembre-octubre. El rango que las envuelve la marca "Abierta" incluso en una ventana intermedia sin fase activa. Es una limitación conocida que documento en lugar de esconder.
- **Sin fecha de fin exacta.** Algunas ayudas (por ejemplo varias de la Cámara de Comercio) no tienen un plazo confirmado en la fuente primaria. No lo invento: se marca para revalidar.
- **Muchos "No especificado".** Campos como `edad_min`, `edad_max`, `documentacion_necesaria`, `codigo_procedimiento` o `bdns` van muy vacíos. Prefiero un hueco honesto a un dato inventado.
- **Etiquetas heterogéneas.** `tipo_ayuda` y `ambito_geografico` llegaron con muchas variantes y matices (por ejemplo "concesión directa"). Las reduje a 4 valores cada una y el matiz se trasladó al campo `requisitos`.
- **Duplicados.** Apareció una fila repetida (`AND-2026-083`) que eliminé.
- **Sesgo de cobertura.** Predominan las ayudas de ámbito andaluz (69 de 130) y la cobertura municipal es pequeña (13 registros). El dataset no pretende ser exhaustivo.
- **Fuentes que no se pueden automatizar.** La BDNS bloquea la extracción por `robots.txt` y la API del BOJA devolvió errores, así que la verificación es manual y cuesta escalarla.
- **Falta de histórico.** Es una foto de un momento (septiembre de 2026). No hay series temporales, así que no se puede estudiar cómo evolucionan las convocatorias de un año a otro.
- **Tasa de error de la fuente original.** El informe de calidad de la recopilación inicial cuantificaba un 21,4 % de error en la muestra auditada. Es la razón por la que cada registro conserva su URL y su fecha de verificación.

## 8. Decisiones de limpieza y transformación previstas

- **Nulos:** no se imputan. Un campo desconocido se escribe como "No especificado", y una fecha de fin que falta lleva el estado "revalidar". Si un dato no se puede verificar, se marca para revisión manual; nunca se corrige automáticamente.
- **Duplicados:** se buscan por `id`, por título y por `url_oficial`, y se eliminan.
- **Normalización:**
  - fechas en formato ISO;
  - importes como numérico y formateados de forma legible al construir el texto;
  - `tipo_ayuda` y `ambito_geografico` en conjuntos cerrados;
  - `palabras_clave` como lista.
- **Variables derivadas:**
  - `estado_calculado`, comparando `fecha_inicio` y `fecha_fin` con la fecha de ejecución. Las prestaciones (paro, IMV, pensiones) se marcan siempre como "Permanente" porque no tienen convocatoria con plazo;
  - la línea de importe del cuerpo, que distingue explícitamente presupuesto total de máximo por beneficiario;
  - el cuerpo en lenguaje natural de cada documento.
- **Agregaciones:** ninguna en la capa gold. Las agregaciones (conteos por ámbito, calendario de cierres) las hago en el análisis exploratorio de la Entrega 4.
- **Datos que se descartan:** el estado original no se usa para recomendar (solo queda como referencia) y los campos casi vacíos (`codigo_procedimiento`, `bdns`) no se usan para filtrar.
- **Criterio de registro válido:** debe tener `id` único, `titulo`, `organismo`, `tipo_ayuda` y `ambito_geografico` dentro de los valores permitidos, `url_oficial` con `https` y, si hay fechas, `fecha_fin` no anterior a `fecha_inicio`. Un script de validación revisa todo esto, avisa de lo que no cumple y no corrige nada por su cuenta.

## 9. Riesgos del modelo de datos

- **Lo más claro:** la estructura y el pipeline. Un CSV consolidado se convierte en documentos con frontmatter y de ahí en un índice; cada paso es reproducible y comprobable.
- **Lo que genera más incertidumbre:** la vigencia y la calidad del dato. Las convocatorias multifase, las ayudas sin fecha de fin y la cobertura limitada pueden provocar recomendaciones incompletas o desfasadas.
- **La fuente que más problemas da:** las convocatorias con varias fases y las publicaciones del BOP, donde los plazos y los importes aparecen dispersos y a veces se contradicen con el texto.
- **Si no pudiera construir la capa gold tal como la he definido:** el sistema seguiría funcionando con el CSV procesado y una búsqueda directa sobre él, pero perdería el filtrado exacto por metadatos, que es justo lo que evita recomendar ayudas cerradas.
- **Alternativa para simplificar:** reducir el frontmatter a los campos imprescindibles (`id`, `titulo`, `tipo_ayuda`, `ambito_geografico`, `estado_calculado`, `importe_max`, `tipo_importe`, `url_oficial`) o centrarme solo en Córdoba capital y las prestaciones estatales, que son los registros más fáciles de mantener al día.
- **Dependencia de servicios externos:** el modelo de lenguaje sí depende de una API con cuota gratuita. Por eso los embeddings y el índice son locales: la capa gold se reconstruye y se consulta sin depender de ella.
