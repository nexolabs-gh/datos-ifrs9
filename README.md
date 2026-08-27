# Datos del curso — Pérdidas Esperadas IFRS 9 con Python (Academia Bayes)

> ⚠️ **En validación — estos archivos van a cambiar.** Una revisión cruzada detectó
> incoherencias entre tablas (fichas de default sin respaldo en el panel, operaciones castigadas
> que siguen con saldo vivo y calendario futuro). Se está corrigiendo el generador y las carteras
> se van a regenerar. **No construyas trabajo sobre esta versión todavía.**

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
| `panel` | operación × mes | Saldos, mora, pagos, forbearance, watchlist (48 meses) |
| `flujos` | operación × cuota futura | Calendario contractual desde el corte (**solo amortizables**) |
| `recuperaciones` | evento × operación en default | Flujos de workout, costos, tipo |
| `defaults` | operación en default | Exposición al incumplimiento, cierre de workout, cura |
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

© 2026 Academia Bayes · Solo uso educativo.
