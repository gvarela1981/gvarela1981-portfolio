# Presupuesto: Infraestructura AWS para Agricultura de Carbono

## 1. Objetivo del Sistema
El resultado esperado es desplegar una infraestructura en la nube (AWS) robusta, geoespacial y escalable, diseñada para el monitoreo de la captura de carbono a trevez del tiempo. Esta infraestructura permitirá al sistema la ingesta mensual de imágenes satelitales (Copernicus/Sentinel-2), el cálculo de índices biofísicos y el almacenamiento a largo plazo, garantizando que los datos sean auditables para la comercialización de créditos de carbono.

---

## 2. Detalle de Arquitectura
La arquitectura se basa en **microservicios desacoplados** que separan la capacidad de cómputo del almacenamiento, permitiendo que cada componente escale de forma independiente.

### A. Provisión para Visualización (Storage S3 Standard)
* **Función:** Alojar el **último raster procesado de cada parcela** (estimado en 100 KB por archivo para ~27 ha).
* **Beneficio:** Optimizado para disponibilidad inmediata, permitiendo que visores GIS Web o de escritorio (como QGIS) carguen las imágenes de forma fluida mediante URLs firmadas, evitando sobrecargar el procesamiento de la base de datos.

### B. Repositorio Histórico (Storage S3 Glacier Deep Archive)
* **Función:** Almacenamiento de ultra bajo costo para el histórico mensual de rasters.
* **Demora en la recuperación:** La recuperación de los archivos para su utilización demora entre 12 y 48 horas.
* **Uso recomendado:** Este storage se recomienda para backup a bajo costo de raster ya procesados, evitando la necesidad de una segunda descarga desde el programa Copernicus. De esta forma se pueden realizar auditorías anuales, reprocesamientos o realizar análisis mas específicos no realizados inicialmente. El bajo costo implica una demora en la recuperacion de los archivos desde el storage de backup. 

### C. Catálogo Geoespacial (GeoDB en un bucket RDS PostGIS t3.small)
* **Función:** Catalogo de polígonos de parcelas y los datos asociados a las mismas. Los datos serán de tipo espaciales, vectoriales (NDVI, Carbono, otros) y metadatos de los archivos raster.
* **Rutas de Acceso:** El catálogo incluye de forma explícita la ruta de descarga del GeoTIFF en el **storage de acceso inmediato** (S3 Standard) y la ruta de acceso al GeoTIFF en el **storage de backup** (S3 Glacier Deep Archive).
* **Disponibilidad:** Proporciona acceso inmediato a datos analíticos para generar gráficos y reportes comparativos de evolución temporal, verificar la existencia de los archivos raster para una parcela especifica, acceder a sus metadatos y obtener las rutas de acceso a los raster para el procesamiento de los mismos. El acceso será permanente sin experimentar demoras.

### D. Procesamiento Serverless (AWS Lambda)
* **Función:** Ejecución del código Python para la descarga y análisis raster.
* **Alcance:** La responsabilidad y el mantenimiento del ciclo de vida del código y la gestión de APIs externas (Copernicus) corre a cargo del equipo técnico del cliente. La infraestructura soporta configuraciones de hasta 10GB de RAM si el geoproceso lo requiere.

---

## 3. Diseño de Arquitectura (Fase MVP)
El MVP inicia con **5 parcelas** piloto, estableciendo una base de datos de 12 rasters por parcela al año.
* **Flujo de Datos:** La Lambda descarga -> Procesa -> Registra vector en PostGIS -> Mueve archivo a Deep Archive.
* **Optimización:** Se mantiene el **último raster de cada parcela** en el storage S3 Standard para visualización inmediata, minimizando costos de transferencia frente a la provisión directa desde la DB.

La arquitectura está diseñada para crecer junto con el volumen de negocio sin necesidad de re-ingeniería:

1. **Fase Inicial (Hasta 10.000 parcelas):** Se mantiene la configuración actual de **~€45,61/mes**.
2. **Fase de Expansión (Análisis masivo):** Si la complejidad de los cálculos espaciales aumenta o se requiere disponibilidad crítica (99.99%), se recomienda el escalado a **Multi-AZ** y duplicar la RAM (pasando a servicio RDS `t3.medium`), lo que situaría el costo operativo en torno a los **€95/mes**.

---

## 4. Escalamiento de Datos e Infraestructura Cloud
El sistema opera bajo el principio de **pago por uso de infraestructura cloud**. Los costos de AWS se ajustan al volumen de datos, sin requerir re-implementación para escalar.

| Tiempo | Parcelas | Impacto en Infraestructura Cloud |
| :--- | :--- | :--- |
| **Inicio (MVP)** | 5 | Consumo mínimo dentro de la capa gratuita de AWS. |
| **6 Meses** | 500 | Escalado lineal en storage S3. La Base de Datos permance del mismo tamaño en t3.small. |
| **12 Meses** | 1,500 | ~18,000 registros de índices/año. DB estimada en ~225MB. |

### Análisis de Almacenamiento (Basado en 100 KB/raster):
* **Storage S3 Standard:** Crecimiento horizontal. 1,500 parcelas ocupan ~150 MB de datos "vivos", de acceso inmediato. El costo de almacenamiento es $0,0245 USD por GB mientras el espacio total se mantenga dentro de los 50TB/mes.
* **Storage S3 Deep Archive:** Crecimiento acumulativo. 1,500 parcelas x 12 meses ocupan \~1.8 GB/año. El impacto financiero debería ser marginal debido al bajo costo por GB (\~$0.0018 USD de almacenamiento puro).

*Los costos están expresados en dólares norteamericanos tomados el 09/02/2026 de la documentación oficial de AWS para la zona Frankfurt*

*Los costos de los storage S3 se han proyectado basándose en las tarifas públicas de AWS eligiendo la región eu-central-1:*
*[Detalle de precios Amazon Storage S3 - https://aws.amazon.com/es/s3/pricing/?nc=sn&loc=4](https://aws.amazon.com/es/s3/pricing/?nc=sn&loc=4)*

*Los costos de la base de datos se han proyectado basándose en el cálculo realizado con la herramienta oficial de AWS, eligiendo la región eu-central-1:*
*[Detalle de precios Amazon RDS para PostgreSQL - (https://calculator.aws/#/createCalculator/RDSPostgreSQL)](https://calculator.aws/#/createCalculator/RDSPostgreSQL)*

---

## 5. Cronograma de Implementación (Roadmap)
Se estima un tiempo total de **4 semanas** para la entrega de la infraestructura operativa y su documentación. La entrega será progresiva, con entregables cada semana sujetos a la aprobación del cliente.

### Fase 1: Setup de Entorno y Control de Gastos (Semana 1)
* Configuración de cuenta AWS, VPC y subredes.
* **Activación de CloudWatch Alarms y AWS Budgets (Alertas de presupuesto).**
* Creación de políticas IAM y roles de ejecución.
* Configuración de buckets S3 (Standard y Glacier).

### Fase 2: Despliegue de Base de Datos y Lógica (Semana 2)
* Instanciación de RDS PostGIS t3.small.
* Creación del esquema de tablas (Catálogo, Índices e Inventario).
* Configuración de las funciones Lambda (Entorno Python y librerías).
* **Configuración de límites de concurrencia en Lambda** para evitar multiples ejecuciones simultáneas y limitar la aparición de costos sorpresa.

*La preparación del entorno python y la estructura de la base de datos esta sujeta la confirmación del cliente sobre las librerías necesarias en el entorno, y la aprobación del esquema de las tablas.*

### Fase 3: Integración y Pruebas (Semana 3)
* Pruebas de conectividad con APIs de Copernicus/Sentinel.
* Pruebas de conectividad con Storage S3 Standard y S3 Glacier Deep Archive
* Validación de flujo de datos (Standard -> Glacier).
* Testing de recuperación de datos raster desde el backup y monitoreo activo de costos.

*El test se realizará de manera conjunta con el equipo técnico del cliente.*

### Fase 4: Optimización y Handover (Semana 4)
* Refinamiento de métricas de rendimiento.
* Documentación técnica de arquitectura.
* Sesión de transferencia de conocimiento (Handover técnico).

---

## 6. Estructura de Costos

### A. Costos de Implementación (Pago Único)
| Concepto | Descripción | Costo (EUR) |
| :--- | :--- | :--- |
| **Configuración SaaS** | Despliegue de S3, Lambdas, VPC y Roles de Seguridad IAM. | €1,800 |
| **Setup de DB** | Configuración RDS PostGIS, implementación de esquema de tablas aprobado. | €1200 |
| **Handover Técnico** | Documentación técnia, documentación operativa de Backup/Restore de la geoDB y transferencia de credenciales. | €800 |
| **TOTAL** | | **€3,800** |

> **Compromiso de Agilidad y Plazos:** > Para el cumplimiento del cronograma de 4 semanas la estrategia de trabajo prioriza la **efectividad en los hitos de validación conjunta**. Se busca minimizar las demoras mediante una comunicación directa en las definiciones y en las sesiones conjuntas de pruebas técnicas. Se buscará que el trabajo conjunto sea ágil y no impacte el plazo de entrega final.

### B. Costos de Mantenimiento Mensual (Estimación AWS - Frankfurt)
*La facturación de estos costos fijos estarán a cargo de Amazon y estan sujetos a cambios por parte del proveedor. La estimación se realizó de acuerdo a los costos de cada componente informados en la documentación de AWS para las instancias alojadas en la Unión Europea (Franckfurt).* 

*La proyección estimada para el escenario de 1,500 parcelas, calculada con un tamaño promedio de **GeoTIFF de 100 KB**.*


| Servicio | Detalle del Recurso | Costo Est. (EUR) |
| :--- | :--- | :--- |
| **Cómputo (RDS)** | Instancia db.t3.small (Single-AZ) | €30,36 |
| **Almacenamiento (RDS)** | 40 GB SSD de Uso General (GP3) | €5,10 |
| **Storage (S3)** | Capas Standard + Glacier Deep Archive | €5,50 |
| **Cómputo (Lambda)** | Procesamiento mensual de imágenes | €4,65 |
| **TOTAL MENSUAL ESTIMADO** | | **~€45,61 / mes** |

*Los costos de AWS se calcularon el día 09/02/2026 utilizando la calculadora provista por AWS para la zona Frankfurt y están sujetos a cambios por parte del proveedor.*
*Acceso a la calculadora de costos: [Configurador de Costos RDS PostgreSQL](https://calculator.aws/#/createCalculator/RDSPostgreSQL)*
*Se adjunta documento adicional con la configuración utilizada para esta estimacion de costo RDS-Postgis-Frankfurt.pdf*

---

## 7. Recomendaciones de Gestión y Control de Riesgos

1. **Eficiencia de Infraestructura:** Se recomienda que el cliente realice revisiones periódicas de métricas para validar que los recursos de AWS (sizing de instancia RDS, tamaño de los Storage y memoria Lambda) están optimizados para el volumen de datos real, buscando evitar que los cambios en el volumen de datos procesados y almacenados mantenga la eficiencia de los recursos y no incremente el impacto de los costos fijos.
2. **Límites de Ejecución y Alertas:** Para proteger la salud financiera, se configurará una alerta de presupuesto al alcanzar los €50 USD. El objetivo es detectar de forma temprana desviaciones de costos asociados a:
    * Pruebas de código intensivas.
    * Ejecuciones simultáneas accidentales o bucles.
    * Descargas fallidas o errores en el consumo de APIs externas.
3. **Circuit Breaker:** Se establecerán límites de concurrencia en Lambda para evitar picos de facturación por errores operativos o pruebas masivas del script Python.

## 8. Forma de pago
La forma de pago sera a convenir.