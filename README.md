# Predictive-Sales-Analysis-

## Análisis de conjunto de datos de ventas y modelo predictivo

Normalmente contiene información de transacciones reales de una tienda online del Reino Unido entre 2009 y 2011.  
Se carga la data en Python, se realiza limpieza de datos, transformación de datos y se crea un modelo sencillo de predicción para el año 2012, con parámetros fiables y probables.

---

## Fuentes de Datos y Recolección

El dataset **Online Retail II** está disponible en el UCI Machine Learning Repository.  
Contiene transacciones de un comercio electrónico del Reino Unido entre el 1 de diciembre de 2009 y el 9 de diciembre de 2011.  

**Sus columnas típicas incluyen:**
- InvoiceNo – Número de factura  
- StockCode – Código del producto  
- Description – Descripción del producto  
- Quantity – Cantidad vendida  
- InvoiceDate – Fecha de la factura  
- UnitPrice – Precio unitario  
- CustomerID – Identificador del cliente  
- Country – País del cliente  

---

## Objetivos
- Limpieza minuciosa de datos  
- Segmentación de ventas reales y devoluciones  
- Identificar productos de marcas logísticas  
- Diversas estrategias de imputación  
- Modelo predictivo para las ventas del año 2012  

---

## Motivación
Crear un modelo predictivo para el siguiente año, después de generar una limpieza e imputación de valores nulos, conocer cómo se comporta la serie de tiempo, analizar las tendencias y los ciclos estacionales.

---

## Metodología y Proceso de Trabajo

Lo primero que realizamos es importar nuestro archivo `.xlsx` en Python. Está en hojas diferentes para cada año, las cuales unimos, obteniendo un total de **1,067,371 filas**.

```python
import pandas as pd

# Cargar las dos hojas
df_2009 = pd.read_excel("online_retail_II.xlsx", sheet_name="Year 2009-2010", engine="openpyxl")
df_2010 = pd.read_excel("online_retail_II.xlsx", sheet_name="Year 2010-2011", engine="openpyxl")

# Unir las hojas
df = pd.concat([df_2009, df_2010], ignore_index=True)


se inicia la limpieza de datos,primero empezamos con eliminar las filas duplicas del todo el dataset,dividimos el conjunto de datos en devoluciones,teniendo en cuenta la columna de cantidades,que sea negativas para devoluciones,positivas para ventas reales.
codigo: df = df.drop_duplicates()
# Ventas reales (Quantity > 0)
ventas = df[df["Quantity"] > 0]

# Devoluciones (Quantity < 0)
devoluciones = df[df["Quantity"] < 0]

la mayoria de valores nulos en el dataframe, estan en la columna Customer ID,siendo 230,876 mas que en la columna Description 

codigo: df.isnull().sum()
Invoice             0
StockCode           0
Description      4275
Quantity            0
InvoiceDate         0
Price               0
Customer ID    235151
Country             0
dtype: int64

se observa valores nulos restantes tanto para el sector devoluciones y ventas en las columnas Description y Customer ID.
pasamos por eliminar valores faltantes para ambas filas,si no tienes ni producto ni cliente, la fila no tiene valor analítico confiable,mejor procedemos a eliminarlas.los resultados fueron que todos los valores faltantes en Description,se eliminaron,quedando solo unos cuantos para la columna Customer ID,¿Qué te dice esto como analista?

codigo: devoluciones = devoluciones[~(devoluciones['Description'].isnull() & devoluciones['Customer ID'].isnull())]
devoluciones.isnull().sum()
Invoice           0
StockCode         0
Description       0
Quantity          0
InvoiceDate       0
Price             0
Customer ID    1473
Country           0
dtype: int64

Que existe un patrón en la base de datos: cuando no se registra el cliente, tampoco se registra el producto.Este tipo de observación es crucial para alertar sobre posibles problemas de captura de datos o procesos internos (por ejemplo, devoluciones automáticas no asociadas a clientes).

procedemos ahora para imputar solo valores faltantes en customer ID,Ver si un Invoice se repite con valores no nulos.En muchos casos, las filas con el mismo Invoice comparten el mismo Customer ID. Podemos usar esto para imputar.
codigo:
# Crear un diccionario con Customer ID por Invoice (solo donde no hay nulos)
invoice_to_customer = devoluciones[devoluciones['Customer ID'].notnull()] \
    .groupby('Invoice')['Customer ID'].first()

# Imputar los valores nulos
devoluciones['Customer ID'] = devoluciones.apply(
    lambda row: invoice_to_customer[row['Invoice']] if pd.isnull(row['Customer ID']) and row['Invoice'] in invoice_to_customer else row['Customer ID'],
    axis=1
)
Las facturas con Customer ID nulo no están duplicadas en otras filas con el mismo número de factura y un ID conocido.Entonces no puedes imputar Customer ID usando Invoice como criterio porque no hay referencia cruzada.

Si ves que cada StockCode tiene muchos Customer ID únicos, NO es confiable imputar así,vemos que muchos stockcode tiene multiples clientes,no es fiable imputar asi.
StockCode
M         229
22423     195
22138     181
POST      155
21843     113
79323W    109
21232     104
85123A     99
79323P     78
21527      76
Name: Customer ID, dtype: int64

Filtrar las filas donde Customer ID es nulo y ver si la Description corresponde a marcas logísticas (es decir, registros no relacionados con productos reales, como 'POSTAGE', 'ADJUSTMENT', 'MANUAL', 'BANK CHARGES', 'CARRIAGE', 'DOTCOM POSTAGE', etc.).filtrar correctamente las filas con Customer ID nulo y con descripciones asociadas a marcas logísticas, ahora simplemente elimínalas de la tabla original devoluciones.En el segmento de devoluciones, muchas filas no corresponden a devoluciones reales de clientes, sino a ajustes administrativos o logísticos.
codigo:# Lista de posibles marcas logísticas (puedes ampliarla)
logisticos = [
    'POSTAGE', 'ADJUSTMENT', 'CARRIAGE', 'MANUAL', 
    'BANK CHARGES', 'DOTCOM POSTAGE', 'CASH CARRY', 'CHECK'
]

# Filtrar registros con Customer ID nulo y descripción logística
marcas_logisticas = nulos_cliente[nulos_cliente['Description'].isin(logisticos)]

print(marcas_logisticas[['StockCode', 'Description', 'Customer ID']])

Descubrimos que hay 953 productos (StockCode) que tienen un único cliente asociado.
Esto es clave porque si un producto solo tiene un cliente que lo compra o devuelve, es razonable asignar ese mismo cliente cuando falte el ID en filas donde aparece ese producto,Se redujeron los valores nulos de 1,473 a 1,252.Esto quiere decir que la imputación logró completar 221 valores nulos (1,473 - 1,252).No todos los valores nulos pudieron ser imputados porque solo se imputaron aquellos donde el producto tiene un único cliente.
codigo:# Filtrar productos con un solo cliente
productos_un_cliente = stockcode_clientes[stockcode_clientes == 1].index

# Crear diccionario StockCode -> Customer ID único
moda_cliente = devoluciones[devoluciones['StockCode'].isin(productos_un_cliente)].groupby('StockCode')['Customer ID'].agg(lambda x: x.mode().iloc[0])

# Función para imputar solo donde Customer ID es nulo y StockCode tiene un cliente único
def imputar_customer_id(row):
    if pd.isnull(row['Customer ID']) and row['StockCode'] in moda_cliente:
        return moda_cliente[row['StockCode']]
    else:
        return row['Customer ID']

# Aplicar imputación
devoluciones['Customer ID'] = devoluciones.apply(imputar_customer_id, axis=1)

dejamos de imputar,remplazamos valores nulos de la columna  customer id con desconocido,con el fin de no perder informacion valiosa.
codigo:devoluciones['Customer ID'].fillna('Desconocido', inplace=True)

limpiamos la columna Description de caracteres no ASCII o caracteres especiales que puedan generar problemas. cargamos el DataFrame devoluciones directamente a 
PostgreSQL usando Python,con el fin de filtrar datos,segmentarlos,calculos basicos y conectar a power bi,para generar graficas.
codigo: import re
# Función para eliminar caracteres no ASCII
def limpiar_texto(texto):
    if isinstance(texto, str):
        # Reemplaza todo lo que no sea ASCII imprimible por espacio vacío
        return re.sub(r'[^\x00-\x7F]+','', texto)
    else:
        return texto

# Aplicar la función a la columna Description
devoluciones['Description'] = devoluciones['Description'].apply(limpiar_texto)
# Subir el DataFrame a PostgreSQL
devoluciones.to_sql('devoluciones', engine, if_exists='replace', index=False)

print("✅ Tabla 'devoluciones' creada correctamente en PostgreSQL.")
## Iniciamos la limpieza de datos con  la informacion de ventas reales:

igual como hicimos para devoluciones,eliminamos filas duplicadas,valores nulos para ambas columnas,marcas logisticas....
Detectamos patrones estacionales o sistemáticos en la pérdida de CustomerID
Picos en diciembre/enero:
En diciembre de ambos años (2009, 2010) el porcentaje de registros sin ID se dispara (30 % → 37.6 %).
En enero 2011 el valor sube hasta 38.4 %.
Esto sugiere un efecto de fin/inicio de año (temporada navideña, rebajas, o cambios operativos en el sistema de facturación) en el cual se registra menos información de cliente. Podría deberse a:
Mayor proporción de compras “como invitado” (no registrado) en la web.
Procesos de logística acelerados (envíos de regalo) que omitían el ID.
Fallas o cambios en el sistema de captura de datos en fechas críticas.
Mínimos a mediados de 2010
Entre marzo y octubre de 2010 la proporción de nulos se mantiene más baja (14 %–21 %), con el mínimo en octubre 2010 (≈ 14.45 %).
Esto puede indicar que el proceso de captura de CustomerID funcionaba adecuadamente durante esos meses, o que había menos promociones que incentivaran compras sin registro.
codig:# 1) Extraer año y mes directamente de la columna datetime
ventas['year']  = ventas['InvoiceDate'].dt.year
ventas['month'] = ventas['InvoiceDate'].dt.month

# 2) Crear la máscara de nulos en CustomerID
ventas['is_null_CID'] = ventas['Customer ID'].isna()

# 3) Agrupar por año y mes y calcular porcentaje de nulos
por_mes = (
    ventas
    .groupby(['year', 'month'])['is_null_CID']
    .mean()               # devuelve proporción de True => proporción de nulos
    .reset_index()
)
por_mes['pct_nulos'] = por_mes['is_null_CID'] * 100

# 4) Mostrar resultado
por_mes[['year', 'month', 'pct_nulos']]
Conclusión preliminar: La pérdida de CustomerID no es completamente aleatoria (MCAR), sino que parece estar relacionada con periodos estacionales (MarÍa/MNAR o MAR), por lo que cualquier imputación o decisión de exclusión debe tener en cuenta este sesgo temporal.
 Interpretación:
Estos son los meses con mayor participación en las ventas totales (en %). Por ejemplo:
Enero 2011: 38.38%
Diciembre 2010: 37.57%
Diciembre 2011: 31.50%
...
Así que si enfocas tu imputación en los primeros 5-8 meses más fuertes, puedes cubrir un 80%+ de las ventas, reduciendo mucho el impacto de los nulos en tus predicciones.Despues Comprobamos si alguna factura tiene más de un cliente ahora,despues de generar la imputacion,para asegurarnos que no hay distorsion de datos.
codigo:# Comprobar si alguna factura tiene más de un cliente ahora
facturas_con_conflicto = ventas_copia[ventas_copia['Customer ID'].notnull()] \
    .groupby('Invoice')['Customer ID'].nunique()
conflictos = facturas_con_conflicto[facturas_con_conflicto > 1]

print(f"Facturas con múltiples Customer ID: {conflictos.shape[0]}")
Facturas con múltiples Customer ID: 0

remplazamos valores nulos en customers id por desconocido,para no perder informacion de ventas lo cual es el objetivo central del analisis.
codigo:ventas_copia['Customer ID'] = ventas_copia['Customer ID'].fillna('desconocido')

creamos la serie de tiempo desde diciembre del año 2009 hasta diciembre del año 2011,agrupamos las ventas en tiempo mensual,calculamos las ventas totales por mes y graficamos para conocer como se comporta la serie,tendencias,ciclos y estaciones.
codigo:import pandas as pd

# Asegurar formato fecha
ventas_copia['InvoiceDate'] = pd.to_datetime(ventas_copia['InvoiceDate'])

# Filtrar rango desde dic 2009 hasta dic 2011
ventas_filtradas = ventas_copia[
    (ventas_copia['InvoiceDate'] >= '2009-12-01') &
    (ventas_copia['InvoiceDate'] <= '2011-12-31')
]

# Agrupar por mes y sumar ventas
ventas_mensuales = ventas_filtradas.groupby(
    pd.Grouper(key='InvoiceDate', freq='M')
)['Price'].sum().reset_index()

# Corregir etiquetas para que el mes sea el inicial y no el fin de mes
ventas_mensuales['Fecha'] = ventas_mensuales['InvoiceDate'] - pd.offsets.MonthEnd(1)
ventas_mensuales.drop(columns='InvoiceDate', inplace=True

generamos un modelo predictivo con la herramienta  de pronóstico de series temporales,Prophet.para predecir valores futuros cuando tienes datos que varían con el tiempo. Características clave:
Maneja estacionalidad: por ejemplo, picos de ventas en diciembre o en vacaciones.Tolera datos faltantes y atípicos sin que se rompa el modelo.Lo cual se adapta a nuestros datos,ya que trata y tolera ventas atipicas pero logica,como las fechas de diciembre ademas de la tendencia.
codigo:import pandas as pd
from prophet import Prophet
import matplotlib.pyplot as plt

# Preparar datos para Prophet (necesita columnas 'ds' para fecha y 'y' para valor)
df_prophet = ventas_mensuales.rename(columns={'Fecha': 'ds', 'VentasTotales': 'y'})

# Crear y ajustar modelo
modelo = Prophet()
modelo.fit(df_prophet)

# Crear DataFrame para predicción 12 meses hacia adelante
futuro = modelo.make_future_dataframe(periods=12, freq='M')

# Realizar la predicción
pronostico = modelo.predict(futuro)

# Mostrar gráfica con pronóstico
modelo.plot(pronostico)
plt.title('Pronóstico de ventas totales mensuales con Prophet')
plt.xlabel('Fecha')
plt.ylabel('Ventas Totales')
plt.show()

# Opcional: mostrar componentes (tendencia, estacionalidad)
modelo.plot_components(pronostico)
plt.show()

Metricas para evaluar y respaldar que el modelo es eficiente y se adapta a nuestros datos:
valuación del modelo Prophet en datos históricos:
📊 Evaluación del modelo Prophet en datos históricos:
RMSE: 25,644.10
MAE: 21,218.95
MAPE: 3.04%
R²: 0.9892
Mediana Error Absoluto (MedAE): 22,922.56
Error Máximo Absoluto (MaxAE): 70,936.30
Desviación estándar de errores: 25,644.10
Theil's U: 0.093  (U<1 → Prophet mejor que ingenuo).

codigo:# Métricas principales
rmse = np.sqrt(mean_squared_error(df_prophet['y'], historico_pred['yhat']))
mae = mean_absolute_error(df_prophet['y'], historico_pred['yhat'])
mape = np.mean(np.abs((df_prophet['y'] - historico_pred['yhat']) / df_prophet['y'])) * 100
r2 = r2_score(df_prophet['y'], historico_pred['yhat'])
medae = median_absolute_error(df_prophet['y'], historico_pred['yhat'])
maxae = np.max(np.abs(errores))
std_error = np.std(errores)

# Theil's U (comparación con modelo ingenuo: último valor real)
naive_forecast = df_prophet['y'].shift(1)
naive_errors = df_prophet['y'][1:] - naive_forecast[1:]
u_stat = (np.sqrt(np.mean((df_prophet['y'][1:] - historico_pred['yhat'][1:])**2)) /
          np.sqrt(np.mean(naive_errors**2)))

# -------------------------
# Resultados
# -------------------------
print("📊 Evaluación del modelo Prophet en datos históricos:")
print(f"RMSE: {rmse:,.2f}")
print(f"MAE: {mae:,.2f}")
print(f"MAPE: {mape:.2f}%")
print(f"R²: {r2:.4f}")
print(f"Mediana Error Absoluto (MedAE): {medae:,.2f}")
print(f"Error Máximo Absoluto (MaxAE): {maxae:,.2f}")
print(f"Desviación estándar de errores: {std_error:,.2f}")
print(f"Theil's U: {u_stat:.3f}  (U<1 → Prophet mejor que ingenuo)")

## Conclusiones en la limpieza de datos:
- existe un patrón en la base de datos: cuando no se registra el cliente, tampoco se registra el producto.Este tipo de observación es crucial para alertar sobre posibles problemas de captura de datos o procesos internos (por ejemplo, devoluciones automáticas no asociadas a clientes.
- hay valores nulos en Description, pero menos frecuentes y dispersos.
En varias filas, los dos campos (Description y Customer ID) están nulos al mismo tiempo.
La mayor densidad de nulos está en la columna Customer ID, lo que indica que podría ser un campo incompleto o poco registrado en devoluciones.
-la myoria de valores nulos estan asociados con el pais Reino Unido,tiene cierta correlacion por ser la tienda online de Reino Unido.

 ## Conclusion generales de los principales indicadores:
- las devoluciones totales reales,es decir,sin contar claramente registros logisticos,para los años de analisis,fueron 1.010.668 $,millones de dolares
- los 5 productos con mayores devoluciones fueron de mayor a menor: Paper craft,Little Birdie,Medium Ceramic top storage,printing smudgest/thrown away y ebay sales,es un canal de ventas y given away,producto logistico
-el grafico de lineas sobre devoluciones del producto,durante los 2 años se observa sin una tendencia marcada,presentado mayores devoluciones en epocas de diciembre y enero
-el producto con mayor devolucion, Paper craft,Little Birdie,representa el 10,25% del total de todos los productos devueltos.
-el pais que presenta mayores devoluciones fue Reino unido,seguido de Francia.
-Para ventas reales,el producto con mayor cantidad de unidades vendidas en total,fue world war 2,gliders asstd designs con un total de 106,140 mil unidades vendidas.
-el producto con mayores ventas monetarias fue regency cakestand 3 tier con un total de 350,590 $ mil dolares.
- el pais con mayores ventas tanto en unidades de productos,como monetarias,fue Reino unido.El menor fue Arabia Saudia.
- La serie de tiempo no suele observarse una tendencia marcada,pero tiene 2 ciclos estacionales con fuertes picos,que corresponden a fechas especiales y logicas en ventas como los meses de diciembre.

## Conclusion del modelo predictivo
## 📊 Interpretación de métricas
### **RMSE:** 25,644.10  
> *Root Mean Square Error* — mide el error promedio penalizando más los errores grandes.  
✅ En este caso, el error promedio es bajo comparado con el rango de tus valores  
(p. ej., si tus ventas están en cientos de miles o millones, es muy aceptable).

### **MAE:** 21,218.95  
> *Mean Absolute Error* — error promedio absoluto sin penalizar tanto los extremos.  
✅ De media, el modelo se equivoca en unas **21 mil unidades** respecto al valor real.

### **MAPE:** 3.04%  
> *Mean Absolute Percentage Error* — error en porcentaje.  
✅ **3%** es excelente para series temporales, indica **muy alta precisión**.

### **R²:** 0.9892  
> Coeficiente de determinación.  
✅ El **98.9%** de la variabilidad de los datos está explicada por el modelo.

### **MedAE:** 22,922.56  
> Mediana del Error Absoluto.  
✅ Similar al MAE, pero evita que valores extremos influyan demasiado.

### **MaxAE:** 70,936.30  
> Máximo Error Absoluto.  
⚠ El peor caso en el que el modelo se desvió respecto al valor real.

### **Desviación estándar de errores:** 25,644.10  
> Indica cuánto varían los errores alrededor del promedio.

### **Theil's U:** 0.093  
> Compara tu modelo con un pronóstico ingenuo (asumir que este mes es igual al anterior).  
✅ Menor que 1 significa que el modelo es mejor que el ingenuo.  
💪 **0.093** es excelente: el modelo es casi **10 veces más preciso** que e

 ## Conclusion generales de los principales indicadores:
- 5,14%,representa el total en valor monetario de devoluciones sobre las ventas
- Paper craft,Little Birdie,representa el 10,25% del total de todos los productos devueltos,le sigue Medium Ceramic top storage.Se recomienda generar analisis de estos  productos mas devueltos,encontrar la razon de las devoluciones,sea por calidad o empaque.
- el Reino Unido concentra las ventas, es normal que  concentrará también las devoluciones, y eso no siempre es señal de problema, sino del volumen y la facilidad de gestión de devoluciones en el mismo pais.
-la tienda online procede de Reino Unido y concentra las mayores ventas,esto es normal por el alcance y posicionamiento local,confianza del cliente local,costos de tiempo de envios,regulacion y pagos.
-el producto regency cakestand 3 tier,presenta los mayores ingresos para la tienda
## Propuesta como analista: 
-5,14% corresponde a devoluciones del total de ventas,al ser una tienda online no representa un porcetanje tan alto,pero hay que revisar la tendencia en los años,si es alcista o se mantiene estable alrededor del promedio.Según la grafica para el total de años en la tabla,solo devoluciones,parece un grafico estable a lo largo del tiempo,con fechas estacionales como diciembre,con muchas mas devoluciones,pero esto es normal y esta relacionado a mayores ventas para esas fechas.
-observando la tendencia del  gráfico y el porcentaje total devoluciones,sobre ventas,parece un costo estructural del canal online.
el producto Paper craft,Little Birdie,seria interesante valorar su aporte monetario por devoluciones,asi como otros productos que puedan generar gran aporte a las ventas e intervenir las causas de sus devoluciones.
-el producto regency cakestand 3 tier,genero mayores ingresos de ventas.si existe alto ingreso,existe mayor demanda.Este producto se puede explorar muchos mas,como otros nichos de mercado,no solo en Reino Unido que ya representa la mayoria de ventas.Por ejemplo en Francia,segundo pais que compra mas en la tienda,mas cerca a Reino Unido,menores costos de transporte.Se podria generar estrategias de ventas para conquistar su mercado,reducir costos de compra de estos productos con materiales diferentes,economicos Y decoraciones mas llamativas;con el fin de  generar mas ganancias.
- EL gráfico de los años de ventas reales,parece en general estable,con sus altos picos de ventas en fechas especiales,el modelo predictivo Prophet es eficiente y seguro para el conjunto de datos según la evaluación de sus métricas y útil para uso predictivo para el año 2012.

## Tecnologías Utilizadas

- Power BI Desktop, Power Query, DAX.
- Git y GitHub para control de versiones y documentación.
- Visual Studio Code y python para limpieza,transformación y modelaje
- postgresql para filtros,agrupaciones,metricas basicas

## Contacto

Proyecto desarrollado por [Luis Sánchez](https://github.com/luismontes031). 
Para consultas, contáctame a [luissanchezmontes31@gmail.com](mailto:luissanchezmontes31@gmail.com).









