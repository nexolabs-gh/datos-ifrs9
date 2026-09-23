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

| Intermedio | Sale de → entra a | Contenido |
|---|---|---|
| `vivas` | Clase 1 → todas | Operaciones vivas al corte (2026-06): su fila del panel + condiciones de la cartera, edad y cosecha |

`aconcagua` = cartera A (clases) · `llanquihue` = cartera B (laboratorios evaluados).
Datos 100 % simulados: no corresponden a ninguna institución real.
