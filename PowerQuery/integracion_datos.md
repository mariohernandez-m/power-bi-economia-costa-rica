# 🔄 Integración y transformación de datos con Power Query

Este documento describe parte del proceso de **extracción, transformación e integración de datos** utilizado en el proyecto:

**Panorama económico y financiero de Costa Rica | 2024–2026**

El objetivo es documentar cómo se integraron distintas fuentes de información en Power BI utilizando **Power Query (M)**, procurando que el modelo pudiera actualizarse posteriormente desde **Power BI Service**.

> **Nota de seguridad:** Los ejemplos incluidos fueron adaptados para documentación pública. No se muestran tokens, credenciales, correos electrónicos, rutas personales de SharePoint/OneDrive ni otros parámetros sensibles utilizados en las conexiones originales.

---

# 🧩 Arquitectura general de datos

El proyecto integra información proveniente de diferentes tipos de fuentes:

- API del Banco Central de Costa Rica.
- Fuentes web oficiales del BCCR.
- Archivos de Excel.
- Archivos almacenados en SharePoint / OneDrive.
- Datos provenientes del INEC.

Debido a que las fuentes poseen estructuras y frecuencias diferentes, fue necesario desarrollar un proceso de transformación que permitiera llevar la información a un formato compatible con el modelo de Power BI.

De forma general, el flujo utilizado fue:

```text
Fuentes oficiales
       ↓
Power Query
       ↓
Limpieza y transformación
       ↓
Normalización de fechas
       ↓
Integración de indicadores
       ↓
Modelo de datos
       ↓
Medidas DAX
       ↓
Visualizaciones
       ↓
Power BI Service
```

---

# 🌐 Conexión con la API del BCCR

Una de las principales fuentes del proyecto corresponde a la API de indicadores económicos del **Banco Central de Costa Rica (BCCR)**.

Para evitar crear una consulta independiente para cada indicador se desarrolló una **función reutilizable en Power Query**.

Esta función recibe el código del indicador y devuelve una tabla estructurada con:

- Fecha.
- Código del indicador.
- Nombre del indicador.
- Valor.

---

## Función reutilizable para indicadores del BCCR

El siguiente código corresponde a una versión adaptada para documentación pública.

```powerquery
(Codigo as text) as table =>
let
    FechaInicio = "2024/01/01",

    FechaFin =
        Date.ToText(
            Date.From(DateTime.LocalNow()),
            "yyyy/MM/dd"
        ),

    BaseUrl =
        "https://apim.bccr.fi.cr/",

    Ruta =
        "SDDE/api/Bccr.GE.SDDE.Publico.Indicadores.API/indicadoresEconomicos/"
        & Codigo &
        "/series",

    Fuente =
        Json.Document(
            Web.Contents(
                BaseUrl,
                [
                    RelativePath = Ruta,
                    Query = [
                        fechaInicio = FechaInicio,
                        fechaFin = FechaFin,
                        idioma = "ES"
                    ],
                    Headers = [
                        Authorization = "Bearer " & pToken,
                        Accept = "application/json"
                    ]
                ]
            )
        ),

    Datos =
        Fuente[datos],

    Indicador =
        Datos{0},

    Series =
        Indicador[series],

    Tabla =
        Table.FromRecords(Series),

    TiposDatos =
        Table.TransformColumnTypes(
            Tabla,
            {
                {"fecha", type date},
                {"valorDatoPorPeriodo", type number}
            }
        ),

    AgregarIndicador =
        Table.AddColumn(
            TiposDatos,
            "Indicador",
            each Indicador[nombreIndicador],
            type text
        ),

    AgregarCodigo =
        Table.AddColumn(
            AgregarIndicador,
            "CodigoIndicador",
            each Codigo,
            type text
        ),

    RenombrarValor =
        Table.RenameColumns(
            AgregarCodigo,
            {
                {"valorDatoPorPeriodo", "Valor"}
            }
        ),

    OrdenColumnas =
        Table.ReorderColumns(
            RenombrarValor,
            {
                "fecha",
                "CodigoIndicador",
                "Indicador",
                "Valor"
            }
        )
in
    OrdenColumnas
```

> `pToken` corresponde a un parámetro de autenticación configurado en el entorno privado del proyecto. Su valor real no se publica en este repositorio.

---

# ♻️ Uso de una función reutilizable

La creación de una función permitió utilizar la misma lógica para consultar diferentes indicadores sin repetir todo el código.

Conceptualmente:

```powerquery
fnBCCR("317")
```

puede utilizarse para obtener un indicador determinado, mientras que:

```powerquery
fnBCCR("318")
```

permite consultar otro utilizando exactamente el mismo proceso de extracción y transformación.

Entre los indicadores integrados se encuentran:

- Tipo de cambio de compra.
- Tipo de cambio de venta.
- Tasa de Política Monetaria.
- Tasa Básica Pasiva.
- Tasa Efectiva en Dólares.
- IMAE.
- MONEX.
- Medio circulante M1.

---

# 💡 ¿Por qué utilizar una función?

La creación de una función reutilizable permitió:

- Reducir código duplicado.
- Estandarizar la extracción de indicadores.
- Facilitar el mantenimiento.
- Incorporar nuevos indicadores de forma más sencilla.
- Mantener una estructura homogénea dentro del modelo.
- Centralizar la lógica de conexión con la API.

Este enfoque facilita la escalabilidad del proyecto.

---

# 🔧 Uso de `Web.Contents`, `RelativePath` y `Query`

Durante el desarrollo se utilizó una estructura basada en:

```powerquery
Web.Contents(
    BaseUrl,
    [
        RelativePath = Ruta,
        Query = [...]
    ]
)
```

en lugar de construir una dirección web completa mediante concatenación dinámica.

Esto permite separar:

### URL base

```text
https://apim.bccr.fi.cr/
```

### Ruta del recurso

```text
SDDE/api/.../indicadoresEconomicos/{Codigo}/series
```

### Parámetros de consulta

Por ejemplo:

```text
fechaInicio
fechaFin
idioma
```

---

## Ventajas de este enfoque

Esta estructura ofrece varias ventajas:

- Mayor organización del código.
- Mejor reutilización de consultas.
- Separación entre URL base y parámetros.
- Mayor compatibilidad con actualizaciones desde Power BI Service.
- Menor dependencia de URLs dinámicas construidas completamente como texto.

---

# 📅 Fecha final dinámica

Para evitar modificar manualmente la fecha final de consulta se utilizó:

```powerquery
Date.ToText(
    Date.From(DateTime.LocalNow()),
    "yyyy/MM/dd"
)
```

Esto permite que la consulta solicite datos hasta la fecha disponible al momento de la actualización.

De esta manera, el proyecto puede incorporar nuevas observaciones sin modificar manualmente el código de Power Query.

---

# 📋 Integración de los indicadores

Los indicadores obtenidos desde la API se integraron posteriormente dentro de una tabla principal de hechos.

Conceptualmente, la estructura utilizada contiene:

```text
Fecha
CódigoIndicador
Indicador
Valor
```

Esto permite almacenar múltiples series económicas en una misma estructura y diferenciarlas mediante su código.

Posteriormente se utiliza una dimensión de indicadores para organizar y clasificar las diferentes variables.

---

# 📊 Estructura conceptual del modelo

Una representación simplificada del modelo es:

```text
                  DimIndicador
                       │
                       │
                       ▼
Calendario ─── FactIndicadoresBCCR
     │
     │
     ├── Crédito
     │
     ├── Reservas
     │
     └── IPC / Inflación
```

La tabla calendario permite establecer un eje temporal común para las diferentes fuentes.

---

# 🗓️ Normalización de fechas

Uno de los principales retos fue integrar series con distintas frecuencias:

- Diarias.
- Mensuales.
- Periódicas.
- Archivos con fechas de corte específicas.

Para facilitar la comparación se normalizaron los campos de fecha dentro de Power Query y posteriormente se relacionaron con una tabla calendario.

Entre las transformaciones aplicadas se encuentran:

```powerquery
Table.TransformColumnTypes(
    Tabla,
    {
        {"fecha", type date}
    }
)
```

La correcta definición del tipo de datos es especialmente importante para:

- Relaciones entre tablas.
- Inteligencia temporal.
- Variaciones interanuales.
- Segmentadores por año.
- Gráficos de evolución.

---

# 💱 Transformación de series cambiarias

Algunos indicadores cambiarios poseen información diaria.

Para determinados análisis fue necesario trabajar con valores mensuales.

Por esta razón se utilizaron transformaciones y medidas que permitieran obtener valores representativos por período, como:

- Promedio mensual de compra.
- Promedio mensual de venta.
- Promedio mensual de MONEX.
- Spread cambiario.

Esto permite comparar las series en una escala temporal común con otros indicadores económicos de frecuencia mensual.

---

# 🌐 Extracción de información desde páginas web

No todas las series utilizadas estaban disponibles bajo la misma estructura de API.

Para determinados indicadores fue necesario consultar tablas publicadas en páginas web oficiales.

Un ejemplo corresponde a información de crédito al sector privado.

Se utilizó una estructura basada en:

```powerquery
let
    Origen =
        Text.FromBinary(
            Web.Contents(
                "https://gee.bccr.fi.cr",
                [
                    RelativePath =
                        "indicadoreseconomicos/Cuadros/frmVerCatCuadro.aspx",

                    Query = [
                        idioma = "1",
                        CodCuadro = " 120"
                    ]
                ]
            ),
            TextEncoding.Utf8
        )
in
    Origen
```

Posteriormente, el contenido HTML puede ser transformado mediante Power Query para obtener la tabla requerida.

---

# 🧹 Transformación de tablas web

Las tablas obtenidas desde fuentes web requieren diferentes procesos de preparación.

Entre las tareas realizadas se encuentran:

- Identificación de la tabla correcta dentro del HTML.
- Eliminación de filas no necesarias.
- Promoción de encabezados.
- Cambio de nombres de columnas.
- Conversión de tipos de datos.
- Limpieza de caracteres.
- Normalización de fechas.
- Eliminación de valores no válidos.

El objetivo es transformar una tabla diseñada originalmente para visualización web en una estructura adecuada para análisis.

---

# 📁 Integración de archivos Excel

Algunas fuentes de información fueron incorporadas mediante archivos de Excel.

Durante las primeras etapas del proyecto estos archivos se encontraban almacenados localmente.

Una conexión local puede utilizar una estructura similar a:

```powerquery
Excel.Workbook(
    File.Contents("C:/ruta/local/archivo.xlsx")
)
```

Este método funciona correctamente dentro de Power BI Desktop, pero puede generar una dependencia del equipo local cuando el reporte se publica en Power BI Service.

---

# ☁️ Migración de archivos locales a SharePoint / OneDrive

Para reducir la dependencia de archivos almacenados únicamente en una computadora, los archivos necesarios fueron trasladados a **SharePoint / OneDrive**.

Posteriormente, las consultas fueron modificadas para utilizar fuentes accesibles desde la nube.

Ejemplo conceptual:

```powerquery
Excel.Workbook(
    Web.Contents(
        "https://organizacion.sharepoint.com/ruta/archivo.xlsx"
    )
)
```

> La dirección mostrada es únicamente ilustrativa. Las rutas reales utilizadas en el proyecto no se publican por razones de seguridad y privacidad.

---

# 📌 Fuentes Excel utilizadas

Entre los archivos integrados se encuentra información relacionada con:

- Reservas internacionales.
- Índice de Precios al Consumidor.
- Series complementarias necesarias para el análisis.

Los archivos fueron preparados para que sus estructuras pudieran ser procesadas mediante Power Query.

---

# 🔐 Autenticación y seguridad

El proyecto utiliza diferentes mecanismos de acceso según el tipo de fuente.

De forma general:

### API

La API utiliza un token de autenticación almacenado como parámetro privado.

```text
pToken
```

El valor real no se publica.

### SharePoint / OneDrive

Los archivos alojados en la nube utilizan autenticación mediante la cuenta organizacional correspondiente.

### Fuentes web públicas

Las páginas públicas del BCCR pueden consultarse como recursos web públicos cuando corresponde.

---

# ⚠️ Información que no se publica

Por razones de seguridad, este repositorio no contiene:

- Tokens de autenticación.
- Contraseñas.
- Correos electrónicos personales o institucionales.
- Direcciones privadas de SharePoint.
- Rutas locales del equipo.
- Credenciales.
- Parámetros sensibles de conexión.

Los fragmentos de código incluidos fueron revisados y adaptados para fines de documentación pública.

---

# 🔄 Compatibilidad con Power BI Service

Uno de los objetivos técnicos del proyecto fue permitir que el modelo pudiera actualizarse después de ser publicado en **Power BI Service**.

Para ello fue necesario revisar la forma en que las diferentes fuentes eran consultadas.

Entre los principales ajustes se encuentran:

- Sustitución de archivos locales por fuentes almacenadas en la nube.
- Uso de URLs base estables.
- Uso de `RelativePath`.
- Uso de parámetros dentro de `Query`.
- Configuración de credenciales para cada fuente.
- Revisión de conexiones en Power BI Service.
- Validación del proceso completo de actualización.

---

# ✅ Validación de actualización

Una vez configuradas las conexiones, el modelo semántico fue actualizado desde **Power BI Service** para verificar que las distintas fuentes pudieran procesarse desde la nube.

Esto permitió validar conjuntamente:

- API del BCCR.
- Fuentes web.
- SharePoint / OneDrive.
- Archivos de Excel.
- Transformaciones de Power Query.
- Modelo de datos.

La actualización exitosa confirmó que el proyecto podía procesar sus fuentes sin depender exclusivamente de la sesión local de Power BI Desktop.

---

# 🧠 Principales retos técnicos

Durante la integración de datos se presentaron varios retos.

## 1. Fuentes con estructuras diferentes

La información provenía de:

- JSON.
- HTML.
- Excel.
- API.
- Archivos almacenados en la nube.

Fue necesario transformar cada fuente hasta obtener estructuras compatibles.

---

## 2. Diferentes frecuencias temporales

Los indicadores no comparten necesariamente la misma frecuencia de publicación.

Algunas series poseen información:

- Diaria.
- Mensual.
- Con fechas de corte particulares.

Esto requirió normalizar fechas y definir criterios adecuados para cada visualización.

---

## 3. Datos no disponibles

No todos los indicadores poseen datos para el mismo último período.

Por esta razón se implementó tratamiento de:

- Valores nulos.
- Períodos sin observaciones.
- Último valor disponible.
- Presentación de `N/D`.

---

## 4. Actualización desde la nube

Las conexiones que funcionan dentro de Power BI Desktop no necesariamente se comportan de la misma forma después de publicar el proyecto.

Fue necesario adaptar algunas consultas para hacerlas compatibles con el proceso de actualización desde Power BI Service.

---

# 📚 Principales aprendizajes

El desarrollo del proyecto permitió aplicar conocimientos relacionados con:

- Power Query.
- Lenguaje M.
- Consumo de APIs.
- Transformación de JSON.
- Extracción de tablas HTML.
- Integración de archivos Excel.
- SharePoint / OneDrive.
- Normalización de fechas.
- Modelado de datos.
- Automatización de consultas.
- Power BI Service.
- Seguridad de credenciales.
- Diseño de procesos de actualización.

---

# 🔗 Relación con DAX

Power Query se utilizó principalmente para:

- Extraer.
- Limpiar.
- Transformar.
- Integrar.
- Estructurar los datos.

Posteriormente, **DAX** se utilizó para desarrollar la capa analítica del modelo, incluyendo:

- Indicadores.
- Variaciones.
- Comparaciones temporales.
- Correlaciones.
- Storytelling dinámico.

La documentación de las principales medidas DAX puede consultarse en:

➡️ **[Medidas DAX principales](../DAX/medidas_principales.md)**

---

# 🏗️ Flujo completo del proyecto

De forma resumida:

```text
BCCR API ────────────────┐
                         │
BCCR Web ────────────────┤
                         │
INEC / IPC ──────────────┤
                         ▼
                 Power Query
                         │
                         ▼
              Limpieza y transformación
                         │
                         ▼
                 Modelo de datos
                         │
                         ▼
                       DAX
                         │
                         ▼
                Visualizaciones
                         │
                         ▼
                 Power BI Service
                         │
                         ▼
                 Actualización
```

---

# ⚠️ Consideraciones

Los ejemplos de Power Query incluidos en este documento tienen fines de:

- Aprendizaje.
- Documentación.
- Demostración técnica.
- Portafolio profesional.

Las fuentes oficiales pueden modificar sus estructuras, mecanismos de acceso o métodos de publicación en el futuro.

Por esta razón, las consultas podrían requerir mantenimiento conforme evolucionen las fuentes de información.

---

# 🔐 Seguridad del repositorio

Este repositorio contiene únicamente ejemplos técnicos y documentación.

No se publican elementos que permitan acceder a cuentas, archivos privados o servicios autenticados.

Cuando un fragmento de código depende de información sensible, se utiliza una referencia genérica o un parámetro sin revelar su contenido.

---

# 📊 Proyecto

**Panorama económico y financiero de Costa Rica | 2024–2026**

Proyecto desarrollado como parte de un portafolio profesional orientado a:

- Análisis de Datos.
- Business Intelligence.
- Power BI.
- Power Query.
- DAX.
- Integración de datos.
- Visualización de información.
- Análisis económico y financiero.
