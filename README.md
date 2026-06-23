# M365 Autonomous Agents

Dos agentes autónomos especializados en Microsoft 365 que operan mediante App Registrations en Azure con acceso controlado a Microsoft Graph API. Los agentes generan dinámicamente los scripts necesarios según tus consultas.

## 📋 Descripción

Este proyecto contiene dos agentes independientes:

### 1. **m365-graph** – Agente de Lectura
Especialista en consultas y lecturas de datos desde Microsoft 365. Genera scripts PowerShell automáticamente según tus necesidades (usuarios, grupos, buzones, calendarios, auditorías, reportes, búsquedas, etc.)

### 2. **m365-mailer** – Agente de Envío de Correos
Especialista en automatización de envío de notificaciones y correos electrónicos. Genera scripts para campañas de mail, alertas, confirmaciones, y notificaciones automáticas.

---

## 🚀 Requisitos Previos

### Instalación
- PowerShell 7.0+ o Windows PowerShell 5.1+
- Microsoft Graph PowerShell SDK: `Install-Module Microsoft.Graph -Scope CurrentUser -Force`
- Acceso administrativo (requerido para variables de entorno del sistema)

### Azure
- Dos App Registrations creadas en tu tenant de Azure
- Secrets generados para cada App
- Admin consent dado en Azure AD

### Variables de Entorno del Sistema
```
M365_TENANT_ID
M365_CLIENT_ID
M365_CLIENT_SECRET
M365_MAILER_CLIENT_ID
M365_MAILER_CLIENT_SECRET
```

---

## 🔒 Arquitectura de Seguridad

Este proyecto implementa dos principios de seguridad críticos:

### 1. **Least Privilege (Menor Privilegio)**

Cada agente tiene acceso **únicamente** a los permisos que necesita:

| Agente | Permisos | Capacidad |
|--------|----------|-----------|
| **m365-graph** | `User.Read.All`, `Mail.Read`, `Calendars.Read.All`, `Group.Read.All`, etc. | 📖 Lectura únicamente |
| **m365-mailer** | `Mail.Send` | ✉️ Envío de correos únicamente |

**Beneficio:** Si un agente es comprometido, el atacante obtiene acceso **solo** a esas capacidades específicas. El agente de lectura no puede enviar correos, y el de correos no puede leer datos.

### 2. **Data Compartmentalization (Compartimentalización de Datos)**

Cada agente tiene **App Registrations independientes** con credenciales separadas:

```
┌─────────────────────────────────────────────────────────┐
│         Azure Tenant (tu organización)                  │
├────────────────────────────┬────────────────────────────┤
│   App Registration #1      │   App Registration #2      │
│   (Graph Reader)           │   (Mail Sender)            │
│                            │                            │
│ Client ID: READER_ID       │ Client ID: MAILER_ID      │
│ Secret: READER_SECRET      │ Secret: MAILER_SECRET     │
│ Perms: Read.All            │ Perms: Mail.Send          │
│                            │                            │
│ m365-graph Agent           │ m365-mailer Agent         │
│ (Variables de Entorno)     │ (Variables de Entorno)    │
└────────────────────────────┴────────────────────────────┘
```

**Beneficios:**
- ✅ Si `READER_SECRET` se expone → solo se compromete lectura
- ✅ Si `MAILER_SECRET` se expone → solo se compromete envío
- ✅ Si una App es comprometida → la otra permanece segura
- ✅ Rotación independiente de secrets (cada agente por su lado)
- ✅ Auditoría separada en Azure (qué hizo cada agente)

### 3. **Variables de Entorno del Sistema (Machine-Level)**

Las credenciales **NUNCA** se guardan en archivos, código o repositorio. Se almacenan en **Variables de Entorno del Sistema** (Machine-level):

**Por qué:**
- ✅ No pasan por el repositorio (evita exposición accidental en Git)
- ✅ Separadas por máquina (credenciales locales, no sincronizadas)
- ✅ Acceso controlado a nivel de SO (permisos de administrador requeridos)
- ✅ No aparecen en logs de comandos PowerShell normales
- ✅ Aisladas de sincronización en la nube (OneDrive, Teams, etc.)

**Comparación:**

| Método | Seguridad | Risk |
|--------|-----------|------|
| Hardcodear en script | ❌ Muy baja | Alto riesgo en Git |
| Guardar en JSON/config | ❌ Baja | Exposición accidental |
| **Variables de Entorno (Machine)** | ✅ **Alta** | **Mínimo** |
| Azure Key Vault | ✅✅ Crítica | Nulo (producción) |

### 4. **Ejecución Automática sin Intervención**

Cada script generado incluye el bloque de autenticación que extrae las credenciales de las variables de entorno:

```powershell
# m365-graph: Lectura
$tenantId     = [System.Environment]::GetEnvironmentVariable("M365_TENANT_ID", "Machine")
$clientId     = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_ID", "Machine")
$clientSecret = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_SECRET", "Machine")

$SecureSecret = ConvertTo-SecureString $clientSecret -AsPlainText -Force
$Credential   = New-Object System.Management.Automation.PSCredential($clientId, $SecureSecret)
Connect-MgGraph -TenantId $tenantId -ClientSecretCredential $Credential
```

```powershell
# m365-mailer: Envío
$SecureSecret = ConvertTo-SecureString $env:M365_MAILER_CLIENT_SECRET -AsPlainText -Force
$Credential   = New-Object System.Management.Automation.PSCredential($env:M365_MAILER_CLIENT_ID, $SecureSecret)
Connect-MgGraph -TenantId $env:M365_TENANT_ID -ClientSecretCredential $Credential -NoWelcome
```

### 📋 Checklist de Seguridad

- [ ] Dos App Registrations creadas en Azure (separadas)
- [ ] Cada App tiene **solo** los permisos mínimos necesarios
- [ ] Credenciales en Variables de Entorno del Sistema (Machine-level)
- [ ] `.gitignore` bloquea archivos de configuración
- [ ] No hay secrets en histórico de Git (`git log --all -- "*.secret"`)
- [ ] Admin consent dado en Azure AD
- [ ] Secrets rotados cada 90 días
- [ ] Acceso auditado en Azure (Sign-in logs)

---

## 🎯 Cómo Usar

### 1. Consulta al Agente vía Consola Claude

Los agentes generan scripts dinámicamente según tus consultas. Simplemente describe lo que necesitas:

**Ejemplos para m365-graph (Lectura):**
```
"Dame un listado de todos los usuarios de Microsoft 365"
"Busca usuarios creados en los últimos 30 días"
"Dame un reporte de buzones con más de 50GB"
"Extrae miembros del grupo 'Finanzas'"
"Audita usuarios con acceso a Exchange Online"
```

**Ejemplos para m365-mailer (Envío):**
```
"Envía un correo a usuario@dominio.com con asunto 'Alerta'"
"Haz una campaña de mail a todos los usuarios informando cambio de contraseña"
"Envía recordatorios de reunión a todos los participantes"
"Notifica a los admins de acciones completadas"
```

### 2. El Agente Genera el Script Automáticamente

El agente:
1. Lee tu consulta
2. Genera un script PowerShell específico para esa tarea
3. Incluye automáticamente la autenticación con las variables de entorno
4. Ejecuta el script
5. Te devuelve los resultados

### 3. Reutiliza o Adapta

Puedes:
- Guardar scripts útiles para uso futuro
- Pedirle al agente que los modifique
- Programarlos con Task Scheduler
- Integrarlos en pipelines de automatización

---

## 📁 Estructura del Proyecto

```
m365-agents/
├── README.md                          # Este archivo
├── m365-graph.md                      # Instrucciones del agente de lectura
├── m365-mailer                        # Instrucciones del agente mailer
├── scripts/                           # Scripts generados previamente (referencia)
│   ├── check_app_perms.ps1           
│   ├── check_app_certs.ps1           
│   └── ...
├── logs/                              # Logs generados por los agentes
├── .claude/settings.local.json        # Configuración local
└── .gitignore                         # Protege credenciales
```

**Nota:** Los scripts en `scripts/` son ejemplos generados previamente. Los agentes generarán nuevos scripts cada vez que los consultes.

---

## 🔧 Instalación y Configuración

### 1. Clonar el repositorio
```bash
git clone https://github.com/tu-usuario/m365-agents.git
cd m365-agents
```

### 2. Instalar Microsoft Graph PowerShell SDK
```powershell
Install-Module Microsoft.Graph -Scope CurrentUser -Force
```

### 3. Crear App Registrations en Azure

**Para m365-graph (Lectura):**
1. Azure Portal → App registrations → New registration
2. Name: "M365-Graph-Reader"
3. Certificates & secrets → New client secret
4. API Permissions → Add: User.Read.All, Mail.Read, Calendars.Read.All, Group.Read.All
5. Grant admin consent

**Para m365-mailer (Envío):**
1. Azure Portal → App registrations → New registration
2. Name: "M365-Mail-Sender"
3. Certificates & secrets → New client secret
4. API Permissions → Add: Mail.Send
5. Grant admin consent

### 4. Configurar Variables de Entorno del Sistema

Abre PowerShell como administrador y ejecuta:

```powershell
# App Reader
[System.Environment]::SetEnvironmentVariable("M365_TENANT_ID", "tu-tenant-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_ID", "app-reader-client-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_SECRET", "app-reader-secret", "Machine")

# App Mailer
[System.Environment]::SetEnvironmentVariable("M365_MAILER_CLIENT_ID", "app-mailer-client-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_MAILER_CLIENT_SECRET", "app-mailer-secret", "Machine")
```

⚠️ **Requiere reinicio de sesión PowerShell**

### 5. Verificar Configuración
```powershell
# Comprueba que las variables están configuradas
[System.Environment]::GetEnvironmentVariable("M365_TENANT_ID", "Machine")
[System.Environment]::GetEnvironmentVariable("M365_CLIENT_ID", "Machine")
# etc.
```

---

## 📋 .gitignore - Protección contra Exposición Accidental

El repositorio debe incluir `.gitignore` para bloquear credenciales:

```gitignore
# Credenciales y secretos
*.secret
*.key
*.pem
secrets/
credentials/
.env
.env.local
.env.production
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

# Historiales
*_history.txt
ConsoleHost_history.txt

# Caché y temporal
temp/
.cache/
obj/
bin/
```

**Antes de hacer push:**
```powershell
git status  # Verifica que NO hay archivos de credenciales
git log --oneline --all -- "*.secret" "*.key"  # Busca secrets en histórico
```

---

## 🛠️ Troubleshooting

### Error: "Connect-MgGraph: Insufficient privileges"
- Verifica que la App Registration tenga los permisos en Azure
- Asegúrate de dar **Admin consent** en Azure AD
- Espera 5 minutos para que los permisos se propaguen

### Error: "Variable de entorno no encontrada"
- Verifica que estén en **Variables del Sistema** (Machine), no usuario
- Reinicia PowerShell completamente
- Ejecuta como administrador

### Error: "ClientSecret expired"
- Crea un nuevo secret en la App Registration de Azure
- Actualiza la variable de entorno con el nuevo secret
- Revoca el secret antiguo en Azure

### Los agentes no generan scripts
- Verifica conexión a internet
- Comprueba que tienes acceso a la consola Claude
- Reinicia la sesión de Claude

---

## 📄 Licencia

Este proyecto está bajo licencia [MIT](LICENSE).

---

## 📧 Contacto

Para preguntas o problemas sobre los agentes, abre un [Issue](../../issues) en GitHub.

---

**Última actualización:** Junio 2026  
**Versión:** 1.0
