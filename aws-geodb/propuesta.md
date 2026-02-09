# Propuesta Técnica: Infraestructura AWS para Agricultura de Carbono

## 1. Objetivo del Sistema
El objetivo principal es desplegar una infraestructura en la nube (AWS) robusta, geoespacial y escalable, diseñada para el monitoreo evolutivo de la captura de carbono. El sistema permitirá la ingesta mensual de imágenes satelitales (Copernicus/Sentinel-2), el cálculo de índices biofísicos y el almacenamiento seguro a largo plazo, garantizando que los datos sean auditables para la comercialización de créditos de carbono.

---

## 2. Descripción de Microservicios
La arquitectura se basa en servicios desacoplados que separan la capacidad de cómputo del almacenamiento persistente, permitiendo que cada componente escale de forma independiente.

### 2a. Provisión para Visualización (S3 Standard)
* **Función:** Alojar el **último raster procesado de cada parcela** (estimado en 100 KB por archivo para ~27 ha).
* **Beneficio:** Optimizado para baja latencia, permitiendo que visores GIS Web o de escritorio carguen las imágenes de forma fluida mediante URLs firmadas, eliminando la sobrecarga de procesamiento de la base de datos.

### 2b. Repositorio Histórico (S3 Glacier Deep Archive)
* **Función:** Almacenamiento de ultra bajo costo para el histórico mensual de rasters.
* **Latencia de recuperación:** Entre 12 y 48 horas.
* **Uso previsto:** Auditorías anuales o análisis profundo de evolución de carbono.

### 2c. Catálogo Geoespacial (RDS PostGIS t3.small)
* **Función:** Repositorio de polígonos de parcelas, datos vectoriales de índices calculados (NDVI, Carbono, otros) y metadatos.
* **Disponibilidad:** Garantiza acceso inmediato a datos analíticos para generar gráficos y reportes comparativos de evolución temporal.

### 2d. Procesamiento Serverless (AWS Lambda)
* **Función:** Ejecución del código Python para la descarga y análisis raster.
* **Responsabilidad:** El mantenimiento del ciclo de vida del código y la gestión de APIs externas (Copernicus) corre a cargo del equipo técnico del cliente.

---

## 3. Diseño de Arquitectura (Fase MVP)
El MVP inicia con **5 parcelas** piloto, estableciendo una base de datos de 12 rasters por parcela al año.
* **Flujo de Datos:** La Lambda descarga -> Procesa -> Registra vector en PostGIS -> Mueve archivo a Deep Archive.
* **Optimización:** Se mantiene el **último raster de cada parcela** en S3 Standard para visualización inmediata, minimizando costos de transferencia frente a la provisión directa desde la DB.

---

## 4. Escalamiento de Datos e Infraestructura Cloud
El sistema opera bajo el principio de **pago por uso de infraestructura cloud**. Los costos de AWS se ajustan al volumen de datos, sin requerir re-implementación para escalar.

| Tiempo | Parcelas | Impacto en Infraestructura Cloud |
| :--- | :--- | :--- |
| **Inicio (MVP)** | 5 | Consumo mínimo dentro de la capa gratuita de AWS. |
| **6 Meses** | 500 | Escalado lineal en S3; base de datos estable en t3.small. |
| **12 Meses** | 1,500 | ~18,000 registros de índices/año. DB estimada en ~225MB. |

### Análisis de Almacenamiento (Basado en 100 KB/raster):
* **S3 Standard:** Crecimiento horizontal. 1,500 parcelas ocupan ~150 MB de datos "vivos".
* **S3 Deep Archive:** Crecimiento acumulativo. 1,500 parcelas x 12 meses ocupan ~1.8 GB/año. El impacto financiero es marginal debido al bajo costo por GB.

---

## 5. Estructura de Costos

### A. Costos de Implementación (Pago Único)
| Concepto | Descripción | Costo (USD) |
| :--- | :--- | :--- |
| **Configuración IaaS** | Infraestructura S3, Lambdas, VPC y Roles IAM. | $1,000 |
| **Setup de DB** | Configuración RDS PostGIS y esquema de tablas. | $600 |
| **Handover Técnico** | Documentación y transferencia de credenciales. | $400 |
| **TOTAL** | | **$2,000** |

### B. Costos de Mantenimiento Mensual (Estimación AWS)
*Proyección para 1,500 parcelas.*

| Servicio | Detalle | Costo Est. (USD) |
| :--- | :--- | :--- |
| **Cómputo (Lambda)** | Procesamiento + Transferencia de descarga. | $5.00 |
| **Base de Datos (RDS)** | Instancia t3.small + Almacenamiento. | $18.00 |
| **S3 Standard** | Almacenamiento activo + Salida a mapas. | $4.50 |
| **S3 Deep Archive** | Almacenamiento histórico acumulado. | $1.50 |
| **TOTAL MENSUAL** | | **~$29.00** |

---

## 6. Recomendaciones de Gestión y Control de Riesgos

1. **Eficiencia de Infraestructura:** Se recomienda que el cliente realice revisiones periódicas de métricas para validar que los recursos de AWS (sizing de instancia RDS y memoria Lambda) están optimizados para el volumen de datos vigente.
2. **Límites de Ejecución y Alertas:** Se configurará una alerta de presupuesto al alcanzar los **$50 USD** para detectar tempranamente desviaciones de costos por:
    * Pruebas de código intensivas.
    * Errores de ejecuciones simultáneas o bucles.
    * Descargas fallidas o errores en el consumo de APIs externas.
3. **Circuit Breaker:** Se establecerán límites de concurrencia en Lambda para evitar picos de facturación por errores operativos del script Python.