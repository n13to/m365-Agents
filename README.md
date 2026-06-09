# 📝 Documentación Técnica: Agente Autónomo M365 con Claude Code

Esta documentación detalla la arquitectura, configuración y uso del agente personalizado m365-graph para automatizar la administración de Microsoft 365 de forma desatendida y segura mediante Claude Code (CLI) y PowerShell.

---

## 1. Arquitectura y Componentes
El ecosistema de este agente se compone de tres pilares fundamentales:

| Componente | Elemento | Propósito y Función |
| :--- | :--- | :--- |
| Identidad (Entra ID) | App Registration | Una identidad de aplicación con permisos de tipo Application (no delegados) y Consentimiento de Administrador concedido en el portal de Azure. |
| Seguridad (Local) | Variables de Entorno | Las credenciales sensibles se almacenan de forma local en el sistema operativo del administrador, quedando ocultas para el modelo de IA. |
| Contexto (Claude) | Archivo Markdown | Instrucciones del sistema que enseñan a Claude cómo realizar el apretón de manos con Microsoft Graph en cada script que genera. |

---

## 2. Guía de Configuración Inicial

### Paso A: Variables de Entorno (Windows)
Para que el agente funcione de forma autónoma, debe leer las credenciales del entorno local. Ejecuta los siguientes comandos una sola vez en una consola de PowerShell como Administrador (reemplazando con tus datos reales):

    [Environment]::SetEnvironmentVariable("M365_TENANT_ID", "TU_TENANT_ID_AQUI", "User")
    [Environment]::SetEnvironmentVariable("M365_CLIENT_ID", "TU_CLIENT_ID_AQUI", "User")
    [Environment]::SetEnvironmentVariable("M365_CLIENT_SECRET", "TU_CLIENT_SECRET_AQUI", "User")

> ¡CRUCIAL! Si tenías Visual Studio Code abierto durante este paso, es obligatorio cerrarlo por completo y volverlo a abrir para que la terminal integrada herede las nuevas variables del sistema operativo.

### Paso B: Archivo de Configuración del Agente
Dentro de tu proyecto de trabajo abierto en VS Code, crea la siguiente estructura exacta:
* Ruta del archivo: .claude/agents/m365-graph.md

Contenido del archivo (m365-graph.md):

---
name: m365-graph
description: Agente autónomo especialista en M365 con conexión automatizada vía App Registration.
tools: Read, Edit, Bash
---

Eres un Administrador de Sistemas Cloud Senior experto en Microsoft 365 y Microsoft Graph PowerShell.

### REGLAS DE AUTENTICACIÓN (CRÍTICO):
1. El entorno ya cuenta con las siguientes variables de entorno del sistema: $env:M365_TENANT_ID, $env:M365_CLIENT_ID y $env:M365_CLIENT_SECRET.
2. Todos tus scripts deben iniciar obligatoriamente con el siguiente bloque de conexión automatizada para no requerir intervención humana:

    $Secret = ConvertTo-SecureString $env:M365_CLIENT_SECRET -AsPlainText -Force
    $GraphParams = @{
        TenantId     = $env:M365_TENANT_ID
        ClientId     = $env:M365_CLIENT_ID
        ClientSecret = $Secret
    }
    Connect-MgGraph @GraphParams

3. Al finalizar el script, incluye siempre Disconnect-MgGraph por buenas prácticas de seguridad.
4. NUNCA escribas los valores reales del TenantID, ClientID o Secret en el código. Usa siempre las variables de entorno.

### REGLAS DE OPERACIÓN:
1. Consola: Asume que estás operando en una terminal con PowerShell 7+ y que el módulo Microsoft.Graph ya está instalado.
2. Seguridad: Antes de ejecutar cualquier comando que modifique o elimine datos (Set-Mg*, Update-Mg*, Remove-Mg*), debes mostrarle el script al usuario en pantalla y explicar el impacto exacto.
3. Optimización: Prefiere el uso de filtros del lado del servidor (-Filter) en lugar de traer miles de objetos y filtrarlos en local con Where-Object. Use Select-Object para limitar propiedades.

---

## 3. Modo de Uso en el Día a Día

### 1. Inicialización en "Modo Manos Libres"
Abre la terminal integrada de VS Code (PowerShell) e inicia la herramienta con la bandera para saltar confirmaciones repetitivas:

    claude --dangerously-skip-permissions

### 2. Activación del Agente
Una vez dentro del prompt interactivo de Claude (Claude >), invoca al especialista:

    agent m365-graph

### 3. Ejemplos de Prompts Reales
Ya puedes pedirle tareas directamente en lenguaje natural:
* "Necesito un reporte en CSV de todos los usuarios que tengan una licencia de 'Power BI Premium' asignada pero que lleven más de 90 días sin iniciar sesión. Guarda el CSV en esta carpeta."
* "Enumera los 5 grupos de Teams más antiguos del tenant indicando su fecha de creación y sus propietarios."

---

## 4. Resolución de Problemas (Troubleshooting)

1. Error: Unexpected token '-graph' in expression
   * Causa: Escribiste el comando directamente en PowerShell.
   * Solución: Entra primero a Claude con `claude --dangerously-skip-permissions` y luego pon `agent m365-graph`.
2. Error: Connect-MgGraph: ClientSecret cannot be null
   * Causa: VS Code no se enteró de las nuevas variables de entorno.
   * Solución: Reinicia VS Code por completo.
3. Error: Authorization_RequestDenied
   * Causa: Tu App Registration no tiene permisos en Azure para esa consulta específica.
   * Solución: Ve a Azure Portal, añade el permiso como tipo Application y dale a "Grant admin consent".
