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
| `manifiesto.json` | — | — | corte, T0, ventana PIT y la huella (sha256) de cada archivo publicado |

**Ojo con `curva_pd`:** la edad cuenta desde que se otorgó el crédito, no desde el corte. La PD de una operación viva
para los próximos meses sale de los hazards **desde su edad actual + 1** en adelante, no de la fila 12 de la tabla.

`aconcagua` = cartera A (clases) · `llanquihue` = cartera B (laboratorios evaluados).
Datos 100 % simulados: no corresponden a ninguna institución real.
