# Diccionario de datos — Banco Llanquihue

> Datos **100% sintéticos**, generados para el curso *Pérdidas Esperadas IFRS 9 con Python* (Academia Bayes). Ninguna persona ni institución real está representada.

**Panel**: 2022-07 a 2026-06 (48 meses) · **fecha de corte del cálculo**: 2026-06 · montos en CLP.

**Semilla**: `20261021` — misma semilla, misma cartera bit a bit.

## Resumen de la cartera

| Métrica | Valor |
|---|---|
| Clientes | 25.000 |
| Operaciones | 40.000 |
| Operaciones vigentes al corte | 32.902 |
| Filas del panel mensual | 719.063 |
| Episodios de incumplimiento | 4.183 |
| Operaciones que entraron en default | 4.166 |
| Castigos (bajas contables) | 849 |
| Tasa de default observada | 10,42% |
| Workouts abiertos a la fecha de corte | 35,4% |
| Tasa de cura entre los defaults | 10,5% |

**Mix de producto**: consumo_cuotas 43,8% · linea_revolvente 26,4% · tarjeta_credito 21,8% · hipotecario 8,0%

## Advertencias de uso

1. La unidad de cálculo del ECL es la **operación**, no el cliente.
2. La tasa de descuento que exige la norma es `tasa_efectiva_original` (EIR), **no** `tasa_interes_anual`.
3. Los **revolventes no tienen calendario contractual**: su exposición futura se construye, no se busca en `flujos`.
4. Hay operaciones con **workout abierto** (`fecha_cierre_workout` vacío). Excluirlas del cálculo de LGD sesga el resultado.
5. Una fracción de operaciones **no tiene `segmento` asignado**. Un `join` interno las hace desaparecer del cálculo sin aviso.
6. `defaults` tiene una fila **por episodio**, no por operación: la llave es `id_operacion` + `n_evento`. Agrupar solo por `id_operacion` mezcla dos incumplimientos distintos.
7. Toda salida del panel está declarada en `motivo_baja`. El **castigo** da de baja el activo: esa operación no aparece al corte ni tiene calendario en `flujos`.
8. El pago publicado **explica** el saldo, fila a fila y en toda la cartera: `saldo_t = saldo_(t-1)·(1+i) + monto_girado − monto_pagado`. Dos convenciones del dataset: el **mes del desembolso no devenga interés** (el primer interés corre con la primera cuota, al mes siguiente; la cartera legada sí devenga desde su primer mes del panel), y en 90+ el saldo deja de devengar desde el mes siguiente al cruce, con el pago del mes en cero porque el efectivo de la cobranza está en `recuperaciones`.
9. Una **cura** recupera el activo, no efectivo: recupera exactamente su exposición y **no genera filas en `recuperaciones`**. Su LGD **nominal** es cero; la **descontada es positiva**, porque ese activo vuelve meses después y el valor del dinero en el tiempo es una pérdida real. Es una de las cosas que se discuten en C4.

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
| `renta_liquida` | float64 · 18,9% vacío | Renta líquida mensual declarada (CLP). **Puede venir vacía** |
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
| `cupo_aprobado` | float64 · 51,8% vacío | Línea aprobada (CLP). Solo revolventes; vacío en amortizables |
| `garantia_tipo` | str | sin garantía · hipotecaria · prendaria |
| `garantia_valor_tasacion` | float64 · 88,2% vacío | Valor de tasación de la garantía (CLP). Vacío si no hay |
| `segmento` | str · 1,5% vacío | Segmento de riesgo (producto × tramo). Unidad de agrupación del motor |

## `llanquihue_panel` — 719.063 filas

Panel mensual operación × mes con el comportamiento observado. Es el insumo de las matrices de transición, del staging y del roll-forward de la provisión.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `mes` | str | Mes del panel (YYYY-MM) |
| `saldo_insoluto` | float64 | Capital vigente **al cierre** del mes (CLP). Sigue la recursión `saldo_t = saldo_(t-1)·(1+i) + monto_girado − monto_pagado`, así que el pago publicado explica el saldo fila a fila. **Dos convenciones del dataset**: (1) el mes del desembolso no devenga interés — el dinero se entrega y el primer interés corre junto con la primera cuota, al mes siguiente; la cartera legada sí devenga desde su primer mes del panel, porque lleva tiempo cursada. (2) En 90+ DPD el saldo deja de devengar desde el mes siguiente al cruce. Ambas son simplificaciones didácticas, **no** exigencias de IFRS 9: la norma (5.4.1 b) manda aplicar la EIR al costo amortizado **neto de provisión** en activos con deterioro crediticio, que no es lo mismo que interés cero |
| `saldo_utilizado` | float64 · 48,6% vacío | Revolventes: monto usado de la línea. Vacío en amortizables |
| `cupo_vigente` | float64 · 48,6% vacío | Revolventes: línea vigente (se recorta tras deterioro). Vacío en amortizables |
| `dias_mora` | int16 | Días de mora al cierre del mes (DPD) |
| `cuota_pactada` | float64 | Cuota exigible del mes (CLP) |
| `monto_girado` | float64 | Entrada de exposición del mes (CLP): el **desembolso** en el mes de cursado de un amortizable, y el **giro** de la línea en un revolvente. Es el término que cierra la ecuación del saldo cuando la exposición sube por dinero entregado y no por interés. Cero el resto de los meses de un amortizable |
| `monto_pagado` | float64 | Monto efectivamente pagado en el mes (CLP). En el mes de la baja incluye el pago que extingue la deuda. **Durante el workout es cero**: el efectivo de un crédito en cobranza vive en `recuperaciones`, y publicarlo en los dos lados lo contaría dos veces |
| `marca_default` | bool | Verdadero si la operación está en 90+ DPD ese mes |
| `marca_forbearance` | bool | Renegociación o reprogramación. **La marca es persistente**: sigue encendida aunque el pago se normalice (criterio cualitativo de SICR) |
| `en_watchlist` | bool | Alerta interna de seguimiento del mes. Se enciende **antes** que la mora |
| `motivo_baja` | str · 99,0% vacío | Causa por la que la operación **deja de existir** después de este mes: `prepago` · `vencimiento` · `castigo` · `resolución de workout`. Vacío mientras sigue vigente. En el mes de la baja el saldo queda en cero y, si la deuda se extingue pagando, `monto_pagado` recoge el pago final: **la exposición nunca desaparece sin causa observable** |

## `llanquihue_flujos` — 752.334 filas

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

## `llanquihue_recuperaciones` — 6.894 filas

Eventos de flujo del proceso de recuperación (workout) de las operaciones que entraron en default. Varias filas por episodio. **El castigo no está acá**: es una baja contable, no un ingreso, y vive en `defaults`.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `n_evento` | int64 | Número de episodio de incumplimiento de esa operación (1, 2, …). Junto con `id_operacion` forma la llave del workout |
| `n_flujo` | int64 | Número correlativo del flujo dentro de su episodio (1, 2, …), en orden de `fecha_flujo`. **`id_operacion` + `n_evento` + `n_flujo` es la llave única de esta tabla**: dos flujos del mismo episodio y el mismo mes son eventos distintos, no un duplicado |
| `fecha_default` | str | Mes de entrada en default (90+ DPD) del episodio |
| `fecha_flujo` | str | Mes en que ocurre el evento de recuperación |
| `monto_recuperado` | int64 | Monto recuperado en el evento (CLP), **nominal**: para la LGD hay que descontarlo a la EIR desde `fecha_default` |
| `tipo` | str | pago voluntario · ejecución de garantía |
| `costos_directos` | int64 | Costos de cobranza asociados al evento (CLP) |
| `fecha_cierre_workout` | str · 17,3% vacío | Mes de cierre del proceso. **Vacío si el workout sigue abierto a la fecha de corte** |
| `marca_cura` | bool | La operación volvió a estar al día de forma sostenida |

## `llanquihue_defaults` — 4.183 filas

Ficha resumen: **un registro por episodio de incumplimiento**, no por operación. Una operación que se cura y vuelve a caer aparece dos veces, con `n_evento` 1 y 2. Es la base natural para estimar LGD.

| Columna | Tipo | Descripción |
|---|---|---|
| `id_operacion` | str | Llave de la operación |
| `n_evento` | int64 | Número de episodio de incumplimiento de esa operación (1, 2, …) |
| `fecha_default` | str | Mes de entrada en default |
| `saldo_al_default` | float64 | Exposición al momento del incumplimiento (CLP). Es el `saldo_insoluto` que el panel publica ese mismo mes |
| `fecha_cierre_workout` | str · 35,4% vacío | Mes de cierre. **Vacío = workout abierto al corte** |
| `marca_cura` | bool | El episodio se cerró porque la operación volvió a estar al día |
| `desenlace` | str | cura · pago voluntario · ejecución de garantía · castigo · abierto |
| `valor_activo_recuperado` | float64 | Saldo con que la operación vuelve a estar sana tras una **cura** (CLP). Una cura recupera el activo —exactamente su exposición— y no genera flujos en `recuperaciones`, así que su LGD nominal es cero. Descontado a la EIR desde la fecha de default **no** lo es: recuperar el mismo peso un año después vale menos. Cero en los demás desenlaces |
| `fecha_castigo` | str · 79,7% vacío | Mes del castigo, si lo hubo. **Ese mes la operación sale del panel**. El castigo se gatilla por **plazo de mora** (CMF Cap. B-2): ~6 meses en consumo sin garantía real, ~36 con garantía real, ~48 en hipotecario. Por eso dentro de esta ventana castigan consumo, tarjetas y líneas, y el hipotecario arrastra su workout sin castigarse |
| `monto_castigado` | float64 · 79,7% vacío | Parte de la exposición dada de baja contablemente en el castigo (CLP): lo que la cobranza no alcanzó a recuperar cuando venció el plazo normativo. Castigo + recuperaciones suman la exposición al incumplimiento |

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