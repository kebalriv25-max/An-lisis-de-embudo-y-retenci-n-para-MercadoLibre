# Análisis de embudo y retencin para MercadoLibre
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
