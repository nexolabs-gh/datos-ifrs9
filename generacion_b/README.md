# `generacion_b/` — datos del curso «Pérdidas Esperadas IFRS 9 con Python» (Generación B)

Todo lo que leen los notebooks de la Generación B vive en esta carpeta, con una sola función:

```python
BASE = "https://raw.githubusercontent.com/nexolabs-gh/datos-ifrs9/main/generacion_b"
def leer(tabla, cartera="aconcagua"):
    return pd.read_parquet(f"{BASE}/{cartera}_{tabla}.parquet")
```

- **Crudos** (`cartera`, `panel`, `flujos`, `recuperaciones`, `defaults`, `macro_historico`,
  `macro_escenarios`): copia bit a bit de los archivos de la raíz del repo.
- **Intermedios**: lo que una clase produce y las siguientes consumen, para que nadie tenga que
  reconstruirlo. Se agregan a medida que avanza el curso.

| Intermedio | Sale de → entra a | Grano | Columnas (unidades) |
|---|---|---|---|
| `vivas` | Clase 1 → todas | una fila por operación viva al corte | la fila del panel en el corte + condiciones de `cartera`; `edad` = meses desde el cursado; `cosecha` = año de cursado |
| `agrupacion` | Clase 3 → 6, 7, 8 | una fila por par (`segmento`, `producto`) | `grupo`: donde se estima la curva de PD (estándar y preferente se juntan; vigilancia va sola). `segmento` puede venir vacío: igual tiene grupo |
| `curva_pd` | Clase 3 → 6, 7, 8 | una fila por (`grupo`, `edad`) | `edad` = meses **desde el cursado** (1, 2, …); `hazard` = probabilidad de caer en 90+ en esa edad si no cayó antes (fracción 0–1, PIT: últimos 12 meses); `pd_acumulada_desde_origen` = 1 − Π(1 − hazard) desde la edad 1; `es_cola` = la edad está después de las 36 observadas y se extiende con el promedio de los 6 últimos hazards |
| `lgd` | Clase 4 → 6, 7, 8 | una fila por `garantia_tipo` | `lgd` = LGD Best Estimate (fracción 0–1): workouts cerrados + abiertos proyectados, descontados a la EIR |
| `curva_pd_ciclo` | Clase 7 → 8 | una fila por (`grupo`, `edad`) | como `curva_pd`, pero estimada con **todos** los meses del panel (el ciclo completo): el nivel de largo plazo al que vuelve el motor desde el mes 49 (método A3, `manifiesto.json → metodo_lote_4`) |
| `manifiesto.json` | — | — | corte, T0, ventana PIT y la huella (sha256) de cada archivo publicado |

**Ojo con `curva_pd`:** la edad cuenta desde que se otorgó el crédito, no desde el corte. La PD de una operación viva
para los próximos meses sale de los hazards **desde su edad actual + 1** en adelante, no de la fila 12 de la tabla.

`aconcagua` = cartera A (clases) · `llanquihue` = cartera B (laboratorios evaluados).
Datos 100 % simulados: no corresponden a ninguna institución real.

---

## Lotes 3 y 4: EAD, etapas, escenarios y fotos de T0

**Convenciones comunes.** Un archivo por cartera: `aconcagua_<tabla>.parquet` (A, clase) y
`llanquihue_<tabla>.parquet` (B, labs evaluados). Corte **2026-06**; **T0 = 2025-06** (el corte menos 12 meses,
la foto de backtesting y roll-forward). Montos en **CLP** (pesos, sin escalar); probabilidades entre 0 y 1;
`mes_futuro` = 1 es el mes siguiente a la fecha de la foto (2026-07 al corte). La llave de operación es
`id_operacion`, la misma de `cartera` y `vivas`. Parámetros del motor en `manifiesto.json → parametros_motor`.

| Archivo | Sale de → entra a | Grano | Filas A / B | Peso A / B |
|---|---|---|---|---|
| `ead` | C5 → C6–C8 | operación viva al corte × mes futuro (1–240) | 4.766.160 / 7.896.480 | 10,8 / 9,0 MB |
| `etapas` | C6 → C7, C8 | operación viva al corte | 19.859 / 32.902 | 0,9 / 1,5 MB |
| `escenarios` | C7 → C8 | escenario × mes futuro (1–240) | 720 / 720 | < 0,1 MB |
| `curva_pd_ciclo` | C7 → C8 | grupo × edad desde el cursado | 3.176 / 3.144 | < 0,1 MB |
| `ecl_escenarios` | C7 → C8 | operación viva al corte × escenario | 59.577 / 98.706 | 0,8 / 1,3 MB |
| `vivas_t0` | C8 | operación viva en T0 | 15.231 / 21.602 | 0,7 / 1,0 MB |
| `curva_pd_t0` | C8 | grupo × edad desde el cursado | 397 / 393 | < 0,1 MB |
| `lgd_t0` | C8 | tipo de garantía | 3 / 3 | < 0,1 MB |
| `ead_t0` | C8 | operación viva en T0 × mes futuro (1–240) | 3.655.440 / 5.184.480 | 8,5 / 6,0 MB |
| `etapas_t0` | C8 | operación viva en T0 | 15.231 / 21.602 | 0,6 / 0,9 MB |

---

## `ead` — la exposición de cada operación, mes a mes (C5)

Grilla **completa**: exactamente 240 filas por operación viva, ceros incluidos. Así un `pivot` no deja huecos (un
NaN multiplicado por una PD marginal de cero da un ECL NaN).

| Columna | Tipo | Unidad | Qué es |
|---|---|---|---|
| `id_operacion` | texto | — | La operación (todas las de `vivas`, ninguna más) |
| `mes_futuro` | int16 | meses | t = 1…240 |
| `ead` | float64 | CLP | Exposición si el incumplimiento ocurre en el mes t: el capital vigente **al inicio** de ese mes |

Cómo se construye (la receta de la v1, sin cambios):

- **Amortizables con calendario**: t = 1 es el saldo insoluto del corte; t ≥ 2 es el `saldo_post` de la cuota
  t − 1 de `flujos`; cero después de vencer. Se multiplica por la supervivencia al prepago, (1 − h)^(t − 1), con h el
  hazard mensual de prepago del producto en la ventana 2025-07…2026-06 (A: consumo 0,443 %, hipotecario 0,042 %).
- **Amortizables sin calendario** (vencidos con saldo impago; 50 en A, 96 en B): el saldo del corte, constante.
- **Revolventes**: `saldo_utilizado` + CCF × máx(`cupo_vigente` − `saldo_utilizado`, 0), constante. CCF de política
  **35 %**: CMF B-3, líneas de libre disposición, leído de la librería del curso (matrices `cmf_b1_b3_2025_01`).
- **La EAD no corta la vida; la corta la PD.** El motor apaga el hazard después de la vida remanente (contractual en
  los amortizables, 36 meses en los revolventes, 1 en los vencidos sin calendario). Por eso revolventes y vencidos
  tienen EAD constante hasta el mes 240, igual que en la v1: más allá de su vida no entra al ECL. El perfil de
  exposición de C5 es la suma de `ead` por `mes_futuro` (A: 352.434 / 321.541 / 264.817 MM en los meses 1, 12 y 36).
- Cuadratura: Σ `ead` del mes 1 = saldo contable + exposición contingente (A: 352.433,7 = 347.480,4 + 4.953,3 MM).

Matriz para el motor: `ead.pivot(index="id_operacion", columns="mes_futuro", values="ead")`.

## `etapas` — la política de staging de C6, al corte

Una fila por operación viva, en el orden de `vivas`. Horizonte de la política: **36 meses** (el de la v1).

| Columna | Tipo | Unidad | Qué es |
|---|---|---|---|
| `id_operacion` | texto | — | La operación |
| `grupo` | texto | — | Grupo del motor (el de `agrupacion`); indexa `curva_pd` |
| `vida_remanente` | int16 | meses | Vida remanente del motor, 1–240: plazo − edad en amortizables (mínimo 1), 36 en revolventes. La política de C6 la usa acotada a 36 |
| `factor_origen` | float64 | veces | Nivel de riesgo del calendario de cursado relativo al de hoy: f(cosecha) ÷ f(ventana reciente), con f = observados ÷ esperados por edad y cascada grupo → producto → cartera → 1,0 (15 eventos mínimos). Cosechas anteriores al panel: 1 ÷ f(reciente) |
| `pd_origen` | float64 | probabilidad | PD lifetime **de origen** sobre la misma ventana que la de hoy (desde la edad actual, a lo largo de la vida remanente acotada a 36): el hazard de `curva_pd` × `factor_origen` |
| `pd_lifetime` | float64 | probabilidad | PD lifetime remanente de hoy, misma ventana, sin escenarios |
| `ratio_sicr` | float64 | veces | `pd_lifetime` ÷ `pd_origen`. **NaN** si `pd_origen` = 0: sin criterio relativo, pero sujeta a backstops y cualitativos (29 en A, 0 en B) |
| `gatillo_mora_90` | bool | — | `dias_mora` ≥ 90: backstop de Stage 3 |
| `gatillo_mora_30` | bool | — | `dias_mora` ≥ 30: backstop de Stage 2 (incluye las 90+) |
| `gatillo_cualitativo` | bool | — | `en_watchlist` o `marca_forbearance` |
| `gatillo_relativo` | bool | — | `ratio_sicr` ≥ 2,0 (NaN no gatilla) |
| `etapa` | int8 | 1, 2, 3 | 3 si mora 90+; si no, 2 si mora 30+, cualitativo o relativo; si no, 1 |
| `gatillo` | texto | — | La combinación, con siete etiquetas exactas: `Stage 3 · backstop 90+ días` · `Stage 2 · solo criterio relativo` · `Stage 2 · solo cualitativo` · `Stage 2 · cualitativo + relativo` · `Stage 2 · solo backstop 30-89` · `Stage 2 · mora + otro gatillo` · `Stage 1 · ningún gatillo` |
| `ecl_12m` | float64 | CLP | Σ de t = 1 a 12 de PD marginal × LGD × EAD(t) × (1 + EIR)^(−t/12) |
| `ecl_lifetime` | float64 | CLP | La misma suma hasta la vida remanente, **con horizonte de 36 meses** (el de la política de C6). La vida completa (240) entra en C7, en `ecl_escenarios` |
| `ecl_politica` | float64 | CLP | El ECL de su etapa: Stage 1 → `ecl_12m`, Stage 2 → `ecl_lifetime`, Stage 3 → LGD × EAD del mes 1 |

En A: Stage 1/2/3 = 32,9 / 65,1 / 2,0 % de las operaciones; ECL de la política 4.627,6 MM (1,33 % del saldo).
Para mover el umbral θ sin recalcular nada: `etapa = 3` si `gatillo_mora_90`; `2` si `gatillo_mora_30`,
`gatillo_cualitativo` o `ratio_sicr ≥ θ`; si no, `1`. El Stage 3 conserva su `ecl_politica`; el resto toma
`ecl_lifetime` o `ecl_12m` según su etapa nueva. La PD de origen a 240 meses (la que usa C7) es la misma fórmula con
`factor_origen` y la vida remanente completa (hasta 240 meses).

## `escenarios` — la trayectoria de cada escenario (C7)

El hazard de una operación en el mes futuro h tiene tres tramos (variante **A3**, la corrección del largo plazo
decidida el 2026-09-27):

- **proyección** (h = 1–24): hazard de `curva_pd` × `multiplicador_pit`(h) — el satélite con la macro del escenario;
- **reversión** (h = 25–48): hazard de `curva_pd` × e^(log m), con
  log m = cota[(1 − `w_ciclo`) · `log_ancla` + `w_ciclo` · log(`curva_pd_ciclo` ÷ `curva_pd`)], **por operación**, a
  su edad, y la cota = recortar log m a [`log_m_piso`, `log_m_tope`]; 0 si alguno de los dos hazards es 0;
- **TTC** (h = 49–240): el hazard de `curva_pd_ciclo` — la curva del ciclo por grupo y edad, igual en los tres.

Siempre con tope 1 y cero después de la vida remanente. Las dos curvas se leen a la edad de la operación en ese mes
(edad al corte + h). Es todo lo que un notebook necesita: no hay que recalcular el satélite.

| Columna | Tipo | Unidad | Qué es |
|---|---|---|---|
| `escenario` | texto | — | `base`, `adverso` u `optimista` |
| `mes_futuro` | int16 | meses | h = 1…240 (h = 1 es 2026-07) |
| `tramo` | texto | — | `proyección` (1–24) · `reversión` (25–48) · `TTC` (49–240): qué fórmula de arriba aplica |
| `peso` | float64 | probabilidad | La ponderación publicada en `macro_escenarios` (0,5 / 0,3 / 0,2); constante por escenario, suman 1 |
| `multiplicador_pit` | float64 | veces | m(h) = f̂(h) ÷ f̂(ventana reciente) sobre la curva PIT, acotado entre el mejor mes observado y 1,5 × el peor. **Solo en la proyección; NaN después** (no hay multiplicador escalar en la reversión ni en el TTC). Reemplaza a la columna `multiplicador` de antes |
| `acotado_proyeccion` | bool | — | El m(h) de la proyección quedó en la cota (A: 0 / 21 / 1 meses en base / adverso / optimista; B: 0 / 12 / 0). `False` fuera de la proyección |
| `log_ancla` | float64 | log | Constante por escenario: log de la proyección **sin acotar** del mes 24 relativa al ancla, log f̂(24) − log f̂(reciente). Es el punto de partida de la reversión |
| `w_ciclo` | float64 | 0–1 | Avance de la reversión hacia la curva del ciclo: 0 en la proyección, (h − 24) ÷ 24 en la reversión, 1 en el TTC |
| `log_m_piso`, `log_m_tope` | float64 | log | La cota, en log del multiplicador (constantes): log del mejor mes observado y de 1,5 × el peor, menos log f̂(reciente). Con ellas `acotado_proyeccion` y el recorte de la reversión |
| `ops_acotadas` | int32 | operaciones | En la reversión, cuántas vivas quedan con su log m en la cota ese mes; 0 fuera de ella (A: 186.499 operación-mes en el adverso, 0 en base y optimista; B: 0). La cota se cuenta y se reporta |
| `factor_overlay` | float64 | veces | Constante: e^(brecha de anclaje) = nivel **observado** ÷ ajustado de la ventana reciente (A: 1,236; B: 1,111). Es el overlay **propuesto y rechazado** de C8 (el hazard de los 240 meses × este factor): duplica el anclaje, porque `curva_pd` ya es la PIT de esa ventana. **No se provisiona ni se firma**. Reemplaza a `multiplicador_overlay` |

Satélite elegido (filtro duro Durbin-Watson 1,6–2,4 y signo, después el indicador multi-criterio): A
`desempleo(t−3) + IMACEC(t−3)`; B `IMACEC(t−3) + IPC(t−3)`.

**Por qué el largo plazo es una curva y no un nivel.** f = observados ÷ esperados con la curva por edad de todo el
panel. La curva PIT (`curva_pd`) está sobre la del ciclo, pero **no en la misma proporción en todas las edades**: en
la ventana reciente, en promedio, 1,877 veces (A; B: 1,821), y en las edades que pesan en los meses 49–240, solo
1,07 veces. Ningún escalar sobre la PIT devuelve la curva del ciclo: 1 ÷ 1,877 (la variante A2, publicada un día)
dejaba esa cola en 0,57× el ciclo, y la media de log f de la v1 (0,214 en A, 0,090 en B), en ~0,23×. Por eso desde el
mes 49 el hazard **es** `curva_pd_ciclo`. A la cartera, el tramo TTC queda en 0,93× la PIT en A (0,84× en B).

## `curva_pd_ciclo` — la curva del ciclo (C7)

Mismas columnas, largo y reglas que `curva_pd` (`grupo`, `edad`, `hazard`, `pd_acumulada_desde_origen`, `es_cola`;
cola = promedio de los 6 últimos hazards observados), con los mismos grupos del corte, pero con **todos los meses del
panel** en vez de la ventana 2025-07…2026-06. La propiedad que la define (y que el builder afirma grupo a grupo): con
ella, los esperados del panel completo son exactamente sus eventos (A: 974 = 974). La PIT no la cumple.

## `ecl_escenarios` — el ECL por operación y escenario (C7)

| Columna | Tipo | Unidad | Qué es |
|---|---|---|---|
| `id_operacion` | texto | — | La operación (las tres corridas cubren todas las vivas) |
| `escenario` | texto | — | `base`, `adverso` u `optimista` |
| `etapa` | int8 | 1, 2, 3 | La etapa **recalculada en ese escenario** (la PD de hoy cambia con el escenario; la de origen no), con vida completa |
| `ecl` | float64 | CLP | ECL de la operación en ese escenario con el hazard A3, vida completa (hasta 240 meses) |

ECL ponderado **sobre salidas**: Σ peso × `ecl` (A: 5.555,5 MM; por escenario 3.507,3 / 10.731,2 / 2.912,8 · B:
6.234,6 MM; 3.524,4 / 12.835,0 / 3.109,4). **Es el ECL del modelo y es lo que se firma**; el ECL a 12 meses no
depende del largo plazo (A: 2.551,7 MM; B: 4.076,1 MM).

## Las fotos de T0 (C8: backtesting y roll-forward)

Todo lo que el motor sabía en **T0 = 2025-06**, sin mirar nada posterior: ventana PIT 2024-07…2025-06, agrupación
decidida con los eventos vistos hasta T0 y LGD con los workouts conocidos hasta T0.

- **`vivas_t0`** — mismas columnas y reglas que `vivas`, en 2025-06 (una fila por operación viva: su fila del panel
  sin `motivo_baja` + condiciones de `cartera`, `edad` y `cosecha`).
- **`curva_pd_t0`** — mismas columnas que `curva_pd` (`grupo`, `edad`, `hazard`, `pd_acumulada_desde_origen`,
  `es_cola`) y el mismo largo. En T0 ningún grupo ni producto alcanzaba 30 eventos: en A y en B hay un solo grupo,
  `cartera_total`.
- **`lgd_t0`** — `garantia_tipo`, `lgd`: la LGD Best Estimate con la información hasta T0 (A: 69,42 / 43,78 /
  19,76 %).
- **`ead_t0`** — mismo formato que `ead`, pero reconstruida **desde el contrato** (amortización francesa con
  `tasa_interes_anual`), porque `flujos` es el calendario publicado al corte y usarlo en T0 miraría el futuro. Prepago
  con la ventana de T0.
- **`etapas_t0`** — mismas columnas que `etapas`, con la foto de T0; `grupo` es la agrupación de T0 (la que indexa
  `curva_pd_t0`).

Roll-forward T0 → corte (el orden es parte del método): inicial (`etapas_t0.ecl_politica`) → bajas → exposición y
edad (EAD y edad del corte, modelo y etapa de T0) → cambio de etapa → modelo (curvas, LGD y agrupación del corte,
vida completa, sin escenarios) → escenarios (la trayectoria A3 completa) → altas → overlay → final = ECL firmado.
**Se firma el modelo: la línea overlay vale 0.** El overlay de la brecha de anclaje (el hazard A3 × `factor_overlay`,
menos el modelo) se reporta aparte, como propuesto y rechazado (A: 849,9 MM; B: 423,7 MM). En A: 297,3 → 5.555,5 MM
(escenarios +10,5; altas 1.910,2); en B: 337,1 → 6.234,6 MM (escenarios −980,1; altas 3.256,7).

## Cómo se verifican

`01_datos/generador/construir_intermedios.py --lote 3|4`, en tres capas y sin escribir nada si algo falla:
(1) controles sobre lo que se publica, leído de vuelta desde los bytes del parquet (grano, unicidad, cobertura de
toda viva, rangos, etiquetas exactas, reglas de etapa y de EAD, la forma de la trayectoria —`multiplicador_pit` solo
en la proyección, `w_ciclo`, ancla y cota coherentes, las vivas en la cota recontadas—, la propiedad de la curva del
ciclo, el ECL publicado = el motor A3 armado solo con lo publicado y la firma = el modelo); (2) un oráculo que ejecuta
**literalmente** las celdas del Lab 4 de la v1 sobre A y B y exige igualdad vector por vector en todo lo que A3 no
toca; (3) los números publicados de la v1 (`numeros_c5` a `numeros_c8`) en A. Cada control se valida mutando los
datos en `mutaciones_intermedios.py`.

**Desvío declarado respecto de la v1 (lote 4).** Dos correcciones de método: el largo plazo es la curva del ciclo
(A3, arriba) y se firma sin overlay. El oráculo de la v1 corre literal y se compara en lo que A3 no toca (satélite,
meses 1–24, ancla y cota, factor del overlay, fotos de T0, roll-forward hasta el paso «modelo»); los números de la v1
que cambian (multiplicador de la cartera desde el mes 36, meses acotados, ECL y etapas por escenario, trampa #2,
roll-forward, lo reportado) se afirman **distintos** de la v1, y sus valores nuevos se contrastan en A con un
diagnóstico independiente (`00_plan/diagnostico_c7_c8/diag_a3.py`) que primero reproduce lo publicado y después cambia
solo el método (B se contrastó con el mismo código). Todo lo que no depende del largo plazo (fotos de T0, `etapas`,
curvas PIT, EAD, LGD) sigue igual bit a bit.
