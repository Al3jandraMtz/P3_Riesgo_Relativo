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

3.- Tabla Loans_Outstanding:

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

### 1.2 Análisis exploratorio

1.- Aplicar medidas de tendencia central

 > [!NOTE]
 >![Captura de pantalla 2024-07-31 150452](https://github.com/user-attachments/assets/4b6fab1d-4148-4873-941f-af7b22c29cff)

2.- Visualizar la distribuiciòn 

 > [!NOTE]
 >![Captura de pantalla 2024-07-31 151810](https://github.com/user-attachments/assets/a21264ce-17bc-4ee5-aeec-ebe227ee2b2b)

Interpretaciones:

Gráfica Preferencia en usos de crédito:
 + La gráfica muestra que a medida que se incrementa el uso de líneas de crédito no aseguradas contra activos personales, el ratio de deuda tiende a aumentar. Sin embargo, hay una variabilidad significativa, especialmente en los niveles más altos de uso de crédito no asegurado.

 Gráfica de Uso de líneas de credito no aseguradas:
 +  La gráfica muestra que los usuarios en sus 40s y principios de los 50s tienen los picos más altos en el uso de líneas de crédito no aseguradas. Después de los 50 años, el uso disminuye notablemente. Esto sugiere que las personas en la mediana edad son más propensas a utilizar líneas de crédito no aseguradas.

  Gráfica de Uso de líneas de crédito aseguradas por patrimonio:
   + La gráfica indica que el uso de líneas de crédito aseguradas por patrimonio aumenta progresivamente desde los 20 años, alcanzando su punto máximo en los usuarios de 50-60 años. Después de esta edad, el uso disminuye gradualmente. Esto sugiere que las personas en su edad media y mayores tienen un mayor uso de líneas de crédito aseguradas por patrimonio.
     
 Observaciones generales: Los picos en el uso de crédito no asegurado ocurren a una edad ligeramente menor que los picos en el uso de crédito asegurado. Y las líneas de crédito aseguradas por patrimonio tienden a ser utilizadas por un rango de edad más amplio y tienen un uso máximo más alto comparado con las líneas de crédito no aseguradas.
  
3.- Calcular cuartiles, deciles o percentiles de los grupos de malos pagadores.

Se analizan las variables que podrian ayudarnos a definir como es un cliente mal pagador.

   Age
 > [!NOTE] 
 > ![Captura de pantalla 2024-07-31 152320](https://github.com/user-attachments/assets/453f1a3b-920b-4927-be67-36f095427df2)

   Last_salary_month
 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 152329](https://github.com/user-attachments/assets/6a286c0d-45ed-4ce9-a0cd-752e89e2035a)

   Number_dependents
 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 152413](https://github.com/user-attachments/assets/29c7c030-a007-4d7b-929e-afaf4e5ae5db)

   Debt_ratio
 > [!NOTE] 
 >!![Captura de pantalla 2024-07-31 152438](https://github.com/user-attachments/assets/b88277d9-6c74-4066-b0b3-9f62e2dcc914)

   Using_lines_not_secured_personal_asset
 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 152507](https://github.com/user-attachments/assets/328f10be-8a85-4d59-8f29-bcf043de79c8)

  Delay_30-59-89
 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 152541](https://github.com/user-attachments/assets/921d8238-2b5b-4953-85fb-9a86857a6811)

4.- Calcular la correlaciòn entre variables nùmericas continuas.

 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 152634](https://github.com/user-attachments/assets/729a35a1-5ac7-46a1-a57a-5814364fd07e)

 Last_month_salary / age
  Interpretaciòn: Hay una correlación positiva muy baja entre la edad  y el salario del último mes. Esto indica que a medida que la edad aumenta, el salario del último mes tiende a aumentar ligeramente, aunque la relación es muy débil.

 Debt_ratio / age
  Interpretaciòn: Hay una correlación positiva extremadamente baja entre la edad y el ratio de deuda. Esto sugiere que la edad tiene muy poca influencia en el ratio de deuda de una persona.

 Debt_ratio / last_month_salary
  Interpretaciòn: Hay una correlación negativa muy baja entre el ratio de deuda (debt_ratio) y el salario del último mes. Esto implica que, a medida que el ratio de deuda aumenta, el salario del último mes tiende a disminuir ligeramente, pero la relación es muy débil.

  Debt_ratio / Using_lines_not_secured_personal_asset
  Interpretaciòn: Hay una correlación positiva extremadamente baja entre el ratio de deuda y el uso de líneas de crédito no aseguradas. Esto indica que el uso de líneas de crédito no aseguradas tiende a aumentar ligeramente con el aumento del ratio de deuda, pero la relación es prácticamente insignificante.

5.- Calcular riesgo relativo

Se analizan las variables para determinar el numero de veces que corre el riesgo de suceder el evento (malos pagadores):

  Age
 
 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 153706](https://github.com/user-attachments/assets/f3bfee0a-026f-4770-a92a-0be0baf02d72)

 Interpretación: 
 + Mayor riesgo relativo (20-51 años): Los usuarios en el rango de edad de 20 a 51 años tienen el mayor riesgo relativo, lo que sugiere que esta cohorte es más propensa a incumplir sus pagos.
 + Menor riesgo relativo (51-96 años): Los usuarios en el rango de edad de 52 a 96 años tienen el menor riesgo relativo, indicando que este grupo es el más confiable en términos de pago.
 + Tendencias de edad: Hay una tendencia general a que el riesgo relativo disminuya con la edad.

Last_salary_month

 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 153748](https://github.com/user-attachments/assets/63e646bd-0d7f-4b05-9d7d-66e6ff0a49c1)

  Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de salario más bajo tienen el mayor riesgo relativo, lo que sugiere que entre son 3.65 veces más propensos a incumplir sus pagos.
 + Menor riesgo relativo: Los usuarios en el rango de salarios más elevados tienen el menor riesgo relativo, indicando que este grupo es el más confiable en términos de pago pues solo tiene un 39% de   
posibilidades de incumplir en sus pagos.
 + Tendencias Hay una tendencia general a que el riesgo relativo disminuya conforme aumentan los ingresos mensuales del cliente.

Number_dependents

 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 153846](https://github.com/user-attachments/assets/ba0678f7-db8e-4045-b735-a81f7af93551)

 Interpretación: 
 + Mayor riesgo relativo: Los usuarios en el rango de dependientes medio (+1) tienen 5.68 veces màs probabilidades de incumplier en sus pagos.
 + Menor riesgo relativo: Los usuarios de los cuartiles 1 y 2  indican que no hay riesgo de que ocurra el evento en estos cuartiles, posiblemente porque no hay dependientes en estos grupos.
 + Tendencias Hay una tendencia general a que el riesgo relativo aumente conforme los dependientes.
   
Debt_ratio

 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 153931](https://github.com/user-attachments/assets/05b0f95f-765e-4b0c-abfd-f762be96bb16)

 Interpretación: 
 + Mayor riesgo relativo: Los usuarios del  tercer cuartil presentan el mayor riesgo relativo (1.41) y la tasa de malos pagadores más alta (2.25%). Este cuartil muestra un punto crítico donde el riesgo de incumplimiento es significativamente mayor.
 + Menor riesgo relativo: Los usuarios del cuartil 1 tienen menor riesgo relativo (0.71) de ser malos pagadores, con una tasa de malos pagadores de 1.35%.
 + Tendencias: A medida que el debt_ratio aumenta, tanto la tasa de malos pagadores como el riesgo relativo tienden a aumentar hasta el tercer cuartil. El cuarto cuartil, aunque tiene los ratios de deuda más altos, muestra un riesgo relativo menor (1.09) que el tercer cuartil, lo que sugiere que algunos de los deudores con ratios extremadamente altos pueden tener factores mitigantes no reflejados solo en el debt_ratio.

Using_lines_not_secured_personal_asset

 > [!NOTE] 
 >![Captura de pantalla 2024-07-31 154416](https://github.com/user-attachments/assets/0578e5de-456c-439d-8166-38c0a81abae4)

Interpretación: 
 + Mayor riesgo relativo: Los usuarios del cuartil 4 tienen el mayor riesgo relativo (41.08) y la tasa más alta de malos pagadores (0.06559). El máximo uso de líneas de crédito no aseguradas en este cuartil es extremadamente alto (22,000), lo que sugiere un riesgo significativamente mayor de incumplimiento asociado con altos niveles de endeudamiento no asegura
 + Menor riesgo relativo: Los usuarios de los cuartil 1 presentan el menor riesgo relativo (0.038) y una muy baja tasa de malos pagadores (0.00089). Es el cuartil con el uso mínimo de líneas de crédito no aseguradas, indicando que un bajo nivel de endeudamiento a través de líneas no aseguradas está asociado con un bajo riesgo de incumplimiento.
 + Tendencias: Estos hallazgos sugieren que un mayor uso de líneas de crédito no aseguradas está fuertemente correlacionado con un aumento en el riesgo de incumplimiento. Esto podría indicar que políticas más estrictas en la concesión de crédito no asegurado o una evaluación más detallada del riesgo crediticio podrían ser necesarias para los individuos en los cuartiles superiores de uso.

## **Hipótesis**

Para la validación de hipótesis en el proceso de evaluación del riesgo relativo, se analizarán distintas variables, como el historial de pagos, la deuda total y los ingresos, entre otros.
El objetivo es determinar cómo estas variables influyen en el riesgo de incumplimiento. Este análisis permitirá clasificar a los clientes en diferentes categorías de riesgo.

 + *Los más jóvenes tienen un mayor riesgo de impago:*  Validada: Los individuos más jóvenes (Generacion Z- Millenials) tienden a tener balances más  altos y límites de crédito más bajos, lo que puede indicar un  mayor riesgo de impago.
   
 + *Las personas con más cantidad de préstamos activos tienen mayor riesgo de ser malos pagadores.* Si se observa que un mayor número de  prestamos activos está asociado con un mayor número de pagos incumplidos en comparación con los pagos cumplidos.
   
 + *Las personas que han retrasado sus pagos por más de 90 días tienen mayor riesgo de ser malos pagadores.* las personas con retrasos en los pagos tienen un mayor riesgo de ser malos pagadores.
   
 + *Existe una diferencia significativa en el riesgo de ser mal pagador entre los diferentes cuartiles de salario del último mes:* Las personas con salarios más bajos tienen un riesgo mucho mayor de ser mal pagadores en comparación con las personas con salarios más altos (cuarto cuartil).
 
 + *Existe una diferencia significativa en el riesgo de ser mal pagador entre los diferentes cuartiles del número de dependientes* Las personas con 0 dependientes en el primer cuartil y las personas con 1 a 13 dependientes en el cuarto cuartil tienen un riesgo mayor de ser mal pagadores en comparación con aquellos en los otros cuartiles.

 ## **Score de Riesgo**

 1.- Creación de variables Dummys
 La regresión logística asigna un peso (coeficiente) a cada característica (variable dummy y otras variables). Estos coeficientes indican la importancia de cada característica para predecir la probabilidad de incumplimiento. (0 u 1)

 > [!NOTE] 
 >![Captura de pantalla 2024-07-30 192248](https://github.com/user-attachments/assets/2a017c20-d897-4498-a89a-a62229f8c031)
  
2.- Calculo de Score Riesgo

El modelo de regresión logística calcula una probabilidad de incumplimiento para cada registro. Esta probabilidad se convierte en un resultado binario utilizando un umbral (generalmente 0.5). Si la probabilidad es mayor que el umbral, el resultado es 1 (incumplimiento), de lo contrario, es 0 (no incumplimiento).

> [!NOTE] 
>![Captura de pantalla 2024-07-30 192451](https://github.com/user-attachments/assets/46185a3e-6bc1-417c-8803-88d4249437b8)

3.- Matriz de confusión

La matriz de confusión te permite evaluar el rendimiento del modelo comparando las predicciones con los valores reales de default_flag.

> [!NOTE] 
>![Captura de pantalla 2024-07-30 192538](https://github.com/user-attachments/assets/46874696-d82b-47c9-b193-724c17d2ca90)

4.- Metricas de presicion: Resultados

> [!NOTE] 
>![Captura de pantalla 2024-07-30 193040](https://github.com/user-attachments/assets/16016e46-9e04-4158-910a-f3dc32afd0e5)

 + Precisión (0.98): Indica que la mayoría de las predicciones positivas son correctas. El modelo tiene un buen rendimiento en identificar casos de incumplimiento.
 + Recall (1.00): El modelo está detectando todos los casos de incumplimiento, sin perder ninguno. Esto es muy positivo, pero puede indicar que el umbral de clasificación podría estar muy ajustado a detectar todos los positivos.
 + Exactitud (0.98): Indica que la proporción total de predicciones correctas es alta. Sin embargo, dado que no hay verdaderos negativos (TN = 0), esto podría ser un indicio de que el modelo está muy sesgado hacia la predicción positiva.
