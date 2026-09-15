# 📐 Medidas DAX principales

Este documento contiene una selección de las medidas DAX utilizadas en el proyecto **Panorama económico y financiero de Costa Rica | 2024–2026**.

El objetivo es documentar parte de la lógica aplicada para el cálculo de indicadores, análisis temporal, tratamiento de valores faltantes, correlaciones y storytelling dinámico dentro de Power BI.

> **Nota:** Las medidas se presentan con fines educativos, de documentación y portafolio profesional. No se incluyen credenciales, tokens, rutas privadas ni parámetros sensibles utilizados para las conexiones de datos.

---

# 📊 Indicadores económicos

## 1. Tasa Básica Pasiva (TBP)

Obtiene el valor de la Tasa Básica Pasiva a partir de la tabla principal de indicadores del BCCR.

```DAX
TBP =
CALCULATE(
    MAX(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 423
)
```

---

## 2. Tasa Efectiva en Dólares (TED)

```DAX
TED =
CALCULATE(
    MAX(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 23698
)
```

---

## 3. Tipo de cambio de compra

```DAX
TC Compra =
CALCULATE(
    MAX(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 317
)
```

---

## 4. Tipo de cambio de venta

```DAX
TC Venta =
CALCULATE(
    MAX(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 318
)
```

---

# 💱 Análisis del mercado cambiario

## 5. Promedio del tipo de cambio de compra

Para el análisis histórico se utiliza el promedio de las observaciones disponibles dentro del contexto temporal seleccionado.

```DAX
TC Compra Promedio =
CALCULATE(
    AVERAGE(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 317
)
```

---

## 6. Promedio del tipo de cambio de venta

```DAX
TC Venta Promedio =
CALCULATE(
    AVERAGE(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 318
)
```

---

## 7. Promedio del tipo de cambio MONEX

En la serie MONEX se excluyen los registros con valores iguales a cero para evitar que afecten el cálculo del promedio histórico.

```DAX
TC MONEX Promedio =
CALCULATE(
    AVERAGE(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 3323,
    FactIndicadoresBCCR[Valor] > 0
)
```

---

## 8. Último valor disponible de MONEX

Esta medida identifica primero la última fecha con información válida y posteriormente obtiene el valor correspondiente.

```DAX
TC MONEX Actual =
VAR UltimaFechaConDato =
    CALCULATE(
        MAX(FactIndicadoresBCCR[fecha]),
        DimIndicador[Codigo] = 3323,
        FactIndicadoresBCCR[Valor] > 0
    )
RETURN
    CALCULATE(
        MAX(FactIndicadoresBCCR[Valor]),
        DimIndicador[Codigo] = 3323,
        FactIndicadoresBCCR[fecha] = UltimaFechaConDato
    )
```

Esta lógica permite evitar que períodos sin información válida sean interpretados como el último dato disponible.

---

## 9. Spread cambiario

El spread cambiario corresponde a la diferencia entre el tipo de cambio promedio de venta y el tipo de cambio promedio de compra.

```DAX
Spread Cambiario =
[TC Venta Promedio] - [TC Compra Promedio]
```

---

# 💵 Liquidez monetaria

## 10. Medio circulante M1

```DAX
M1 =
CALCULATE(
    MAX(FactIndicadoresBCCR[Valor]),
    DimIndicador[Codigo] = 1445
)
```

---

## 11. M1 expresado en billones

Para facilitar la interpretación del indicador dentro de las visualizaciones se transforma la escala original.

```DAX
M1 Billones =
DIVIDE(
    [M1],
    1000000
)
```

---

# 📈 Análisis temporal

## 12. Crecimiento interanual del crédito

Esta medida compara el nivel actual de crédito con el correspondiente al mismo período del año anterior.

```DAX
Credito YoY Historico =
VAR CreditoActualMes =
    [Credito]

VAR CreditoHace12Meses =
    CALCULATE(
        [Credito],
        DATEADD(
            Calendario[Fecha],
            -1,
            YEAR
        )
    )

RETURN
    DIVIDE(
        CreditoActualMes - CreditoHace12Meses,
        CreditoHace12Meses
    )
```

El resultado se presenta en formato porcentual.

Esta medida permite analizar la evolución del crédito eliminando parcialmente los efectos derivados de la estacionalidad mensual.

---

# 🔗 Análisis de correlaciones

Para explorar relaciones estadísticas entre diferentes variables económicas se calcularon coeficientes de **correlación de Pearson**.

Las correlaciones se calculan dinámicamente según el contexto temporal aplicado en el dashboard.

Esto significa que el resultado puede cambiar al seleccionar:

- 2024
- 2025
- 2026
- Todo el período disponible

---

## 13. Correlación entre inflación y TPM

```DAX
Correlacion Inflacion TPM =
VAR TablaMeses =
    FILTER(
        ADDCOLUMNS(
            VALUES(Calendario[MesInicio]),
            "Inflacion", CALCULATE([Inflacion Interanual]),
            "TPMValor", CALCULATE([TPM])
        ),
        NOT(ISBLANK([Inflacion])) &&
        NOT(ISBLANK([TPMValor]))
    )

VAR PromedioInflacion =
    AVERAGEX(
        TablaMeses,
        [Inflacion]
    )

VAR PromedioTPM =
    AVERAGEX(
        TablaMeses,
        [TPMValor]
    )

VAR Numerador =
    SUMX(
        TablaMeses,
        ([Inflacion] - PromedioInflacion) *
        ([TPMValor] - PromedioTPM)
    )

VAR Denominador =
    SQRT(
        SUMX(
            TablaMeses,
            POWER(
                [Inflacion] - PromedioInflacion,
                2
            )
        )
        *
        SUMX(
            TablaMeses,
            POWER(
                [TPMValor] - PromedioTPM,
                2
            )
        )
    )

RETURN
    DIVIDE(
        Numerador,
        Denominador
    )
```

La medida construye primero una tabla temporal con las observaciones mensuales disponibles y posteriormente aplica la fórmula del coeficiente de correlación de Pearson.

---

## 14. Visualización de correlaciones sin información disponible

Cuando no existen suficientes datos para calcular una correlación, el dashboard muestra `N/D`.

```DAX
Corr Inflacion TPM Mostrar =
VAR Resultado =
    [Correlacion Inflacion TPM]

RETURN
    IF(
        ISBLANK(Resultado),
        "N/D",
        FORMAT(
            Resultado,
            "0.00"
        )
    )
```

Este tratamiento evita presentar valores potencialmente engañosos cuando los datos disponibles no permiten realizar el cálculo.

---

# 📝 Storytelling dinámico

Además de indicadores numéricos, el proyecto incorpora medidas DAX destinadas a generar textos interpretativos que responden automáticamente a los filtros aplicados por el usuario.

Esto permite complementar las visualizaciones con una lectura descriptiva del comportamiento de los indicadores.

---

## 15. Resumen dinámico del panorama económico

```DAX
Resumen Panorama =
VAR AnioSeleccionado =
    IF(
        HASONEVALUE(Calendario[Año]),
        FORMAT(
            SELECTEDVALUE(Calendario[Año]),
            "0"
        ),
        "2024–2026"
    )

RETURN
    "En " &
    AnioSeleccionado &
    ", la TPM se ubica en " &
    FORMAT(
        [TPM Actual],
        "0.00"
    ) &
    "%, la inflación interanual en " &
    FORMAT(
        [Inflacion Actual],
        "0.00"
    ) &
    "% y el crédito registra una variación interanual de " &
    FORMAT(
        [Crecimiento Interanual Credito],
        "0.00%"
    ) &
    "."
```

Esta medida permite que el texto se actualice automáticamente según el año seleccionado.

---

## 16. Resumen dinámico del mercado financiero y cambiario

```DAX
Resumen Mercado =
VAR Periodo =
    IF(
        HASONEVALUE(Calendario[Año]),
        FORMAT(
            SELECTEDVALUE(Calendario[Año]),
            "0"
        ),
        "2024–2026"
    )

RETURN
    "En " &
    Periodo &
    ", la TPM se ubica en " &
    FORMAT(
        [TPM Actual],
        "0.00"
    ) &
    "%, la TBP en " &
    FORMAT(
        [TBP Actual],
        "0.00"
    ) &
    "% y el tipo de cambio de venta en ₡" &
    FORMAT(
        [TC Venta Actual],
        "0.00"
    ) &
    " por dólar."
```

---

# ❓ Tratamiento de valores no disponibles

Las diferentes fuentes utilizadas en el proyecto poseen frecuencias y fechas de publicación distintas.

Por esta razón, algunos indicadores pueden no disponer de información para determinados períodos.

Para diferenciar correctamente un valor igual a cero de un dato realmente no disponible se utilizan medidas específicas.

---

## 17. Visualización del IMAE

```DAX
IMAE Mostrar =
VAR Resultado =
    [IMAE Actual]

RETURN
    IF(
        ISBLANK(Resultado),
        "N/D",
        FORMAT(
            Resultado,
            "0.00"
        )
    )
```

De esta forma, cuando no existe información disponible, el dashboard presenta:

`N/D`

en lugar de mostrar un valor incorrecto o interpretar la ausencia de información como cero.

---

## 18. Visualización de la variación de reservas

```DAX
Reservas YoY Mostrar =
VAR Resultado =
    [Crecimiento Interanual Reservas]

RETURN
    IF(
        ISBLANK(Resultado),
        "N/D",
        FORMAT(
            Resultado,
            "0.00 %"
        )
    )
```

---

# 🗓️ Tabla calendario

La integración temporal del modelo utiliza una tabla calendario para relacionar las diferentes fuentes de información.

Entre sus campos se encuentran:

- Fecha
- Año
- Mes
- Número de mes
- Año-Mes
- Trimestre
- Inicio de mes

---

## 19. Inicio de mes

```DAX
MesInicio =
DATE(
    YEAR(Calendario[Fecha]),
    MONTH(Calendario[Fecha]),
    1
)
```

Este campo se utiliza especialmente para:

- Ejes temporales mensuales.
- Análisis de correlaciones.
- Agrupación de series.
- Comparaciones entre indicadores.
- Integración de fuentes con distintas frecuencias.

---

# 🧠 Diseño de las medidas

Las medidas fueron desarrolladas buscando que el modelo responda dinámicamente al contexto de filtros utilizado por el usuario.

La lógica implementada incluye:

- Cálculo de valores actuales.
- Identificación del último dato disponible.
- Cálculo de valores históricos.
- Comparaciones interanuales.
- Variaciones absolutas y porcentuales.
- Promedios históricos.
- Tratamiento de valores faltantes.
- Correlaciones dinámicas.
- Storytelling.
- Interpretaciones automáticas.
- Adaptación de resultados según el año seleccionado.

---

# 📊 Uso de las medidas en el dashboard

Las medidas documentadas se utilizan en diferentes elementos del reporte, entre ellos:

### Tarjetas KPI

Para presentar los valores más recientes de:

- TPM
- TBP
- TED
- Tipo de cambio
- IMAE
- Crédito
- Reservas internacionales

### Gráficos de líneas

Para analizar la evolución temporal de:

- Inflación
- Política monetaria
- Actividad económica
- Crédito
- Tipos de cambio
- Tasas de interés

### Gráficos de dispersión

Para estudiar relaciones entre:

- Inflación y TPM
- TPM y crecimiento del crédito
- M1 e IMAE

### Textos dinámicos

Para generar interpretaciones que cambian según los filtros aplicados.

---

# 🔗 Interpretación de las correlaciones

El coeficiente de correlación de Pearson puede tomar valores entre:

`-1` y `1`

De forma general:

- Valores cercanos a **1** indican una asociación lineal positiva.
- Valores cercanos a **-1** indican una asociación lineal negativa.
- Valores cercanos a **0** indican una asociación lineal débil o inexistente.

Para facilitar la interpretación dentro del dashboard se utilizaron categorías basadas en la magnitud absoluta de la correlación:

- Menor a `0.20`: muy débil
- Entre `0.20` y `0.39`: débil
- Entre `0.40` y `0.59`: moderada
- Entre `0.60` y `0.79`: fuerte
- Igual o superior a `0.80`: muy fuerte

Estas clasificaciones se utilizan únicamente como una guía descriptiva para facilitar la lectura de los resultados.

---

# ⚠️ Correlación no implica causalidad

Las correlaciones incluidas en el proyecto tienen un propósito **exploratorio**.

Una asociación estadística entre dos variables no demuestra que una variable provoque cambios directamente en la otra.

Las relaciones observadas pueden estar influenciadas por:

- Rezagos temporales.
- Condiciones económicas externas.
- Cambios regulatorios.
- Expectativas de los agentes económicos.
- Condiciones financieras.
- Variables no incorporadas en el modelo.
- Cambios estructurales en la economía.

Para establecer relaciones causales sería necesario complementar este análisis con metodologías estadísticas y econométricas adicionales.

---

# 📌 Consideraciones sobre las medidas

Las medidas presentadas en este documento representan una selección de la lógica utilizada dentro del modelo.

El proyecto contiene medidas adicionales para:

- Valores iniciales y finales de los períodos.
- Variaciones absolutas.
- Variaciones porcentuales.
- Hallazgos dinámicos.
- Comparaciones temporales.
- Análisis de correlaciones.
- Manejo de datos faltantes.
- Interpretaciones automáticas.

El objetivo de este documento no es reproducir completamente el modelo semántico, sino mostrar ejemplos representativos de las técnicas utilizadas durante su construcción.

---

# ⚠️ Nota metodológica

Los resultados dependen de:

- La disponibilidad de información en las fuentes oficiales.
- Las fechas de actualización de cada indicador.
- La frecuencia de publicación de cada serie.
- Los filtros aplicados dentro del dashboard.
- Las transformaciones realizadas durante la preparación de los datos.

Algunas series pueden no disponer de información para el mismo período más reciente.

Por esta razón, determinados indicadores o análisis pueden mostrar `N/D` cuando no existe información suficiente.

---

# 🔐 Seguridad

Este repositorio no contiene:

- Tokens de autenticación.
- Contraseñas.
- Credenciales.
- Rutas privadas.
- Direcciones personales de SharePoint o OneDrive.
- Información confidencial.
- Parámetros sensibles de conexión.

Las medidas incluidas se publican exclusivamente con fines de **aprendizaje, documentación técnica y portafolio profesional**.

---

# 📚 Proyecto relacionado

Estas medidas forman parte del proyecto:

**Panorama económico y financiero de Costa Rica | 2024–2026**

El dashboard fue desarrollado utilizando principalmente:

- Power BI
- Power Query
- DAX
- Power BI Service
- API del Banco Central de Costa Rica
- Datos del INEC
- SharePoint / OneDrive

El objetivo del proyecto es demostrar la aplicación de técnicas de **análisis de datos, Business Intelligence, modelado, visualización y comunicación de resultados** mediante información económica y financiera pública.


