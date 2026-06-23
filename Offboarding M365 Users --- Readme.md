# Microsoft 365 Offboarding Process - Agentes Autónomos Basados en Grafos

## Descripción General

Este proyecto contiene un **sistema completo de offboarding de empleados en Microsoft 365** implementado como una arquitectura de **agentes autónomos especializados** basada en grafos de estado, con reintentos automáticos, manejo de errores granular y auditoría completa.

### Flujo de 9 Nodos Especializados

```
┌─────────────┐
│  TRIGGER    │  (Nueva fila en SharePoint)
└──────┬──────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ NODE_1: ORCHESTRATOR (Fetch User Data)                   │
│ Leer de SPO, validar en Azure AD                        │
└──────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ NODE_2-3: SECURITY EXECUTOR (Revoke/Block)              │
│ Revocar sesiones + bloquear sign-in                    │
└──────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ NODE_4-5: DATA PRESERVATION (Mailbox/OneDrive)          │
│ Convertir buzón + delegar OneDrive (⚠️ CRÍTICO)         │
└──────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ NODE_6-7: CLEANUP EXECUTOR (Groups/Licenses)            │
│ Remover de grupos + retirar licencias                  │
└──────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ NODE_8-9: FINALIZE (Update SPO + Notify)                │
│ Actualizar SharePoint + enviar notificación            │
└──────────────────────────────────────────────────────────┘
```

---

## Agentes Disponibles

### 1. **m365-offboarding-orchestrator**
**Rol:** Coordinador principal  
**Nodos:** NODE_1 (Fetch User Data)  
**Responsabilidad:** Iniciar flujo, leer datos de SharePoint, validar usuario en Azure AD  
**Crítico:** ✓ SÍ (detiene si usuario no existe)

### 2. **m365-offboarding-security-executor**
**Rol:** Especialista en seguridad inmediata  
**Nodos:** NODE_2 (Revoke Sessions), NODE_3 (Block SignIn)  
**Responsabilidad:** Invalidar sesiones, bloquear acceso  
**Crítico:** ✗ NO (continúa si falla)

### 3. **m365-offboarding-data-preservation**
**Rol:** Especialista en preservación de datos  
**Nodos:** NODE_4 (Convert Mailbox), NODE_5 (Delegate OneDrive)  
**Responsabilidad:** Convertir buzón a compartido, delegar acceso  
**Crítico:** ✓ SÍ en NODE_4 (pausa si falla)

### 4. **m365-offboarding-cleanup-executor**
**Rol:** Especialista en limpieza y optimización  
**Nodos:** NODE_6 (Remove Groups), NODE_7 (Remove Licenses)  
**Responsabilidad:** Remover de grupos, retirar licencias  
**Crítico:** ✗ NO (continúa si falla)

---

## Requisitos Previos

### App Registration en Azure
Requiere **UNA App Registration** con permisos:

```
API Permissions:
✓ User.Read.All
✓ User.ReadWrite.All
✓ Group.Read.All
✓ Group.ReadWrite.All
✓ Directory.ReadWrite.All
✓ Mail.Read.All
✓ Mail.Send
✓ Files.Read.All
✓ Files.ReadWrite.All
✓ Sites.Read.All
✓ Sites.ReadWrite.All
```

**Admin Consent:** Requerido

### Variables de Entorno del Sistema

```powershell
# CRÍTICAS - Configura PRIMERO
[System.Environment]::SetEnvironmentVariable("M365_TENANT_ID", "tu-tenant-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_ID", "app-client-id", "Machine")
[System.Environment]::SetEnvironmentVariable("M365_CLIENT_SECRET", "app-secret", "Machine")

# CONFIGURACIÓN DE SHAREPOINT
[System.Environment]::SetEnvironmentVariable("SHAREPOINT_SITE_ID", "site-id", "Machine")
[System.Environment]::SetEnvironmentVariable("SHAREPOINT_LIST_ID", "list-id", "Machine")

# NOTIFICACIONES
[System.Environment]::SetEnvironmentVariable("IT_TEAM_EMAIL", "it-team@org.com", "Machine")
[System.Environment]::SetEnvironmentVariable("HR_TEAM_EMAIL", "hr-team@org.com", "Machine")

# POLÍTICAS DE REINTENTO (Opcional - valores por defecto)
[System.Environment]::SetEnvironmentVariable("MAX_RETRIES_REVOKE_SESSIONS", "3", "Machine")
[System.Environment]::SetEnvironmentVariable("MAX_RETRIES_CONVERT_MAILBOX", "5", "Machine")
[System.Environment]::SetEnvironmentVariable("MAX_RETRIES_REMOVE_LICENSES", "5", "Machine")
```

⚠️ **Requiere reinicio de PowerShell después de configurar**

---

## Instalación

### 1. Clonar/Descargar Archivos
```bash
git clone https://github.com/tu-usuario/m365-offboarding-process.git
cd m365-offboarding-process
```

### 2. Copiar Agentes a Directorio Local
```powershell
Copy-Item m365-offboarding-*.md $env:USERPROFILE\.claude\agents\offboarding-process\
```

### 3. Instalar Módulos Requeridos
```powershell
Install-Module Microsoft.Graph -Scope CurrentUser -Force
Install-Module ExchangeOnlineManagement -Scope CurrentUser -Force  # Para conversión de buzón
```

### 4. Configurar Variables de Entorno
(Ver sección anterior)

### 5. Probar Conectividad
```powershell
# Debería conectar exitosamente
$tenantId = [System.Environment]::GetEnvironmentVariable("M365_TENANT_ID", "Machine")
$clientId = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_ID", "Machine")
$clientSecret = [System.Environment]::GetEnvironmentVariable("M365_CLIENT_SECRET", "Machine")

$SecureSecret = ConvertTo-SecureString $clientSecret -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential($clientId, $SecureSecret)
Connect-MgGraph -TenantId $tenantId -ClientSecretCredential $Credential -NoWelcome

Get-MgUser -Top 1  # Debe retornar al menos 1 usuario
```

---

## Uso

### Opción A: Ejecución Manual (Testing)

```powershell
# 1. Ejecutar Orchestrator (NODE_1)
# Consulta: "Dame un script para leer la fila 12345 de la lista de offboarding de SPO y validar el usuario en Azure AD"
# El agente m365-offboarding-orchestrator genera el script automáticamente

# 2. Ejecutar Security (NODE_2-3)
# Consulta: "Revoca todas las sesiones y bloquea sign-in del usuario john.doe@org.com"
# El agente m365-offboarding-security-executor genera el script

# 3. Ejecutar Data Preservation (NODE_4-5)
# Consulta: "Convierte el buzón a compartido y delega a jane.smith@org.com"
# El agente m365-offboarding-data-preservation genera el script (⚠️ Crítico)

# 4. Ejecutar Cleanup (NODE_6-7)
# Consulta: "Remueve de todos los grupos y retira licencias"
# El agente m365-offboarding-cleanup-executor genera el script

# 5. Finalize
# Actualizar SharePoint + enviar notificación
```

### Opción B: Ejecución Automática (Producción)

Usa **Power Automate** o **Azure Automation Runbook**:

1. Trigger: Nueva fila creada en lista de SPO
2. Ejecutar script de NODE_1 (leer datos)
3. Ejecutar scripts de NODE_2-9 secuencialmente
4. Actualizar SharePoint y notificar

---

## Estructura de Carpetas

```
offboarding-process/
├── README.md                                      (Este archivo)
├── m365-offboarding-orchestrator.md              (NODE_1)
├── m365-offboarding-security-executor.md         (NODE_2-3)
├── m365-offboarding-data-preservation.md         (NODE_4-5)
├── m365-offboarding-cleanup-executor.md          (NODE_6-7)
├── offboarding-config.json                       (Configuración ejemplo)
├── logs/
│   ├── orchestrator-2025-01-15-143000.json      (Logs de ejecución)
│   ├── security-2025-01-15-143005.json
│   ├── data-2025-01-15-143015.json
│   └── cleanup-2025-01-15-143030.json
└── scripts/                                       (Scripts generados)
    ├── node-1-fetch-user.ps1
    ├── node-2-revoke-sessions.ps1
    ├── node-3-block-signin.ps1
    ├── node-4-convert-mailbox.ps1
    ├── node-5-delegate-onedrive.ps1
    ├── node-6-remove-groups.ps1
    ├── node-7-remove-licenses.ps1
    ├── node-8-update-sharepoint.ps1
    └── node-9-notify-completion.ps1
```

---

## Archivos de Configuración

### offboarding-config.json (Ejemplo)

```json
{
  "tenant": {
    "id": "00000000-0000-0000-0000-000000000000",
    "name": "organization.com"
  },
  "appRegistration": {
    "clientId": "app-client-id",
    "scopes": [
      "User.ReadWrite.All",
      "Group.ReadWrite.All",
      "Mail.Send"
    ]
  },
  "sharepoint": {
    "siteId": "site-id",
    "listId": "list-id",
    "columns": {
      "userPrincipalName": "UserPrincipalName",
      "managerUpn": "ManagerUPN",
      "department": "Department",
      "status": "Status"
    }
  },
  "notifications": {
    "itTeamEmail": "it-team@organization.com",
    "hrTeamEmail": "hr-team@organization.com",
    "senderUserId": "service-account-id"
  },
  "retryPolicy": {
    "revokeSessions": { "maxAttempts": 3, "backoffSeconds": "exponential" },
    "blockSignIn": { "maxAttempts": 3, "backoffSeconds": "exponential" },
    "convertMailbox": { "maxAttempts": 5, "backoffSeconds": "exponential" },
    "delegateOneDrive": { "maxAttempts": 3, "backoffSeconds": "linear" },
    "removeGroups": { "maxAttempts": 2, "backoffSeconds": "linear" },
    "removeLicenses": { "maxAttempts": 5, "backoffSeconds": "exponential" }
  },
  "criticalNodes": {
    "NODE_4": { "isCritical": true, "action": "PAUSE_IF_FAILED", "notifyIT": true },
    "NODE_2": { "isCritical": false, "action": "LOG_AND_CONTINUE" },
    "NODE_3": { "isCritical": false, "action": "LOG_AND_CONTINUE" },
    "NODE_5": { "isCritical": false, "action": "LOG_WARNING" },
    "NODE_6": { "isCritical": false, "action": "LOG_AND_CONTINUE" },
    "NODE_7": { "isCritical": false, "action": "LOG_AND_CONTINUE" }
  }
}
```

---

## Flujo de Ejecución Completo

```
┌─ INICIO ─────────────────────────────────────────────────────────┐
│                                                                   │
│ 1. SharePoint Row creada manualmente o vía integraciones         │
│    (RRHH, SuccessFactors, Workday, etc.)                         │
│                                                                   │
│ 2. Power Automate / Webhook dispara el flujo                     │
│                                                                   │
│ 3. NODE_1 (Orchestrator)                                         │
│    └─ Lee row de SPO                                             │
│    └─ Valida usuario en Azure AD                                 │
│    └─ Si error → STOP, notifica RRHH                            │
│                                                                   │
│ 4. NODE_2 (Security) - Revoke Sessions (3x reintentos)          │
│    └─ Si error → LOG WARNING, CONTINUA                          │
│                                                                   │
│ 5. NODE_3 (Security) - Block SignIn (3x reintentos)             │
│    └─ Si error → LOG WARNING, CONTINUA                          │
│                                                                   │
│ 6. NODE_4 (Data) - Convert Mailbox (5x reintentos) ⚠️ CRÍTICO   │
│    └─ Si error → PAUSA flujo, notifica IT INMEDIATAMENTE        │
│    └─ Si éxito → CONTINUA                                        │
│                                                                   │
│ 7. NODE_5 (Data) - Delegate OneDrive (3x reintentos)            │
│    └─ Si error → LOG WARNING, CONTINUA                          │
│                                                                   │
│ 8. NODE_6 (Cleanup) - Remove Groups (2x reintentos por grupo)   │
│    └─ Si error → LOG WARNING, CONTINUA                          │
│                                                                   │
│ 9. NODE_7 (Cleanup) - Remove Licenses (5x reintentos)           │
│    └─ Si error → LOG WARNING, CONTINUA                          │
│                                                                   │
│ 10. NODE_8 - Update SharePoint Status = "Completado"            │
│     └─ Si error → LOG WARNING, CONTINUA                         │
│                                                                   │
│ 11. NODE_9 - Notify IT & HR                                     │
│     └─ Si error → LOG WARNING, igualmente MARCA COMO COMPLETADO │
│                                                                   │
│ 12. FIN - Offboarding completado o pausado                      │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Auditoría y Logging

Todos los pasos registran en archivos JSON con estructura:

```json
{
  "ExecutionId": "uuid-unique-per-run",
  "User": "john.doe@organization.com",
  "Manager": "jane.smith@organization.com",
  "StartTime": "2025-01-15T14:30:00Z",
  "EndTime": "2025-01-15T14:45:00Z",
  "DurationMinutes": 15,
  "Status": "completed|failed|paused",
  "Nodes": [
    {
      "Name": "NODE_1",
      "Status": "success",
      "Duration": "2s",
      "Details": {}
    },
    ...
  ],
  "Errors": [
    {
      "Node": "NODE_5",
      "Severity": "warning",
      "Message": "OneDrive not found",
      "Timestamp": "2025-01-15T14:32:00Z"
    }
  ],
  "Summary": {
    "StepsCompleted": 9,
    "StepsFailed": 0,
    "WarningsLogged": 1,
    "GroupsRemoved": 12,
    "LicensesRemoved": 3
  }
}
```

---

## Casos de Error Crítico

| Escenario | Acción |
|-----------|--------|
| NODE_1: Usuario no existe | ❌ DETENER - Notificar RRHH |
| NODE_4: Mailbox convert falla (5x) | ❌ PAUSA TODO - Notificar IT inmediatamente |
| NODE_4: Todos demás nodos OK | ✓ COMPLETADO (se notifica que NODE_4 requiere atención manual) |

---

## Troubleshooting

### Error: "Connect-MgGraph: Insufficient privileges"
**Solución:** 
- Verificar que App Registration tenga todos los permisos requeridos
- Dar "Admin consent" en Azure AD
- Esperar 5 minutos a que se propague

### Error: "Mailbox conversion failed"
**Solución:**
- Verificar que EXO PowerShell esté instalado: `Install-Module ExchangeOnlineManagement`
- Conectar a EXO: `Connect-ExchangeOnline -Credential $credential`
- Verificar que usuario no esté en estado especial (delegado, recurso, etc.)

### Error: "OneDrive not found"
**Solución:**
- Esto es normal si el usuario nunca accedió a OneDrive
- El flujo continúa de todas formas (no-crítico)
- Datos están en buzón compartido

### Usuario aún tiene acceso después de offboarding
**Causa:** Tokens en caché  
**Solución:** Esperar 15-30 minutos para que se propague globalmente

---

## Mejores Prácticas

✅ **SIEMPRE:**
- Probar con usuario piloto primero
- Verificar log de ejecución tras cada offboarding
- Alertar a manager antes de convertir buzón
- Mantener copia de seguridad de OneDrive
- Auditar cambios en Azure AD

❌ **NUNCA:**
- Ejecutar en paralelo del mismo usuario
- Omitir NODE_4 (conversión de mailbox)
- Cambiar APP secret sin actualizar variables de entorno
- Ejecutar durante mantenimiento de M365

---

## Soporte y Contacto

Para preguntas o problemas:
1. Revisar logs en carpeta `logs/`
2. Consultar sección de Troubleshooting
3. Verificar permisos de App Registration
4. Abrir Issue en GitHub

---

**Versión:** 1.0 (PoC)  
**Última actualización:** Enero 2025  
**Estado:** Production Ready
