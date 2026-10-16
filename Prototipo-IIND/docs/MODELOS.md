# Modelos predictivos y algoritmos del prototipo · Grupo 14 · Reto 19

Este documento explica qué modelos se construyeron sobre la base del reto, cómo se validaron, qué encontraron y cómo guían las funcionalidades del prototipo. Todo número sale de `scripts/train_models.py` sobre `Base_Datos_Reto_Universidad.xlsx` (sha256 `5029790b7a29…`, 600 clientes, corte 2026-10-08). La base es 100 % sintética y académica: sus cifras no son métricas de Lulo Bank.

## 1. Resumen

El motor del prototipo decidía con reglas: una señal de comportamiento (peajes, nómina, transferencias fijas, bolsillos) con una fuerza supuesta de 1,0, 0,6 o 0. Esa fuerza era una hipótesis, porque la base no trae resultados de ofertas por cliente. Los modelos sirven para reemplazar esa hipótesis por evidencia donde la base la tiene, y para decir con claridad dónde no la tiene.

| Modelo | Pregunta | Resultado | Uso en el prototipo |
|---|---|---|---|
| M1 Propensión por ramo | ¿Quién se parece a los clientes que ya tienen cada seguro? | Le gana al azar en SOAT, Vida y Desempleo (AUC 0,59 a 0,64); no en Hogar, Mascotas ni Viajes | Ajusta el valor esperado de la oferta (reglas v1.2) |
| M2 Segmentos | ¿Qué tipos de cliente hay según sus hábitos? | Tres grupos con estructura débil (silueta 0,11) | Tarjeta «Tu forma de usar Lulo» en Tu año; columna en el plan de campaña |
| M3 Respuesta al push | ¿Hay clientes que abren menos las notificaciones? | No hay diferencias reales entre clientes (ρ = 0,0099) | Puede mandar la oferta a banner; hoy no cambia ningún canal |
| M4 Demanda por ramo | ¿Cuántas pólizas se emitirán los próximos meses? | Tendencia creciente significativa en Desempleo, Hogar y Mascotas | Panel: planeación de capacidad de la campaña |
| M5 Momento del mes | ¿Se compra más justo después de la nómina? | No: χ² = 4,83 frente a un crítico de 7,82 | Confirma que el momento lo dan los eventos, no el calendario |
| M6 Tamaño del piloto | ¿Cuántos clientes se necesitan para medir el efecto? | Entre 510 y 879 por grupo para detectar +50 % | Diseño del piloto real con el grupo de control |

El hallazgo central coincide con la literatura que sostiene la propuesta (Boustani et al., 2024): **el perfil demográfico no predice la tenencia de seguros en ningún ramo** (AUC entre 0,48 y 0,60, alrededor del azar), mientras que **la conducta sí** (AUC de hasta 0,64). La variable más fuerte en los tres ramos que funcionan es la misma: visitar la sección de Seguros en la app.

## 2. Datos y variables

Cada cliente se describe con 41 variables calculadas en `lulo_app/ml/vector.py`, a la fecha de corte y con una ventana de 365 días. Solo se usan campos canónicos (fecha, monto, canal, categoría y MCC, más el perfil y el uso de la app); nunca nombres de comercios o de terceros. El mismo código calcula las variables al entrenar y en producción, así que no hay diferencias entre los dos.

- **Perfil (12):** rango de edad, ocupación, segmento Pro, antigüedad, saldo, ingreso, crédito, CDT y bolsillos.
- **Conducta (26):** número de movimientos; compras por categoría (peajes, mascotas, aerolíneas, hoteles, salud, e-commerce, restaurantes, transporte, supermercados, servicios); transferencias fijas a terceros; racha y meses de nómina; gasto frente a ingreso (consumption ratio); gasto mensual; abonos por tipo de bolsillo; y participación de cada canal de pago.
- **Uso de la app (3):** sesiones, visitas a Seguros y tasa de apertura de notificaciones.

**Objetivo de M1.** Que el cliente tenga el ramo, sea con Lulo (Clientes.seguros_vigentes) o pagado a otra aseguradora (Pagos_Seguros). Es un modelo de parecido (*look-alike*): estima quién se parece a quienes ya compraron, no quién responderá a una oferta. Esa segunda pregunta solo la puede contestar el piloto con grupo de control (M6).

| Ramo | Clientes que lo tienen | Tasa base |
|---|---:|---:|
| SOAT | 170 | 28,3 % |
| Vida | 93 | 15,5 % |
| Hogar | 66 | 11,0 % |
| Desempleo | 39 | 6,5 % |
| Mascotas | 38 | 6,3 % |
| Viajes | 18 | 3,0 % |

## 3. M1 · Propensión por ramo

**Método.**
1. Regresión logística con regularización L1, variables estandarizadas. La fuerza de la regularización (C) se elige por validación cruzada interna entre 0,05 y 0,5.
2. Se excluyen las variables que la regla del ramo ya usa para su señal (por ejemplo, peajes para el SOAT o abonos al bolsillo de mascotas para Mascotas). Así el modelo aporta solo lo que la regla no ve y no contradice la hipótesis de negocio de la señal.
3. **Validación:** AUC con validación cruzada estratificada de 5 pliegues repetida 5 veces.
4. **Prueba de permutación:** se repite la validación con las etiquetas barajadas 200 veces. El valor p es la proporción de barajadas con AUC igual o mayor que el real.
5. **Selección por estabilidad:** el modelo final solo conserva las variables que el L1 elige en al menos el 80 % de 100 remuestreos *bootstrap*.
6. **Comparaciones:** solo perfil, solo conducta y un *gradient boosting* de referencia.

**Regla de uso.** Un ramo usa el modelo solo si p < 0,05 y AUC ≥ 0,55. Si no, el motor sigue con la regla y la afinidad vale 1.

| Ramo | AUC (± desv.) | p | AUC solo perfil | AUC solo conducta | Boosting | Variables estables | Uso |
|---|---|---|---|---|---|---|---|
| SOAT | 0,63 ± 0,05 | 0,005 | 0,50 | 0,63 | 0,61 | Visitas a Seguros (+0,44) | Ajusta el VE |
| Vida | 0,59 ± 0,07 | 0,040 | 0,48 | 0,60 | 0,54 | Visitas a Seguros (+0,41), abonos a bolsillo de viajes (−0,36) | Ajusta el VE |
| Desempleo | 0,64 ± 0,08 | 0,030 | 0,60 | 0,62 | 0,59 | Crédito activo (+0,48), visitas a Seguros (+0,48) | Ajusta el VE |
| Hogar | 0,50 | 0,811 | 0,50 | 0,50 | 0,48 | ninguna | Regla |
| Mascotas | 0,50 | 0,975 | 0,50 | 0,50 | 0,41 | ninguna | Regla |
| Viajes | 0,51 ± 0,13 | 0,343 | 0,50 | 0,51 | 0,48 | inestable | Regla |

**Lectura.**
- **La señal es modesta.** Un AUC de 0,6 ordena mejor que el azar, pero está lejos de identificar a un comprador. Por eso el modelo no decide solo: multiplica el valor esperado de una oferta que la regla ya aprobó.
- **El *gradient boosting* no mejora a la regresión** en ningún ramo. Con 600 clientes, más complejidad solo agrega ruido; la regresión además se explica en una línea.
- **En Hogar, Mascotas y Viajes no hay señal aprovechable.** En Viajes hay apenas 18 clientes que lo tienen. Esos ramos siguen con la regla, y eso queda registrado en cada decisión.
- **El signo negativo de «abonos a bolsillo de viajes» en Vida es estable pero no tiene explicación de negocio clara.** Conviene revisarlo con datos reales antes de darle importancia.

**Cómo entra al motor.** `afinidad = probabilidad del modelo / tasa base del ramo`, acotada entre 0,5 y 2,0. El valor esperado pasa a ser:

> conversión previa × comisión esperada a 12 meses × fuerza de la señal × afinidad

En el plan de campaña a la fecha de corte, el modelo cambia el ramo ofrecido a 50 de los 600 clientes (8 %): 25 pasan de Vida a Desempleo, 16 de Desempleo a Mascotas, 8 de Desempleo a Hogar y 1 de SOAT a Viajes. Las ofertas con fecha límite (SOAT por vencer y desembolso) siguen yendo primero.

## 4. M2 · Segmentos de comportamiento

**Método.** K-medias sobre 16 hábitos estandarizados (peajes, mascotas, vuelos, hoteles, transferencias fijas, racha de nómina, gasto frente a ingreso, gasto mensual, abonos por bolsillo, sesiones, visitas a Seguros, crédito y Pro). Se probó k de 3 a 7 y se eligió el de mayor silueta (k = 3; siluetas entre 0,10 y 0,11). El nombre de cada grupo sale de su variable más distinta del promedio, hacia arriba o hacia abajo.

| Segmento | Clientes | Rasgos | Ramo más afín (lift de tenencia) |
|---|---:|---|---|
| Gasto contenido | 316 | Gasta poco frente a su ingreso, gasto mensual bajo, casi no compra vuelos | Desempleo (× 1,22) |
| Gasta lo que entra | 233 | Gasta buena parte de su ingreso, gasto mensual alto, compra vuelos y hoteles | Mascotas (× 1,15) |
| Con mascota | 51 | Ahorra para su mascota, no ahorra para emergencias | Vida (× 1,52) |

**Lectura.** Una silueta de 0,11 indica que los clientes no forman grupos naturales marcados. Por eso los segmentos no entran a la decisión del motor. Sirven para describir al cliente en lenguaje claro (Tu año) y para leer el plan de campaña.

## 5. M3 · Respuesta al push

**Método.** Modelo beta-binomial con Bayes empírico. Las aperturas de cada cliente siguen una binomial cuya probabilidad varía entre clientes según una Beta(α, β). α y β se estiman por máxima verosimilitud con los 439 clientes que recibieron notificaciones. La probabilidad de que un cliente abra un push es (abiertas + α) / (enviadas + α + β), que no castiga a quien tiene pocas notificaciones.

**Resultado.** α = 33,4 y β = 66,3, con una correlación entre envíos del mismo cliente de ρ = 0,0099 (prácticamente cero). Todos los clientes quedan entre 32,5 % y 34,5 % (percentiles 10 y 90), alrededor de la tasa agregada de 33,5 %.

**Lectura.** En esta base **no hay clientes que abran sistemáticamente menos**: las diferencias que se ven en los conteos son ruido de muestras pequeñas (de 0 a 6 notificaciones). La regla de fatiga actual (ignoró al menos 3 y al menos el doble de las que abrió) manda 23 clientes a banner por ese ruido. Se deja así porque fue una regla aprobada, pero conviene revisarla con datos reales. El modelo queda activo con un umbral de 20 %; con esta base no cambia ningún canal.

## 6. M4 · Demanda por ramo

**Método.** Regresión lineal de las pólizas emitidas por mes (Seguros_Mensual, 12 meses por ramo) contra el tiempo, con intervalo de predicción al 80 % (t de Student con 10 grados de libertad). La pendiente es significativa si supera 2,23 errores estándar.

| Ramo | Promedio mensual | Pendiente | Pronóstico nov-2026 (80 %) |
|---|---:|---|---|
| SOAT | 234,8 | +2,3 / mes, no significativa | 249 (218 a 281) |
| Vida | 130,2 | +1,7 / mes, no significativa | 141 (116 a 166) |
| Desempleo | 77,5 | +0,8 / mes, significativa | 83 (77 a 89) |
| Hogar | 60,9 | +1,0 / mes, significativa | 67 (59 a 75) |
| Mascotas | 54,0 | +1,0 / mes, significativa | 60 (55 a 65) |
| Viajes | 42,0 | +0,5 / mes, no significativa | 45 (39 a 52) |

**Lectura.** Con un año de datos no se puede estimar estacionalidad. El pronóstico sirve para dimensionar la campaña, no para tomar decisiones por cliente.

## 7. M5 · ¿Hay un momento del mes para comprar?

**Método.** Para cada pago a otra aseguradora se cuentan los días desde el último abono de nómina. Los 76 pagos con nómina previa se agrupan en ventanas de 0 a 7, 8 a 14, 15 a 21 y 22 a 31 días, y se comparan con una distribución uniforme mediante una prueba χ² (3 grados de libertad, 5 %).

| Días desde la nómina | Observados | Esperados |
|---|---:|---:|
| 0 a 7 | 17 | 19,0 |
| 8 a 14 | 19 | 16,6 |
| 15 a 21 | 10 | 16,6 |
| 22 a 31 | 30 | 23,8 |

**Resultado.** χ² = 4,83 frente a un valor crítico de 7,82: **no hay un momento preferido del mes**. Esto respalda el diseño del motor, donde el momento de la oferta lo da un evento del cliente (un desembolso, un vencimiento, un abono) y no una fecha del calendario. Si el momento importa, solo se va a poder demostrar con el grupo de control.

## 8. M6 · Tamaño del piloto

**Método.** Prueba z de dos proporciones (α = 5 %, potencia 80 %). La conversión base de cada ramo es ventas sobre impactados de su campaña previa (hoja Campanas_Previas).

| Ramo | Conversión base | Clientes por grupo: +20 % | +50 % | +100 % |
|---|---:|---:|---:|---:|
| SOAT | 12,9 % | 2.871 | 510 | 147 |
| Vida | 9,2 % | 4.201 | 752 | 219 |
| Desempleo | 8,0 % | 4.900 | 879 | 257 |
| Hogar | 8,0 % | 4.900 | 879 | 257 |
| Mascotas | 10,6 % | 3.605 | 643 | 187 |
| Viajes | 10,2 % | 3.769 | 673 | 196 |

**Lectura.** Con 600 clientes y un control del 10 % (60 clientes) no se puede detectar ningún efecto. El piloto real necesita la base completa del banco: para ver un aumento de 50 % en el SOAT hacen falta 510 clientes en control y 510 en tratamiento.

## 9. Cómo guían las funcionalidades

- **Motor de disparadores (Presente):** la afinidad de M1 ordena las ofertas dentro de lo que la regla ya aprobó. La traza de cada decisión dice si se usó el modelo o la regla, y con qué afinidad.
- **Pantalla de oferta:** cuando el modelo sube la afinidad (al menos 1,1) y el cliente es elegible, la app agrega una línea «Clientes como tú», con los hábitos que la explican en lenguaje claro.
- **Tu año en Lulo (Pasado):** muestra el segmento de M2 como «Tu forma de usar Lulo este año».
- **Canal:** M3 puede enviar la oferta a banner si la probabilidad de abrir un push queda bajo el umbral.
- **Panel de la demo:** nueva pestaña «Modelos» con las métricas de validación, los resultados de M1 a M6 y el detalle del cliente que se está viendo.
- **API:** `GET /api/v3/models` y la llave `models` en `/api/v3/meta`.
- **Ley 1581:** si el cliente no autoriza el uso de sus datos, el motor no calcula ningún modelo sobre él.

## 10. Límites y siguientes pasos

1. **La base es sintética.** Los modelos validan el método, no el comportamiento real de los clientes de Lulo. Con datos reales hay que volver a correr `scripts/train_models.py`, y la regla de uso decide sola qué ramos usan el modelo.
2. **Tenencia no es respuesta.** M1 dice quién se parece a quien ya compró. El efecto de ofrecer en un momento se mide con el grupo de control; con datos reales de respuesta, el siguiente paso es un modelo *uplift*.
3. **Las muestras son pequeñas.** Algunos ramos tienen pocos positivos (18 en Viajes, 39 en Desempleo), así que los intervalos son anchos; la prueba de permutación protege contra falsos hallazgos, no contra efectos pequeños.
4. **Hay que revisar dos cosas con datos reales:** la regla de fatiga, dado lo que encontró M3, y el signo de «bolsillo de viajes» en Vida.

## 11. Reproducir

```bash
pip install -r requirements-dev.txt          # incluye scikit-learn y numpy (solo desarrollo)
python scripts/train_models.py               # entrena y escribe lulo_app/ml/modelos.json (≈ 45 s)
python scripts/train_models.py --check       # verifica que el JSON del repo coincide
python -m pytest                             # 67 pruebas
python -m lulo_app.core backtest             # 0 violaciones
```

El entrenamiento es determinista (semilla 19): dos corridas producen el mismo archivo. En producción (Vercel) no se instala nada nuevo: `lulo_app/ml/models.py` lee el JSON y calcula con Python puro.
