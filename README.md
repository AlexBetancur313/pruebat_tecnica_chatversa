# Agente de Automatización de Documentos (Cotizaciones y CxC)

Este proyecto es un workflow de n8n diseñado como solución a la prueba técnica de **Iyara (evaluacion_automatizacion_datos_iyata_v7.pdf)**. El objetivo es automatizar el proceso completo de generación de documentos financieros (Cotizaciones y Cuentas de Cobro) a partir de una entrada de lenguaje natural, como un mensaje de WhatsApp.

El workflow actúa como un agente de IA que recibe una solicitud, la comprende, genera un documento PDF profesional, lo archiva en Google Drive, lo registra en Google Sheets y (opcionalmente) notifica al usuario.

## 🚀 Características Principales

- **Procesamiento de Lenguaje Natural (PLN):** Utiliza un nodo `Information Extractor` (compatible con OpenAI) para analizar texto no estructurado (ej. "Necesito una cotización para 10 tornillos y 5 tuercas") y convertirlo en JSON estructurado.
- **Generación Dinámica de PDF:** Crea documentos PDF con un diseño profesional y limpio, maquetado con tablas HTML (`<table>`) para máxima compatibilidad con los motores de renderizado de PDF.
- **Enrutamiento Inteligente:** Un nodo `IF` detecta si el documento solicitado es una "Cotización" o una "CxC" y dirige el flujo a diferentes carpetas de Google Drive y hojas de Google Sheets.
- **Gestión de Consecutivos:** Lee una hoja de Google Sheets para obtener el último número de documento y genera el siguiente (ej. "047", "048").
- **Integración Total con Google:**
  - **Google Drive:** Almacena el PDF final en la carpeta correcta (`/Cotizaciones` o `/CxC`).
  - **Google Sheets:** Registra cada transacción en un "libro" contable correspondiente.
- **Notificaciones:** Preparado para enviar confirmaciones vía WhatsApp usando una API local (Evolution API).
- **Manejo de Errores y Seguridad:** Incluye una validación inicial que bloquea peticiones de números no autorizados.

---

## ⚙️ Componentes Clave

El workflow utiliza los siguientes servicios y nodos principales:

- **n8n:** Orquestador principal del flujo.
- **Webhook:** Punto de entrada para recibir solicitudes (ej. desde una API de WhatsApp).
- **IA (LLM):** Nodo `Information Extractor` para el PLN.
- **Google Sheets:** Para leer datos (consecutivos, info del emisor) y escribir registros.
- **Google Drive:** Para almacenar los PDFs generados.
- **PdForge:** (o similar) Para convertir el HTML dinámico en un archivo PDF.
- **API de WhatsApp (HTTP Request):** Para enviar notificaciones.

---

## 📈 Detalle del Flujo de Trabajo (Paso a Paso)

El workflow sigue esta secuencia lógica:

1.  **Recepción (Webhook):** Espera una llamada HTTP con los datos de la solicitud (ej. el mensaje del cliente y su número).
2.  **Validación (IF):** Un nodo `IF` (`IF1`) comprueba si el número del remitente está autorizado.
    - **Si Falla:** Envía un mensaje de "Número no autorizado" y termina.
    - **Si Pasa:** Continúa al siguiente paso.
3.  **Extracción (Information Extractor):** El mensaje del cliente se pasa a un LLM (OpenAI) que extrae los datos clave (`tipo_operacion`, `cliente`, `items`) basado en un schema JSON predefinido.
4.  **Recolección de Datos (Google Sheets):**
    - `Get Info Emisor`: Obtiene los datos de _tu empresa_ (nombre, NIT, etc.) desde una hoja de Google Sheets para ponerlos en el PDF.
    - `obtener informacion de consecutivos`: Lee otra hoja para obtener el último ID de documento.
5.  **Procesamiento (Code - `Organizar el JSON con info`):** Este nodo de código es el cerebro central:
    - Calcula el nuevo ID (ej. 46 + 1 = 47).
    - Calcula los totales (`subtotal`, `iva`, `total_factura`) basado en los `items` extraídos.
    - Genera el `fileName` (ej. "047 CxC Sra. Clara Eriquez.pdf").
    - Combina todos los datos (del extractor, del emisor, y los cálculos) en un solo objeto JSON.
6.  **Generación de PDF:**
    - `Code - Codigo para generar html...`: Toma el JSON y lo inserta en una plantilla HTML robusta (diseñada con `<table>` para compatibilidad con PDF).
    - `Convertir HTML a PDF`: El nodo `pdforge` recibe el HTML y devuelve una `signedUrl` (URL temporal) del PDF generado.
7.  **Fusión de Datos (Merge):** **Este es un paso crítico.** El nodo `Merge` recombina la `signedUrl` (salida del PDF) con los datos originales de `Organizar el JSON con info` (que se habían perdido en el paso anterior). Ahora tenemos un solo item con _toda_ la información.
8.  **Descarga (HTTP Request):** El nodo `Descarga el PDF` usa la `signedUrl` para descargar el archivo PDF a n8n como un dato binario.
9.  **Enrutamiento (IF - `El tipo de documento es:`):** El flujo se divide según el `tipoOperacion`.
10. **Rama "Cotización":**
    - `Upload file`: Sube el PDF binario a la carpeta de "Cotizaciones" en Google Drive.
    - `Add row(s) in sheet1`: Registra los detalles en la hoja de Google Sheets de "Cotizaciones".
    - `Edit Fields` / `Envio de Confirmacion`: Prepara y envía la notificación de WhatsApp.
11. **Rama "CxC":**
    - `Upload file1`: Sube el PDF binario a la carpeta de "CxC".
    - `Add row(s) in sheet`: Registra los detalles en la hoja de Google Sheets de "CxC".
    - `Edit Fields1` / `Envio de Confirmacion...`: Prepara y envía la notificación de WhatsApp.

---

## 🔧 Configuración y Prerrequisitos

Para que este workflow funcione, es necesario configurar las siguientes credenciales y IDs:

### 1. Credenciales

Asegúrate de tener credenciales válidas en n8n para:

- **Google (OAuth2):** Con permisos para Google Drive y Google Sheets.
- **PdForge (API Key):** Para la conversión a PDF.
- **API de WhatsApp (Header Auth):** Una `apikey` para tu API de WhatsApp (ej. Evolution API).

### 2. Configuración de Nodos

Deberás actualizar los siguientes nodos con tus propios IDs:

- **Google Sheets (Get Info Emisor):**
  - `Spreadsheet ID`: El ID de tu hoja de Google Sheets.
  - `Range`: El rango donde están los datos de tu empresa (ej. `DatosEmisor!A2:E2`).
- **Google Sheets (obtener informacion de consecutivos):**
  - `Spreadsheet ID`: El ID de tu hoja de Google Sheets.
  - `Range`: El rango donde está el último ID (ej. `Consecutivos!B1`).
- **Nodos de Google Drive (Upload file y Upload file1):**
  - `Folder ID`: El ID de la carpeta de destino en Google Drive para "Cotizaciones" y "CxC" respectivamente.
- **Nodos de Google Sheets (Add row(s)...):**
  - `Spreadsheet ID`: El ID de tu hoja de registro.
  - `Sheet Name`: El nombre de la hoja específica para "Cotizaciones" y "CxC".
- **Nodos HTTP (Envio de Confirmacion...):**
  - `URL`: La URL de tu API de WhatsApp (actualmente `http://192.168.0.15:8080/...`).

### 3. Hojas de Cálculo de Configuración

El workflow depende de varias Hojas de Cálculo de Google para operar. A continuación se anexa el enlace al archivo que contiene la configuración de los consecutivos, la información del emisor y los registros de documentos:

- **Archivo Maestro de Google Sheets:** [evaluacion_automatizacion_datos_iyata_v7.xlsx](https://docs.google.com/spreadsheets/d/1UbHJ7vmsS3t5_ROl8UalW3LFcX4yWtet/edit?usp=sharing&ouid=106382798697741552583&rtpof=true&sd=true)

Este archivo contiene las siguientes pestañas:

- `Dashboard`: Consecutivos e IDs de carpetas.
- `Registro CxC`: Historial de Cuentas de Cobro generadas.
- `Registro Cotizaciones`: Historial de Cotizaciones generadas.
- `Info Emisor`: Los datos de la empresa/persona que emite los documentos.
- `Lista Contactos`: (Opcional) Para validación de números autorizados.

---

## ⚠️ Nota sobre el Estado del Proyecto

Este repositorio documenta la arquitectura y el desarrollo de la solución de automatización con n8n (correspondiente a la Parte A de la evaluación).

Es importante notar que, debido a la **alta complejidad** de la prueba (especialmente los retos de limpieza y análisis de datos en la Parte B) y al **tiempo disponible** para su desarrollo, no todos los puntos de la evaluación (`evaluacion_automatizacion_datos_iyata_v7.pdf`) fueron completados. El foco principal del trabajo se centró en la construcción del flujo de generación de documentos.
