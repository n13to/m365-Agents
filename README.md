# M365 Autonomous Agents

Dos agentes autónomos especializados en Microsoft 365 que operan mediante App Registrations en Azure con acceso controlado a Microsoft Graph API.

## 📋 Descripción

Este proyecto contiene dos agentes independientes:

### 1. **m365-graph** – Agente de Lectura
Especialista en consultas y lecturas de datos desde Microsoft 365 (usuarios, grupos, buzones, calendarios, etc.)

**Permisos:** Lectura en Microsoft Graph  
**Casos de uso:** Auditorías, reportes, consultas de datos, búsquedas

### 2. **m365-mailer** – Agente de Envío de Correos
Especialista en automatización de envío de notificaciones y correos electrónicos vía Graph API.

**Permisos:** Envío de correos en Microsoft Graph  
**Casos de uso:** Notificaciones automatizadas, alertas, confirmaciones

---

## 🔐 Requisitos Previos

### App Registrations en Azure

Necesitas **dos App Registrations** diferentes en tu tenant de Azure:

#### App 1: Graph Reader
```
Client ID: <M365_CLIENT_ID>
Tenant ID: <M365_TENANT_ID>
Client Secret: <M365_CLIENT_SECRET>

Permisos requeridos (API Permissions):
- User.Read.All
- Group.Read.All
- Mail.Read
- Calendars.Read.All
- (adiciona según necesites)
```

#### App 2: Mail Sender
```
Client ID: <M365_MAILER_CLIENT_ID>
Tenant ID: <M365_TENANT_ID>
Client Secret: <M365_MAILER_CLIENT_SECRET>

Permisos requeridos (API Permissions):
- Mail.Send
```

### Variables de Entorno del Sistema

Registra estas variables en **Variables de Entorno del Sistema** (no de usuario):

```powershell
# App de lectura (Graph Reader)
M365_TENANT_ID          = "tu-tenant-id"
M365_CLIENT_ID          = "app-reader-client-id"
M365_CLIENT_SECRET      = "app-reader-client-secret"

# App de envío (Mail Sender)
M365_MAILER_CLIENT_ID       = "app-mailer-client-id"
M365_MAILER_CLIENT_SECRET   = "app-mailer-client-secret"
```

---

## 🚀 Instalación

### 1. Clonar el repositorio
```bash
git clone https://github.com/tu-usuario/m365-agents.git
cd m365-agents
```

### 2. Instalar Microsoft Graph PowerShell SDK
```powershell
Install-Module Microsoft.Graph -Scope CurrentUser -Force
```

### 3. Configurar variables de entorno
Edita las variables del sistema con tus credenciales de Azure (ver sección anterior).

### 4. Verificar la conexión
```powershell
# Script de prueba
.\scripts\check_app_perms.ps1
```

---

## 📖 Uso

### Agente m365-graph (Lectura)

Los scripts se conectan automáticamente usando las credenciales del sistema:

```powershell
# Listar usuarios
.\List-MgUsers.ps1

# Consultar buzones
.\get_recent_user.ps1

# Crear logs de auditoría
.\Create-AgentLog.ps1
```

**Bloque de autenticación automática:**
```powershell
$tenantId     = [System.Environment]::GetEnvironmentVariable("M365_TENANT_ID", "Machine")
$clientId     = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_ID", "Machine")
$clientSecret = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_SECRET", "Machine")

$SecureSecret = ConvertTo-SecureString $clientSecret -AsPlainText -Force
$Credential   = New-Object System.Management.Automation.PSCredential($clientId, $SecureSecret)
Connect-MgGraph -TenantId $tenantId -ClientSecretCredential $Credential
```

### Agente m365-mailer (Envío de Correos)

Los scripts se conectan automáticamente con las credenciales del mailer:

```powershell
# Enviar correo (ejemplo)
Send-MgUserMessage -UserId "usuario@dominio.com" -Message $messageBody
```

**Bloque de autenticación automática:**
```powershell
$SecureSecret = ConvertTo-SecureString $env:M365_MAILER_CLIENT_SECRET -AsPlainText -Force
$Credential   = New-Object System.Management.Automation.PSCredential($env:M365_MAILER_CLIENT_ID, $SecureSecret)
Connect-MgGraph -TenantId $env:M365_TENANT_ID -ClientSecretCredential $Credential -NoWelcome
```

---

## 📁 Estructura del Proyecto

```
m365-agents/
├── README.md                          # Este archivo
├── m365-graph.md                      # Documentación del agente de lectura
├── m365-mailer                        # Configuración del agente mailer
├── scripts/
│   ├── check_app_certs.ps1           # Verificar certificados de apps
│   ├── check_app_perms.ps1           # Verificar permisos de apps
│   ├── decode_approles.ps1           # Decodificar roles de app
│   └── ...
├── List-MgUsers.ps1                  # Ejemplo: listar usuarios
├── Create-AgentLog.ps1               # Ejemplo: crear logs
├── get_recent_user.ps1               # Ejemplo: usuarios recientes
└── .claude/settings.local.json        # Configuración local
```

---

## 🔒 Seguridad

⚠️ **Importante:**
- **Nunca** commits credenciales al repositorio
- Las credenciales deben estar en **Variables de Entorno del Sistema**, no en archivos
- Usa secretos diferentes para cada agente (lectura vs. envío)
- Revisa regularmente los permisos otorgados en Azure
- Revoca secretos expirados en Azure AD

---

## ✅ Validación y Testing

### Verificar conexión exitosa
```powershell
.\scripts\check_app_perms.ps1
```

### Validar permisos
```powershell
.\scripts\decode_approles.ps1
```

### Ver certificados/secretos
```powershell
.\scripts\check_app_certs.ps1
```

---

## 📝 Logs y Auditoría

Los agentes generan logs en:
```
./logs/
├── graph-reader-<date>.log
├── mailer-<date>.log
└── agent-audit.log
```

Para crear nuevos logs:
```powershell
.\Create-AgentLog.ps1
```

---

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor:
1. Crea un branch (`git checkout -b feature/mi-feature`)
2. Commit los cambios (`git commit -am 'Agrega nueva feature'`)
3. Push al branch (`git push origin feature/mi-feature`)
4. Abre un Pull Request

---

## 📄 Licencia

Este proyecto está bajo licencia [MIT](LICENSE).

---

## 📧 Contacto

Para preguntas o problemas, abre un [Issue](../../issues) en GitHub.

---

## 🛠️ Troubleshooting

### Error: "Connect-MgGraph: Insufficient privileges"
- Verifica que la App Registration tenga los permisos necesarios en Azure
- Admin consent debe estar dado en Azure AD

### Error: "Variable de entorno no encontrada"
- Verifica que las variables estén en **Variables del Sistema** (no de usuario)
- Reinicia PowerShell después de crear las variables
- Usa: `[System.Environment]::GetEnvironmentVariable("M365_TENANT_ID", "Machine")`

### Error: "ClientSecret expired"
- Crea un nuevo secret en la App Registration de Azure
- Actualiza la variable de entorno con el nuevo secret

---

**Última actualización:** Junio 2026  
**Versión:** 1.0
