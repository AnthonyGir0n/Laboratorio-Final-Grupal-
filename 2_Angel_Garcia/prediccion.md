# Hipótesis y Predicción de Escalamiento Horizontal

- **Grupo:** 6 (`GRUPO=6` → `10,400` iteraciones SHA-256 por lectura)
- **Integrantes:** Angel Daniel García Mansilla · Anthony Paul Girón Gordon · Julio Abrahan Chet Chamalé · Marcos Emilio Lopez Chamale
- **Momento de registro:** Posterior a las corridas iniciales con 1 instancia (`base1` y `base2`, `18:15–18:20`) y previo a iniciar `api-2` y `api-3`.

---

## 1. Estimaciones Cuantitativas (1 réplica vs. 3 réplicas)

### A. Throughput esperado
- **Valor base (`base1`):** `32.83 lecturas/s` (`33.08/s` reportado por `k6`)
- **Proyección con 3 réplicas:** **`~49 lecturas/s`** *(intervalo estimado: `44 – 55 lecturas/s`, incremento de `~1.5x` en vez de `3x`)*.
- **Justificación técnica:** En `docker-compose.yml`, las instancias no son simétricas: `api-1` y `api-2` disponen de `0.5 CPU` cada una, pero `api-3` está restringida a `0.25 CPU`. Dado que Traefik aplica balanceo *Round Robin* con pesos iguales, asigna exactamente `1/3` del tráfico a cada contenedor sin evaluar su capacidad. Como `api-3` solo puede procesar la mitad que `api-1` (`~16.4 lecturas/s` para las `10,400` iteraciones SHA-256 del Grupo 6), actúa como cuello de botella y fija el tope global del clúster en aproximadamente `3 × 16.4 ≈ 49.2 lecturas/s`.

### B. Latencia mediana (`p50`)
- **Valor base (`base1`):** `320.2 ms`
- **Proyección con 3 réplicas:** **`~50 ms`** *(intervalo estimado: `25 – 100 ms`, reducción drástica)*.
- **Justificación técnica:** Dos tercios (`66.7 %`) de las peticiones caerán en `api-1` o `api-2`, las cuales operarán con holgura de CPU y prácticamente sin encolamiento. Al ordenarse las muestras, el percentil 50 quedará ubicado dentro de ese bloque de respuestas rápidas.

### C. Latencia de cola (`p99`)
- **Valor base (`base1`):** `479.0 ms`
- **Proyección con 3 réplicas:** **`~650 ms`** *(intervalo estimado: `550 – 800 ms`, degradación respecto a 1 réplica)*.
- **Justificación técnica:** Al tratarse de un bucle cerrado con `10` usuarios virtuales (Ley de Little, $L = \lambda \cdot W$), los VUs que son ruteados hacia `api-3` tardan mucho más en liberarse y se represan en dicha instancia. En consecuencia, el `1 %` de solicitudes más lentas (`p99`) corresponderá íntegramente a peticiones encoladas en `api-3`.

---

## 2. Premisas del Modelo

- **Consumo constante de CPU por petición:** En la línea base con `GRUPO=6` (`10,400` iteraciones SHA-256), cada lectura demandó cerca de `0.5 CPU / 32.83 req/s ≈ 15.2 ms` de tiempo efectivo de procesador.
- **Ausencia de saturación externa:** Ni PostgreSQL (`1.0 CPU`) ni Traefik constituyen el factor limitante bajo este volumen de tráfico.
- **Generador de carga sincrónico:** Cada uno de los 10 usuarios virtuales de k6 espera la respuesta HTTP antes de disparar la siguiente petición.

---

## 3. Indicadores Esperados en Monitoreo

| Indicador | Comportamiento previsto durante `escalado` |
| :--- | :--- |
| **Tasa de solicitudes por instancia** | Idéntica en las 3 réplicas (`~16 req/s` en `api-1`, `api-2` y `api-3`). |
| **Uso de CPU (`docker stats`)** | `api-3` saturada al tope de su cuota (`~25 %`), mientras `api-1` y `api-2` trabajan a media capacidad (`~24 %` sobre `50 %` posibles). |
| **Distribución de latencias** | Separación bimodal entre las respuestas rápidas (`api-1`/`api-2`) y las lentas (`api-3`). |
