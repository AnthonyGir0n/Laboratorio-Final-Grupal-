# Laboratorio Final — Rendimiento, Observabilidad y Sistemas Distribuidos

- **Curso:** Arquitectura de Sistemas II
- **Grupo:** 6
- **Proyecto:** Servicio de Ingesta y Monitoreo Smart City (`smartcity-perf`)

---

## Integrantes del Grupo 6

- **Angel Daniel García Mansilla**
- **Anthony Paul Girón Gordon**
- **Julio Abrahan Chet Chamalé**
- **Marcos Emilio Lopez Chamale**

---

## Resumen del Entorno

Este laboratorio evalúa el comportamiento bajo carga, la observabilidad y la tolerancia a fallos del subsistema de recepción de lecturas de sensores de **Smart City**. El tráfico HTTP entra por **Traefik** (balanceador de carga), se distribuye hacia hasta tres instancias de una API construida con **FastAPI** (`api-1`, `api-2`, `api-3`), y persiste los registros en **PostgreSQL**. La telemetría se recolecta mediante **Prometheus** y se visualiza en **Grafana**, mientras que las pruebas de estrés se ejecutan con **k6** y se procesan con `analizar.py`.

---

## Accesos y Endpoints Locales

- **Balanceador / API (`POST /lecturas`, `GET /salud`, `GET /metrics`):** `http://localhost:8000`
- **Dashboard de Traefik:** `http://localhost:8080`
- **Consola de Prometheus:** `http://localhost:9090`
- **Panel de Grafana:** `http://localhost:3000` *(credenciales: `smartcity` / `smartcity`)*

---

## Comandos para Levantar y Ejecutar Pruebas

### 1. Inicializar contenedores
```powershell
docker compose --profile herramientas pull
docker compose build
docker compose up -d traefik api-1 postgres prometheus grafana
```

### 2. Preparar entorno de análisis en Python
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install pandas matplotlib
```

### 3. Correr pruebas de carga y generar métricas
```powershell
docker compose run --rm k6 run --out csv=/resultados/base1.csv /scripts/carga.js
python analizar.py resultados/base1.csv
```

### 4. Apagar el laboratorio
```powershell
docker compose down
```

---

## Organización del Repositorio

- **`.env`**: Parámetro de configuración del grupo (`GRUPO=6`).
- **`docker-compose.yml`**: Definición de servicios, límites de CPU y reglas de ruteo de Traefik.
- **`prediccion.md`**: Hipótesis previa al escalamiento horizontal a 3 réplicas.
- **`respuestas.md`**: Informe técnico completo (Partes A, B, C, D y preguntas de defensa).
- **`analizar.py`**: Script para cálculo de percentiles, media, desviación estándar e histogramas.
- **`api/`**: Servicio FastAPI (`app.py`), utilidad de bloqueo (`congelar.py`) y `Dockerfile`.
- **`db/init.sql`**: Esquema e índice de la tabla `lecturas` en PostgreSQL.
- **`k6/carga.js`**: Script de prueba de carga (10 usuarios virtuales durante 2 minutos).
- **`prometheus/` y `grafana/`**: Configuración de scrapeo y aprovisionamiento automático del dashboard.
- **`resultados/`**: Archivos CSV crudos, resúmenes estadísticos e histogramas generados.
- **`capturas/`**: Respaldo gráfico de todas las fases experimentales.
"# Laboratorio-Final-Grupal-" 
