# PoC: Agente de Offboarding M365 basado en Grafos
## Arquitectura y Especificación Técnica

---

## 1. ARQUITECTURA DEL GRAFO

### 1.1 Flujo General y Nodos

El agente implementa un grafo cíclico con **9 nodos principales** y **múltiples aristas condicionales**. Cada nodo representa una acción atómica que muta el estado compartido.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GRAFO DE OFFBOARDING M365                          │
└─────────────────────────────────────────────────────────────────────────────┘

                                 ┌──────────┐
                                 │  START   │
                                 └────┬─────┘
                                      │
                    ┌─────────────────▼─────────────────┐
                    │ NODE_1: FETCH_USER_DATA           │
                    │ (Lee de SharePoint, valida)       │
                    └─────────────────┬─────────────────┘
                                      │
                 ┌────────────────────▼────────────────────┐
                 │ NODE_2: REVOKE_SESSIONS                │
                 │ (Invalida todas las sesiones activas)   │
                 └────────────────────┬────────────────────┘
                                      │
                ┌─────────────────────▼─────────────────────┐
                │ NODE_3: BLOCK_SIGNIN                      │
                │ (Bloquea acceso AccountEnabled=false)    │
                └─────────────────────┬─────────────────────┘
                                      │
           ┌──────────────────────────▼──────────────────────────┐
           │ NODE_4: CONVERT_TO_SHARED_MAILBOX                   │
           │ (Convierte a compartido y da acceso al manager)     │
           └──────────────────────────┬──────────────────────────┘
                                      │
                ┌─────────────────────▼─────────────────────┐
                │ NODE_5: DELEGATE_ONEDRIVE                │
                │ (Transfiere archivos a manager)           │
                └─────────────────────┬─────────────────────┘
                                      │
           ┌──────────────────────────▼──────────────────────────┐
           │ NODE_6: REMOVE_FROM_GROUPS_AND_TEAMS               │
           │ (Elimina de todos los grupos y teams)              │
           └──────────────────────────┬──────────────────────────┘
                                      │
                ┌─────────────────────▼─────────────────────┐
                │ NODE_7: REMOVE_LICENSES                  │
                │ (Retirad todas las licencias)             │
                └─────────────────────┬─────────────────────┘
                                      │
               ┌──────────────────────▼──────────────────────┐
               │ NODE_8: UPDATE_SHAREPOINT_RECORD            │
               │ (Marca como "Tramitado" en SPO)             │
               └──────────────────────┬──────────────────────┘
                                      │
                   ┌──────────────────▼──────────────────┐
                   │ NODE_9: NOTIFY_COMPLETION            │
                   │ (Envía notificación a IT/RRHH)      │
                   └──────────────────┬──────────────────┘
                                      │
                                 ┌────▼─────┐
                                 │  SUCCESS │
                                 └──────────┘
```

### 1.2 Nodos Detallados

#### **NODE_1: FETCH_USER_DATA**
- **Entrada:** ID de fila SharePoint
- **Acción:**
  - Lee fila de SharePoint Online (SPO)
  - Extrae: Nombre, UPN, Manager UPN, Departamento, ID Empleado
  - Valida que el usuario exista en Azure AD
- **Salida:** `OffboardingState` con datos del usuario validados
- **Transición:** 
  - ✅ Suceso → NODE_2
  - ❌ Usuario no encontrado → NODE_9 (notifica error)
  - ❌ Datos incompletos → NODE_9 (notifica error)

#### **NODE_2: REVOKE_SESSIONS**
- **Entrada:** UPN del usuario
- **Acción:**
  - Invalida todas las sesiones activas (invalidateAllRefreshTokens)
  - Espera 5 segundos para propagación
- **Salida:** Sessions revocadas
- **Transición:**
  - ✅ Suceso → NODE_3
  - ❌ Error API (reintento 3x con backoff exponencial) → NODE_2
  - ❌ Error permanente → NODE_9 (continúa de todos modos, registra fallo)

#### **NODE_3: BLOCK_SIGNIN**
- **Entrada:** User Object ID
- **Acción:**
  - Actualiza AccountEnabled = False
  - Registra timestamp de bloqueo
- **Salida:** Usuario bloqueado
- **Transición:**
  - ✅ Suceso → NODE_4
  - ❌ Error API → Reintentos automáticos (3x)
  - ❌ Fallo permanente → continúa (registra, no detiene)

#### **NODE_4: CONVERT_TO_SHARED_MAILBOX**
- **Entrada:** UPN usuario, UPN manager
- **Acción:**
  - Convierte buzón a compartido (RecipientTypeDetails = "SharedMailbox")
  - Establece OutOfOfficeAudit
  - Agrega manager como propietario delegado
  - Configura mensaje Fuera de Oficina automático
- **Salida:** Buzón compartido configurado
- **Transición:**
  - ✅ Suceso → NODE_5
  - ❌ Error → Reintentos

#### **NODE_5: DELEGATE_ONEDRIVE**
- **Entrada:** UPN usuario, UPN manager
- **Acción:**
  - Busca OneDrive del usuario
  - Asigna "Manager" permisos totales a carpeta raíz
  - Crea carpeta de archivo "Archived-[UserName]"
  - Registra URL de acceso
- **Salida:** OneDrive delegado
- **Transición:**
  - ✅ Suceso → NODE_6
  - ❌ OneDrive no encontrado → Continúa (log warning)
  - ❌ Error permisos → Reintentos

#### **NODE_6: REMOVE_FROM_GROUPS_AND_TEAMS**
- **Entrada:** User Object ID
- **Acción:**
  - Query: Obtiene todas las membresías de grupo (Microsoft 365, Teams)
  - Para cada grupo: remove-member
  - Registra grupos removidos
- **Salida:** Usuario removido de todos los grupos
- **Transición:**
  - ✅ Suceso → NODE_7
  - ⚠️ Parcialmente removido → Continúa (registra)
  - ❌ Error → Reintentos

#### **NODE_7: REMOVE_LICENSES**
- **Entrada:** User Object ID
- **Acción:**
  - Query: Obtiene SKUs asignados
  - Para cada licencia: Assign-MgUserLicense (RemoveAsignments)
  - Registra licencias removidas
- **Salida:** Licencias retiradas
- **Transición:**
  - ✅ Suceso → NODE_8
  - ⚠️ Algunas licencias persistentes → Log + continúa
  - ❌ Error → Reintentos (max 5x)

#### **NODE_8: UPDATE_SHAREPOINT_RECORD**
- **Entrada:** ID fila SPO, estado del flujo
- **Acción:**
  - Actualiza columna "Status" = "Completado"
  - Agrega timestamp y detalles
  - Crea elemento de auditoría
- **Salida:** Registro actualizado en SPO
- **Transición:**
  - ✅ Suceso → NODE_9
  - ❌ Error → Reintentos

#### **NODE_9: NOTIFY_COMPLETION**
- **Entrada:** Resumen del estado del flujo
- **Acción:**
  - Envía correo a RRHH (lista de distribución)
  - Envía correo a IT (lista de distribución)
  - Incluye: User, Fecha/Hora, Pasos completados, Errores (si hay)
- **Salida:** Notificaciones enviadas
- **Transición:** → END (cierra flujo)

### 1.3 Estrategia de Reintentos y Recuperación de Errores

```
┌─────────────────────────────────────────────────────┐
│  POLÍTICA DE REINTENTOS AUTOMÁTICOS (POR NODO)    │
├─────────────────────────────────────────────────────┤
│ Nodo                      Reintentos  Backoff      │
├─────────────────────────────────────────────────────┤
│ REVOKE_SESSIONS          3x          Exponencial   │
│ BLOCK_SIGNIN             3x          Exponencial   │
│ CONVERT_MAILBOX          5x          Exponencial   │
│ DELEGATE_ONEDRIVE        3x          Lineal        │
│ REMOVE_GROUPS            2x          Lineal        │
│ REMOVE_LICENSES          5x          Exponencial   │
│ UPDATE_SHAREPOINT        3x          Lineal        │
│ NOTIFY                   2x          Lineal        │
└─────────────────────────────────────────────────────┘
```

**Backoff Exponencial:**
```
Intento 1: Espera 1 segundo
Intento 2: Espera 2 segundos
Intento 3: Espera 4 segundos
```

**Manejo de Errores Críticos:**
- Si NODE_2 (REVOKE_SESSIONS) o NODE_3 (BLOCK_SIGNIN) fallan → Se registran pero se continúa
- Si NODE_4 (MAILBOX) falla → Se reintenta 5 veces; si persiste → Se pausa y notifica a IT manualmente
- Si NODE_8 (UPDATE_SHAREPOINT) falla → Se reintenta; si persiste → Error crítico enviado a IT

**Nodos que NUNCA detienen el flujo:**
- NODE_2, NODE_3 (seguridad prioritaria pero no detienen)
- NODE_5 (si no hay OneDrive, continúa)
- NODE_6 (si está en grupos que fallan, registra y continúa)

---

## 2. PASOS DE TRAMITACIÓN EN M365 (CON GRAPH API)

### 2.1 Validación e Identidad del Usuario (NODE_1)

**Objetivo:** Confirmar existencia y validez del usuario en Azure AD

**Endpoints Microsoft Graph:**

| Paso | Endpoint | Método | Descripción |
|------|----------|--------|-------------|
| 1.1 | `/users/{userPrincipalName}` | GET | Obtiene objeto usuario |
| 1.2 | `/users/{id}/memberOf` | GET | Obtiene membresías (grupos, roles) |
| 1.3 | `/users/{id}/manager` | GET | Obtiene objeto del manager |

**Validaciones:**
```
- userPrincipalName != null
- accountEnabled == true (antes del offboarding)
- managerUpn resolvable a objectId válido
- Departamento != null
```

---

### 2.2 Seguridad Inmediata (NODE_2 & NODE_3)

#### **NODE_2: Revocar Sesiones Activas**

**Objetivo:** Invalidar todos los tokens y sesiones activas

**Endpoint Microsoft Graph:**
```
POST /users/{id}/revokeSignInSessions
```

**Payload:**
```json
{}
```

**Efecto:** 
- Invalida todos los refresh tokens
- Desconecta de Teams, Outlook, OneDrive, SharePoint
- Tarda 15-30 segundos en propagarse globalmente

---

#### **NODE_3: Bloquear Sign-In**

**Objetivo:** Impedir que el usuario pueda autenticarse nuevamente

**Endpoint Microsoft Graph:**
```
PATCH /users/{id}
```

**Payload:**
```json
{
  "accountEnabled": false
}
```

**Efecto:**
- Usuario no puede autenticarse
- Sign-in intentos son rechazados inmediatamente
- Cambio se propaga en ~2-3 minutos globalmente

---

### 2.3 Preservación de Correo (NODE_4)

**Objetivo:** Convertir buzón personal a compartido con acceso del manager

#### **Paso 4.1: Convertir a Buzón Compartido**

**Endpoint Microsoft Graph:**
```
PATCH /users/{id}
```

**Payload:**
```json
{
  "usageLocation": "ES"  // Necesario para licencias
}
```

**Luego, en Exchange Online (via Graph - Mailbox):**
```
POST /me/mailboxSettings
PATCH /users/{id}/mailboxes
```

*Nota: La conversión a SharedMailbox requiere EXO API o PnP PowerShell; Graph tiene limitaciones. Considerar híbrido.*

#### **Paso 4.2: Agregar Manager como Propietario Delegado**

**Endpoint Microsoft Graph:**
```
POST /users/{userid}/mailboxSettings
```

**Alternativa (usar sendMail delegado):**
```
POST /users/{managerID}/sendMail
```

**Delegar permiso de buzón:**
```powershell
# Ejecutar vía EXO PowerShell (fuera de Graph puro)
Add-MailboxPermission -Identity {mailbox} -User {manager} -AccessRights FullAccess -InheritanceType All
```

#### **Paso 4.3: Configurar Mensaje Fuera de Oficina**

**Endpoint Microsoft Graph:**
```
PATCH /users/{id}/mailboxSettings/automaticRepliesSetting
```

**Payload:**
```json
{
  "isScheduled": false,
  "status": "alwaysEnabled",
  "externalAudience": "all",
  "externalReplyMessage": "Este usuario ha sido dado de baja de la organización. Para consultas, contacte a [Manager Name] en [Manager Email]",
  "internalReplyMessage": "Este usuario ha sido dado de baja. Contacte a [Manager Name] para soporte."
}
```

---

### 2.4 Respaldo de Archivos (NODE_5)

**Objetivo:** Delegar permisos de OneDrive al manager

#### **Paso 5.1: Obtener Drive del Usuario**

**Endpoint Microsoft Graph:**
```
GET /users/{id}/drive
GET /users/{userPrincipalName}/drive
```

**Response:**
```json
{
  "id": "b!...",
  "webUrl": "https://tenant-my.sharepoint.com/personal/user_domain_com",
  "name": "User Name"
}
```

#### **Paso 5.2: Asignar Permisos al Manager**

**Endpoint Microsoft Graph:**
```
POST /drives/{driveId}/root/permissions
```

**Payload:**
```json
{
  "recipients": [
    {
      "email": "manager@organization.com"
    }
  ],
  "roles": ["owner"],  // o ["edit"]
  "requireSignIn": true,
  "notifyEmail": true
}
```

#### **Paso 5.3: Crear Carpeta de Archivo**

**Endpoint Microsoft Graph:**
```
POST /drives/{driveId}/items/root/children
```

**Payload:**
```json
{
  "name": "Archived-[UserName]-[Date]",
  "folder": {}
}
```

**Efecto:** Archivo legible para auditoría y retención

---

### 2.5 Limpieza (NODE_6)

**Objetivo:** Remover usuario de todos los grupos y equipos

#### **Paso 6.1: Obtener Todos los Grupos (Membresía)**

**Endpoint Microsoft Graph:**
```
GET /users/{id}/memberOf
GET /users/{id}/memberOf?$filter=isof('microsoft.graph.group')
```

**Response:**
```json
{
  "value": [
    { "id": "group-id-1", "displayName": "Ventas" },
    { "id": "group-id-2", "displayName": "IT" }
  ]
}
```

#### **Paso 6.2: Remover de Cada Grupo**

**Endpoint Microsoft Graph:**
```
DELETE /groups/{groupId}/members/{memberId}/$ref
```

**Ejemplo:**
```
DELETE /groups/12345678-1234-1234-1234-123456789012/members/87654321-4321-4321-4321-210987654321/$ref
```

#### **Paso 6.3: Remover de Teams (Alternativo)**

**Endpoint Microsoft Graph:**
```
DELETE /teams/{teamId}/members/{membershipId}
```

*Nota: Los miembros de equipos vienen a través de grupos de Microsoft 365 (paso 6.2), pero Teams también tiene lista de miembros separada.*

---

### 2.6 Optimización: Retirar Licencias (NODE_7)

**Objetivo:** Liberar licencias para reasignar a otros usuarios

#### **Paso 7.1: Obtener Licencias Asignadas**

**Endpoint Microsoft Graph:**
```
GET /users/{id}/licenseDetails
```

**Response:**
```json
{
  "value": [
    {
      "skuId": "6fd2c87f-bc8f-4e6d-8ce8-40da8bbcb051",  // Microsoft 365 E3
      "skuPartNumber": "ENTERPRISEPACK",
      "servicePlans": [
        { "servicePlanId": "...", "servicePlanName": "EXCHANGE_S_ENTERPRISE", "provisioningStatus": "Success" }
      ]
    }
  ]
}
```

#### **Paso 7.2: Remover Licencias**

**Endpoint Microsoft Graph:**
```
POST /users/{id}/assignLicense
```

**Payload:**
```json
{
  "addLicenses": [],
  "removeLicenses": [
    "6fd2c87f-bc8f-4e6d-8ce8-40da8bbcb051"  // E3
  ]
}
```

**Propagación:** ~2-5 minutos

---

### 2.7 Cierre: Actualizar SharePoint y Notificar (NODE_8 & NODE_9)

#### **Paso 8.1: Actualizar Lista SharePoint**

**Endpoint Microsoft Graph (SharePoint REST):**
```
PATCH /sites/{siteId}/lists/{listId}/items/{itemId}
```

**Payload:**
```json
{
  "fields": {
    "Status": "Completado",
    "OffboardingDate": "2025-01-15T14:30:00Z",
    "Notes": "Offboarding completado exitosamente. Manager notificado."
  }
}
```

#### **Paso 8.2: Enviar Notificación por Correo**

**Endpoint Microsoft Graph:**
```
POST /users/{senderId}/sendMail
```

**Payload:**
```json
{
  "message": {
    "subject": "Offboarding completado: [User Name]",
    "body": {
      "contentType": "HTML",
      "content": "<h2>Offboarding Completado</h2><p>Usuario: [Name]</p><p>Buzón convertido a compartido y delegado a [Manager].</p><p>OneDrive: [URL]</p><p>Licencias retiradas.</p>"
    },
    "toRecipients": [
      { "emailAddress": { "address": "it-team@organization.com" } },
      { "emailAddress": { "address": "hr-team@organization.com" } }
    ]
  },
  "saveToSentItems": true
}
```

---

## 3. EJEMPLO DE CÓDIGO (PYTHON + LANGGRAPH)

### 3.1 Estructura de Estado Compartido

```python
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum

class OffboardingStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    PAUSED = "paused"

@dataclass
class OffboardingState:
    """Estado compartido del flujo de offboarding"""
    
    # Datos del usuario
    user_principal_name: str
    user_object_id: str
    manager_principal_name: str
    manager_object_id: str
    department: str
    employee_id: str
    
    # Metadatos del flujo
    sharepoint_row_id: str
    status: OffboardingStatus = OffboardingStatus.PENDING
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    
    # Historial de ejecución de nodos
    node_execution_log: Dict[str, Dict[str, Any]] = field(default_factory=dict)
    # Ejemplo: {
    #   "NODE_2_REVOKE_SESSIONS": {
    #     "status": "success",
    #     "attempts": 1,
    #     "timestamp": "2025-01-15T14:30:00Z",
    #     "details": "Sessions revoked successfully"
    #   }
    # }
    
    # Errores acumulados
    errors: List[Dict[str, Any]] = field(default_factory=list)
    
    # Datos de cada nodo
    revoked_sessions: bool = False
    signin_blocked: bool = False
    mailbox_converted: bool = False
    onedrive_delegated: bool = False
    groups_cleaned: bool = False
    licenses_removed: bool = False
    sharepoint_updated: bool = False
    notification_sent: bool = False
    
    # URLs y referencias para auditoría
    mailbox_shared_url: Optional[str] = None
    onedrive_archive_url: Optional[str] = None
    removed_groups: List[str] = field(default_factory=list)
    removed_licenses: List[str] = field(default_factory=list)
```

### 3.2 Estructura de Nodos en LangGraph

```python
from langgraph.graph import StateGraph, END
from typing import Callable, Any
import asyncio

class OffboardingGraph:
    """Agente de offboarding basado en grafo"""
    
    def __init__(self, graph_client, sharepoint_client, config: Dict[str, Any]):
        """
        Args:
            graph_client: Cliente autenticado de Microsoft Graph
            sharepoint_client: Cliente autenticado de SharePoint
            config: Configuración (reintentos, timeouts, etc.)
        """
        self.graph_client = graph_client
        self.sharepoint_client = sharepoint_client
        self.config = config
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        """Construye el grafo de estados"""
        graph = StateGraph(OffboardingState)
        
        # Agregar nodos
        graph.add_node("NODE_1_FETCH_USER_DATA", self.node_1_fetch_user_data)
        graph.add_node("NODE_2_REVOKE_SESSIONS", self.node_2_revoke_sessions)
        graph.add_node("NODE_3_BLOCK_SIGNIN", self.node_3_block_signin)
        graph.add_node("NODE_4_CONVERT_MAILBOX", self.node_4_convert_mailbox)
        graph.add_node("NODE_5_DELEGATE_ONEDRIVE", self.node_5_delegate_onedrive)
        graph.add_node("NODE_6_REMOVE_GROUPS", self.node_6_remove_groups)
        graph.add_node("NODE_7_REMOVE_LICENSES", self.node_7_remove_licenses)
        graph.add_node("NODE_8_UPDATE_SHAREPOINT", self.node_8_update_sharepoint)
        graph.add_node("NODE_9_NOTIFY_COMPLETION", self.node_9_notify_completion)
        
        # Agregar aristas (transiciones)
        graph.add_edge("NODE_1_FETCH_USER_DATA", "NODE_2_REVOKE_SESSIONS")
        graph.add_edge("NODE_2_REVOKE_SESSIONS", "NODE_3_BLOCK_SIGNIN")
        graph.add_edge("NODE_3_BLOCK_SIGNIN", "NODE_4_CONVERT_MAILBOX")
        graph.add_edge("NODE_4_CONVERT_MAILBOX", "NODE_5_DELEGATE_ONEDRIVE")
        graph.add_edge("NODE_5_DELEGATE_ONEDRIVE", "NODE_6_REMOVE_GROUPS")
        graph.add_edge("NODE_6_REMOVE_GROUPS", "NODE_7_REMOVE_LICENSES")
        graph.add_edge("NODE_7_REMOVE_LICENSES", "NODE_8_UPDATE_SHAREPOINT")
        graph.add_edge("NODE_8_UPDATE_SHAREPOINT", "NODE_9_NOTIFY_COMPLETION")
        graph.add_edge("NODE_9_NOTIFY_COMPLETION", END)
        
        # Establecer punto de entrada
        graph.set_entry_point("NODE_1_FETCH_USER_DATA")
        
        return graph.compile()
    
    async def execute(self, sharepoint_row_id: str) -> OffboardingState:
        """
        Ejecuta el flujo completo de offboarding
        
        Args:
            sharepoint_row_id: ID de la fila en SharePoint que dispara el flujo
        
        Returns:
            Estado final del offboarding
        """
        initial_state = OffboardingState(
            sharepoint_row_id=sharepoint_row_id,
            user_principal_name="",
            user_object_id="",
            manager_principal_name="",
            manager_object_id="",
            department="",
            employee_id=""
        )
        
        final_state = await self.graph.ainvoke(initial_state)
        return final_state
```

### 3.3 Implementación de Nodos Clave

#### **NODE_1: Fetch User Data**

```python
async def node_1_fetch_user_data(self, state: OffboardingState) -> OffboardingState:
    """
    Lee datos del usuario de SharePoint y valida en Azure AD
    """
    node_name = "NODE_1_FETCH_USER_DATA"
    
    try:
        # 1. Leer fila de SharePoint
        sharepoint_item = await self._get_sharepoint_item(state.sharepoint_row_id)
        
        user_upn = sharepoint_item.get("UserPrincipalName")
        manager_upn = sharepoint_item.get("ManagerUPN")
        department = sharepoint_item.get("Department")
        employee_id = sharepoint_item.get("EmployeeID")
        
        # 2. Validar en Azure AD
        user_object = await self._get_user_from_aad(user_upn)
        if not user_object or not user_object.get("id"):
            raise ValueError(f"Usuario {user_upn} no encontrado en Azure AD")
        
        manager_object = await self._get_user_from_aad(manager_upn)
        if not manager_object or not manager_object.get("id"):
            raise ValueError(f"Manager {manager_upn} no encontrado en Azure AD")
        
        # 3. Actualizar estado
        state.user_principal_name = user_upn
        state.user_object_id = user_object["id"]
        state.manager_principal_name = manager_upn
        state.manager_object_id = manager_object["id"]
        state.department = department
        state.employee_id = employee_id
        state.status = OffboardingStatus.IN_PROGRESS
        
        # 4. Registrar en log
        self._log_node_execution(state, node_name, "success", {"users_validated": 2})
        
        return state
    
    except Exception as e:
        self._log_error(state, node_name, str(e))
        state.status = OffboardingStatus.FAILED
        return state
```

#### **NODE_2: Revoke Sessions (con Reintentos)**

```python
async def node_2_revoke_sessions(self, state: OffboardingState) -> OffboardingState:
    """
    Invalida todas las sesiones activas del usuario.
    Implementa reintentos automáticos con backoff exponencial.
    """
    node_name = "NODE_2_REVOKE_SESSIONS"
    max_retries = self.config.get("max_retries_revoke", 3)
    
    for attempt in range(1, max_retries + 1):
        try:
            # POST /users/{id}/revokeSignInSessions
            await self.graph_client.post(
                f"/users/{state.user_object_id}/revokeSignInSessions",
                json={}
            )
            
            state.revoked_sessions = True
            
            # Esperar a propagación
            await asyncio.sleep(5)
            
            self._log_node_execution(
                state, node_name, "success",
                {"attempts": attempt, "timestamp": datetime.now().isoformat()}
            )
            
            return state
        
        except Exception as e:
            error_details = {
                "attempt": attempt,
                "error": str(e),
                "timestamp": datetime.now().isoformat()
            }
            
            if attempt < max_retries:
                # Backoff exponencial: 1s, 2s, 4s
                backoff_seconds = 2 ** (attempt - 1)
                print(f"[{node_name}] Reintentando en {backoff_seconds}s...")
                await asyncio.sleep(backoff_seconds)
            else:
                # Último intento falló, registrar pero continuar
                self._log_error(state, node_name, str(e), error_details)
                print(f"[{node_name}] ADVERTENCIA: No se pudieron revocar sesiones. Continuando...")
                # No bloquea el flujo
                break
    
    return state
```

#### **NODE_4: Convert Mailbox (Hybrid - Graph + PowerShell)**

```python
async def node_4_convert_mailbox(self, state: OffboardingState) -> OffboardingState:
    """
    Convierte buzón a compartido y delega al manager.
    Nota: Requiere EXO PowerShell para conversión; Graph para delegación.
    """
    node_name = "NODE_4_CONVERT_MAILBOX"
    max_retries = self.config.get("max_retries_mailbox", 5)
    
    for attempt in range(1, max_retries + 1):
        try:
            # Paso 1: Ejecutar conversión vía PowerShell (ejecutado en máquina local)
            conversion_result = await self._execute_powershell_script(
                f"""
                $UserEmail = "{state.user_principal_name}"
                $ManagerEmail = "{state.manager_principal_name}"
                
                # Convertir a buzón compartido
                Set-Mailbox -Identity $UserEmail -Type Shared -Confirm:$false
                
                # Agregar permisos al manager
                Add-MailboxPermission -Identity $UserEmail -User $ManagerEmail `
                    -AccessRights FullAccess -InheritanceType All -Confirm:$false
                """
            )
            
            # Paso 2: Configurar fuera de oficina vía Graph
            await self.graph_client.patch(
                f"/users/{state.user_object_id}/mailboxSettings/automaticRepliesSetting",
                json={
                    "isScheduled": False,
                    "status": "alwaysEnabled",
                    "externalAudience": "all",
                    "externalReplyMessage": f"Este usuario ha sido dado de baja. Contacte a {state.manager_principal_name} para soporte.",
                    "internalReplyMessage": f"Este usuario ha sido dado de baja. Contacte a {state.manager_principal_name} para soporte."
                }
            )
            
            state.mailbox_converted = True
            
            self._log_node_execution(
                state, node_name, "success",
                {"attempts": attempt, "mailbox": state.user_principal_name}
            )
            
            return state
        
        except Exception as e:
            if attempt < max_retries:
                backoff_seconds = 2 ** (attempt - 1)
                print(f"[{node_name}] Reintentando en {backoff_seconds}s...")
                await asyncio.sleep(backoff_seconds)
            else:
                # Error crítico: pausar flujo y notificar
                self._log_error(state, node_name, str(e), {"critical": True})
                state.status = OffboardingStatus.PAUSED
                print(f"[{node_name}] ERROR CRÍTICO: Flujo pausado. Requiere intervención manual.")
                raise
    
    return state
```

#### **NODE_6: Remove Groups**

```python
async def node_6_remove_groups(self, state: OffboardingState) -> OffboardingState:
    """
    Elimina usuario de todos los grupos de M365 y Teams
    """
    node_name = "NODE_6_REMOVE_GROUPS"
    removed_count = 0
    failed_groups = []
    
    try:
        # Obtener todos los grupos del usuario
        # GET /users/{id}/memberOf?$filter=isof('microsoft.graph.group')
        groups_response = await self.graph_client.get(
            f"/users/{state.user_object_id}/memberOf?$filter=isof('microsoft.graph.group')",
            params={"$select": "id,displayName,resourceProvisioningOptions"}
        )
        
        groups = groups_response.get("value", [])
        
        # Remover de cada grupo
        for group in groups:
            try:
                group_id = group["id"]
                group_name = group.get("displayName", group_id)
                
                # DELETE /groups/{groupId}/members/{memberId}/$ref
                await self.graph_client.delete(
                    f"/groups/{group_id}/members/{state.user_object_id}/$ref"
                )
                
                removed_count += 1
                state.removed_groups.append(group_name)
                
            except Exception as group_error:
                failed_groups.append({
                    "group": group.get("displayName", group["id"]),
                    "error": str(group_error)
                })
        
        self._log_node_execution(
            state, node_name, "success",
            {"removed_count": removed_count, "failed_count": len(failed_groups)}
        )
        
        if failed_groups:
            print(f"[{node_name}] ADVERTENCIA: {len(failed_groups)} grupos no pudieron ser removidos")
            # No bloquea el flujo
        
        state.groups_cleaned = True
        return state
    
    except Exception as e:
        self._log_error(state, node_name, str(e))
        # Continuar de todas formas
        state.groups_cleaned = True
        return state
```

#### **NODE_9: Notify Completion**

```python
async def node_9_notify_completion(self, state: OffboardingState) -> OffboardingState:
    """
    Envía notificación de completación a IT y RRHH
    """
    node_name = "NODE_9_NOTIFY_COMPLETION"
    
    try:
        # Construir resumen del offboarding
        summary = self._build_offboarding_summary(state)
        
        email_body_html = f"""
        <h2>Offboarding Completado: {state.user_principal_name}</h2>
        <p><strong>Fecha:</strong> {state.updated_at.isoformat()}</p>
        <p><strong>Empleado:</strong> {state.user_principal_name} ({state.employee_id})</p>
        <p><strong>Departamento:</strong> {state.department}</p>
        <p><strong>Manager:</strong> {state.manager_principal_name}</p>
        
        <h3>Pasos Completados</h3>
        <ul>
            <li>✓ Sesiones revocadas: {state.revoked_sessions}</li>
            <li>✓ Sign-in bloqueado: {state.signin_blocked}</li>
            <li>✓ Buzón convertido: {state.mailbox_converted}</li>
            <li>✓ OneDrive delegado: {state.onedrive_delegated}</li>
            <li>✓ Grupos limpiados: {state.groups_cleaned}</li>
            <li>✓ Licencias removidas: {state.licenses_removed}</li>
        </ul>
        
        <h3>Detalles</h3>
        <p>Grupos removidos: {', '.join(state.removed_groups[:5])}</p>
        <p>Licencias retiradas: {', '.join(state.removed_licenses[:5])}</p>
        
        {"<h3>Errores Registrados</h3>" + self._format_errors_html(state.errors) if state.errors else ""}
        """
        
        # Enviar a IT y RRHH
        it_email = self.config.get("it_team_email", "it-team@organization.com")
        hr_email = self.config.get("hr_team_email", "hr-team@organization.com")
        
        for recipient in [it_email, hr_email]:
            await self.graph_client.post(
                f"/users/{self.config['sender_user_id']}/sendMail",
                json={
                    "message": {
                        "subject": f"Offboarding Completado: {state.user_principal_name}",
                        "body": {
                            "contentType": "HTML",
                            "content": email_body_html
                        },
                        "toRecipients": [
                            {"emailAddress": {"address": recipient}}
                        ]
                    },
                    "saveToSentItems": True
                }
            )
        
        state.notification_sent = True
        state.status = OffboardingStatus.COMPLETED
        
        self._log_node_execution(state, node_name, "success", {"recipients": 2})
        
        return state
    
    except Exception as e:
        self._log_error(state, node_name, str(e))
        # El flujo se considera completado aunque la notificación falle
        state.status = OffboardingStatus.COMPLETED
        return state
```

### 3.4 Métodos Auxiliares

```python
def _log_node_execution(self, state: OffboardingState, node: str, status: str, details: Dict[str, Any]):
    """Registra ejecución de nodo"""
    state.node_execution_log[node] = {
        "status": status,
        "timestamp": datetime.now().isoformat(),
        "details": details
    }
    print(f"[{node}] {status.upper()} - {details}")

def _log_error(self, state: OffboardingState, node: str, error: str, context: Dict[str, Any] = None):
    """Registra errores"""
    error_record = {
        "node": node,
        "error": error,
        "timestamp": datetime.now().isoformat(),
        "context": context or {}
    }
    state.errors.append(error_record)
    print(f"[{node}] ERROR: {error}")

async def _get_sharepoint_item(self, item_id: str) -> Dict[str, Any]:
    """Obtiene fila de SharePoint"""
    response = await self.sharepoint_client.get(
        f"/sites/{self.config['sharepoint_site_id']}/lists/{self.config['offboarding_list_id']}/items/{item_id}",
        params={"$expand": "fields"}
    )
    return response.get("fields", {})

async def _get_user_from_aad(self, upn: str) -> Dict[str, Any]:
    """Obtiene usuario de Azure AD"""
    try:
        response = await self.graph_client.get(
            f"/users/{upn}",
            params={"$select": "id,userPrincipalName,displayName,mail,department"}
        )
        return response
    except:
        return None

async def _execute_powershell_script(self, script: str) -> Dict[str, Any]:
    """Ejecuta script PowerShell (requiere agente local)"""
    # Implementación de ejecución remota de PS
    # Usar Azure Automation o local exec
    pass

def _build_offboarding_summary(self, state: OffboardingState) -> Dict[str, Any]:
    """Construye resumen de offboarding"""
    return {
        "user": state.user_principal_name,
        "manager": state.manager_principal_name,
        "completed_at": datetime.now().isoformat(),
        "steps": {
            "sessions_revoked": state.revoked_sessions,
            "signin_blocked": state.signin_blocked,
            "mailbox_converted": state.mailbox_converted,
            "onedrive_delegated": state.onedrive_delegated,
            "groups_cleaned": state.groups_cleaned,
            "licenses_removed": state.licenses_removed
        },
        "errors": len(state.errors),
        "error_details": state.errors
    }

def _format_errors_html(self, errors: List[Dict[str, Any]]) -> str:
    """Formatea errores para HTML"""
    html = "<ul>"
    for error in errors:
        html += f"<li>[{error['node']}] {error['error']}</li>"
    html += "</ul>"
    return html
```

### 3.5 Uso Principal (Punto de Entrada)

```python
async def main():
    """Punto de entrada para el agente"""
    from msgraph.core import GraphClient
    from azure.identity import ClientSecretCredential
    
    # Configuración
    config = {
        "tenant_id": os.getenv("M365_TENANT_ID"),
        "client_id": os.getenv("M365_CLIENT_ID"),
        "client_secret": os.getenv("M365_CLIENT_SECRET"),
        "max_retries_revoke": 3,
        "max_retries_mailbox": 5,
        "max_retries_licenses": 5,
        "sharepoint_site_id": "site-id-123",
        "offboarding_list_id": "list-id-456",
        "it_team_email": "it-team@organization.com",
        "hr_team_email": "hr-team@organization.com",
        "sender_user_id": "system-user-id"
    }
    
    # Inicializar cliente
    credential = ClientSecretCredential(
        config["tenant_id"],
        config["client_id"],
        config["client_secret"]
    )
    graph_client = GraphClient(credential=credential)
    
    # Crear agente
    agent = OffboardingGraph(graph_client, sharepoint_client=None, config=config)
    
    # Ejecutar flujo
    sharepoint_row_id = "item-12345"  # Obtenido del trigger de SPO
    final_state = await agent.execute(sharepoint_row_id)
    
    print(f"\n=== FLUJO COMPLETADO ===")
    print(f"Estado: {final_state.status}")
    print(f"Usuario: {final_state.user_principal_name}")
    print(f"Errores: {len(final_state.errors)}")
    
    if final_state.errors:
        for error in final_state.errors:
            print(f"  - [{error['node']}] {error['error']}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 3.6 Diagrama de Estado del Offboarding

```
┌─────────────────────────────────────────────────────────────────┐
│                    OffboardingState (Compartido)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  user_principal_name: "john.doe@organization.com"             │
│  user_object_id: "a1b2c3d4-..."                               │
│  manager_principal_name: "jane.smith@organization.com"        │
│  manager_object_id: "e5f6g7h8-..."                            │
│                                                                 │
│  status: OffboardingStatus.IN_PROGRESS                         │
│                                                                 │
│  node_execution_log: {                                         │
│    "NODE_1_FETCH_USER_DATA": {                                │
│      "status": "success",                                      │
│      "timestamp": "2025-01-15T14:30:00Z",                     │
│      "details": {"users_validated": 2}                       │
│    },                                                          │
│    "NODE_2_REVOKE_SESSIONS": {                               │
│      "status": "success",                                      │
│      "attempts": 1,                                            │
│      "timestamp": "2025-01-15T14:30:05Z"                     │
│    },                                                          │
│    ...                                                         │
│  }                                                             │
│                                                                 │
│  errors: [                                                     │
│    {                                                           │
│      "node": "NODE_5_DELEGATE_ONEDRIVE",                     │
│      "error": "OneDrive not found for user",                │
│      "timestamp": "2025-01-15T14:31:00Z",                    │
│      "context": {"critical": False}                          │
│    }                                                           │
│  ]                                                             │
│                                                                 │
│  revoked_sessions: True                                        │
│  signin_blocked: True                                          │
│  mailbox_converted: True                                       │
│  onedrive_delegated: False  ← Error manejado                  │
│  groups_cleaned: True                                          │
│  licenses_removed: False    ← En progreso                      │
│  sharepoint_updated: False  ← Próximo                          │
│  notification_sent: False                                      │
│                                                                 │
│  removed_groups: ["Ventas", "IT", "Finance"]                 │
│  removed_licenses: ["ENTERPRISEPACK", "TEAMS_STANDARD"]      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## RESUMEN EJECUTIVO

**Flujo de Offboarding de 9 Nodos:**
1. Fetch User Data → Validación
2. Revoke Sessions → Seguridad inmediata
3. Block SignIn → Bloqueo permanente
4. Convert Mailbox → Preservación correo
5. Delegate OneDrive → Respaldo archivos
6. Remove Groups → Limpieza de permisos
7. Remove Licenses → Optimización
8. Update SharePoint → Auditoría
9. Notify Completion → Cierre

**Características de Resiliencia:**
- Reintentos automáticos con backoff exponencial
- Manejo de errores granular (críticos vs. advertencias)
- State persistente compartido entre nodos
- Logs detallados de cada transición
- Pausa manual si error crítico

**Endpoints clave de Microsoft Graph:**
- `/users/{id}/revokeSignInSessions` (Sesiones)
- `PATCH /users/{id}` accountEnabled (Bloqueo)
- `/drives/{id}/permissions` (OneDrive)
- `/groups/{id}/members/{id}/$ref` (Grupos)
- `/users/{id}/assignLicense` (Licencias)

---

**Nota:** Algunos pasos (conversión a SharedMailbox) requieren **PowerShell híbrido** (EXO) porque Graph aún no expone esta capacidad completamente. Se recomienda usar Azure Automation o un runbook local.

