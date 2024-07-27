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
 > ![Captura de pantalla 2024-07-26 174938](https://github.com/user-attachments/assets/ed01baf3-8a58-419b-bc43-d53a255d165e)

   Last_salary_month
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 174956](https://github.com/user-attachments/assets/1622c21a-ca74-4c14-bcef-180d51ffc4bb)

   Number_dependents
 > [!NOTE] 
 > ![Captura de pantalla 2024-07-26 175010](https://github.com/user-attachments/assets/61aed98b-957c-4556-8ba7-2c48831f08d9)

   Debt_ratio
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 175030](https://github.com/user-attachments/assets/beefac87-32d7-47ee-a8d3-26cbf69b7aa4)

   Using_lines_not_secured_personal_asset
 > [!NOTE] 
 > ![Captura de pantalla 2024-07-26 175049](https://github.com/user-attachments/assets/70fb3835-6086-45b3-a01b-bafe4dfb3dd5)

  number_times_delayed_payment_loan_30_59_days
 > [!NOTE] 
 > ![Captura de pantalla 2024-07-26 175339](https://github.com/user-attachments/assets/2d2992a7-9c57-4589-a469-dc8a8e6ae63d)

 Consulta:

   ~~~
 #CALCULAR LOS CUARTILES DE MALOS PAGADORES POR VARIABLE
WITH base_data AS (
    SELECT 
        age,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
),
---Calcular los cuartiles dependiendo la edad.
quartiles AS (
    SELECT 
        age,
        default_flag,
        NTILE(4) OVER (ORDER BY age) AS age_quartile
    FROM base_data ---datos de donde provienen las variables
),
-- Calcula el número total de malos pagadores 
quartile_risk AS (
    SELECT 
        age_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers,
    FROM quartiles
    GROUP BY age_quartile
),
---rango de edad (mínimo y máximo) para cada cuartil.
quartile_ranges AS (
    SELECT
        age_quartile,
        MIN(age) AS min_age,
        MAX(age) AS max_age
    FROM quartiles
    GROUP BY age_quartile
)
SELECT 
    q.age_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_age,
    r.max_age
FROM quartile_risk q
JOIN quartile_ranges r
ON q.age_quartile = r.age_quartile
ORDER BY age_quartile ASC;

#CALCULAR LOS CUARTILES DE MALOS PAGADORES LAST_MONTH
WITH base_data AS (
    SELECT 
        last_month_salary,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
),
---Calcular los cuartiles dependiendo la edad.
quartiles AS (
    SELECT 
        last_month_salary,
        default_flag,
        NTILE(4) OVER (ORDER BY last_month_salary) AS last_month_salary_quartile
    FROM base_data ---datos de donde provienen las variables
),
-- Calcula el número total de malos pagadores 
quartile_risk AS (
    SELECT 
        last_month_salary_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers,
    FROM quartiles
    GROUP BY last_month_salary_quartile
),
---rango de edad (mínimo y máximo) para cada cuartil.
quartile_ranges AS (
    SELECT
        last_month_salary_quartile,
        MIN(last_month_salary) AS min_last_month_salary,
        MAX(last_month_salary) AS max_last_month_salary
    FROM quartiles
    GROUP BY last_month_salary_quartile
)
SELECT 
    q.last_month_salary_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_last_month_salary,
    r.max_last_month_salary
FROM quartile_risk q
JOIN quartile_ranges r
ON q.last_month_salary_quartile = r.last_month_salary_quartile
ORDER BY last_month_salary_quartile ASC;

#CALCULAR LOS CUARTILES DE MALOS PAGADORES NUMBER_DEPENDENTS
WITH base_data AS (
    SELECT 
        number_dependents,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
),
---Calcular los cuartiles dependiendo la edad.
quartiles AS (
    SELECT 
        number_dependents,
        default_flag,
        NTILE(4) OVER (ORDER BY number_dependents) AS number_dependents_quartile
    FROM base_data ---datos de donde provienen las variables
),
-- Calcula el número total de malos pagadores 
quartile_risk AS (
    SELECT 
        number_dependents_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers,
    FROM quartiles
    GROUP BY number_dependents_quartile
),
---rango de edad (mínimo y máximo) para cada cuartil.
quartile_ranges AS (
    SELECT
        number_dependents_quartile,
        MIN(number_dependents) AS min_number_dependents,
        MAX(number_dependents) AS max_number_dependents
    FROM quartiles
    GROUP BY number_dependents_quartile
)
SELECT 
    q.number_dependents_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_number_dependents,
    r.max_number_dependents
FROM quartile_risk q
JOIN quartile_ranges r
ON q.number_dependents_quartile = r.number_dependents_quartile
ORDER BY number_dependents_quartile ASC;

#CALCULAR LOS CUARTILES DE MALOS PAGADORES DEBT_RATIO
WITH base_data AS (
    SELECT 
        debt_ratio,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
),
---Calcular los cuartiles dependiendo la edad.
quartiles AS (
    SELECT 
        debt_ratio,
        default_flag,
        NTILE(4) OVER (ORDER BY debt_ratio) AS debt_ratio_quartile
    FROM base_data ---datos de donde provienen las variables
),
-- Calcula el número total de malos pagadores 
quartile_risk AS (
    SELECT 
        debt_ratio_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers,
    FROM quartiles
    GROUP BY debt_ratio_quartile
),
---rango de edad (mínimo y máximo) para cada cuartil.
quartile_ranges AS (
    SELECT
        debt_ratio_quartile,
        MIN(debt_ratio) AS min_debt_ratio,
        MAX(debt_ratio) AS max_debt_ratio
    FROM quartiles
    GROUP BY debt_ratio_quartile
)
SELECT 
    q.debt_ratio_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_debt_ratio,
    r.max_debt_ratio
FROM quartile_risk q
JOIN quartile_ranges r
ON q.debt_ratio_quartile = r.debt_ratio_quartile
ORDER BY debt_ratio_quartile ASC;

#CALCULAR LOS CUARTILES DE MALOS PAGADORES using_lines_not_secured_personal_assets
WITH base_data AS (
    SELECT 
        using_lines_not_secured_personal_assets,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
),
---Calcular los cuartiles dependiendo la edad.
quartiles AS (
    SELECT 
        using_lines_not_secured_personal_assets,
        default_flag,
        NTILE(4) OVER (ORDER BY using_lines_not_secured_personal_assets) AS using_lines_not_secured_personal_assets_quartile
    FROM base_data ---datos de donde provienen las variables
),
-- Calcula el número total de malos pagadores 
quartile_risk AS (
    SELECT 
        using_lines_not_secured_personal_assets_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers,
    FROM quartiles
    GROUP BY using_lines_not_secured_personal_assets_quartile
),
---rango de edad (mínimo y máximo) para cada cuartil.
quartile_ranges AS (
    SELECT
        using_lines_not_secured_personal_assets_quartile,
        MIN(using_lines_not_secured_personal_assets) AS min_using_lines_not_secured_personal_assets,
        MAX(using_lines_not_secured_personal_assets) AS max_using_lines_not_secured_personal_assets
    FROM quartiles
    GROUP BY using_lines_not_secured_personal_assets_quartile
)
SELECT 
    q.using_lines_not_secured_personal_assets_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_using_lines_not_secured_personal_assets,
    r.max_using_lines_not_secured_personal_assets
FROM quartile_risk q
JOIN quartile_ranges r
ON q.using_lines_not_secured_personal_assets_quartile = r.using_lines_not_secured_personal_assets_quartile
ORDER BY using_lines_not_secured_personal_assets_quartile ASC;

#CALCULAR LOS CUARTILES DE MALOS PAGADORES delay_30_59_90
-- Calcular los cuartiles de malos pagadores delay_30_59_90
WITH base_data AS (
    SELECT 
        delay_30_59_90,
        default_flag
    FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo`
    WHERE default_flag = 1
),
-- Calcular los cuartiles dependiendo del delay_30_59_90
quartiles AS (
    SELECT 
        delay_30_59_90,
        default_flag,
        NTILE(4) OVER (ORDER BY delay_30_59_90) AS delay_30_59_90_quartile
    FROM base_data
),
-- Calcula el número total de malos pagadores y el total de registros para cada cuartil
quartile_risk AS (
    SELECT 
        delay_30_59_90_quartile,
        COUNT(*) AS total_count,
        SUM(default_flag) AS total_bad_payers
    FROM quartiles
    GROUP BY delay_30_59_90_quartile
),
-- Rango de delay_30_59_90 (mínimo y máximo) para cada cuartil
quartile_ranges AS (
    SELECT
        delay_30_59_90_quartile,
        MIN(delay_30_59_90) AS min_delay_30_59_90,
        MAX(delay_30_59_90) AS max_delay_30_59_90
    FROM quartiles
    GROUP BY delay_30_59_90_quartile
)
SELECT 
    q.delay_30_59_90_quartile,
    q.total_count,
    q.total_bad_payers,
    r.min_delay_30_59_90,
    r.max_delay_30_59_90
FROM quartile_risk q
JOIN quartile_ranges r
ON q.delay_30_59_90_quartile = r.delay_30_59_90_quartile
ORDER BY q.delay_30_59_90_quartile ASC;
   ~~~


4.- Calcular la correlaciòn entre variables nùmericas continuas.

 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 180718](https://github.com/user-attachments/assets/3852f5d4-ca38-4e3a-9975-8a3cefdc1480)

 Last_month_salary / age
  Interpretaciòn: Esto sugiere que hay una relación débilmente positiva entre "age" y "last_month_salary". Es decir, a medida que una de estas variables aumenta, la otra también tiende a aumentar        
  ligeramente.

 Debt_ratio / age
  Interpretaciòn: Esto indica que prácticamente no hay una relación lineal entre "age" y "debt_ratio". Cambios en "age" no tienen un impacto significativo en "debt_ratio".

 Debt_ratio / last_month_salary
  Interpretaciòn: Esta correlación negativa débil sugiere que hay una tendencia muy leve de que cuando "debt_ratio" aumenta, "last_month_salary" disminuye, pero la relación es casi inexistente.

  Debt_ratio / Using_lines_not_secured_personal_asset
  Interpretaciòn: Esto indica que hay una relación extremadamente débilmente positiva entre "debt_ratio" y "using_lines_not_secured_personal_assets". Es decir, cuando una de estas variables aumenta, la  
  otra tiende a aumentar ligeramente, pero la relación es muy débil.

 Consulta: 
  ~~~
 SELECT 
 CORR(age,last_month_salary) AS age_last_month,
 CORR (age,debt_ratio) AS age_debt_ratio,
 CORR (debt_ratio,last_month_salary) AS debt_last_month,
 CORR (debt_ratio, using_lines_not_secured_personal_assets) AS debt_usin_lines
  FROM `riesgo-relativo-p3.Data_Set.Data_Set_Completo` 
 ~~~


5.- Calcular riesgo relativo

Se analizan las variables para determinar el numero de veces que corre el riesgo de suceder el evento (malos pagadores):

  Age
 
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 183433](https://github.com/user-attachments/assets/e8a970d7-29e4-479d-8ef1-106f6b587faa)

 Interpretación: 
 + Mayor riesgo relativo (30-46 años): Los usuarios en el rango de edad de 30 a 46 años tienen el mayor riesgo relativo, lo que sugiere que esta cohorte es más propensa a incumplir sus pagos.
 + Menor riesgo relativo (59-96 años): Los usuarios en el rango de edad de 59 a 96 años tienen el menor riesgo relativo, indicando que este grupo es el más confiable en términos de pago.
 + Tendencias de edad: Hay una tendencia general a que el riesgo relativo disminuya con la edad.

   Last_salary_month
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 183548](https://github.com/user-attachments/assets/cc75eadb-3ee8-47b3-b30d-95bfaf30a1db)

  Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de salario más bajo tienen el mayor riesgo relativo, lo que sugiere que entre son 4 veces más propensos a incumplir sus pagos.
 + Menor riesgo relativo: Los usuarios en el rango de salarios más elevados tienen el menor riesgo relativo, indicando que este grupo es el más confiable en términos de pago pues solo tiene un 37% de   
   posibilidades de incumplir en sus pagos.
 + Tendencias Hay una tendencia general a que el riesgo relativo disminuya conforme aumentan los ingresos mensuales del cliente.

   Number_dependents
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 183636](https://github.com/user-attachments/assets/78bbf668-3897-4e92-975b-d72408386e50)

 Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de dependientes medio (+1) tienen 3.46 veces màs probabilidades de incumplier en sus pagos.
 + Menor riesgo relativo: Los usuarios de los cuartiles 1 y 2  indican que no hay riesgo de que ocurra el evento en estos cuartiles, posiblemente porque no hay dependientes en estos grupos.
 + Tendencias Hay una tendencia general a que el riesgo relativo aumente conforme los dependientes.
   
   Debt_ratio
 > [!NOTE] 
 ![Captura de pantalla 2024-07-26 183722](https://github.com/user-attachments/assets/82b60898-0388-4d29-8eb3-cfa383a015f2)

 Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de dependientes medio (+1) tienen 3.46 veces màs probabilidades de incumplier en sus pagos.
 + Menor riesgo relativo: Los usuarios de los cuartiles 1 y 2  indican que no hay riesgo de que ocurra el evento en estos cuartiles, posiblemente porque no hay dependientes en estos grupos.
 + Tendencias Hay una tendencia general a que el riesgo relativo aumente conforme los dependientes.

   Using_lines_not_secured_personal_asset
 > [!NOTE] 
 >![Captura de pantalla 2024-07-26 183813](https://github.com/user-attachments/assets/7e65a9bd-cf2d-4457-b011-650eb0862d12)

Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de dependientes medio (+1) tienen 3.46 veces màs probabilidades de incumplier en sus pagos.
 + Menor riesgo relativo: Los usuarios de los cuartiles 1 y 2  indican que no hay riesgo de que ocurra el evento en estos cuartiles, posiblemente porque no hay dependientes en estos grupos.
 + Tendencias Hay una tendencia general a que el riesgo relativo aumente conforme los dependientes.

  delay_30_59_90
 > [!NOTE] 
 > ![Captura de pantalla 2024-07-26 183945](https://github.com/user-attachments/assets/dcff22e9-6f23-492a-9fd3-0c31f79fcf43)

Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de dependientes medio (+1) tienen 3.46 veces màs probabilidades de incumplier en sus pagos.
 + Menor riesgo relativo: Los usuarios de los cuartiles 1 y 2  indican que no hay riesgo de que ocurra el evento en estos cuartiles, posiblemente porque no hay dependientes en estos grupos.
 + Tendencias Hay una tendencia general a que el riesgo relativo aumente conforme los dependientes.

## **Hipótesis**

Para la validación de hipótesis en el proceso de evaluación del riesgo relativo, se analizarán distintas variables, como el historial de pagos, la deuda total y los ingresos, entre otros.
El objetivo es determinar cómo estas variables influyen en el riesgo de incumplimiento. Este análisis permitirá clasificar a los clientes en diferentes categorías de riesgo.

 + *Los más jóvenes tienen un mayor riesgo de impago:*  Los individuos más jóvenes (Generacion Z- Millenials) tienden a tener balances más        altos y límites de crédito más bajos, lo que puede indicar un  mayor riesgo de impago.
   
 + *Las personas con más cantidad de préstamos activos tienen mayor riesgo de ser malos pagadores.* Si se observa que un mayor número de          prestamos activos está asociado con un mayor número de pagos incumplidos en comparación con los pagos cumplidos.
   
 + *Las personas que han retrasado sus pagos por más de 90 días tienen mayor riesgo de ser malos pagadores.* as personas con retrasos en los      pagos tienen un mayor riesgo de ser malos pagadores.
   
 + *Existe una diferencia significativa en el riesgo de ser mal pagador entre los diferentes cuartiles de salario del último mes:* Las            personas con salarios más bajos (primer cuartil) tienen un riesgo mucho mayor de ser mal pagadores en comparación con las personas con        salarios más altos (cuarto cuartil).
   
 + *Existe una diferencia significativa en el riesgo de ser mal pagador entre los diferentes cuartiles del número de dependientes* Las           personas con 0 dependientes en el primer cuartil y las personas con 1 a 13 dependientes en el cuarto cuartil tienen un riesgo mayor de ser    mal pagadores en comparación con aquellos en los otros cuartiles.
   
 + *No hay diferencia significativa en el riesgo de ser mal pagador entre los diferentes cuartiles del ratio de deuda.* Los individuos en el     primer y segundo cuartil (con ratios de deuda más bajos) tienen un riesgo menor de ser mal pagadores en comparación con aquellos en el        tercer y cuarto cuartil (con ratios de deuda más altos).
