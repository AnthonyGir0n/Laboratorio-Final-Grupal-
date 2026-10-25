# Informe de Resultados y Defensa — Laboratorio Final: Medir antes de afirmar

- **Curso:** Arquitectura de Sistemas II
- **Grupo:** 6 (`GRUPO=6` configurado en `.env`, equivalente a `8000 + 6 × 400 = 10 400` iteraciones SHA-256 por lectura)
- **Integrantes:**
  - Angel Daniel García Mansilla
  - Anthony Paul Girón Gordon
  - Julio Abrahan Chet Chamalé
  - Marcos Emilio Lopez Chamale

> **Nota sobre huso horario:** Todos los registros horarios del documento se expresan en hora local de Guatemala (`UTC-6`). La interfaz web de Prometheus grafica en `UTC` (`+6 horas`, es decir `00:14–00:45 UTC` del día siguiente).

---

## Bitácora General de Ejecución y Verificación Inicial

Antes de iniciar las mediciones formales, se validó el estado de los contenedores en PowerShell (`18:13:52`), el endpoint `/salud`, la inserción de prueba en `/lecturas` (verificando `"iteraciones_hash": 10400`), el registro en Traefik (`1 servidor activo` en `http://172.21.0.5:8000`), los targets en Prometheus, el tablero de Grafana y una corrida rápida de humo (`10 s`, `327` lecturas, `32.41 req/s`) con 100 % de éxito (`capturas/01_*`).

- **Prueba de humo (`10 s`):** `18:14:49` a `18:15:03` — Evidencia: `capturas/01_verificacion_k6_humo.png`
- **Fase de calentamiento (`30 s`, no contabilizada):** `18:15:12` a `18:15:45`
- **Corrida `base1` (`2 min`, 1 réplica):** `18:15:45` a `18:17:49` — Evidencia: `capturas/02_k6_resumen_base1.png`
- **Corrida `base2` (`2 min`, 1 réplica):** `18:18:04` a `18:20:06` — Evidencia: `capturas/02_k6_resumen_base2.png`

---

## Bloque A — Análisis Estadístico del Rendimiento (1 Réplica)

### 1. Cuadro comparativo de las corridas base (`base1` vs. `base2`)

Datos procesados mediante `python analizar.py resultados/base1.csv` y `python analizar.py resultados/base2.csv` (con `0` errores HTTP en ambos casos: `3973` y `3987` peticiones exitosas respectivamente). Los gráficos de distribución se encuentran en `resultados/base1_histograma.png` y `resultados/base2_histograma.png`.

| Métrica Evaluada | Corrida `base1` | Corrida `base2` | Variación Relativa `(base2 − base1) / base1` |
| :--- | :---: | :---: | :---: |
| **Throughput (lecturas/s)** | `32.83` | `32.95` | `+0.37 %` |
| **Latencia Media (ms)** | `303.9` | `299.7` | `−1.38 %` |
| **Mediana / p50 (ms)** | `320.2` | `314.1` | `−1.91 %` |
| **Percentil 90 / p90 (ms)** | `429.4` | `421.1` | `−1.93 %` |
| **Percentil 95 / p95 (ms)** | `446.9` | `444.2` | `−0.60 %` |
| **Percentil 99 / p99 (ms)** | `479.0` | `481.4` | `+0.50 %` |
| **Latencia Máxima (ms)** | `645.0` | `598.0` | `−7.29 %` |
| **Desviación Estándar (ms)** | `82.4` | `82.5` | `+0.12 %` |

### 2. Discusión e interpretación estadística

- **Contraste del `p95` entre k6 y `analizar.py`:**
  Los valores coinciden de forma exacta: la salida por consola de k6 marca `p(95)=446.90ms` (`capturas/02_k6_resumen_base1.png`) y `analizar.py` arroja `446.9 ms`. Esto se debe a que ambos operan sobre el mismo conjunto de `3973` muestras; cualquier diferencia marginal en centésimas proviene únicamente del redondeo a un decimal y del método de interpolación lineal entre observaciones adyacentes.

- **Morfología del histograma en `base1`:**
  Lejos de presentar una campana unimodal o simétrica, el histograma (`resultados/base1_histograma.png`) exhibe **múltiples crestas separadas por intervalos regulares de ~100 ms** (alrededor de `120`, `235`, `335`, `435` y `530 ms`). Además, ocurre un fenómeno particular: **el promedio (`303.9 ms`) es inferior a la mediana (`320.2 ms`)**.
  *Explicación física:* El contenedor `api-1` tiene configurado `cpus: 0.5`. El planificador CFS de Docker asigna cuotas en ventanas de `100 ms`, permitiendo `50 ms` de ejecución seguidos de `50 ms` de *throttling* (pausa forzosa). Con `GRUPO=6` (`10,400` iteraciones SHA-256 por solicitud), cada petición que encuentra la cuota agotada sufre ciclos discretos de enfriamiento de `100 ms`, agrupando las duraciones en "peldaños". El clúster inferior alrededor de `235 ms` jala la media hacia abajo (`303.9 ms`), demostrando que el promedio oculta la estructura escalonada de los tiempos de respuesta.

- **Traducción del `p95` a lenguaje de negocio:**
  > *"De cada 20 lecturas enviadas por los sensores, 19 se procesan y confirman en 446.9 ms o menos, mientras que solo 1 supera ese tiempo."*

- **Verificación mediante la Ley de Little ($L = \lambda \cdot W$):**
  Tomando $\lambda = 32.83\text{ lecturas/s}$ y $W = 0.3039\text{ s}$, la concurrencia media calculada es:
  $$L = 32.83 \times 0.3039 = 9.98 \text{ solicitudes en vuelo}$$
  Esto representa el `99.8 %` de los `10` usuarios virtuales configurados. La brecha restante de `0.02` se explica por dos factores:
  1. `analizar.py` calcula los segundos como `fin - inicio + 1 = 121 s` (enteros), mientras que la duración efectiva en k6 fue de `120.1 s` ($\lambda_{\text{k6}} = 33.08\text{ req/s}$).
  2. El tiempo de cliente entre una respuesta y la siguiente solicitud (armar el payload JSON, validar el `check` e iniciar la siguiente iteración): en el CSV, `iteration_duration` promedia `304.24 ms` frente a `303.90 ms` de `http_req_duration` (`~0.34 ms` fuera de la red/servidor).

- **Ruido experimental y umbral de significancia:**
  Al repetir exactamente el mismo experimento (`base1` vs. `base2`), el throughput fluctuó `0.37 %`, la media `1.38 %` y el `p95` varió `0.60 %` (mientras que el máximo osciló `7.29 %` por depender de un único valor extremo). En consecuencia, cualquier cambio de código o infraestructura que produzca una ganancia inferior a `~1.4 %` cae dentro del margen de variabilidad natural del entorno.

---

## Bloque B — Observabilidad con Prometheus y Grafana

### 1. Inspección de `/metrics` en la API
Al consultar `curl.exe -s http://localhost:8000/metrics | Select-String smartcity_solicitud` (`capturas/03_metricas_crudas.png`), el histograma `smartcity_solicitud_duracion_segundos_bucket` expone las cotas superiores (`le`): `0.1`, `0.5`, `1.0`, `2.5`, `5.0`, `10.0` y `+Inf`. Estas corresponden exactamente a la tupla `buckets` declarada en `api/app.py`, sumando el corte `+Inf` que Prometheus añade por defecto para capturar el total acumulado (`_count`).

### 2. Evaluación de consultas PromQL

- **Serie acumulada (`smartcity_solicitud_duracion_segundos_count`):**
  Visible en `capturas/03_prometheus_contador_crudo.png`. Al ser un contador monótono creciente, la gráfica únicamente sube (alcanzando `9228` peticiones acumuladas tras la prueba de humo, calentamientos, `base1` y `base2`) o se mantiene horizontal cuando no hay carga. Por sí sola no permite leer el caudal instantáneo de peticiones por segundo.

- **Caudal instantáneo (`rate(smartcity_solicitud_duracion_segundos_count[30s])`):**
  Visible en `capturas/03_prometheus_rate.png`. Durante la meseta de `base1` y `base2`, la tasa mediana se ubicó en **`32.85 – 32.95 lecturas/s`**, coincidiendo con una desviación menor al `1 %` respecto a `analizar.py` (`32.83` y `32.95`) y k6 (`33.08` y `33.20`).

- **Cálculo del `p95` en Prometheus al cierre de `base1` (`18:17:45` local / `00:17:45 UTC`):**
  ```promql
  histogram_quantile(0.95, sum by (le, instance) (rate(smartcity_solicitud_duracion_segundos_bucket[2m]))) * 1000
  ```
  El valor devuelto por Prometheus es **`474.2 ms`** (`capturas/03_prometheus_p95_base1.png`).

### 3. Discrepancia entre el percentil real y la aproximación de Prometheus

| Método de Cálculo | `p95` en `base1` (ms) | Error Relativo frente al CSV |
| :--- | :---: | :---: |
| **Cálculo exacto (`analizar.py` / k6)** | `446.9` | — |
| **Interpolación por buckets (Prometheus)** | `474.2` | **`+6.1 %`** |

- **¿Por qué sobreestima a `474.2 ms`?**
  Revisando los conteos por bucket en `capturas/03_metricas_crudas.png` (de `9228` lecturas acumuladas, `194` entraron en `le="0.1"` y `9221` en `le="0.5"`), el **`97.8 %` de las muestras cayó dentro de un único bucket muy ancho (`100 ms` a `500 ms`)**. Como Prometheus desconoce la distribución interna dentro de ese rango, aplica **interpolación lineal asumiendo distribución uniforme** entre `0.1 s` y `0.5 s`:
  $$\text{Posición objetivo (95 \%)} = 0.95 \times 3973 = 3774.4$$
  $$\text{Fracción en el bucket } (0.1, 0.5] \approx 0.9355 \implies p95_{\text{Prom}} = 0.1 + 0.9355 \times (0.5 - 0.1) = 0.4742\text{ s} = 474.2\text{ ms}$$
  Dado que las peticiones reales del Grupo 6 se concentran entre `200` y `460 ms`, la hipótesis de uniformidad dentro del bucket `[100 ms, 500 ms]` empuja artificialmente el percentil hacia `474.2 ms`.

- **Diferencia entre medición en el cliente (k6) y en el servidor (API):**
  El middleware de FastAPI cronometra únicamente desde que la corrutina entra a `medir()` hasta que retorna la respuesta. No contabiliza el salto de red cliente-Traefik-API, el enrutamiento en Traefik, ni la cola de sockets en `uvicorn`. Comparando las medias en `base1`: el servidor reportó `~281.6 ms` (`2598.71 s / 9228 req` en `/metrics`) frente a los `303.9 ms` medidos por k6, lo que revela **`~22.3 ms` de sobrecarga externa al código de la API**.

- **¿Cuál dato presentar a gerencia?**
  El `p95` de **k6 / `analizar.py` (`446.9 ms`)**, ya que refleja el tiempo extremo a extremo experimentado por el cliente y se calcula sobre cada observación individual sin distorsión de buckets. Prometheus debe emplearse para monitoreo continuo de tendencias y disparo de alertas.

- **Cómo corregir la precisión en `api/app.py` y su costo:**
  Redefinir la tupla `buckets` con mayor granularidad entre `100 ms` y `1000 ms`, por ejemplo:
  `buckets=(0.05, 0.1, 0.15, 0.2, 0.25, 0.3, 0.35, 0.4, 0.45, 0.5, 0.75, 1.0, 2.5, 5.0, 10.0)`.
  *Trade-off:* Cada límite `le` adicional multiplica la cardinalidad de series temporales en Prometheus (por cada combinación de `instance`, `ruta` y `estado`), incrementando el consumo de RAM, disco y ancho de banda de scrapeo.

### 4. Auditoría del Dashboard de Grafana durante `base1` y `base2` (`capturas/03_grafana_base1_base2.png`)

| Panel del Dashboard | Categoría RED | Valor en `base1` (`18:16–18:17`) | Diagnóstico de Confiabilidad |
| :--- | :---: | :---: | :--- |
| **¿Cuántas lecturas por segundo acepta el sistema?** | **R** (*Rate*) | `~32.9 req/s` | **Correcto.** Concuerda con el throughput real (`32.83`). |
| **¿Cómo se reparte el tráfico entre réplicas?** | **R** (*Rate*) | `api-1`: `~32.9 req/s` | **Correcto.** Al haber 1 sola instancia, absorbe el 100 % del flujo. |
| **p95 del sistema (ms)** | **D** (*Duration*) | `~474.2 ms` | **Incorrecto.** Presenta sesgo por anchura de buckets (`+6.1 %`) y además usa `avg(histogram_quantile(... by (le, instance)))`, promediando percentiles (fallo oculto cuando solo hay 1 réplica). |
| **Latencia p50 y p99 del sistema (ms)** | **D** (*Duration*) | `p50 ≈ 298.0 ms` / `p99 ≈ 495.0 ms` | **Aproximado.** Agrega bien por `le`, pero la interpolación lineal en el bucket `[0.1, 0.5]` desvía el `p50` (vs `320.2 ms`) y el `p99` (vs `479.0 ms`). |
| **¿Qué porcentaje de lecturas falla en la API?** | **E** (*Errors*) | `No data` | **Deficiente.** Al no existir muestras con `estado!="201"`, la división retorna vacío (`No data`) en lugar de `0 %`, y además ignora errores previos a la API. |
| **Traefik: solicitudes por segundo según código HTTP** | **R / E** | `HTTP 201 ≈ 33.0 req/s` (`200 ≈ 0`) | **Correcto.** Refleja el caudal en el proxy y permite ver códigos de error si ocurrieran. |
| **¿Qué réplicas reporta Prometheus como activas?** | *(Salud de scrapeo, fuera de RED)* | `api-1 = 1` | **Parcial.** Solo indica si Prometheus logra leer `/metrics`, no si la réplica procesa tráfico real. |

---
