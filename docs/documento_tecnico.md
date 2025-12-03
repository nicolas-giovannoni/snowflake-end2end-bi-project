# DOCUMENTO TÉCNICO – Snowflake End-to-End BI Project


## 1. Introducción
   
Este documento describe el desarrollo técnico del proyecto “Snowflake End-to-End BI Project”,
cuyo objetivo fue construir un pipeline end-to-end para analizar la performance de agencias
de cobranza mediante:

- Snowflake para ingesta, almacenamiento y modelado de datos
- SQL para creación de modelos y vistas analíticas
- Looker Studio para dashboards ejecutivos

El dataset es sintético e incluye:
- Operaciones de crédito
- Gestiones
- Pagos
- Agencias
- Calendario

## 2. Arquitectura General

CSV Local
    → Snowflake Stage
        → Tablas modelo estrella
            → Vistas analíticas
                → Looker Studio
                    → Dashboards ejecutivos

Componentes:
- Warehouse: WH_BI_XS
- Database: BD_RECUPERO
- Schemas: STG_RECUPERO, ANALITICA
- Stage: STG_RECUPERO
- Visualización: Looker Studio

## 3. Modelo de Datos (Esquema Estrella)

Dimensiones:
- DIM_TIEMPO → Año, mes, nombre del mes, fecha
- DIM_AGENCIA → Agencia, región
- DIM_OPERACION → Operación, saldo inicial, saldo actual, tramo y tipo de cartera

Hechos:
- FACT_GESTION → resultado de gestión, contactos, promesas
- FACT_PAGO → montos pagados por operación
  
## 4. Ingesta de Datos

4.1 Creación del Stage:
CREATE STAGE IF NOT EXISTS STG_RECUPERO
  FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1);

4.2 Carga de archivos CSV:
PUT file://stg_recupero/*.csv @STG_RECUPERO;
LIST @STG_RECUPERO;

4.3 Carga a tablas:
COPY INTO DIM_AGENCIA
FROM @STG_RECUPERO/dim_agencia.csv
FILE_FORMAT=(TYPE='CSV' SKIP_HEADER=1);

## 5. Transformaciones y Vistas Analíticas

### 5.1 Vista de stock por agencia:
CREATE OR REPLACE VIEW VW_STOCK_AGENCIA AS
SELECT
    o.ID_AGENCIA,
    a.NOMBRE_AGENCIA,
    a.REGION,
    SUM(o.SALDO_INICIAL) AS SALDO_INICIAL_AGENCIA,
    SUM(o.SALDO_ACTUAL) AS SALDO_ACTUAL_AGENCIA
FROM DIM_OPERACION o
JOIN DIM_AGENCIA a ON o.ID_AGENCIA = a.ID_AGENCIA
GROUP BY 1,2,3;

### 5.2 Vista de recupero mensual:
CREATE OR REPLACE VIEW VW_RECUPERO_MENSUAL_AGENCIA AS
SELECT
    t.ANIO,
    t.MES,
    t.NOMBRE_MES,
    o.ID_AGENCIA,
    a.NOMBRE_AGENCIA,
    a.REGION,
    SUM(p.MONTO_PAGO) AS MONTO_RECUPERADO
FROM FACT_PAGO p
JOIN DIM_OPERACION o ON p.ID_OPERACION = o.ID_OPERACION
JOIN DIM_TIEMPO t ON p.ID_TIEMPO = t.ID_TIEMPO
JOIN DIM_AGENCIA a ON o.ID_AGENCIA = a.ID_AGENCIA
GROUP BY 1,2,3,4,5,6;

### 5.3 Vista de gestiones mensuales:
Incluye totales, contactos, promesas, contactabilidad y % de promesas.

CREATE OR REPLACE VIEW VW_GESTIONES_MENSUAL_AGENCIA AS
SELECT
    t.ANIO,
    t.MES,
    t.NOMBRE_MES,
    o.ID_AGENCIA,
    a.NOMBRE_AGENCIA,
    a.REGION,
    COUNT(*) AS GESTIONES_TOTALES,
    SUM(CASE WHEN g.RESULTADO_GESTION IN ('Contacto efectivo','Promesa de pago')
             THEN 1 ELSE 0 END) AS GESTIONES_CONTACTO,
    SUM(CASE WHEN g.RESULTADO_GESTION = 'Promesa de pago'
             THEN 1 ELSE 0 END) AS GESTIONES_PROMESA,
    SUM(CASE WHEN g.RESULTADO_GESTION IN ('Contacto efectivo','Promesa de pago')
             THEN 1 ELSE 0 END) / NULLIF(COUNT(*),0) AS TASA_CONTACTABILIDAD,
    SUM(CASE WHEN g.RESULTADO_GESTION = 'Promesa de pago'
             THEN 1 ELSE 0 END) / NULLIF(COUNT(*),0) AS PORC_GESTIONES_CON_PROMESA
FROM FACT_GESTION g
JOIN DIM_OPERACION o ON g.ID_OPERACION = o.ID_OPERACION
JOIN DIM_TIEMPO t ON g.ID_TIEMPO = t.ID_TIEMPO
JOIN DIM_AGENCIA a ON o.ID_AGENCIA = a.ID_AGENCIA
GROUP BY 1,2,3,4,5,6;

### 5.4 Vista final para BI (dashboard):
CREATE OR REPLACE VIEW VW_DASH_AGENCIA_MENSUAL AS
SELECT
    e.ANIO,
    e.MES,
    e.NOMBRE_MES,
    TO_DATE(TO_VARCHAR(e.ANIO) || '-' || LPAD(e.MES, 2, '0') || '-01') AS FECHA_MES,
    e.ID_AGENCIA,
    e.NOMBRE_AGENCIA,
    e.REGION,
    e.SALDO_INICIAL_AGENCIA,
    e.SALDO_ACTUAL_AGENCIA,
    e.MONTO_RECUPERADO,
    e.EFICIENCIA_SOBRE_INICIAL,
    g.GESTIONES_TOTALES,
    g.GESTIONES_CONTACTO,
    g.GESTIONES_PROMESA,
    g.TASA_CONTACTABILIDAD,
    g.PORC_GESTIONES_CON_PROMESA
FROM VW_EFICIENCIA_MENSUAL_AGENCIA e
LEFT JOIN VW_GESTIONES_MENSUAL_AGENCIA g
  ON e.ANIO = g.ANIO AND e.MES = g.MES AND e.ID_AGENCIA = g.ID_AGENCIA;

## 6. Dashboards en Looker Studio

Página 1 – Panel Ejecutivo:
- Recupero total
- Eficiencia sobre saldo inicial
- Gestiones totales
- Recupero por agencia
- KPIs operativos

Página 2 – Calidad Operativa:
- Contactabilidad mensual
- Promesas mensuales
- Gestiones vs contactos vs promesas
- Tabla operativa detallada

Página 3 – Evolución del recupero mensual (tendencia general):
- Evolución del recupero mensual
- Evolución del recupero por agencia (líneas comparativas)
- Ranking acumulado de agencias por recupero
- Evolución del saldo actual de cartera
- KPIs agregados del período (recupero total, eficiencia promedio, variaciones mensuales)

## 7. Métricas Clave

Contactabilidad = Contactos / Gestiones  
Promesas = Promesas / Gestiones  
Eficiencia = Recupero / Saldo Inicial  
Recupero = SUM(MONTO_PAGO)  
Gestiones Totales = COUNT(*)  

## 8. Conclusiones

El proyecto demuestra:
- Carga y modelado en Snowflake
- Aplicación de modelo estrella
- Creación de vistas analíticas
- Integración con Looker Studio
- KPI típicos del negocio de cobranzas
- Flujo end-to-end estilo Data Engineer / BI Developer

## 9. Autor

[Nicolás Giovannoni](https://www.linkedin.com/in/nicolas-giovannoni2806/)

