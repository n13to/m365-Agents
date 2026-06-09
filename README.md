# 📝 Documentación Técnica: Agente Autónomo M365 con Claude Code

Esta documentación detalla la arquitectura, configuración y uso del agente personalizado `m365-graph` para automatizar la administración de Microsoft 365 de forma desatendida y segura mediante **Claude Code (CLI)** y **PowerShell**.

---

## 1. Arquitectura y Componentes
El ecosistema de este agente se compone de tres pilares fundamentales:

| Componente | Elemento | Propósito y Función |
| :--- | :--- | :--- |
| **Identidad (Entra ID)** | App Registration | Una identidad de aplicación con permisos de tipo "Application" (no delegados) y Consentimiento de Administrador concedido en el portal de Azure. |
| **Seguridad (Local)** | Variables de Entorno | Las credenciales sensibles se almacenan de forma local en el sistema operativo del administrador, quedando ocultas para el modelo de IA. |
| **Contexto (Claude)** | Archivo Markdown | Instrucciones del sistema que enseñan a Claude cómo realizar el apretón de manos con Microsoft Graph en cada script que genera. |

---

## 2. Guía de Configuración Inicial

### Paso A: Variables de Entorno (Windows)
Para que el agente funcione de forma autónoma, debe leer las credenciales del entorno local. Ejecuta los siguientes comandos una sola vez en una consola de PowerShell como Administrador (reemplazando con tus datos reales):

```powershell
[Environment]::SetEnvironmentVariable("M365_TENANT_ID", "TU_TENANT_ID_AQUI", "User")
[Environment]::SetEnvironmentVariable("M365_CLIENT_ID", "TU_CLIENT_ID_AQUI", "User")
[Environment]::SetEnvironmentVariable("M365_CLIENT_SECRET", "TU_CLIENT_SECRET_AQUI", "User")
