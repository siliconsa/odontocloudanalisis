# Reglas de Preauditoría Odontológica — Sistema COS (Círculo Odontológico Santafesino)

Este documento describe las reglas **reales** de preauditoría implementadas en el sistema `coscloudapi` (motor de facturación/gestión del Círculo Odontológico Santafesino), extraídas directamente del código fuente que las ejecuta. No es un set de reglas genérico de la industria: es la documentación de lo que el sistema efectivamente valida, código por código del nomenclador, antes de aprobar o rechazar una práctica facturada por un profesional.

## 1. Dónde vive esta lógica

- **`routes/preauditoria.js`** → **`controllers/preauditoria.js`** → **`models/preauditoria.js`**: el endpoint `POST /preauditoria` dispara la preauditoría sobre las prácticas cargadas. `models/preauditoria.js` (9.637 líneas) contiene un `switch(row.codigo)` con más de 95 `case`, uno por cada código de prestación del nomenclador que tiene una regla específica.
- **`functions/funciones_preauditoria.js`**: ~30 funciones auxiliares reutilizadas por los distintos `case` del switch, que arman el SQL de cada control (vigencia/frecuencia, control de extracción, edad permitida, dientes permitidos, cantidad de caras, tope máximo por código, verificación de padrón de obra social, etc.).
- **Códigos sin `case` explícito**: caen en el `default:` del switch y se **aprueban automáticamente** (`resultadopreauditoria = 83`), sin ningún control. Es decir: la ausencia de regla equivale a aprobación directa.

## 2. Conceptos y controles reutilizados en casi todos los códigos

Estos "bloques" se repiten con variaciones (parámetros distintos) en la mayoría de los `case`. Para no repetir la explicación código por código, se listan acá y en cada código sólo se indican sus parámetros:

| Control | Qué hace | Función fuente |
|---|---|---|
| **RB preexistente** | Rechaza si en el Relevamiento Bucal (RB) del paciente esa pieza figura con una práctica (típicamente una extracción) ya aprobada y pendiente de facturar. Evita, por ejemplo, facturar un tratamiento de conducto o una restauración sobre un diente que según el RB "debería" extraerse. | `BuscarPreexistenteAutorizadoEnRB` |
| **Control de Extracción** | Rechaza si esa misma pieza ya tiene facturada una extracción (lista fija de códigos: 10.01.00, 10.01.01 a 10.01.08, 10.01.10, 10.01.13, 10.01.50, 10.02.04, 10.09.00, 10.18.00). Evita seguir tratando un diente que ya fue extraído. | `ExtraccionControlN` / variantes |
| **Vigencia / frecuencia** | Rechaza si el mismo código (o un grupo de códigos "hermanos") ya se facturó para ese paciente/diente/profesional dentro de una ventana de meses determinada. Es el control de "no repetir antes de tiempo". Muchas variantes: por diente, por arcada, con o sin restricción de profesional, con o sin restricción a obra social prepaga o a una obra social puntual (frecuentemente IAPOS, obra_social_id=18, que casi siempre tiene reglas propias y más estrictas). | `VigenciaN`, `VigenciaN2..N5`, `VigenciaControlDienteN`, `VigenciaControlDiente2N/2AN/2BN`, `VigenciaControlArcosN`, `VigenciaCantidadMaxXDiente`, `VigenciaControlCarasN` |
| **Edad permitida** | Rechaza si la edad del paciente (calculada desde la fecha de nacimiento) está fuera del rango habilitado para ese código. | `EdadPermitida` |
| **Dientes permitidos** | Rechaza si la pieza facturada no está en la lista de piezas habilitadas para ese código (ej. sólo molares, sólo piezas temporarias). | `DientesPermitidos` |
| **Cantidad de caras** | Rechaza si la cantidad de caras cargadas en el odontograma para esa práctica está fuera del rango permitido (ej. restauración simple 1–3 caras, compuesta 2–4, compleja exactamente 5). | `CarasCantidad` |
| **Tope máximo por código** | Rechaza si ya se alcanzó una cantidad máxima histórica de facturaciones de ese código (para el paciente, a veces por profesional). | `TopeMaximoPorCodigoN`, `ExistenciaCodigo` |
| **Relevamiento Bucal obligatorio** | Si la obra social exige RB, valida que exista uno vigente (normalmente 17–18 meses) y aprobado (estado 83) antes de habilitar ciertas prácticas. Si no existe, rechaza pidiendo el envío del RB. | `VigenciaRelevamientoBucal`, `BuscaPracticaAutorizadaEnRBVigente` |

Cuando cualquiera de estos controles falla, el código queda en estado **84 (rechazada)** con el/los mensaje(s) correspondiente(s) concatenados; si falta el Relevamiento Bucal, el estado es **85**; si todos los controles pasan, el estado es **83 (aprobada)**.

**IAPOS (obra_social_id = 18)** aparece una y otra vez como caso especial: en general tiene ventanas de vigencia más largas y controles más estrictos que el resto de las obras sociales.

---

## 3. Reglas por código de prestación

### 3.1 Consultas y relevamiento (01.xx – 06.xx)

**01.01.00 — Consulta de Relevamiento Bucal**
- Si la obra social exige RB: busca un RB vigente (≤17 meses, estado aprobado). Si no lo encuentra, **rechaza siempre** pidiendo el envío del RB (código 85) — la bifurcación por tipo de obra social en el código no cambia este desenlace en la práctica.
- Si pasa ese control: no permite repetir 01.01.00 (ni 05.00.00 en menores de 12 años) para el mismo profesional u obra social en los últimos 17 meses.

**01.02.00 — Consulta de Urgencia** — vigencia de 3 meses contra el propio código, mismo profesional/obra social. (Existe una regla alternativa por mes calendario, comentada/deshabilitada en el código "hasta que Alejandra lo indique luego de la pandemia".)

**01.06.00** *(sin nombre en catálogo base)* — no puede facturarse más de una vez por mes calendario, misma obra social prepaga.

**05.00.00 — Consulta Preventiva y de Relevamiento** — dos etapas: (1) si la obra social exige RB, debe existir un RB vigente (18 meses) o se rechaza pidiendo el envío; (2) vigencia de 12 meses contra el propio código y 01.01.00, más edad permitida 0–12 años.

**05.02.00 — Topicación con Flúor** — vigencia de 6 meses si es IAPOS, 4 meses el resto; edad 0–12 años (límite superior no inclusive).

**05.02.01 — Topicación con Flúor en Embarazadas** — vigencia de 3 meses, sin control de edad.

**05.02.02 — Topicación con Flúor en Discapacitados** — vigencia de 4 meses, sin control de edad.

**05.03.00 — Inactivación de Policaries Activas** — edad 0–10 años; tope de 1 sola vez en la vida por paciente (excluyendo el mes en curso del conteo).

**05.04.00 — Detección y Control de Placa Bacteriana** — vigencia 12 meses; edad 0–12 años.

**05.05.00 — Sellantes de Fosas y Fisuras** (la regla más elaborada del bloque):
- Control de RB preexistente y de extracción.
- Vigencia por diente condicionada a edad y pieza: 0–8 años en molar permanente (16,17,26,27,36,37,46,47) → 12 meses; 0–18 años en cualquier otra pieza permitida → 23 meses; fuera de esos rangos de edad, rechazo directo.
- Piezas permitidas distintas según sea IAPOS (sólo molares 16,17,26,27,36,37,46,47) o el resto de obras sociales (listado más amplio de premolares y molares).

**06.01.00** — si la obra social exige RB, exige que la práctica esté autorizada en un RB vigente (18 meses); si no lo exige, aprobación automática.

**06.02.00 / 06.03.00** — exigen tener facturado antes el código 06.01.00 para el mismo profesional/obra social (06.03.00 sólo cuenta antecedentes de obra social prepaga).

**07.01.00 — Motivación** — edad menor de 10 años; vigencia 6 meses por profesional; tope histórico de 2 veces (IAPOS) o 4 veces (resto) por profesional.

**07.02.00 — Motivación en Pacientes con Discapacidad Mental** — sin límite de edad; vigencia 4 meses; tope histórico de 8 veces por profesional.

**07.03.00 — Coronas Metálicas para Dientes Primarios** — RB preexistente, control de extracción, vigencia por diente 24 meses, edad 0–10 años, sólo piezas temporarias (51–55, 61–65, 71–75, 81–85).

### 3.2 Restauraciones (02.xx)

**02.01.00 — Restauración Convencional**: RB preexistente + control de extracción + cantidad de caras 1–3 + vigencia (IAPOS: 48 meses contra restauración compleja/compuesta en la misma obra social, con reintento a 24 meses sin obra social puntual; resto: 24 meses) + tope de repeticiones por diente (1 o 2 según antecedentes) + para IAPOS, si hay cara Incisal/General, exige práctica autorizada en RB vigente.

**02.01.01 (menores de 10)** — versión simplificada: RB preexistente, control de extracción, edad 0–9, caras 1–2, vigencia 48 meses contra 02.09.00 y contra el grupo de restauraciones compuestas, sin las reglas IAPOS ni el tope de repeticiones.

**02.01.02** — igual a 02.01.01 sin control de edad, vigencia 24 meses.

**02.01.50 (exfoliación)** — igual patrón sin edad, vigencia 24 meses contra 02.09.00 y el grupo compuesto, exigiendo obra social prepaga.

**02.02.00 — Restauración Compuesta** — igual a 02.01.00 pero caras 2–4 y comparando vigencia contra restauración compleja (02.09.00) y con retención adicional (02.09.10); sin el tope de repeticiones ni el control final de RB de 02.01.00.

**02.02.01 (menores de 10) / 02.02.02 / 02.02.50 (exfoliación)** — mismas variantes que el bloque 02.01.xx, ajustando caras a 2–4 y las ventanas de vigencia (48, 24 y 24 meses respectivamente). *(02.02.02 tiene un detalle notable: el control de vigencia de caras se ejecuta prácticamente sin filtro de código, lo que parece un descuido de implementación más que una regla intencional.)*

**02.09.00 — Restauración Compleja** — RB preexistente, control de extracción, caras exactamente 5, vigencia como OR entre dos ramas: no-IAPOS 24 meses / IAPOS 48 meses contra el grupo de restauraciones compuestas. El mensaje de rechazo se simplificó a "Rechazo: Vigencia" por pedido explícito de un referente de negocio en 2020 (hay un comentario en el código citando ese pedido).

**02.09.01 (menores de 10) / 02.09.02** — mismas reglas que 02.09.00 sumando (02.09.01) o no (02.09.02) el control de edad 0–10.

**02.09.10 — Restauración Compleja con Retención Adicional** — RB preexistente, control de extracción, caras 3–5, vigencia 48 meses contra el propio código, sin distinción de obra social ni profesional.

### 3.3 Endodoncia / tratamientos de conducto (03.xx)

Familia muy homogénea: **03.01.00 (uniradicular)**, **03.02.00 (2 conductos)**, **03.03.00 (3 conductos)**, **03.04.00 (4 conductos)** comparten la misma estructura — RB preexistente, control de extracción, y vigencia por diente de 36 meses (mismo profesional) / 24 meses (cualquier profesional) contra los códigos "hermanos" (los otros números de conducto). Cada uno tiene:
- una variante **"en menores de 10 años"** (03.01.01, 03.02.01, 03.03.01, 03.04.01) que agrega el control de edad 0–10;
- una variante **sin sufijo de edad pero con lista de comparación ampliada** (03.01.02, 03.02.02, 03.03.02, 03.04.02) que además compara contra las variantes ".02" entre sí.

**03.05.00 — Biopulpectomía Parcial**: RB preexistente + control de extracción + control histórico (prácticamente sin vencimiento) de que la pieza no tenga ya facturado 03.05.00 o 03.06.01 + sólo en piezas 54,55,64,65,74,75,84,85. La variante de menores de 10 (03.05.01) simplifica: no incluye el control histórico ni el de piezas permitidas. 03.05.02 es la versión mínima (sólo RB + extracción).

**03.06.00 — Necropulpectomía Parcial (Momificación)**: mismo patrón que 03.05.00 (control histórico cruzado contra 03.06.01, piezas 54/55/64/65/74/75/84/85). 03.06.01 agrega edad 0–10. 03.06.02 y 03.06.50 (exfoliación) son versiones mínimas sin control histórico ni de edad.

**03.07.00 — Protección Pulpar Indirecta**: RB preexistente + control de extracción + sólo premolares y molares permanentes (14-18, 24-28, 34-38, 44-48). 03.07.01 agrega edad 0–10.

### 3.4 Prótesis y coronas (04.xx)

Patrón común a casi todo el bloque: **RB preexistente** (si la pieza tiene extracción aprobada pendiente en el RB) + **control de extracción** (lista estándar) + **vigencia por diente contra obra social prepaga**, con ventanas que varían según el código:

| Código | Nombre | Controles propios | Vigencia |
|---|---|---|---|
| 04.01.00 | — | Extracción + vigencia | 48 meses |
| 04.01.01 | Incrustación cavidad simple | Sólo RB preexistente | — |
| 04.01.02 | Incrustación cavidad compuesta | RB + extracción + vigencia | 48 meses |
| 04.01.03 | — | Extracción + vigencia | 48 meses |
| 04.01.04 | Coronas coladas | RB + extracción + vigencia | 48 meses |
| 04.01.05 | Coronas coladas c/frente estético acrílico | Sólo RB preexistente | — |
| 04.01.06 | — | Extracción + vigencia | 84 meses |
| 04.01.08 | Perno muñón simple | RB + extracción + vigencia | 84 meses |
| 04.01.09 | Perno muñón seccionado | RB + extracción + vigencia | 84 meses |
| 04.01.11 | Corona en acrílico | RB + extracción + vigencia | 60 meses |
| 04.01.12 | Coronas provisorias de acrílico | RB + vigencia (sin extracción) | 12 meses |
| 04.01.13 | — | Sólo vigencia | 84 meses |
| 04.01.14 | — | RB + vigencia (sin extracción) | 84 meses |

**04.02.01 — Prótesis parcial acrílico 4-7 dientes**: vigencia por **arcada** (superior/inferior) 60 meses contra obra social prepaga. **04.02.02 (parcial acrílico 8 dientes)** y **04.02.03 (parcial colada cromo-cobalto hasta 6 dientes)** se controlan **cruzados entre sí**, misma arcada, 60 meses. **04.02.04 / 04.02.05 (parcial colada cromo-cobalto más de 6 dientes)**: vigencia global 60 meses (04.02.04 sólo cuenta obra social prepaga; 04.02.05 cuenta cualquier obra social).

**04.03.01 / 04.03.02 — Prótesis completa superior/inferior**: vigencia 60 meses contra obra social prepaga.

**04.04.12** — mismo patrón, vigencia 12 meses.

**04.05.01** — vigencia por diente 84 meses, sin distinguir tipo de obra social.

### 3.5 Periodoncia (08.xx)

**08.11.00 — Consulta Periodontal** — vigencia 24 meses por profesional.

**08.12.00 / 08.12.50 — Tratamiento de Gingivitis por Arcada**: si la obra social exige RB, valida autorización previa en RB (18 meses); edad mínima 12 años; vigencia por arcada cruzada entre ambos códigos (24 meses IAPOS / 12 meses resto); además exige que no haya controles post-tratamiento (08.14.00/08.15.00) en los últimos 12 meses.

**08.13.00 — Enseñanza de Higiene Oral en el Adulto**: vigencia prácticamente única en la vida (6000 meses) si es IAPOS por el mismo profesional, o 12 meses para el resto sin restringir profesional; edad mínima 12 años.

**08.14.00 / 08.15.00 — Controles post-tratamiento**: vigencia cruzada entre ambos y con 08.12.00 (12 meses IAPOS / 4 meses resto), control global sin restringir profesional.

**08.16.00 / 08.17.00 — Tratamiento de enfermedad periodontal (bolsas ≤4mm / >4mm)**: RB preexistente + control de extracción + vigencia por diente cruzada entre ambos y con 08.14.00/08.15.00 (24 meses).

### 3.6 Radiografías y estudios (09.xx)

**09.02.04 (Pantomografía) / 09.02.05 (Teleradiografía)**: vigencia 24 meses (IAPOS) / 12 meses (resto), filtrando por obra social del paciente.

**09.65.10 a 09.65.15** (estudios/prácticas complementarias, sin nombre en catálogo base): vigencia cruzada de 12 meses entre el grupo 09.65.12–09.65.15, con tope combinado de 1 aparición del grupo en 12 meses. 09.65.11.nousar (código marcado como en desuso) mantiene vigencia propia de 12 meses.

### 3.7 Cirugía y extracciones (10.xx)

**10.01.00, 10.01.50, 10.05.00, 10.08.00, 10.09.00, 10.10.00, 10.11.00, 10.12.00, 10.18.00** (extracción dentaria y variantes: exfoliación, reimplante, alargamiento de corona, extracción de restos radiculares, germenectomía, liberación de dientes retenidos, apicectomía, extracción con alveolectomía): comparten exactamente el mismo patrón — **RB preexistente** + **control cruzado contra la lista estándar de códigos de extracción sobre esa misma pieza** — sin controles de edad, vigencia temporal o piezas permitidas adicionales.

**10.41.01 — Tratamiento inicial prequirúrgico, fisura labio-palatino**: vigencia prácticamente única en la vida (999 meses) contra obra social prepaga.

### 3.8 Módulos especiales / registros administrativos (06.xx tratados en 3.1)

*(Sin más códigos con `case` propio detectados en el archivo fuente además de los listados arriba; cualquier código de prestación fuera de esta lista se aprueba automáticamente sin controles.)*

---

## 4. Verificación de padrón (afiliación)

Además del switch de preauditoría por código, `funciones_preauditoria.js` incluye funciones separadas para validar la afiliación vigente del paciente contra padrones externos antes de aceptar cualquier práctica:

- **`VerificarPadronIAPOSWS`**: consulta un servicio SOAP externo de IAPOS para verificar estado de afiliación (activo/baja/inactivo) por DNI, con `FormatoDNI` normalizando el número de documento según la regla de LIBERATIS (prefijos `10`, `200` o `300` según rango de DNI y sexo).
- **`VerificarPadron` / `VerificarPadronOS`**: consultan una base MySQL externa (servidor web del Círculo Odontológico Santafesino) contra un padrón de afiliados de distintas obras sociales/planes (Amasca, Bounus, Cogas, Luz y Fuerza, OSAE, OSPA Vial, Sanatorio Santa Fe, OSDOP, Tadeo Czerweny, Arte de Curar, DOS, etc.), mapeando el plan interno al identificador del padrón externo.
- **`registrarPadronIAPOS`**: sincroniza altas/bajas de afiliados IAPOS (plan_os_id=34) contra el padrón local según la respuesta del webservice.

## 5. Notas de calidad del código fuente (relevantes para quien vaya a mantener o auditar estas reglas)

- Varias reglas usan `eval()` sobre cadenas armadas dinámicamente para evaluar la combinación de controles — un patrón fuera de las buenas prácticas actuales, a tener en cuenta si se migra o refactoriza esta lógica.
- Hay reglas deshabilitadas mediante comentarios en el código (ej. control por mes calendario en 01.02.00, "a habilitar cuando Alejandra lo indique") — es decir, existen decisiones de negocio pendientes de activar que no están documentadas fuera del código.
- Algunos mensajes de rechazo son genéricos o reciclados entre reglas distintas (ej. "Extracción Preexistente en Relevamiento Bucal" se reutiliza para controles que no son estrictamente de extracción), lo que puede confundir al usuario final del sistema.
- Se detectó al menos un caso (02.02.02) donde el control de vigencia de caras se ejecuta con la lista de códigos vacía, lo que en la práctica anula el filtro — posible bug más que regla intencional.
- El catálogo base de prestaciones (`prestacion_nomenclador_cos`) tiene 137 códigos cargados, pero el switch de preauditoría sólo implementa reglas explícitas para ~97 de ellos; el resto se aprueba automáticamente por no tener `case` propio.
- Las credenciales de conexión a la base MySQL externa de verificación de padrón están **hardcodeadas en texto plano** en `funciones_preauditoria.js` (`VerificarPadron`, `VerificarPadronOS`) — riesgo de seguridad a señalar aparte si se decide actuar sobre él.

> Este documento refleja el estado del código al momento de esta revisión (commit `3fcffbc` de `coscloudapi`). Cualquier cambio posterior en `models/preauditoria.js` o `functions/funciones_preauditoria.js` puede volver desactualizadas estas descripciones.
