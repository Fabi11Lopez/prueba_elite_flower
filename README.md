# 🚕 Data Lake Analytics - NYC TLC (HVFHS)

## 📌 1. Objetivo del Proyecto
Este repositorio contiene el diseño y código fuente para la implementación de un Data Lake en AWS utilizando Databricks. El objetivo es procesar y analizar viajes de alto volumen de plataformas como Uber y Lyft (dataset HVFHS de NYC TLC para enero de 2025), garantizando escalabilidad, calidad de datos y buenas prácticas de ingeniería.


## 🏗️ 2. Arquitectura del Data Lake en AWS
La solución está diseñada bajo una **Arquitectura Medallón**, separando el almacenamiento en Amazon S3 y el cómputo distribuido en Databricks (PySpark).


* **Capa Bronze (Ingesta):** Almacenamiento de datos crudos (HVFHS y Catálogo de Zonas) tal cual provienen de la fuente. Funciona como un registro histórico inmutable de tipo *append-only*.
* **Capa Silver (Transformación):** Limpieza, normalización de timestamps, casteo estricto de tipos de datos (decimales para métricas financieras) y enriquecimiento espacial mediante *JOIN* con el catálogo de zonas.
* **Capa Gold (Presentación):** Modelado dimensional y agregaciones diarias para disponibilizar los KPIs de negocio requeridos listos para el consumo de herramientas de BI (como Amazon QuickSight o Athena).

### 💾 Formato de Almacenamiento y Particionado
* **Formato:** Se utiliza **Delta Lake** en las capas Silver y Gold por su soporte nativo de transacciones ACID, evolución de esquemas y capacidades de *Time Travel*.
* **Particionado:** La tabla Silver está particionada lógicamente por la columna derivada `pickup_date`. Esto optimiza drásticamente los tiempos de lectura y reduce costos computacionales al evitar escaneos completos de la tabla en consultas analíticas diarias.

---

## ⚙️ 3. Desarrollo de ETLs y Reglas de Calidad
El pipeline implementa las siguientes transformaciones críticas:
1.  **Detección y corrección de tipos lógicos:** Los campos `base_passenger_fare`, `tolls`, `sales_tax` y demás *fees* se convierten a `decimal(10,2)` para garantizar precisión financiera.
2.  **Normalización Temporal:** Conversión de strings a `timestamp` e inferencia de husos horarios para las fechas de *pickup* y *dropoff*.
3.  **Filtros de Integridad:** Se descartan viajes sin zona de origen (`PULocationID` nulo) y viajes ilógicos (donde la fecha de fin es menor a la de inicio).
4.  **Generación de KPIs (Gold):** Cálculo preciso de viajes promedio por hora, ingresos totales, tiempo y distancia promedio por día.

---

## 🛡️ 4. Estrategia de Incrementalidad, Fallas y Reprocesos
Para asegurar la fiabilidad del Data Lake ante escenarios de producción, se establecen las siguientes directrices:

* **Idempotencia mediante Partition Overwrite:** El procesamiento Silver implementa un reemplazo dinámico de particiones (`replaceWhere` en Delta Lake) limitado al periodo procesado (Enero 2025). Esto permite re-ejecutar el pipeline ante fallas sin duplicar información histórica.
* **Manejo de Errores en Datos:** La ingesta Silver actúa como un escudo. Las fechas o formatos inválidos que PySpark no puede castear se evalúan, y los duplicados lógicos (misma licencia, fecha y zona) son removidos antes de la escritura mediante `dropDuplicates()`.

### 🚀 Optimización Avanzada: Manejo de Cambios (CDC Conceptual)
Como evolución lógica de la arquitectura propuesta, el manejo avanzado de eventos se abordará de la siguiente manera:
* **Upserts (Merge):** Transición de `overwrite` a comandos `MERGE INTO` de Delta Lake para actualizar eficientemente registros de viajes corregidos de forma asíncrona por las plataformas.
* **Llegadas Tardías:** El particionado por `pickup_date` garantiza que los datos rezagados se inserten en su partición histórica correcta sin alterar el job del día actual.
* **Soft Deletes:** En lugar de borrados físicos por viajes invalidados, se propone una bandera booleana (`is_active = false`) en la capa Silver para mantener trazabilidad.
* **Control de Snapshots:** Aprovechamiento nativo del log de transacciones de Delta Lake para consultar estados pasados (Time Travel) o realizar *Rollbacks*

### 🌍 Arquitectura y Separación de Entornos

Para garantizar el ciclo de vida del desarrollo de software (SDLC) y la integridad de los datos, la arquitectura implementa una **separación lógica** mediante prefijos en S3, orquestada por variables de configuración dinámicas:

* **🧪 Desarrollo (`dev`)**
    * **Ruta S3:** `s3://datalake/dev/...`
    * **Uso:** Espacio de trabajo transitorio ("Sandbox") para pruebas unitarias, validación de código y experimentación. Los datos en este entorno son volátiles y pueden ser reiniciados en cualquier momento.
    * **Rama Git:** `desarrollo`.

* **🚀 Producción (`prod`)**
    * **Ruta S3:** `s3://datalake/prod/...`
    * **Uso:** Fuente única de verdad. Contiene datos inmutables y fiables que alimentan los dashboards ejecutivos. Solo el pipeline automatizado (Job) tiene permisos de escritura en este prefijo.
    * **Rama Git:** `main`.


### 🚧 Gestión de Entornos y Seguridad (Alternativa a Unity Catalog)

Debido a las restricciones técnicas del entorno **Databricks Community Edition** (ausencia de Unity Catalog y persistencia limitada de estados entre contextos de ejecución), se optó por definir las variables de configuración de ruta y credenciales de manera explícita en el código (`00_config`). Esta decisión se tomó para garantizar la **reproducibilidad inmediata** de la Prueba de Concepto (PoC) por parte de los evaluadores, eliminando dependencias de configuración externas.

**Propuesta de Arquitectura para Entorno Productivo (AWS):**

En un despliegue empresarial real, esta configuración "hardcoded" se reemplazaría por una arquitectura de **Seguridad y Configuración Dinámica** basada en los siguientes componentes:

1.  **Gestión de Secretos (Databricks Secrets / AWS Secrets Manager):**
    * Las credenciales nunca residirían en el código. Se utilizaría la inyección de secretos en tiempo de ejecución:
        `password = dbutils.secrets.get(scope="production", key="aws-secret-key")`
    * Esto permite rotar las claves en AWS sin detener ni modificar los pipelines.

2.  **Variables de Entorno del Clúster (Spark Config):**
    * El entorno (`DEV`, `QA`, `PROD`) se definiría a nivel de infraestructura del clúster (ej: `spark.env = "PROD"`).
    * El código leería esta variable al iniciar y seleccionaría automáticamente la ruta S3 correspondiente (`s3://datalake/{env}/...`), evitando errores humanos de escritura en producción.

3.  **Seguridad Basada en Roles (IAM Instance Profiles):**
    * Se implementaría el principio de **Mínimo Privilegio**. El clúster de Producción tendría un Rol de IAM asociado que solo permite escritura en el bucket `s3://.../prod`, mientras que el clúster de Desarrollo tendría acceso restringido únicamente a `s3://.../dev`.