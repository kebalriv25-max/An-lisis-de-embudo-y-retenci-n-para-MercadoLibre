# CAP 1: Análisis de embudo y retención para MercadoLibre
Análisis de producto en MercadoLibre, dentro del equipo de Crecimiento y Retención.
## 1.1 Objetivos del proyecto
- Construir embudos multietapa en SQL usando CTEs.
- Calcular tasas de conversión entre pasos y detectar caídas.
- Analizar la retención de usuarios por cohortes.
- Simular mejoras en conversión o retención.
- Validar resultados y comunicar hallazgos ejecutivos.
## 1.2 Dataset del proyecto
Tablas disponibles
- mercadolibre_funnel
- mercadolibre_retention
Definición del Macro Journey (Embudo General)
El negocio esta interesado particularmente en el siguiente embudo de conversión.

## 1.3 Etapa del Embudo	Evento	Descripción	Métrica Clave
<img width="706" height="754" alt="image" src="https://github.com/user-attachments/assets/d038c8a4-7901-4dcc-9c00-d0fb1f81c7ad" />

## 1.4 Contexto del negocio
La dirección de producto busca responder:
## ¿En qué etapa se pierden más usuarios?
- Entre el [01/01/2025] y el [08/31/2025], ¿cuál es la tasa de conversión entre cada etapa clave del embudo?.
- ¿En qué paso se observa la mayor caída porcentual de usuarios?
- ¿Cómo varía esta pérdida por país (country)?

Métricas clave:
- Tasa de conversión por etapa
- Agrupación de eventos por device_category y referral_source

## ¿Qué tan bien retenemos a los usuarios a lo largo del tiempo?
- Para los usuarios que se registraron entre el [01/01/2025] y el [06/01/2025], ¿cuál es la tasa de retención en D7, D14, D21, D28?
- ¿Cómo se comporta la retención por país (country)?

Métrica clave: 
- Retención por cohorte (D7, D14, D21, D28)

---
# CAP 2: CONTRUYENDO EL EMBUDO DE CONVERSIÓN
- La meta es calcular, con CTEs, cuántos usuarios alcanzan cada etapa del Embudo General, desde la primera visita hasta la compra.
## 2.1. Creaamos CTEs por etapa
### Objetivo:
- Construir bloques de usuarios únicos por evento (CTEs) en el rango 2025-01-01 → 2025-08-31, unirlos y contar usuarios por etapa del embudo. Al unirlos, nos aseguramos de que todos los usuarios pasaron por cada etapa del embudo.
- A partir de los conteos por etapa del embudo, calcular el porcentaje de conversión desde la etapa inicial (first_visit) hacia cada etapa.

```sql
WITH first_visit AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'first_visit'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
select_item AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name IN ('select_item', 'select_promotion')
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_to_cart AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'add_to_cart'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
begin_checkout AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'begin_checkout'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_shipping_info AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'add_shipping_info'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_payment_info AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'add_payment_info'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
purchase AS (
  SELECT DISTINCT user_id
  FROM mercadolibre_funnel
  WHERE event_name = 'purchase'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
), 
funnel_counts AS(
SELECT
  COUNT(fv.user_id) AS usuarios_first_visit,
  COUNT(si.user_id) AS usuarios_select_item,
  COUNT(a.user_id) AS usuarios_add_to_cart,
  COUNT(bc.user_id) AS usuarios_begin_checkout,
  COUNT(asi.user_id) AS usuarios_add_shipping_info,
  COUNT(api.user_id) AS usuarios_add_payment_info,
  COUNT(p.user_id) AS usuarios_purchase
FROM first_visit fv
LEFT JOIN select_item si        ON fv.user_id = si.user_id
LEFT JOIN add_to_cart a         ON fv.user_id = a.user_id
LEFT JOIN begin_checkout bc     ON fv.user_id = bc.user_id
LEFT JOIN add_shipping_info asi ON fv.user_id = asi.user_id
LEFT JOIN add_payment_info api  ON fv.user_id = api.user_id
LEFT JOIN purchase p            ON fv.user_id = p.user_id
)

SELECT
ROUND(usuarios_select_item * 100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_select_item,
ROUND(usuarios_add_to_cart *100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_add_to_cart,
ROUND(usuarios_begin_checkout *100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_begin_checkout,
ROUND(usuarios_add_shipping_info * 100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_add_shipping_info,
ROUND(usuarios_add_payment_info * 100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_add_payment_info,
ROUND(usuarios_purchase * 100.0 / NULLIF(usuarios_first_visit, 0), 2) AS conversion_purchase
FROM funnel_counts;
```
### Embudo General
<img width="1031" height="157" alt="image" src="https://github.com/user-attachments/assets/6f0b1214-adc4-4606-bf06-b6d28d42e03b" />

## 2.2. Segmentar el Embudo General por país
### Objetivo: 
- agrupar las conversiones del embudo por país (country) y detectar en que etapa del funnel se pierde más a los usuarios.
```sql
WITH first_visits AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'first_visit'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
select_item AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name IN ('select_item', 'select_promotion')
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_to_cart AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'add_to_cart'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
begin_checkout AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'begin_checkout'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_shipping_info AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'add_shipping_info'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
add_payment_info AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'add_payment_info'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
purchase AS (
  SELECT DISTINCT user_id, country
  FROM mercadolibre_funnel
  WHERE event_name = 'purchase'
    AND event_date BETWEEN '2025-01-01' AND '2025-08-31'
),
funnel_counts AS(
SELECT fv.country,
  COUNT(fv.user_id) AS usuarios_first_visit,
  COUNT(si.user_id) AS usuarios_select_item,
  COUNT(a.user_id) AS usuarios_add_to_cart,
  COUNT(bc.user_id) AS usuarios_begin_checkout,
  COUNT(asi.user_id) AS usuarios_add_shipping_info,
  COUNT(api.user_id) AS usuarios_add_payment_info,
  COUNT(p.user_id) AS usuarios_purchase
FROM first_visits fv
LEFT JOIN select_item si        ON fv.user_id = si.user_id AND fv.country = si.country
LEFT JOIN add_to_cart a         ON fv.user_id = a.user_id AND fv.country = a.country
LEFT JOIN begin_checkout bc     ON fv.user_id = bc.user_id AND fv.country = bc.country
LEFT JOIN add_shipping_info asi ON fv.user_id = asi.user_id AND fv.country = asi.country
LEFT JOIN add_payment_info api  ON fv.user_id = api.user_id AND fv.country = api.country
LEFT JOIN purchase p            ON fv.user_id = p.user_id AND fv.country = p.country
GROUP BY fv.country
)
    
SELECT 
country,
usuarios_select_item *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_select_item,
usuarios_add_to_cart *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_to_cart,
usuarios_begin_checkout *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_begin_checkout,
usuarios_add_shipping_info *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_shipping_info,
usuarios_add_payment_info *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_add_payment_info,
usuarios_purchase *100.0 / NULLIF(usuarios_first_visit, 0) AS conversion_purchase
FROM funnel_counts
ORDER BY conversion_purchase DESC;
```
### Embudo General por país
<img width="1037" height="528" alt="image" src="https://github.com/user-attachments/assets/fb2d41a9-be44-4582-85e4-43360ccbf30e" />

## 2.3. Medimos la retención real de usuarios
### Objetivo: 
- Para cada país, contar usuarios activos acumulados desde su registro, en el rango 2025-01-01 → 2025-08-31, al día 7, día 14, día 21 y día 28.
```sql
   SELECT 
    country,
    -- Contar usuarios activos acumulados al día 7
    COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 7 THEN user_id END) AS users_d7,
    -- Contar usuarios activos acumulados al día 14
    COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 14 THEN user_id END) AS users_d14,
    -- Contar usuarios activos acumulados al día 21
    COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 21 THEN user_id END) AS users_d21,
    -- Contar usuarios activos acumulados al día 28
    COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 28 THEN user_id END) AS users_d28
FROM 
    mercadolibre_retention
WHERE 
    activity_date BETWEEN '2025-01-01' AND '2025-08-31'
GROUP BY 
    country
ORDER BY 
    country;
```
### Retención real por usuario
<img width="915" height="482" alt="image" src="https://github.com/user-attachments/assets/9a88658b-ab49-4a5c-b06f-dbde7ce96ce6" />

- Convertir los conteos activos realizados anteriormente, en porcentajes de retención por país al día 7, día 14, día 21 y día 28.
```sql
SELECT 
    country,
    -- Porcentaje de Retención Día 7
    ROUND(
        COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 7 THEN user_id END) * 100.0 / NULLIF(COUNT(DISTINCT user_id), 0), 1) AS retention_d7_pct,

    -- Porcentaje de Retención Día 14
    ROUND(
        COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 14 THEN user_id END) * 100.0 
        / NULLIF(COUNT(DISTINCT user_id), 0), 
    1) AS retention_d14_pct,

    -- Porcentaje de Retención Día 21
    ROUND(
        COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 21 THEN user_id END) * 100.0 
        / NULLIF(COUNT(DISTINCT user_id), 0), 
    1) AS retention_d21_pct,

    -- Porcentaje de Retención Día 28
    ROUND(
        COUNT(DISTINCT CASE WHEN active = 1 AND day_after_signup >= 28 THEN user_id END) * 100.0 
        / NULLIF(COUNT(DISTINCT user_id), 0), 
    1) AS retention_d28_pct

FROM 
    mercadolibre_retention
WHERE 
    activity_date BETWEEN '2025-01-01' AND '2025-08-31'
GROUP BY 
    country
ORDER BY 
    country;
```
### Retención por país
<img width="959" height="485" alt="image" src="https://github.com/user-attachments/assets/3bc7d007-eb84-45ea-92fb-73fa31565fa2" />

- Ahora vamos a analizar la retención por cohort, asignando el cohort en formato YYYY-MM a cada usuario (usando su primera fecha de registro).
- Para cada cohorte mensual (YYYY-MM), se va a calcular el % de usuarios activos al día 7, 14, 21, y 28  desde su registro.
```sql
- WITH cohort AS (
SELECT
user_id,
TO_CHAR(DATE_TRUNC('month', MIN(signup_date)), 'YYYY-MM') AS cohort
FROM mercadolibre_retention
GROUP BY user_id
),
activity AS (
    SELECT r.user_id, c.cohort, r.day_after_signup, r.active
    FROM mercadolibre_retention r
    LEFT JOIN cohort c
    ON r.user_id = c.user_id
    WHERE activity_date BETWEEN '2025-01-01' AND '2025-08-31'
)
SELECT cohort, 
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN day_after_signup >= 7 AND active = 1 THEN user_id END) / NULLIF(COUNT(DISTINCT user_id),0),1) AS retention_d7_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN day_after_signup >= 14  AND active = 1 THEN user_id END) / NULLIF(COUNT(DISTINCT user_id),0),1) AS retention_d14_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN day_after_signup >= 21 AND active = 1 THEN user_id END) / NULLIF(COUNT(DISTINCT user_id),0),1) AS retention_d21_pct,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN day_after_signup >= 28 AND active = 1 THEN user_id END) / NULLIF(COUNT(DISTINCT user_id),0),1) AS retention_d28_pct

FROM activity
GROUP BY cohort
ORDER BY cohort;
```
### Retención por cohorte
<img width="961" height="408" alt="image" src="https://github.com/user-attachments/assets/3e3d33b0-cf69-45f0-89fb-9a2923a88785" />

# CAP 3: Informe Ejecutivo

## 3.1. Informe ejecutivo (C → F → I)
#### Respondemos nuestras principales interrogantes:
#### 🎈 Entre el [01/01/2025] y el [08/31/2025], ¿cuál es la tasa de conversión entre cada etapa clave del embudo.
- ¿En qué paso se observa la mayor caída porcentual de usuarios?
- ¿Cómo varía esta pérdida por país (country)?

#### a) Context
- Análisis del embudo de conversión completo (desde select_item hasta purchase) para usuarios activos entre el 01/01/2025 y el 31/08/2025, desglosado por país.
#### b) Hallazgo
- La caída más drástica ocurre entre Select Item (76.9%) y Add to Cart (11.0%); perdemos casi al 85% de los interesados ahí.
- Paraguay presenta una anomalía crítica: tiene 0.00% de conversión desde Begin Checkout en adelante, indicando un posible error técnico.
- Uruguay tiene el mejor desempeño en Add to Cart (22.7%), doblando el promedio general.
#### c) Implicación
- Revisar la etapa de "add_to_cart" ya que es el mayor "tapón" del embudo.
- Investigar qué está haciendo bien Uruguay para replicarlo en otros mercados.

#### 🎈 ¿Qué tan bien retenemos a los usuarios a lo largo del tiempo?
- Para los usuarios que se registraron entre el [01/01/2025] y el [06/01/2025], ¿cuál es la tasa de retención en D7, D14, D21, D28?
- ¿Cómo se comporta la retención agrupados por país (country)?

#### a) Context
- Evaluación de la retención de usuarios (actividad recurrente) a los 7, 14, 21 y 28 días después del registro. Se analizan las cohortes registradas entre Enero y Junio de 2025.
#### b) Hallazgo
- La retención inicial (D7) es muy fuerte y estable (~86-87%) en todas las cohortes de Enero a Junio.
- Sin embargo, la retención se desploma dramáticamente hacia el D28, cayendo a niveles de 2% - 3%.
- A partir del día 14 (D14 ~55%) la pérdida de usuarios se acelera significativamente.
#### c) Implicación
- El onboarding es exitoso (buen D7), pero el producto no logra generar un hábito a largo plazo.
- Se deben implementar campañas de re-engagement (notificaciones, emails, ofertas) agresivas entre el día 14 y el 21 para frenar la caída antes del mes.

#### 🔍Reflexión
- Definitivamente se tendría que mejorar la etapa de Add to Cart (Agregar al carrito). Según los datos, es donde ocurre la fuga masiva de usuarios (baja del 76% al 11%). Al estar tan arriba en el embudo, cualquier mejora porcentual aquí tendría un efecto multiplicador mucho mayor en las ventas finales que optimizar etapas posteriores como el pago, donde ya llegan muy pocos usuarios.
- Los usuarios tienen mucho interés en ver productos ( select_item), pero mucho miedo o fricción para comprometerse a comprar (add_to_cart).                                                         - En retención, los usuarios se quedan la primera semana (D7 > 86%), probablemente por la novedad de la app, pero perdemos su atención casi por completo antes de cumplir el mes (D28 < 3%). Esto sugiere que la app es fácil de empezar a usar, pero difícil de convertir en un hábito diario.			
