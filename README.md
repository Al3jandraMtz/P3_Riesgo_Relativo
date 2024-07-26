# Proyecto 3.- Riesgo Relativo

Automatizar y optimizar el proceso de análisis crediticio para gestionar eficazmente el riesgo de incumplimiento

### Temas

- [Introducción](#introducción)
- [Herramientas](#herramientas)
- [Procesamiento](#procesamiento)
- [Hipótesis](#hipótesis)
- [Conclusiónes](#Conclusiónes)
- [Recomendaciones](#Recomendaciones)
- [Recursos](#Recursos)

## Introducción
 El riesgo relativo (RR) es una medida utilizada en epidemiología y estudios de investigación para comparar la incidencia de un resultado o evento entre dos grupos. Aquí está una explicación más detallada:
  * Grupo expuesto: Este grupo está compuesto por personas que han estado expuestas a un factor de riesgo específico. Por ejemplo, si estamos estudiando el riesgo de desarrollar enfermedades cardíacas en       fumadores, el grupo expuesto sería el de los fumadores.
  * Grupo no expuesto: Este grupo está compuesto por personas que no han estado expuestas al factor de riesgo. Siguiendo el mismo ejemplo, el grupo no expuesto sería el de los no fumadores.
    
El riesgo relativo se calcula como la razón de las tasas de incidencia entre estos dos grupos:

  > [!NOTE]
  > ![1](https://github.com/user-attachments/assets/90959969-7424-4843-9d1c-352066d53971))

  * Si el RR es igual a 1, significa que no hay diferencia en la incidencia entre los dos grupos. En otras palabras, la exposición al factor de riesgo no parece afectar la probabilidad de desarrollar la        enfermedad.
  * Si el RR es mayor que 1, indica que el grupo expuesto tiene un mayor riesgo de desarrollar la enfermedad en comparación con el grupo no expuesto. Por ejemplo, si RR = 1.5, significa que el grupo             expuesto tiene un 50% más de riesgo que el grupo no expuesto.
Si el RR es menor que 1, sugiere que el grupo expuesto tiene un menor riesgo de desarrollar la enfermedad en comparación con el grupo no expuesto. Por ejemplo, si RR = 0.8, significa que el grupo expuesto tiene un 20% menos de riesgo que el grupo no expuesto.
En resumen, el riesgo relativo nos ayuda a comprender cómo la exposición a un factor de riesgo afecta la probabilidad de un resultado específico en comparación con un grupo no expuesto. 


### Objetivo
El objetivo del análisis es armar un score crediticio a partir de un análisis de datos y la evaluación del riesgo relativo que pueda clasificar a los solicitantes en diferentes categorías de riesgo basadas en su probabilidad de incumplimiento. Esta clasificación permitirá al banco tomar decisiones informadas sobre a quién otorgar crédito, reduciendo así el riesgo de préstamos no reembolsables. Además, la integración de la métrica existente de pagos atrasados fortalecerá la capacidad del modelo para identificar riesgos, lo que en última instancia contribuirá a la solidez financiera y la eficiencia operativa del banco.

### **Herramientas**
  1. Google BigQuery
  2. Google Colab
  3. Google Looker Studio

## **Procesamiento**

### 1.1 Limpieza de datos 

1.- Tabla Default:
  Nulos:
  + user_id: 0 nulls
  + default_flag: 0 nulls
Duplicados:
  + user_id: 0 
  + default_flag: 0
Outliers:
  + default_flag: 0 Outliers

Consulta
  ~~~
  -- Identificar Nulos
  SELECT
    'user_id' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.default`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
  UNION ALL
  SELECT
    'default_flag' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.default`
  WHERE
    SAFE_CAST(default_flag AS INT64) IS NULL;
  
  -- Identificar y manejar valores duplicados
  WITH base_data AS (
    SELECT
     user_id,
     default_flag
    FROM
      `riesgo-relativo-p3.Data_Set.default`
  ),
  duplicados AS (
    SELECT
      user_id,
      default_flag,
      COUNT(*) AS duplicado
    FROM
      base_data
    GROUP BY
      user_id,
      default_flag
    HAVING
      COUNT(*) > 1
  )
  SELECT
    user_id,
    default_flag,
    duplicado
  FROM
    duplicados;
  
  -- Identificar y manejar datos fuera del alcance del análisis (No se aplica correlación entre variables)
  SELECT
    *
  FROM
   `riesgo-relativo-p3.Data_Set.default`
  WHERE
    user_id IS NULL OR default_flag IS NULL;
  
  -- Identificar y manejar datos inconsistentes en variables categóricas
  SELECT
    user_id,
    default_flag
  FROM
    `riesgo-relativo-p3.Data_Set.default`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL OR SAFE_CAST(default_flag AS INT64) IS NULL;
  
  #Identificar y manejar datos discrepantes en variables numéricas (OUTLIERS)
  WITH ordered_values AS (
      SELECT
          default_flag,
          NTILE(4) OVER (ORDER BY default_flag) AS quartile
      FROM `riesgo-relativo-p3.Data_Set.default`
  ),
  quartiles AS (
      SELECT
          MAX(CASE WHEN quartile = 1 THEN default_flag ELSE NULL END) AS Q1,
          MAX(CASE WHEN quartile = 2 THEN default_flag ELSE NULL END) AS Q2,  -- Usaremos Q2 como mediana
          MAX(CASE WHEN quartile = 3 THEN default_flag ELSE NULL END) AS Q3,
          MAX(CASE WHEN quartile = 4 THEN default_flag ELSE NULL END) AS Q4
      FROM ordered_values
  ),
  iqr_calc AS (
      SELECT
          Q1,
          Q3,
          Q3 - Q1 AS IQR,
          Q2 AS median_value
      FROM quartiles
  ),
  outliers AS (
      SELECT
          user_id,  -- Asumiendo que tienes una columna `user_id` para identificar las filas
          default_flag,
          CASE
              WHEN default_flag < (SELECT Q1 FROM iqr_calc) - 1.5 * (SELECT IQR FROM iqr_calc) THEN (SELECT Q1 FROM iqr_calc) - 1.5 * (SELECT IQR FROM iqr_calc) - default_flag
              WHEN default_flag > (SELECT Q3 FROM iqr_calc) + 1.5 * (SELECT IQR FROM iqr_calc) THEN default_flag - (SELECT Q3 FROM iqr_calc) - 1.5 * (SELECT IQR FROM iqr_calc)
          END AS distance_from_limit
      FROM `riesgo-relativo-p3.Data_Set.default`
      WHERE default_flag < (SELECT Q1 FROM iqr_calc) - 1.5 * (SELECT IQR FROM iqr_calc)
         OR default_flag > (SELECT Q3 FROM iqr_calc) + 1.5 * (SELECT IQR FROM iqr_calc)
  )
  SELECT *
  FROM outliers
  ORDER BY distance_from_limit DESC
  LIMIT 10;  -- Puedes ajustar este límite según cuántos outliers relevantes desees ver
  
  # Comprobar y cambiar tipo de dato (Se corrobora que sea INT64)
  
  #Crear nuevas variables (No se aplican nuevas variables)
  ~~~

2.- Tabla Loans_detail:
  Nulos:
    + user_id_nulos: 0 nulls
    + more_90_nulos: 0 nulls
    + using_lines_nulos: 0 nulls
    + numer_times_30_59_nulos: 0 nulls
    + debt_ratio_nulos: o nulls
    + number_times_60_89_nulos: 0 nulls
Duplicados: 0 
Valores fuera del alcance del analísis:
  Se evalúan las variables para detectar la correlación entre:
 
  + number_times_delayed_payment_loan_30_59_days / number_times_delayed_payment_loan_60_89_days : 0.98655 (Esto indica una alta similitud entre ellas.)
  + more_90_days_overdue, number_times_delayed_payment_loan_60_89_days : 0.99217 
  + more_90_days_overdue, number_times_delayed_payment_loan_30_59_day: 0.98291

  > [!NOTE]
  > ![2](https://github.com/user-attachments/assets/f095f852-5dcd-452f-ab82-d92813144db1)

  * Interpretación: Tiene una correlación muy similar con ambas variables de retraso en el pago

Desviación estandar:
  + more_90_days_overdue: 4.1213646684267
  + number_times_delayed_payment_loan_30_59: 4.144020
  + number_times_delayed_payment_loan_60_89_day: 4.105514

  > [!NOTE]
  > ![3](https://github.com/user-attachments/assets/36e93ada-a92e-4c97-b08b-229da7848dbb)

  * Interpretación: Las tres variables tienen desviaciones estándar similares, lo que sugiere que sus valores están dispersos en torno a sus medias de manera similar.

  Dado que *more_90_days_overdue* tiene una correlación similar con ambas variables de retraso en el pago y su desviación estándar es similar a las otras dos, podríamos considerar eliminarla para evitar     redundancia en nuestro análisis.

Outliers:
  > [!NOTE]
  > ![15](https://github.com/user-attachments/assets/6ff9facf-27a8-455d-8916-41d601879ebd)

   + Número de outliers en using_lines_not_secured_personal_assets: 177 ya que no contamos con información adicional, no se eliminan ni imputan estos datos.
  > [!NOTE]
  >![16](https://github.com/user-attachments/assets/514a9cdc-78ea-48d6-9035-0ecbb12de1ac)

  Vista outliders:
  > [!NOTE]
  > ![17](https://github.com/user-attachments/assets/ab67c30e-12f6-4cd0-ac53-054e905c2101)

   + Número de outliers en debt_ratio: 7579 Ya que no contamos con información adicional, no se eliminan ni se imputan estos datos.
  > [!NOTE]
   > ![22](https://github.com/user-attachments/assets/1394e668-9bfc-481f-8339-553341f517c9)

   Vista Outliders:
   > [!NOTE]
   > ![23](https://github.com/user-attachments/assets/32e1cb30-66c6-459a-a30c-7ab06f7c27d6)
     
   + Número de outliers en more_90_days_overdue: 1946
   > [!NOTE]
   ![Boxplot90](https://github.com/user-attachments/assets/9cf9b3c6-4884-4c55-b5a2-2bf8091401f5)

   Vista Outliders:
   > [!NOTE]
    >![18](https://github.com/user-attachments/assets/55c80bc5-5259-473e-8d2a-e2639d8e7d33)

   + Número de outliers en number_times_delayed_payment_loan_30_59_days: 5812 se deciden imputar (eliminar) los valores mayores a 20 (63 user_id)
   > [!NOTE]
   >![18](https://github.com/user-attachments/assets/55c80bc5-5259-473e-8d2a-e2639d8e7d33)

   Vista Outliders:
   > [!NOTE]
   >![19](https://github.com/user-attachments/assets/ece79371-c1a2-4d15-91fa-bedcf9b639d3)

   + Número de outliers en number_times_delayed_payment_loan_60_89_days: 1865 se deciden imputar (eliminar) los valores mayores a 20 (63 user_id).
   > [!NOTE]
   > ![20](https://github.com/user-attachments/assets/b4cbcb95-5635-45b8-8c8b-5d23c402dff9)

   Vista Outliders:
   > [!NOTE]
   >![21](https://github.com/user-attachments/assets/d0e07b55-a6fd-49c9-9fcb-f151271d3138)
     
  Nuevas Variables:
  + delay-30_59_90: Se realiza la limpieza de los Outliers, asi como la suma de las 3 variables de tiempo de retraso para crear esta columna.
  + segmentacion_delay: Segmentación en base a los dias de retraso.
  + Incumplimiento_debt_ratio: Segmentación para riesgo de incumplimiento de pago en debt_ratio.
      + Debt ratio bajo: Indica que una persona tiene poca deuda en relación con sus ingresos disponibles o activos
        + Mayor capacidad de manejar deuda.
        + Mejor capacidad de obtener crédito adicional.
        + Impacto positivo en la calificación crediticia.
      + Debt_ratio alto: Significa que una persona tiene una cantidad significativa de deuda en comparación con sus ingresos disponibles o             activos.
        + Mayor riesgo de incumplimiento.
        + Menor capacidad de endeudamiento adicional.
        + Posible impacto negativo en la calificación crediticia.
          
           Asignando los siguientes valores:
           + Riesgo_alto: Cuando debt_ratio > 60% (0.6)
           + Riesgo_medio: Cuando debt_ratio > 40-60% (0.4- 0.6)
           + Riesgo_bajo: Cuando debt_ratio < 30% (0.3)
             
  + Incumplimiento_unsec_line: La segmentación de riesgo se utiliza para clasificar a los clientes en diferentes niveles según su solvencia         crediticia. Aquí están los tres niveles comunes:
      + Alto Riesgo: Clientes con una mayor probabilidad de incumplimiento o retraso en los pagos.
      + Medio Riesgo: Clientes con un riesgo moderado, que pueden cumplir con sus obligaciones, pero con cierta incertidumbre.
      + Bajo Riesgo: Clientes con una alta probabilidad de cumplir con sus pagos de manera puntual23.
      
          Asignando los siguientes valores:
           + Riesgo_alto: Cuando debt_ratio > 60% (0.6)
           + Riesgo_medio: Cuando debt_ratio > 40-60% (0.4- 0.6)
           + Riesgo_bajo: Cuando debt_ratio < 30% (0.3)

Cosntulta:
   ~~~
     -- Identificar Nulos
  SELECT
    'user_id' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'more_90_days_overdue' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(more_90_days_overdue AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'using_lines_not_secured_personal_assets' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(using_lines_not_secured_personal_assets AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'number_times_delayed_payment_loan_30_59_days' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(number_times_delayed_payment_loan_30_59_days AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'debt_ratio' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(debt_ratio AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'number_times_delayed_payment_loan_60_89_days' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(number_times_delayed_payment_loan_60_89_days AS INT64) IS NULL; 
    
  -- Identificar y manejar valores duplicados
  WITH
    base_data AS (
    SELECT
      user_id,
      more_90_days_overdue,
      using_lines_not_secured_personal_assets,
      number_times_delayed_payment_loan_30_59_days,
      debt_ratio,
      number_times_delayed_payment_loan_60_89_days
    FROM
      `riesgo-relativo-p3.Data_Set.loans_detail` ),
    duplicado AS (
    SELECT
      user_id,
      more_90_days_overdue,
      using_lines_not_secured_personal_assets,
      number_times_delayed_payment_loan_30_59_days,
      debt_ratio,
      number_times_delayed_payment_loan_60_89_days,
      COUNT(*) AS duplicado
    FROM
      base_data
    GROUP BY
      user_id,
      more_90_days_overdue,
      using_lines_not_secured_personal_assets,
      number_times_delayed_payment_loan_30_59_days,
      debt_ratio,
      number_times_delayed_payment_loan_60_89_days
    HAVING
      COUNT(*) > 1 )
  SELECT
    user_id,
    more_90_days_overdue,
    using_lines_not_secured_personal_assets,
    number_times_delayed_payment_loan_30_59_days,
    debt_ratio,
    number_times_delayed_payment_loan_60_89_days,
    duplicado
  FROM
    duplicado; 
    
  -- Identificar y manejar datos fuera del alcance del análisis 
   -- Correlación y Desviación Estándar entre number_times_delayed_payment_loan_30_59_days y number_times_delayed_payment_loan_60_89_days
  SELECT
    CORR(number_times_delayed_payment_loan_30_59_days, number_times_delayed_payment_loan_60_89_days) AS correlation_30_59_60_89,
    STDDEV(number_times_delayed_payment_loan_30_59_days) AS stddev_30_59,
    STDDEV(number_times_delayed_payment_loan_60_89_days) AS stddev_60_89
  FROM `riesgo-relativo-p3.Data_Set.loans_detail`;
  
  -- Correlación y Desviación Estándar entre more_90_days_overdue y number_times_delayed_payment_loan_60_89_days
  SELECT
    CORR(more_90_days_overdue, number_times_delayed_payment_loan_60_89_days) AS correlation_90_60_89,
    STDDEV(more_90_days_overdue) AS stddev_90,
    STDDEV(number_times_delayed_payment_loan_60_89_days) AS stddev_60_89
  FROM `riesgo-relativo-p3.Data_Set.loans_detail`;
  
  -- Correlación y Desviación Estándar entre more_90_days_overdue y number_times_delayed_payment_loan_30_59_days
  SELECT
    CORR(more_90_days_overdue, number_times_delayed_payment_loan_30_59_days) AS correlation_90_30_59,
    STDDEV(more_90_days_overdue) AS stddev_90,
    STDDEV(number_times_delayed_payment_loan_30_59_days) AS stddev_30_59
  FROM `riesgo-relativo-p3.Data_Set.loans_detail`;
    
    -- Identificar y manejar datos inconsistentes en variables categóricas
  SELECT
    user_id,
    more_90_days_overdue,
    using_lines_not_secured_personal_assets,
    number_times_delayed_payment_loan_30_59_days,
    debt_ratio,
    number_times_delayed_payment_loan_60_89_days,
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
    OR SAFE_CAST(more_90_days_overdue AS INT64) IS NULL
    OR SAFE_CAST(using_lines_not_secured_personal_assets AS INT64) IS NULL
  OR SAFE_CAST(number_times_delayed_payment_loan_30_59_days AS INT64) IS NULL
  OR SAFE_CAST(debt_ratio AS INT64) IS NULL
  OR SAFE_CAST(number_times_delayed_payment_loan_60_89_days AS INT64) IS NULL;
  
  #Identificar y manejar datos discrepantes en variables numéricas (OUTLIERS)
  WITH ordered_values AS (
    SELECT
      more_90_days_overdue,
      using_lines_not_secured_personal_assets,
      number_times_delayed_payment_loan_30_59_days,
      debt_ratio,
      number_times_delayed_payment_loan_60_89_days,
      NTILE(4) OVER (ORDER BY more_90_days_overdue) AS quartile_90_days,
      NTILE(4) OVER (ORDER BY using_lines_not_secured_personal_assets) AS quartile_unsecured,
      NTILE(4) OVER (ORDER BY number_times_delayed_payment_loan_30_59_days) AS quartile_30_59,
      NTILE(4) OVER (ORDER BY debt_ratio) AS quartile_debt_ratio,
      NTILE(4) OVER (ORDER BY number_times_delayed_payment_loan_60_89_days) AS quartile_60_89
    FROM
      `riesgo-relativo-p3.Data_Set.loans_detail`
  ),
  quartiles AS (
    SELECT
      MAX(CASE WHEN quartile_90_days = 1 THEN more_90_days_overdue ELSE NULL END) AS Q1_90_days,
      MAX(CASE WHEN quartile_90_days = 2 THEN more_90_days_overdue ELSE NULL END) AS Q2_90_days,
      MAX(CASE WHEN quartile_90_days = 3 THEN more_90_days_overdue ELSE NULL END) AS Q3_90_days,
      MAX(CASE WHEN quartile_90_days = 4 THEN more_90_days_overdue ELSE NULL END) AS Q4_90_days,
      
      MAX(CASE WHEN quartile_unsecured = 1 THEN using_lines_not_secured_personal_assets ELSE NULL END) AS Q1_unsecured,
      MAX(CASE WHEN quartile_unsecured = 2 THEN using_lines_not_secured_personal_assets ELSE NULL END) AS Q2_unsecured,
      MAX(CASE WHEN quartile_unsecured = 3 THEN using_lines_not_secured_personal_assets ELSE NULL END) AS Q3_unsecured,
      MAX(CASE WHEN quartile_unsecured = 4 THEN using_lines_not_secured_personal_assets ELSE NULL END) AS Q4_unsecured,
      
      MAX(CASE WHEN quartile_30_59 = 1 THEN number_times_delayed_payment_loan_30_59_days ELSE NULL END) AS Q1_30_59,
      MAX(CASE WHEN quartile_30_59 = 2 THEN number_times_delayed_payment_loan_30_59_days ELSE NULL END) AS Q2_30_59,
      MAX(CASE WHEN quartile_30_59 = 3 THEN number_times_delayed_payment_loan_30_59_days ELSE NULL END) AS Q3_30_59,
      MAX(CASE WHEN quartile_30_59 = 4 THEN number_times_delayed_payment_loan_30_59_days ELSE NULL END) AS Q4_30_59,
      
      MAX(CASE WHEN quartile_debt_ratio = 1 THEN debt_ratio ELSE NULL END) AS Q1_debt_ratio,
      MAX(CASE WHEN quartile_debt_ratio = 2 THEN debt_ratio ELSE NULL END) AS Q2_debt_ratio,
      MAX(CASE WHEN quartile_debt_ratio = 3 THEN debt_ratio ELSE NULL END) AS Q3_debt_ratio,
      MAX(CASE WHEN quartile_debt_ratio = 4 THEN debt_ratio ELSE NULL END) AS Q4_debt_ratio,
      
      MAX(CASE WHEN quartile_60_89 = 1 THEN number_times_delayed_payment_loan_60_89_days ELSE NULL END) AS Q1_60_89,
      MAX(CASE WHEN quartile_60_89 = 2 THEN number_times_delayed_payment_loan_60_89_days ELSE NULL END) AS Q2_60_89,
      MAX(CASE WHEN quartile_60_89 = 3 THEN number_times_delayed_payment_loan_60_89_days ELSE NULL END) AS Q3_60_89,
      MAX(CASE WHEN quartile_60_89 = 4 THEN number_times_delayed_payment_loan_60_89_days ELSE NULL END) AS Q4_60_89
    FROM
      ordered_values
  ),
  iqr_calc AS (
    SELECT
      Q1_90_days, Q3_90_days, Q3_90_days - Q1_90_days AS IQR_90_days, Q2_90_days AS median_90_days,
      Q1_unsecured, Q3_unsecured, Q3_unsecured - Q1_unsecured AS IQR_unsecured, Q2_unsecured AS median_unsecured,
      Q1_30_59, Q3_30_59, Q3_30_59 - Q1_30_59 AS IQR_30_59, Q2_30_59 AS median_30_59,
      Q1_debt_ratio, Q3_debt_ratio, Q3_debt_ratio - Q1_debt_ratio AS IQR_debt_ratio, Q2_debt_ratio AS median_debt_ratio,
      Q1_60_89, Q3_60_89, Q3_60_89 - Q1_60_89 AS IQR_60_89, Q2_60_89 AS median_60_89
    FROM
      quartiles
  )
  SELECT
    COUNT(CASE WHEN more_90_days_overdue < (SELECT Q1_90_days FROM iqr_calc) - 1.5 * (SELECT IQR_90_days FROM iqr_calc) OR more_90_days_overdue > (SELECT Q3_90_days FROM iqr_calc) + 1.5 * (SELECT IQR_90_days FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_90_days,
    COUNT(CASE WHEN using_lines_not_secured_personal_assets < (SELECT Q1_unsecured FROM iqr_calc) - 1.5 * (SELECT IQR_unsecured FROM iqr_calc) OR using_lines_not_secured_personal_assets > (SELECT Q3_unsecured FROM iqr_calc) + 1.5 * (SELECT IQR_unsecured FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_unsecured,
    COUNT(CASE WHEN number_times_delayed_payment_loan_30_59_days < (SELECT Q1_30_59 FROM iqr_calc) - 1.5 * (SELECT IQR_30_59 FROM iqr_calc) OR number_times_delayed_payment_loan_30_59_days > (SELECT Q3_30_59 FROM iqr_calc) + 1.5 * (SELECT IQR_30_59 FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_30_59,
    COUNT(CASE WHEN debt_ratio < (SELECT Q1_debt_ratio FROM iqr_calc) - 1.5 * (SELECT IQR_debt_ratio FROM iqr_calc) OR debt_ratio > (SELECT Q3_debt_ratio FROM iqr_calc) + 1.5 * (SELECT IQR_debt_ratio FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_debt_ratio,
    COUNT(CASE WHEN number_times_delayed_payment_loan_60_89_days < (SELECT Q1_60_89 FROM iqr_calc) - 1.5 * (SELECT IQR_60_89 FROM iqr_calc) OR number_times_delayed_payment_loan_60_89_days > (SELECT Q3_60_89 FROM iqr_calc) + 1.5 * (SELECT IQR_60_89 FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_60_89
  FROM
    `riesgo-relativo-p3.Data_Set.loans_detail`;
  
  # Comprobar y cambiar tipo de dato (Se corrobora que sea INT64)
  
  #Crear nuevas variables 
  WITH datos_filtrados AS (
    SELECT 
      user_id,
      debt_ratio,
      using_lines_not_secured_personal_assets,
      -- Filtrar valores mayores a 20
      CASE 
        WHEN number_times_delayed_payment_loan_30_59_days > 20 THEN NULL
        ELSE number_times_delayed_payment_loan_30_59_days
      END AS number_times_delayed_payment_loan_30_59_days,
      
      CASE 
        WHEN number_times_delayed_payment_loan_60_89_days > 20 THEN NULL
        ELSE number_times_delayed_payment_loan_60_89_days
      END AS number_times_delayed_payment_loan_60_89_days,
      
      CASE 
        WHEN more_90_days_overdue > 20 THEN NULL
        ELSE more_90_days_overdue
      END AS more_90_days_overdue
    FROM
      `proyecto2-super-caja.Data_set.loans_detail`
  ),
  
  datos_suma AS (
    SELECT
      user_id,
      debt_ratio,
      using_lines_not_secured_personal_assets,
      COALESCE(number_times_delayed_payment_loan_30_59_days, 0) +
      COALESCE(number_times_delayed_payment_loan_60_89_days, 0) +
      COALESCE(more_90_days_overdue, 0) AS delay_30_59_90
    FROM
      datos_filtrados
  ),
  
  segmentacion AS (
    SELECT
      user_id,
      debt_ratio,
      using_lines_not_secured_personal_assets,
      delay_30_59_90,
      
      -- Segmentación Basada en Debt Ratio
      CASE
        WHEN debt_ratio < 0.30 THEN 'Bajo Riesgo'
        WHEN debt_ratio >= 0.30 AND debt_ratio <= 0.60 THEN 'Medio Riesgo'
        ELSE 'Alto Riesgo'
      END AS Incumplimiento_debt_ratio,
      
      -- Segmentación Basada en Uso de Líneas de Crédito No Aseguradas
      CASE
        WHEN using_lines_not_secured_personal_assets < 0.30 THEN 'Bajo Riesgo'
        WHEN using_lines_not_secured_personal_assets >= 0.30 AND using_lines_not_secured_personal_assets <= 0.60 THEN 'Medio Riesgo'
        ELSE 'Alto Riesgo'
      END AS Incumplimiento_unsec_line,
      
      -- Segmentación Basada en la Nueva Columna de Suma
      CASE
        WHEN delay_30_59_90 = 0 THEN 'Ningún Retraso'
        WHEN delay_30_59_90 > 0 AND delay_30_59_90 <= 5 THEN 'Poco Retraso'
        ELSE 'Muchos Retrasos'
      END AS segmentacion_delay
    FROM
      datos_suma
  )
  
  -- Seleccionar los resultados finales
  SELECT
    user_id,
    debt_ratio,
    Incumplimiento_debt_ratio,
    using_lines_not_secured_personal_assets,
    Incumplimiento_unsec_line,
    delay_30_59_90,
    segmentacion_delay
  FROM
    segmentacion
  ORDER BY
    user_id;
   ~~~

2.- Tabla Loans_detail:

  Nulos:
    + loan_id_ 0 nulls
    + user_id: 0 nulls
    + loan_type: 0 nulls
  Duplicados:
    + loan_id_ 0
    + user_id: 305335 No se imputan, ni se eliminan ya que se aprecia que los duplicados son por que los clientes tienen varios tipos de credito.
    + loan_type: 0
  Outliers:
    + loan_id_ 0 
    + user_id: 0 
    + loan_type: 0 
  Cambiar tipo de dato:
    + loan_id_ 0 
    + user_id: 0 
    + loan_type: Se realiza la estandarización en minusculas de la palabra "real estate" asi como de la palabra "others".
  Nuevas Variables:
  + real_estate_loan_type: Cantidad de prestamos por cliente
  + others_loan_type: Cantidad de prestamos por cliente
  + total_loan_type: Total de prestamos por cliente.

Consulta: 
  ~~~
  -- Identificar Nulos
  SELECT
    'user_id' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
  UNION ALL
  SELECT
    'loan_id' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`
  WHERE
    SAFE_CAST(loan_id AS INT64) IS NULL
  UNION ALL
  SELECT
    'loan_type' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`
  WHERE
    SAFE_CAST(loan_type AS INT64) IS NULL;
  
  -- Identificar y manejar valores duplicados
  WITH base_data AS (
    SELECT
      user_id,
      loan_id,
      loan_type
    FROM
      `riesgo-relativo-p3.Data_Set.loans_outstanding`
  ),
  duplicado AS (
    SELECT
      user_id,
      loan_id,
      loan_type,
      COUNT(*) AS duplicado
    FROM
      base_data
    GROUP BY
      user_id,
      loan_id,
      loan_type
    HAVING
      COUNT(*) > 1
  )
  SELECT
    user_id,
    loan_id,
    loan_type,
    duplicado
  FROM
    duplicado;
  
  -- Identificar y manejar datos inconsistentes en variables categóricas
  SELECT
    user_id,
    loan_id,
    loan_type
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
    OR SAFE_CAST(loan_id AS INT64) IS NULL;
  
  -- Identificar y manejar datos discrepantes en variables numéricas (OUTLIERS)
  WITH ordered_values AS (
    SELECT
      loan_id,
      NTILE(4) OVER (ORDER BY loan_id) AS quartile_loan_id
    FROM
      `riesgo-relativo-p3.Data_Set.loans_outstanding`
  ),
  quartiles AS (
    SELECT
      MAX(CASE WHEN quartile_loan_id = 1 THEN loan_id ELSE NULL END) AS Q1_loan,
      MAX(CASE WHEN quartile_loan_id = 2 THEN loan_id ELSE NULL END) AS Q2_loan,
      MAX(CASE WHEN quartile_loan_id = 3 THEN loan_id ELSE NULL END) AS Q3_loan,
      MAX(CASE WHEN quartile_loan_id = 4 THEN loan_id ELSE NULL END) AS Q4_loan
    FROM
      ordered_values
  ),
  iqr_calc AS (
    SELECT
      Q1_loan,
      Q3_loan,
      Q3_loan - Q1_loan AS IQR_loan,
      Q2_loan AS median_loan
    FROM
      quartiles
  )
  SELECT
    COUNT(CASE WHEN loan_id < (SELECT Q1_loan FROM iqr_calc) - 1.5 * (SELECT IQR_loan FROM iqr_calc) OR loan_id > (SELECT Q3_loan FROM iqr_calc) + 1.5 * (SELECT IQR_loan FROM iqr_calc) THEN 1 ELSE NULL END) AS outliers_loan
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`;
  
  -- Comprobar y cambiar tipo de dato
  -- LOWER (convierte las letras mayúsculas en minúsculas)
  SELECT 
    LOWER(
      REPLACE(loan_type, 'other', 'others')
    ) AS normalized_loan_type
  FROM 
    `riesgo-relativo-p3.Data_Set.loans_outstanding`;
  
  -- Crear nuevas variables
  SELECT
    user_id,
    SUM(CASE
        WHEN loan_type = 'real estate' THEN 1
        ELSE 0
    END) AS real_estate_loan_type,
    SUM(CASE
        WHEN loan_type = 'other' THEN 1
        ELSE 0
    END) AS others_loan_type,
    SUM(CASE
        WHEN loan_type = 'real estate' THEN 1
        ELSE 0
    END) + SUM(CASE
        WHEN loan_type = 'other' THEN 1
        ELSE 0
    END) AS total_loan_type
  FROM
    `riesgo-relativo-p3.Data_Set.loans_outstanding`
  GROUP BY
    user_id;
  
  -- Seleccionar los resultados finales
  SELECT
    user_id,
    real_estate_loan_type,
    others_loan_type,
    total_loan_type
  FROM
    (
      SELECT
        user_id,
        SUM(CASE
            WHEN loan_type = 'real estate' THEN 1
            ELSE 0
        END) AS real_estate_loan_type,
        SUM(CASE
            WHEN loan_type = 'other' THEN 1
            ELSE 0
        END) AS others_loan_type,
        SUM(CASE
            WHEN loan_type = 'real estate' THEN 1
            ELSE 0
        END) + SUM(CASE
            WHEN loan_type = 'other' THEN 1
            ELSE 0
        END) AS total_loan_type
      FROM
        `riesgo-relativo-p3.Data_Set.loans_outstanding`
      GROUP BY
        user_id
    ) AS segmentacion
  ORDER BY
    user_id;
  ~~~  

4.- Tabla user_info:

  Nulos:
  + user_id: 0 nulls
  + age: 0 nulls
  + sex: 0 nulls
  + last_mont_salary: 7199 nulls (valor 0)
  + number_dependents: 943 nulls
  Duplicados:
  + user_id: 0 
  + age: 0 
  + sex: 0 
  + last_mont_salary: 0 
  + number_dependents: 0
  Outliers:
  + age: 10 Outliders se determina eliminar las edades mayores a 96 años
     > [!NOTE]
     > ![5](https://github.com/user-attachments/assets/0bf0e07c-4cd4-4e7c-827c-fa45f06afbaf)

     Vista Outliders:
     > [!NOTE]
     > ![6](https://github.com/user-attachments/assets/1726400c-4d3b-423a-a1aa-bff1df92568a)
     
   + number_dependents: 3,230 Outliers al no tener información clara sobre cual seria la cantidad limite de numero de dependientes por user_id se determina dejar los valores tal como estan.
     > [!NOTE]
     > ![12](https://github.com/user-attachments/assets/f1c2509e-9f2b-4467-b3a3-9701e1986f05)

      Vista Outliders:
     > [!NOTE]
     >![14](https://github.com/user-attachments/assets/665583bf-d935-4b6f-ad9d-8abb6ff6917d)
     
   + last_month_salary: 1170 Outliers  se llega a la decisión realizar la imputación de los datos mediante el reemplazo de los valores atípicos con el valor promedio de los dos segmentos (Sueldo bajo)          (Sueldo alto)
     > [!NOTE]
     > ![10](https://github.com/user-attachments/assets/73302ffc-f545-4ce8-87a2-7553148c4a71)

     VistaLast_month_salary desglosados:
     > [!NOTE]
     >![13](https://github.com/user-attachments/assets/3aa70616-ca01-4445-863f-07c12ecc77c5)

  Nuevas Variables:
  + Generational_group: Segmentación por grupo generacional.
  + birth_year: año de nacimiento.

Consulta:
   ~~~  
   -- Identificar Nulos
  SELECT
    'user_id' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`
  WHERE
    SAFE_CAST(user_id AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'age' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`
  WHERE
    SAFE_CAST(age AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'last_month_salary' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`
  WHERE
    SAFE_CAST(last_month_salary AS INT64) IS NULL
  UNION ALL
    --
  SELECT
    'number_dependents' AS column_name,
    COUNT(*) AS null_count
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`
  WHERE
    SAFE_CAST(number_dependents AS INT64) IS NULL; -- Identificar y manejar valores duplicados
  WITH
    base_data AS (
    SELECT
      user_id,
      age,
      sex,
      last_month_salary,
      number_dependents,
    FROM
      `riesgo-relativo-p3.Data_Set.user_info` ),
    duplicado AS (
    SELECT
      user_id,
      age,
      sex,
      last_month_salary,
      number_dependents,
      COUNT(*) AS duplicado
    FROM
      base_data
    GROUP BY
      user_id,
      age,
      sex,
      last_month_salary,
      number_dependents
    HAVING
      COUNT(*) > 1 )
  SELECT
    user_id,
    age,
    sex,
    last_month_salary,
    number_dependents,
    duplicado
  FROM
    duplicado; -- Identificar y manejar datos fuera del alcance del análisis (NO aplica) -- Identificar y manejar datos inconsistentes en variables categóricas
  SELECT
    age,
    sex,
    last_month_salary,
    number_dependents,
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`
  WHERE
    SAFE_CAST(age AS INT64) IS NULL
    OR SAFE_CAST(sex AS INT64) IS NULL
    OR SAFE_CAST(last_month_salary AS INT64) IS NULL
    OR SAFE_CAST(number_dependents AS INT64) IS NULL; #Identificar y manejar datos discrepantes en variables numéricas (OUTLIERS)
  WITH
    ordered_values AS (
    SELECT
      age,
      sex,
      last_month_salary,
      number_dependents,
      NTILE(4) OVER (ORDER BY age) AS quartile_age,
      NTILE(4) OVER (ORDER BY last_month_salary) AS quartile_last_month_salary,
      NTILE(4) OVER (ORDER BY number_dependents) AS quartile_number_dependents
    FROM
      `riesgo-relativo-p3.Data_Set.user_info` ),
    quartiles AS (
    SELECT
      MAX(CASE
          WHEN quartile_age = 1 THEN age
          ELSE NULL
      END
        ) AS Q1_age,
      MAX(CASE
          WHEN quartile_age = 2 THEN age
          ELSE NULL
      END
        ) AS Q2_age,
      MAX(CASE
          WHEN quartile_age = 3 THEN age
          ELSE NULL
      END
        ) AS Q3_age,
      MAX(CASE
          WHEN quartile_age = 4 THEN age
          ELSE NULL
      END
        ) AS Q4_age,
      MAX(CASE
          WHEN quartile_last_month_salary = 1 THEN last_month_salary
          ELSE NULL
      END
        ) AS Q1_last_month_salary,
      MAX(CASE
          WHEN quartile_last_month_salary = 2 THEN last_month_salary
          ELSE NULL
      END
        ) AS Q2_last_month_salary,
      MAX(CASE
          WHEN quartile_last_month_salary = 3 THEN last_month_salary
          ELSE NULL
      END
        ) AS Q3_last_month_salary,
      MAX(CASE
          WHEN quartile_last_month_salary = 4 THEN last_month_salary
          ELSE NULL
      END
        ) AS Q4_last_month_salary,
      MAX(CASE
          WHEN quartile_number_dependents = 1 THEN number_dependents
          ELSE NULL
      END
        ) AS Q1_number_dependents,
      MAX(CASE
          WHEN quartile_number_dependents = 2 THEN number_dependents
          ELSE NULL
      END
        ) AS Q2_number_dependents,
      MAX(CASE
          WHEN quartile_number_dependents = 3 THEN number_dependents
          ELSE NULL
      END
        ) AS Q3_number_dependents,
      MAX(CASE
          WHEN quartile_number_dependents = 4 THEN number_dependents
          ELSE NULL
      END
        ) AS Q4_number_dependents
    FROM
      ordered_values ),
    iqr_calc AS (
    SELECT
      Q1_age,
      Q3_age,
      Q3_age - Q1_age AS IQR_age,
      Q2_age AS median_age,
      Q1_last_month_salary,
      Q3_last_month_salary,
      Q3_last_month_salary - Q1_last_month_salary AS IQR_last_month_salary,
      Q2_last_month_salary AS median_last_month_salary,
      Q1_number_dependents,
      Q3_number_dependents,
      Q3_number_dependents - Q1_number_dependents AS IQR_number_dependents,
      Q2_number_dependents AS median_number_dependents
    FROM
      quartiles )
  SELECT
    COUNT(CASE
        WHEN age < ( SELECT Q1_age FROM iqr_calc) - 1.5 * ( SELECT IQR_age FROM iqr_calc) OR age > ( SELECT Q3_age FROM iqr_calc) + 1.5 * ( SELECT IQR_age FROM iqr_calc) THEN 1
        ELSE NULL
    END
      ) AS outliers_age,
    COUNT(CASE
        WHEN last_month_salary < ( SELECT Q1_last_month_salary FROM iqr_calc) - 1.5 * ( SELECT IQR_last_month_salary FROM iqr_calc) OR last_month_salary > ( SELECT Q3_last_month_salary FROM iqr_calc) + 1.5 * ( SELECT IQR_last_month_salary FROM iqr_calc) THEN 1
        ELSE NULL
    END
      ) AS outliers_last_month_salary,
    COUNT(CASE
        WHEN number_dependents < ( SELECT Q1_number_dependents FROM iqr_calc) - 1.5 * ( SELECT IQR_number_dependents FROM iqr_calc) OR number_dependents > ( SELECT Q3_number_dependents FROM iqr_calc) + 1.5 * ( SELECT IQR_number_dependents FROM iqr_calc) THEN 1
        ELSE NULL
    END
      ) AS outliers_number_dependents
  FROM
    `riesgo-relativo-p3.Data_Set.user_info`;
   ~~~  

5.- Unión de tablas

Consulta:
   ~~~
-- Crear una tabla temporal con la moda de last_month_salary por default_flag
WITH salary_mode AS (
  -- Subconsulta para calcular la moda de last_month_salary por default_flag
  SELECT
    default_flag,
    last_month_salary AS mode_salary
  FROM (
    SELECT
      default_flag,
      last_month_salary,
      COUNT(*) AS frequency,
      ROW_NUMBER() OVER (PARTITION BY default_flag ORDER BY COUNT(*) DESC, last_month_salary) AS rn
    FROM
      `riesgo-relativo-p3.Data_Set.Data_Set`
    WHERE
      last_month_salary IS NOT NULL AND last_month_salary > 0
    GROUP BY
      default_flag, last_month_salary
  )
  WHERE
    rn = 1
)

-- Imputar valores de last_month_salary con la moda calculada para cada default_flag y reemplazar ceros y nulos
SELECT
  DF.user_id,
  DF.default_flag,
  UI.age,
  UI.generational_group,
  UI.sex,
  COALESCE(
    CASE
      WHEN UI.last_month_salary = 0 THEN salary_mode.mode_salary
      ELSE UI.last_month_salary
    END,
    CASE
      WHEN DF.default_flag = 0 THEN 5000
      WHEN DF.default_flag = 1 THEN 2500
      ELSE NULL
    END
  ) AS last_month_salary,
  COALESCE(UI.number_dependents, 0) AS number_dependents, 
  LD.debt_ratio,
  LD.Incumplimiento_debt_ratio,
  LD.using_lines_not_secured_personal_assets,
  LD.Incumplimiento_unsec_line,
  COALESCE(LT.real_estate_loan_type, 0) AS real_estate_loan_type,
  COALESCE(LT.others_loan_type, 0) AS others_loan_type,
  COALESCE(LT.total_loan_type, 0) AS total_loan_type,
  LD.delay_30_59_90,
  LD.segmentacion_delay
FROM
  `riesgo-relativo-p3.Data_Set.default` AS DF
LEFT JOIN
  `riesgo-relativo-p3.Data_Set.user_info1` AS UI
ON
  DF.user_id = UI.user_id
LEFT JOIN
  `riesgo-relativo-p3.Data_Set.Loans_detail` AS LD
ON
  DF.user_id = LD.user_id
LEFT JOIN
  `riesgo-relativo-p3.Data_Set.Loans_type` AS LT
ON
  DF.user_id = LT.user_id
LEFT JOIN
  salary_mode
ON
  DF.default_flag = salary_mode.default_flag
ORDER BY
  DF.user_id;
~~~

### 1.2 Análisis exploratorio

1.- Aplicar medidas de tendencia central

 > [!NOTE]
 >![24](https://github.com/user-attachments/assets/534c7a2c-8b25-4e21-a076-56e0629c361e)

2.- Visualizar la distribuiciòn 

 > [!NOTE]
 >![25](https://github.com/user-attachments/assets/437361bb-1423-4b31-aa0b-4d95db9e4748)

Interpretaciones:

 Gráfica de “Others_loan_type”:
 
   +Baby Boomers: Tienen el valor más bajo en esta categoría.
   + Generación X: Un poco más alto que los Baby Boomers, pero aún bajo.
   + Generación Z: Similar a la Generación X.
   + Millennials: Tienen el valor más alto en esta categoría, aunque sigue siendo relativamente bajo.
 Gráfica de “Real_estate_loan_type”:
   + Baby Boomers: Tienen un valor moderado.
   + Generación X: Similar a los Baby Boomers.
   + Generación Z: Un poco más alto que las generaciones anteriores.
   + Millennials: Tienen el valor más alto, superando los 20.
     
 Observaciones generales: los Millennials parecen tener una mayor participación en ambos tipos de préstamos, especialmente en los préstamos inmobiliarios.
  
3.- Calcular cuartiles, deciles o percentiles de los grupos de malos pagadores.

Se analizan las variables que podrian ayudarnos a definir como es un cliente mal pagador.

   Age
 > [!NOTE] 
 > ![27](https://github.com/user-attachments/assets/1f285cf1-96a6-4c1a-b801-2fc52ca9b0af)

   Last_salary_month
 > [!NOTE] 
 >![28](https://github.com/user-attachments/assets/ade76e63-4324-4486-8c8b-3486a4c0fa3e)

   Number_dependents
 > [!NOTE] 
 > ![29](https://github.com/user-attachments/assets/94685706-7e1c-4c33-9ace-104ce6c7d094)

   Debt_ratio
 > [!NOTE] 
 >![30](https://github.com/user-attachments/assets/c76804ee-e332-4913-8463-5dc496db4868)

   Using_lines_not_secured_personal_asset
 > [!NOTE] 
 > ![31](https://github.com/user-attachments/assets/179899f6-3fe3-443b-80a1-7166576988a7)

  number_times_delayed_payment_loan_30_59_days
 > [!NOTE] 
 > ![](Imagenes/32.png)

   
