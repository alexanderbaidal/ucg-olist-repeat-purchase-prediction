# **Personalización Data-Driven en E-Commerce: Potenciando la Retención de Clientes** 

### **Sector/Tipo de Dataset** 

- **Sector:** E-Commerce del sector privado (Online Retail Business) 

- **Dataset:** Datos de clientes y ventas de una plataforma de e-commerce (registros transaccionales y de comportamiento curados; serán provistos) 

### **Contexto y descripción del problema** 

**Contexto de negocio:** _ShopMart_ (una empresa ficticia de venta online de tamaño mediano) opera en un mercado de e-commerce altamente competitivo. La compañía ofrece una amplia gama de productos a través de su sitio web y app móvil, atendiendo a miles de clientes. En los últimos años, _ShopMart_ ha acumulado un gran volumen de datos sobre interacciones de clientes —incluyendo historiales de compra, comportamiento de navegación, calificaciones de productos y comunicaciones de marketing—. La dirección ha identificado la **retención de clientes y la experiencia personalizada** como áreas estratégicas clave para mejorar la rentabilidad y la lealtad. La investigación muestra que adquirir nuevos clientes es significativamente más costoso que retener a los existentes (del orden de 5 veces más caro). Al mismo tiempo, los líderes del sector han demostrado el poder de la personalización: por ejemplo, alrededor del **35% de las ventas de Amazon están impulsadas por su recommendation engine con AI** . Estos hechos subrayan el valor de negocio de aprovechar los datos y la AI para involucrar mejor a los clientes de _ShopMart_ . 

**Impulsores del problema:** Actualmente, _ShopMart_ enfrenta retos comunes del e-commerce: tasas de **churn** (pérdida de clientes) más altas de lo deseado, tasas de recompra estancadas y esfuerzos de marketing insuficientemente adaptados a preferencias individuales. Los clientes tienen muchas opciones online; si _ShopMart_ no presenta productos relevantes u ofertas oportunas, pueden migrar a competidores. La gerencia cree que los **datos ricos** disponibles pueden transformarse en insights accionables para personalizar la experiencia de compra y abordar proactivamente el churn. Han planteado un desafío amplio al equipo de data science: usar técnicas de **AI/ML** para explotar los datos de clientes y **mejorar el customer lifetime value (CLV)** . El proyecto nace de una necesidad real de negocio: incrementar ingresos y lealtad entendiendo y atendiendo mejor a los clientes. 

**Antecedentes del dato:** Se proporcionará un **dataset curado** que simula los datos de clientes y ventas de _ShopMart_ . Probablemente incluirá (sin limitarse a): 

- **Información de clientes:** Datos básicos de perfil (p. ej., customer ID, fecha de registro, quizá demografía o ubicación) y métricas de engagement (como nivel de lealtad o interacciones previas con customer service, si están disponibles). 

- **Historia transaccional:** Registros detallados de pedidos (fechas, productos comprados, cantidades, precios, categorías, método de pago, etc.) para un periodo de tiempo; capturan frecuencia y comportamiento de compra. 

- **Catálogo de productos:** Información de productos (product IDs, categorías, precios, quizá descripciones o ratings) para relacionar el comportamiento del cliente con atributos de producto. 

- **Browsing/clickstream data (si se incluye):** Logs de navegación (vistas de páginas de producto, items añadidos al carrito, search queries) y puntos de contacto de marketing (clicks en emails, etc.), que enriquecen la comprensión del interés incluso cuando no hubo compra. 

Todos los datos han sido limpiados y preparados a alto nivel, pero se espera que los **estudiantes realicen Exploratory Data Analysis (EDA)** y posiblemente un preprocesamiento adicional para extraer insights. El dataset refleja patrones típicos de e-commerce (algunos clientes muy activos y otros esporádicos, long tail de ventas por producto, picos estacionales, etc.), proporcionando un terreno realista para aplicar habilidades de AI y data science. No se requiere recolectar datos externos; el foco es **aprovechar al máximo la información provista** para abordar el problema. 

**Relevancia de negocio:** Resolver este caso tiene implicaciones directas para el éxito de _ShopMart_ . Al analizar los datos de clientes para detectar patrones, la empresa puede **identificar quiénes tienen riesgo de churn, qué segmentos responden a qué tipos de productos o promociones, y qué recomendaciones u ofertas incrementarían el engagement** . Los insights pueden informar decisiones estratégicas como campañas de marketing dirigidas, sistemas de recomendación personalizados en el sitio, programas de lealtad u ofertas proactivas de retención para clientes de alto riesgo. En esencia, la empresa busca **aumentar el CLV y la satisfacción** personalizando la experiencia según necesidades y comportamientos. Lograrlo mejora las ventas recurrentes, reduce el desperdicio en marketing y fortalece la posición competitiva mediante mayor lealtad y word-of-mouth. 

### **Pregunta analítica guía** 

**Desafío abierto:** _¿Cómo puede ShopMart aprovechar sus datos de clientes y transacciones para predecir o influir en el comportamiento del cliente de formas que mejoren la retención e incrementen las ventas?_ 

La pregunta es intencionalmente amplia, invitando a múltiples enfoques analíticos. Debe interpretarse en el contexto de _ShopMart_ y refinarse en sub-preguntas específicas tras explorar los datos. En esencia, el equipo debe definir **qué estrategia data-driven ayudaría más a mantener el engagement y la lealtad** . Por ejemplo, el análisis podría ayudar a responder: 

- “¿Qué clientes probablemente dejarán de comprar (churn) en el corto plazo y qué señales lo indican?” 

- “¿Qué segmentos de clientes podemos identificar y cómo dirigir a cada uno con marketing o recomendaciones personalizadas?” 

- “¿Cómo recomendar productos relevantes a cada usuario para aumentar cross-selling y up-selling?” 

- “¿Cuál es el lifetime value esperado de distintos clientes y cómo actuar para maximizarlo?” 

La pregunta central debe enmarcarse de modo que **use machine learning o AI** para abordar un driver clave de retención o experiencia personalizada. Debe ser respondible con el dataset provisto y alinearse con la meta de mejorar el CLV. Al final, el equipo propondrá y validará una solución con AI (o insights) que responda directamente a esta pregunta guía. No hay **una sola “respuesta correcta”** : parte de la tarea es decidir qué ángulo atacar (p. ej., churn prediction, customer segmentation, recommendation engine, etc.) y justificar por qué ese enfoque impulsa la retención **con evidencia del análisis** . 

### **Restricciones y flexibilidad** 

Este caso está diseñado para ser **realista pero flexible** , dando libertad para explorar distintas rutas dentro de un alcance definido: 

- **Dataset curado provisto:** No es necesario recolectar datos externos; todo lo requerido se entregará. El dataset de e-commerce es lo bastante completo para soportar múltiples ángulos (conductual, transaccional, predictivo). Se espera que el equipo realice **data cleaning** , transformación y EDA para extraer features e insights útiles. El alcance del dataset (periodo y rango de productos) define el sandbox de trabajo. No es necesario, por ejemplo, integrar social media data; el foco es explotar el **dato interno** de _ShopMart_ . 

- **Restricciones de negocio y técnicas:** La solución propuesta debe ser **práctica para una empresa mediana** . Considere: complejidad del modelo (implementable con recursos de la compañía), privacidad (manejo de datos conforme a estándares) y factibilidad de despliegue (p. ej., un recommendation system en tiempo real tendría que integrarse con la plataforma). Para el proyecto académico, no se exige un sistema en producción: el énfasis está en el **proof-of-concept** mediante análisis y modelado. Hay libertad para elegir herramientas y algoritmos (el equipo domina Python y librerías como scikit-learn, TensorFlow, PyTorch, etc.), pero justifique que las elecciones **tienen sentido según las características del dato** (p. ej., usar una red neuronal solo si la complejidad y el tamaño lo ameritan; usar modelos más simples si bastan). 

- **Enfoque de modelado abierto:** El equipo tiene **libertad para elegir** la aproximación de AI/ML que mejor responda a la pregunta guía, existiendo múltiples estrategias viables. El problema puede abordarse como tarea supervisada (p. ej., predecir churn o el **next purchase amount** ), descubrimiento no supervisado (p. ej., clustering de clientes), o recomendación (p. ej., collaborative filtering o content-based). Dado que el reto es amplio, el equipo debe **acotar el foco tras el EDA** : decidir qué problema específico (o combinación) maximiza impacto de negocio. No se impone modelo ni target predefinido: corresponde al equipo definir un objetivo de modelado razonable (con aprobación docente si aplica) alineado a retención/personalización. 

- **Plazos y tamaño del equipo:** El proyecto lo realiza un equipo pequeño de hasta 3 estudiantes durante la **Trayectoria 1** (aprox. **4–5 meses** de trabajo parcial). Deben planificar todas las fases: entender el contexto, analizar datos, modelar y reportar. La gestión del tiempo es crucial: dedicar semanas tempranas a EDA y definición del problema; luego implementar e iterar modelos, dejando tiempo para documentación y refinamiento. El alcance debe ser realista: es mejor implementar **uno o dos componentes clave** a fondo que muchos superficiales (p. ej., **churn prediction + customer segmentation de apoyo** , o **prototipo de recommendation system** con evaluación de impacto). 

- **Guía y recursos:** Aunque el problema es abierto, hay mentoría docente y recursos del curso. Se espera aplicar conceptos vistos (data mining, machine learning, deep learning, etc.). **No hay “recetario” paso a paso** : se valora el pensamiento crítico para decidir métodos. Si se requieren supuestos (p. ej., definición operativa de “churn” o umbrales de high vs. low value), deben declararse claramente y ser razonables. En resumen, hay restricciones de alcance de datos y relevancia de negocio, pero **amplia libertad** para explorar e innovar. 

### **Notas sobre consideraciones éticas/sociales** 

Trabajar en este caso ofrece una oportunidad para reflexionar sobre dimensiones éticas y sociales de la AI en negocio. Considere e integre en sus entregables: 

- **Privacidad de datos de clientes:** Tratar los datos como sensibles, incluso si están pseudonimizados (IDs en lugar de nombres). En un despliegue real, asegurar consentimiento, opt-out, seguridad y cumplimiento (GDPR/leyes locales). En el proyecto, **evitar PII innecesaria** en el modelado y documentar prácticas seguras. 

- **Equidad y sesgo:** Verificar si el dato o el modelo reflejan sesgos. Por ejemplo, si “alto valor” privilegia a ciertos grupos (p. ej., regiones con mayor poder adquisitivo). Usar etiquetas neutrales para segmentos (p. ej., “bargain shoppers” vs. “premium shoppers”) y garantizar oportunidades equitativas (no dirigir descuentos solo a un grupo). Evaluar métricas por segmentos y, si hay disparidades, proponer correcciones. 

- **Transparencia y autonomía del cliente:** La personalización debe **mejorar** la experiencia sin engañar o coaccionar. Proveer explicaciones (“Recomendado porque compraste X”), opciones de control y evitar spam (frequency capping). Usar el modelo de churn para **asistir** a los equipos, no para penalizar. 

- **Evitar resultados sociales negativos:** Evitar explotar a consumidores vulnerables (p. ej., fomento de gasto excesivo). Incluir controles (alertas de gasto), mantener supervisión humana y aprendizaje continuo para corregir errores de segmentación o recomendaciones. 

- **Cumplimiento regulatorio:** Señalar si se usan atributos sensibles y justificarlo o excluirlos. Reconocer el escrutinio regulatorio creciente sobre personalización y fairness. 

## 🗂 DATASET 

<u>https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce</u> 

**Conjunto de Datos de Comercio Electrónico Brasileño Olist –** Un conjunto de datos público multi-tabla de una plataforma de comercio electrónico latinoamericana, útil para el análisis del comportamiento del cliente y la tasa de abandono (churn). Origen y Descripción: Publicado por Olist, un marketplace brasileño, que contiene 100k pedidos realizados entre 2016–2018. Los datos abarcan múltiples aspectos de las transacciones: artículos del pedido, productos, vendedores, clientes, pagos y reseñas. Esto permite ver cada compra desde la realización del pedido hasta la entrega y los comentarios del cliente. Formato y Tamaño: Se proporciona en varios archivos CSV (~ — por ejemplo, orders.csv tiene 100k pedidos, order_items.csv 112k artículos, etc.). Tamaño total ~20 MB. Variables: Las tablas clave incluyen: Clientes (ID de cliente único con ciudad, estado), Pedidos (ID de pedido, ID de cliente, marca de tiempo del pedido, marcas de tiempo de entrega, estado del pedido), Artículos del Pedido (ID de pedido, ID de producto, ID de vendedor, precio, costo de flete), Productos (metadatos del producto como categoría, longitud del nombre, longitud de la descripción), Pagos (tipo de pago e información de cuotas) y Reseñas (puntuación y comentario de la reseña, con marca de tiempo de la reseña). Usando esto, se pueden derivar características como el historial de compras de cada cliente, la frecuencia, la recencia y si devolvieron artículos o tuvieron problemas (a partir de las reseñas). Licencia: Creative Commons BY-NC-SA 4.0 – de uso gratuito para fines académicos/no comerciales (la página de Kaggle enumera explícitamente esta licencia). Advertencias: No hay una etiqueta de 'churn' (abandono) predefinida; se puede inferir (p.ej. sin pedidos en los últimos X meses de datos). Los datos son transaccionales y estáticos (una instantánea de 2016–2018). También tenga en cuenta que los nombres de las categorías de productos están en portugués y algunos están codificados. Asegúrese de unir las tablas mediante ID para obtener una imagen completa (cada tabla por sí sola no es suficiente). A pesar de que la licencia limita el uso comercial, es excelente para proyectos educativos. 

