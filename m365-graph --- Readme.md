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

## 🔐 Arquitectura de Seguridad

Este proyecto implementa dos principios de seguridad críticos:

### 1. **Least Privilege (Menor Privilegio)**
Cada agente tiene acceso **únicamente** a los permisos que necesita:
- **m365-graph**: solo puede **leer** datos (User.Read.All, Mail.Read, etc.)
- **m365-mailer**: solo puede **enviar correos** (Mail.Send)

Si un agente es comprometido, el atacante solo obtiene acceso a sus capacidades específicas, no al sistema completo.

### 2. **Data Compartmentalization (Compartimentalización de Datos)**
Cada agente tiene sus **propias credenciales independientes**:
- Diferentes App Registrations en Azure
- Diferentes Client IDs y Secrets
- Secrets almacenados en variables de entorno separadas

Esto asegura **aislamiento de datos**: si el secret de lectura se expone, el agente de envío permanece protegido y viceversa.

---

## 🔐 Requisitos Previos

### App Registrations en Azure (Separadas por Principio de Menor Privilegio)

Necesitas **dos App Registrations DIFERENTES** en tu tenant de Azure, cada una con permisos mínimos específicos:

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

### Variables de Entorno del Sistema (Compartimentalización de Datos)

Las credenciales **DEBEN** almacenarse en **Variables de Entorno del Sistema** (Machine-level), NO en archivos ni en código:

**Por qué variables de entorno:**
- ✅ No se commitean al repositorio (evita exposición accidental en Git)
- ✅ Separadas por máquina (credenciales locales, no sincronizadas)
- ✅ Acceso controlado a nivel de SO (permisos de administrador requeridos)
- ✅ No aparecen en logs de comandos (mayor privacidad)
- ❌ Nunca hardcodear credenciales en scripts
- ❌ Nunca guardar en archivos .config o .json en el repo

**Registra estas variables (nivel de máquina):**

```powershell
# App de lectura (Graph Reader) - SOLO lectura
[System.Environment]::SetEnvironmentVariable("M365_TENANT_ID", "tu-tenant-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_ID", "app-reader-client-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_SECRET", "app-reader-secret", "Machine")

# App de envío (Mail Sender) - SOLO envío de mail
[System.Environment]::SetEnvironmentVariable("M365_MAILER_CLIENT_ID", "app-mailer-client-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_MAILER_CLIENT_SECRET", "app-mailer-secret", "Machine")
```

⚠️ **Requiere permisos de administrador y reinicio de sesión**

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

## 🔒 Seguridad - Principios Implementados

### Least Privilege (Menor Privilegio)

Cada App Registration tiene **SOLO** los permisos que necesita:

| Agente | Permisos | Acceso |
|--------|----------|--------|
| **m365-graph** | `User.Read.All`, `Mail.Read`, `Calendars.Read.All` | 📖 Lectura únicamente |
| **m365-mailer** | `Mail.Send` | ✉️ Envío de correos únicamente |

Si un agente es comprometido, el atacante está limitado a esos permisos específicos.

### Data Compartmentalization (Compartimentalización de Datos)

Cada agente es **completamente independiente**:

```
┌─────────────────────────────────────────────────────┐
│           Azure Tenant (tu organización)            │
├──────────────────────┬──────────────────────────────┤
│   App Registration   │    App Registration          │
│    (Graph Reader)    │     (Mail Sender)            │
│                      │                              │
│ Client ID: XXX       │  Client ID: YYY             │
│ Secret: secretA      │  Secret: secretB            │
│ Perms: Read.All      │  Perms: Mail.Send           │
│                      │                              │
│ M365_CLIENT_ID       │  M365_MAILER_CLIENT_ID      │
│ M365_CLIENT_SECRET   │  M365_MAILER_CLIENT_SECRET  │
│                      │                              │
│ (Variables OS)       │  (Variables OS)             │
└──────────────────────┴──────────────────────────────┘
```

**Beneficios:**
- ✅ Si `secretA` se expone → solo se compromete lectura
- ✅ Si `secretB` se expone → solo se compromete envío
- ✅ Si una App es comprometida → la otra permanece segura
- ✅ Rotación independiente de secrets

### Almacenamiento de Credenciales

⚠️ **CRÍTICO - Nunca violar esto:**

| ❌ NUNCA | ✅ SIEMPRE |
|---------|----------|
| Hardcodear en scripts | Variables de Entorno del Sistema |
| Guardar en archivos JSON | Machine-level (no user-level) |
| Commitar al repositorio | Regenerar tras exposición |
| Usar repo privados como "vault" | Usar Azure Key Vault en producción |
| Compartir en chats/emails | Rotar automáticamente cada 90 días |

### Checklist de Seguridad

- [ ] Dos App Registrations creadas y separadas en Azure
- [ ] Cada App tiene **solo** los permisos necesarios
- [ ] Credenciales en Variables de Entorno del Sistema (Machine-level)
- [ ] `.gitignore` bloquea archivos de configuración con secrets
- [ ] No hay secrets en histórico de Git
- [ ] Admin consent dado en Azure AD
- [ ] Secretos rotados cada 90 días
- [ ] Acceso a las Apps registrado y auditado

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

## 📋 .gitignore - Protección contra Exposición Accidental

Asegúrate de que `.gitignore` bloquea archivos con credenciales:

```gitignore
# Credenciales y secretos
*.secret
*.key
*.pem
secrets/
credentials/
.env
.env.local
appsettings.*.json
web.config
app.config

# Logs con datos sensibles
*.log
logs/

# Archivos de configuración local
.claude/settings.local.json
local_config.json
config.machine.json

# Historiales de PowerShell
*_history.txt
ConsoleHost_history.txt

# Temporal y caché
temp/
.cache/
obj/
bin/
```

**Verificar antes de pushear:**
```powershell
git status  # Asegúrate de que NO hay archivos de credenciales
git log --oneline --all -- "*.secret" "*.key"  # Verifica histórico
```

---


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
