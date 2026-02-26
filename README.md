# Análisis de embudo y retención para MercadoLibre
Análisis de producto en MercadoLibre, dentro del equipo de Crecimiento y Retención.
# Objetivos del proyecto
- Construir embudos multietapa en SQL usando CTEs.
- Calcular tasas de conversión entre pasos y detectar caídas.
- Analizar la retención de usuarios por cohortes.
- Simular mejoras en conversión o retención.
- Validar resultados y comunicar hallazgos ejecutivos.
#Dataset del proyecto
Tablas disponibles
- mercadolibre_funnel
- mercadolibre_retention
Definición del Macro Journey (Embudo General)
El negocio esta interesado particularmente en el siguiente embudo de conversión.

# Etapa del Embudo	Evento	Descripción	Métrica Clave
<img width="706" height="754" alt="image" src="https://github.com/user-attachments/assets/d038c8a4-7901-4dcc-9c00-d0fb1f81c7ad" />

# Contexto del negocio
La dirección de producto busca responder:
# ¿En qué etapa se pierden más usuarios?
- Entre el [01/01/2025] y el [08/31/2025], ¿cuál es la tasa de conversión entre cada etapa clave del embudo?.
- ¿En qué paso se observa la mayor caída porcentual de usuarios?
- ¿Cómo varía esta pérdida por país (country)?

Métricas clave:
- Tasa de conversión por etapa
- Agrupación de eventos por device_category y referral_source

# ¿Qué tan bien retenemos a los usuarios a lo largo del tiempo?
- Para los usuarios que se registraron entre el [01/01/2025] y el [06/01/2025], ¿cuál es la tasa de retención en D7, D14, D21, D28?
- ¿Cómo se comporta la retención por país (country)?

Métrica clave: 
- Retención por cohorte (D7, D14, D21, D28)

---
# CONTRUYENDO EL EMBUDO DE CONVERSIÓN
### La meta es calcular, con CTEs, cuántos usuarios alcanzan cada etapa del Embudo General, desde la primera visita hasta la compra.
# 1. Creaamos CTEs por etapa
## Objetivo:
- Construir bloques de usuarios únicos por evento (CTEs) en el rango 2025-01-01 → 2025-08-31, unirlos y contar usuarios por etapa del embudo. Al unirlos, nos aseguramos de que todos los usuarios pasaron por cada etapa del embudo.


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
<img width="1025" height="165" alt="image" src="https://github.com/user-attachments/assets/5d31e6af-544e-48bb-bc9c-ff109d5f4130" />

# 2. Calcular conversiones entre etapas
## Objetivo: 
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
<img width="1031" height="157" alt="image" src="https://github.com/user-attachments/assets/6f0b1214-adc4-4606-bf06-b6d28d42e03b" />

# 3. Segmentar el Embudo General por país
## Objetivo: 
- agrupa las conversiones del embudo por país (country) y detecta en que etapa del funnel se pierde más a los usuarios.
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
<img width="1037" height="528" alt="image" src="https://github.com/user-attachments/assets/fb2d41a9-be44-4582-85e4-43360ccbf30e" />


