# Datos del curso — Pérdidas Esperadas IFRS 9 con Python (Academia Bayes)

Datasets **100% sintéticos** para uso académico del curso. Ninguna persona ni institución real
está representada. Generados por Academia Bayes / Nexo Labs.

| Dataset | Institución ficticia | Uso en el curso |
|---|---|---|
| `aconcagua_*` | Banco Aconcagua | Demostraciones en clase |
| `llanquihue_*` | Banco Llanquihue | Laboratorios y proyecto evaluado |

**La unidad de cálculo del ECL es la operación, no el cliente.** Ocho tablas por cartera:

| Tabla | Grano | Contenido |
|---|---|---|
| `clientes` | deudor | Demografía y renta declarada |
| `cartera` | **operación** | Producto, plazo, tasa contractual, **EIR**, garantía, segmento |
| `panel` | operación × mes | Saldos, mora, pagos, **giros**, forbearance, watchlist y **motivo de baja** (48 meses) |
| `flujos` | operación × cuota futura | Calendario contractual desde el corte (**solo amortizables vigentes**) |
| `recuperaciones` | evento × episodio de default | Flujos de workout, costos, tipo. Llave: `id_operacion` + `n_evento` + `n_flujo` |
| `defaults` | **episodio de default** | Exposición al incumplimiento, cierre de workout, cura, desenlace, activo recuperado y castigo |
| `macro_historico` | mes | Serie observada: desempleo, actividad, TPM, IPC, precios de vivienda |
| `macro_escenarios` | escenario × mes | Tres trayectorias prospectivas con sus ponderaciones |

**Panel**: 2022-07 a 2026-06 (48 meses). **Fecha de corte del cálculo**: 2026-06. Montos en CLP.

Detalle columna por columna en `diccionario_aconcagua.md` y `diccionario_llanquihue.md`.

## Carga rápida en Google Colab

```python
import pandas as pd

BASE = "https://raw.githubusercontent.com/nexolabs-gh/datos-ifrs9/main"
cartera = pd.read_parquet(f"{BASE}/llanquihue_cartera.parquet")
panel   = pd.read_parquet(f"{BASE}/llanquihue_panel.parquet")
```

Cada tabla está en `parquet` (principal) y `csv.gz` (espejo).

## Advertencias de uso

1. La tasa de descuento que exige la norma es `tasa_efectiva_original` (**EIR**), no
   `tasa_interes_anual`.
2. Los **revolventes no tienen calendario contractual**: no aparecen en `flujos`. Su exposición
   futura se construye con un factor de conversión sobre la línea disponible.
3. Hay operaciones con **workout abierto** (`fecha_cierre_workout` vacío). Excluirlas del cálculo
   de LGD sesga el resultado.
4. Una fracción de operaciones **no trae `segmento`** asignado. Un `join` interno las hace
   desaparecer del cálculo sin aviso.
5. `defaults` tiene una fila **por episodio de incumplimiento**, no por operación: la llave es
   `id_operacion` + `n_evento`. Una operación que se cura y vuelve a caer aparece dos veces, con
   su propia exposición y su propio workout. Agrupar solo por `id_operacion` mezcla dos
   incumplimientos distintos.
6. Toda salida del panel está declarada en `motivo_baja` (`prepago`, `vencimiento`, `castigo`,
   `resolución de workout`), con el saldo llevado a cero ese mes. **Ninguna exposición desaparece
   sin causa observable**: es lo que permite cuadrar el roll-forward de la provisión.
7. El **castigo es una baja contable, no una recuperación**: vive en `defaults.fecha_castigo` y
   `monto_castigado`, no como un flujo de `recuperaciones`. Sumarlo como ingreso infla la LGD.
   Se gatilla por **plazo de mora** (CMF Cap. B-2): ~6 meses en consumo sin garantía real, ~36
   con garantía real, ~48 en hipotecario — por eso castigan consumo, tarjetas y líneas, y el
   hipotecario arrastra su workout sin castigarse. El castigo **corta la cobranza**: recuperado +
   castigado suman exactamente la exposición al incumplimiento.
8. Los montos de `recuperaciones` son **nominales**: la LGD workout se calcula descontando cada
   flujo a la EIR desde la fecha de default.
9. Una **cura** no recupera efectivo: recupera el **activo**, y recupera exactamente su
   exposición, así que su LGD es cero y **no genera filas en `recuperaciones`**. El saldo con que
   el crédito vuelve a estar sano está en `defaults.valor_activo_recuperado`.
10. El pago publicado **explica** el saldo, fila a fila y en toda la cartera:
    `saldo_t = saldo_(t-1)·(1+i) + monto_girado − monto_pagado`. Dos convenciones: el **mes del
    desembolso no devenga interés** (el primer interés corre con la primera cuota, al mes
    siguiente; la cartera legada sí devenga desde su primer mes del panel), y en 90+ DPD se
    **suspende el devengo** (tratamiento de Stage 3, desde el mes siguiente al cruce) con el pago
    del mes en cero, porque el efectivo de la cobranza vive en `recuperaciones`.

## Cómo se validan estos datos

Cada cartera pasa **95 criterios de aceptación** contra los archivos publicados —no contra la
memoria del generador—: coherencia entre tablas, roll-forward mensual del saldo, signos y cotas de
cada movimiento bruto, nulidad validada celda a celda, claves foráneas, reconciliación completa
del calendario de pagos, castigos que respetan el plazo normativo de su producto, ausencia de masa
artificial en las distribuciones y reproducibilidad **bit a bit** desde la semilla.

Y los criterios críticos se validan **por mutación**: se inyecta el defecto que cada uno dice
detectar y se exige que ese criterio falle. Un control que no puede fallar no cuenta como verde.
Son 64 mutaciones dirigidas, no una por cada criterio.

© 2026 Academia Bayes · Solo uso educativo.
