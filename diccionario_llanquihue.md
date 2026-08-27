# Diccionario de datos — Banco Llanquihue

> Datos **100% sintéticos**, generados para el curso *Pérdidas Esperadas IFRS 9 con Python* (Academia Bayes). Ninguna persona ni institución real está representada.

**Panel**: 2022-07 a 2026-06 (48 meses) · **fecha de corte del cálculo**: 2026-06 · montos en CLP.

**Semilla**: `20261021` — misma semilla, misma cartera bit a bit.

## Resumen de la cartera

| Métrica | Valor |
|---|---|
| Clientes | 25.000 |
| Operaciones | 40.000 |
| Operaciones vigentes al corte | 35.022 |
| Filas del panel mensual | 723.838 |
| Operaciones que entraron en default | 4.458 |
| Tasa de default observada | 10,19% |
| Workouts abiertos a la fecha de corte | 46,2% |
| Tasa de cura entre los defaults | 13,2% |

**Mix de producto**: consumo_cuotas 44,2% · linea_revolvente 25,9% · tarjeta_credito 21,9% · hipotecario 8,0%

## Advertencias de uso

1. La unidad de cálculo del ECL es la **operación**, no el cliente.
2. La tasa de descuento que exige la norma es `tasa_efectiva_original` (EIR), **no** `tasa_interes_anual`.
3. Los **revolventes no tienen calendario contractual**: su exposición futura se construye, no se busca en `flujos`.
4. Hay operaciones con **workout abierto** (`fecha_cierre_workout` vacío). Excluirlas del cálculo de LGD sesga el resultado.
5. Una fracción de operaciones **no tiene `segmento` asignado**. Un `join` interno las hace desaparecer del cálculo sin aviso.

## `llanquihue_clientes` — 25.000 filas

Un registro por deudor. El cliente **no** es la unidad de cálculo del ECL: sirve para segmentar y para agrupar operaciones del mismo deudor.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_cliente` | str | Identificador único del cliente |
| `fecha_alta_cliente` | str | Mes de alta en la institución (YYYY-MM) |
| `edad` | int64 | Edad en años al inicio del panel |
| `sexo` | str | M / F |
| `region` | str | Región de residencia (Chile) |
| `estado_civil` | str | soltero / casado / divorciado / viudo |
| `nivel_educacional` | str | básica / media / técnica / universitaria / postgrado |
| `tipo_empleo` | str | dependiente / independiente / jubilado |
| `renta_liquida` | float64 · 19,2% vacío | Renta líquida mensual declarada (CLP). **Puede venir vacía** |
| `n_dependientes` | int64 | Número de cargas familiares |

## `llanquihue_cartera` — 40.000 filas

**Una fila por operación: es la unidad de cálculo del ECL.** Contiene las condiciones contractuales vigentes desde el reconocimiento inicial.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Identificador único de la operación |
| `id_cliente` | str | Deudor de la operación |
| `producto` | str | consumo_cuotas · hipotecario · linea_revolvente · tarjeta_credito |
| `fecha_cursado` | str | Mes de reconocimiento inicial. **Define la PD de origen del SICR**. Puede ser anterior al inicio del panel (cartera legada) |
| `monto_original` | int64 | Monto cursado (CLP) |
| `plazo_meses` | int64 | Vida contractual en meses. **-1 = sin vencimiento** (revolventes) |
| `tasa_interes_anual` | float64 | Tasa contractual anual |
| `tasa_efectiva_original` | float64 | **EIR — tasa efectiva original.** Incorpora comisiones, seguros y gastos de originación. Es la tasa de descuento que exige IFRS 9; **no** es la tasa contractual |
| `tipo_amortizacion` | str | francés · sin vencimiento |
| `cupo_aprobado` | float64 · 52,2% vacío | Línea aprobada (CLP). Solo revolventes; vacío en amortizables |
| `garantia_tipo` | str | sin garantía · hipotecaria · prendaria |
| `garantia_valor_tasacion` | float64 · 88,1% vacío | Valor de tasación de la garantía (CLP). Vacío si no hay |
| `segmento` | str · 1,5% vacío | Segmento de riesgo (producto × tramo). Unidad de agrupación del motor |

## `llanquihue_panel` — 723.838 filas

Panel mensual operación × mes con el comportamiento observado. Es el insumo de las matrices de transición, del staging y del roll-forward de la provisión.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `mes` | str | Mes del panel (YYYY-MM) |
| `saldo_insoluto` | float64 | Capital vigente al cierre del mes (CLP). Mientras la operación paga sigue el calendario contractual; en mora se congela y devenga interés |
| `saldo_utilizado` | float64 · 48,6% vacío | Revolventes: monto usado de la línea. Vacío en amortizables |
| `cupo_vigente` | float64 · 48,6% vacío | Revolventes: línea vigente (se recorta tras deterioro). Vacío en amortizables |
| `dias_mora` | int16 | Días de mora al cierre del mes (DPD) |
| `cuota_pactada` | float64 | Cuota exigible del mes (CLP) |
| `monto_pagado` | float64 | Monto efectivamente pagado en el mes (CLP) |
| `marca_default` | bool | Verdadero si la operación está en 90+ DPD ese mes |
| `marca_forbearance` | bool | Renegociación o reprogramación. **La marca es persistente**: sigue encendida aunque el pago se normalice (criterio cualitativo de SICR) |
| `en_watchlist` | bool | Alerta interna de seguimiento del mes. Se enciende **antes** que la mora |

## `llanquihue_flujos` — 765.822 filas

Calendario contractual de pagos futuros desde la fecha de corte. **Solo operaciones amortizables vigentes**: los revolventes no tienen calendario, y su exposición futura hay que construirla con un factor de conversión sobre la línea.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `mes` | str | Mes futuro del flujo (YYYY-MM) |
| `n_cuota` | int64 | Número de cuota contado desde la fecha de corte (1 = mes siguiente) |
| `flujo_capital` | float64 | Amortización de capital del período (CLP) |
| `flujo_interes` | float64 | Interés del período (CLP) |
| `flujo_total` | float64 | Cuota total del período (CLP) |
| `saldo_post` | float64 | Saldo insoluto después de pagar la cuota (CLP) |

## `llanquihue_recuperaciones` — 10.622 filas

Eventos de flujo del proceso de recuperación (workout) de las operaciones que entraron en default. Varias filas por operación.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `fecha_default` | str | Mes de entrada en default (90+ DPD) |
| `fecha_flujo` | str | Mes en que ocurre el evento de recuperación |
| `monto_recuperado` | int64 | Monto recuperado en el evento (CLP) |
| `tipo` | str | pago voluntario · ejecución de garantía · castigo |
| `costos_directos` | int64 | Costos de cobranza asociados al evento (CLP) |
| `fecha_cierre_workout` | str · 28,2% vacío | Mes de cierre del proceso. **Vacío si el workout sigue abierto a la fecha de corte** |
| `marca_cura` | bool | La operación volvió a estar al día de forma sostenida |

## `llanquihue_defaults` — 4.458 filas

Ficha resumen: un registro por operación que entró en default. Es la base natural para estimar LGD.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `fecha_default` | str | Mes de entrada en default |
| `saldo_al_default` | float64 | Exposición al momento del incumplimiento (CLP) |
| `fecha_cierre_workout` | str · 46,2% vacío | Mes de cierre. **Vacío = workout abierto al corte** |
| `marca_cura` | bool | La operación se curó |

## `llanquihue_macro_historico` — 48 filas

Serie macroeconómica mensual observada durante el panel. Es el ciclo que efectivamente vivió la cartera.

| Columna | Tipo | Descripción |
|---|---|---|
| `mes` | str | Mes (YYYY-MM) |
| `desempleo_pct` | float64 | Tasa de desempleo (%) |
| `imacec_var_anual_pct` | float64 | Variación anual del índice de actividad (%) |
| `tpm_pct` | float64 | Tasa de política monetaria (%) |
| `ipc_var_anual_pct` | float64 | Inflación anual (%) |
| `indice_precios_vivienda` | float64 | Índice de precios de vivienda (base 100 al inicio del panel) |

## `llanquihue_macro_escenarios` — 180 filas

Tres trayectorias prospectivas desde la fecha de corte, con sus ponderaciones declaradas. Insumo del cálculo multiescenario.

| Columna | Tipo | Descripción |
|---|---|---|
| `escenario` | str | base · adverso · optimista |
| `ponderacion` | float64 | Ponderación declarada del escenario (suman 1) |
| `mes` | str | Mes proyectado (YYYY-MM) |
| `horizonte_meses` | int64 | Meses desde la fecha de corte (1 = mes siguiente) |
| `desempleo_pct` | float64 | Tasa de desempleo proyectada (%) |
| `imacec_var_anual_pct` | float64 | Variación anual de actividad proyectada (%) |
| `tpm_pct` | float64 | Tasa de política monetaria proyectada (%) |
| `ipc_var_anual_pct` | float64 | Inflación anual proyectada (%) |
| `indice_precios_vivienda` | float64 | Índice de precios de vivienda proyectado |

---

*Generado automáticamente por `01_datos/generador/diccionario.py` desde los datos publicados. Los números de esta página se leen del output real: no se escriben a mano.*