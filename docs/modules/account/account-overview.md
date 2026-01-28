# 💳 ACCOUNT - Módulo de Cuentas

**Módulo ID**: `account`  
**Versión**: 1.0  
**Última actualización**: 2026-01-30  
**Propósito**: Permitir a los equipos operativos (customer service, back-office y administradores) consultar, validar y actualizar la información financiera y personal de cada cuenta de tarjeta de crédito, garantizando que los datos sensibles se muestren bajo control y que las transacciones de actualización sean atómicas.

---

## 📋 Descripción general

El módulo ACCOUNT expone dos pantallas principales (consulta y actualización) que se construyen sobre Material-UI, hooks compartidos y llamadas REST al backend. El flujo típico inicia con un agente del back-office ingresando un Account ID de 11 dígitos, validando la existencia del registro en la tabla `account` y mostrando información de cuenta, cliente y tarjetas. Si el rol permite edición, se activa el modo de actualización donde el usuario puede modificar límites, datos de contacto y métricas de riesgo; todas las modificaciones pasan por validaciones de negocio antes de enviarlas al backend.

## 🧑‍💼 Actores y viajes principales

- **Representante de Customer Service**: Busca cuentas por su ID, revisa saldos y tarjetas asociadas y responde consultas en < 500ms para cumplir acuerdos de servicio.
- **Administrador / Supervisor**: Activa el modo de edición, ajusta límites o datos personales, confirma los cambios en un diálogo y recibe retroalimentación si alguna validación falla.
- **Oficial de Cumplimiento**: Verifica que el SSN y el número de tarjeta siempre se presenten enmascarados por defecto.
- **Sistemas externos (e.g., scoring)**: Aunque actualmente no hay integración activa, el módulo prevé ganchos en el servicio de validación para extender las reglas.

## 🏗️ Arquitectura del módulo

### Componentes clave

1. **`AccountViewScreen.tsx`** (`app/components/account/AccountViewScreen.tsx`) – Página de consulta con el header transaccional (`SystemHeader`), formulario centrado, tarjetas de información y botones de `Show Sensitive Data` / `Show Test Accounts`. Usa `LoadingSpinner` para indicar búsquedas y valida localmente que el Account ID sea un número de 11 dígitos diferente de ceros.
2. **`AccountUpdateScreen.tsx`** (`app/components/account/AccountUpdateScreen.tsx`) – Formulario dividido en tarjetas (account info, financial info, customer info) con distintos `<TextField>`, `<Switch>` para habilitar edición y un `<Dialog>` de confirmación antes de hacer `PUT`. Escucha teclas F3/Escape para salir, F5 para guardar y F12 para resetear.
3. **`useAccountView.ts`** (`app/hooks/useAccountView.ts`) – Hook basado en `useMutation` que consume `apiClient.get('/account-view?accountId=...')`, maneja la inicialización (`/account-view/initialize`) y centraliza `loading`, `error`, `data` y `clearData`.
4. **`useAccountUpdate.ts`** (`app/hooks/useAccountUpdate.ts`) – Orquesta la búsqueda (`GET /accounts/{id}`) y la actualización (`PUT /accounts/{id}`), expone detección de cambios (`hasChanges`), `resetForm` y `updateLocalData` para que la UI muestre botones de guardado cuando hay diferencias.
5. **`apiClient` + `useMutation`** (`app/services/api.ts`, `app/hooks/useApi.ts`) – Añaden Authorization desde `localStorage`, gestionan timeouts/errores y normalizan respuestas reales vs. MSW.
6. **Handlers de MSW** (`app/mocks/accountHandlers.ts`, `app/mocks/accountUpdateHandlers.ts`) – Simulan los endpoints y validaciones antes descritas, acelerando pruebas locales sin backend.

### Flujo de estado

`AccountViewScreen` → `useAccountView` → `apiClient` → `GET /account-view?...`  
`AccountUpdateScreen` → `useAccountUpdate` → `apiClient` → `GET /accounts/{id}` / `PUT /accounts/{id}`  
`apiClient` agrega `Authorization` (dependencia con AUTH) y detecta si la respuesta es `ApiResponse`, `success/error` o datos raw del backend.

## 🔗 APIs documentadas

- **`GET /api/account-view?accountId={11 dígitos}`**  
  - Request: query string contiene `accountId` padded a 11 dígitos.  
  - Response: estructura `AccountViewResponse` (ver más abajo). Las propiedades críticas: `accountStatus` (Y/N), `creditLimit`, `currentBalance`, `groupId`, `customerSsn`, `cards`.  
  - Comportamiento: devuelve `errorMessage` si el account no se encuentra o el filtro es inválido; `infoMessage` para datos de solo lectura.
- **`GET /api/account-view/initialize`**  
  - Request: sin parámetros.  
  - Response: payload de `AccountViewResponse` con metadatos (`transactionId`, `programName`) y datos mock para mostrar un estado inicial.  
- **`GET /api/accounts/{accountId}`**  
  - Request: path param `accountId`.  
  - Response: `ApiResponse<AccountUpdateData>` con `success: true` y el DTO lleno con `activeStatus`, límites, datos de cliente, FICO, etc.  
- **`PUT /api/accounts/{accountId}`**  
  - Request: body `AccountUpdateData` (no wrapping `success`).  
  - Response: `ApiResponse<AccountUpdateData>` indicando `success` y `message`. En caso de validación se regresa `success: false` con `errors`.

## 📊 Modelos de datos

```typescript
interface AccountViewResponse {
  currentDate: string;
  currentTime: string;
  transactionId: string;
  programName: string;
  accountId?: number;
  accountStatus?: 'Y' | 'N';
  currentBalance?: number;
  creditLimit?: number;
  cashCreditLimit?: number;
  currentCycleCredit?: number;
  currentCycleDebit?: number;
  openDate?: string;
  expirationDate?: string;
  reissueDate?: string;
  groupId?: string;
  customerSsn?: string;
  ficoScore?: number;
  firstName?: string;
  lastName?: string;
  ssn?: string;
  cardNumber?: string;
  infoMessage?: string;
  errorMessage?: string;
  inputValid: boolean;
}
```

```typescript
interface AccountUpdateData {
  accountId: number;
  activeStatus: 'Y' | 'N';
  currentBalance: number;
  creditLimit: number;
  cashCreditLimit: number;
  currentCycleCredit: number;
  currentCycleDebit: number;
  groupId: string;
  customerId: number;
  firstName: string;
  middleName?: string;
  lastName: string;
  addressLine1: string;
  addressLine2?: string;
  stateCode: string;
  zipCode: string;
  countryCode: string;
  phoneNumber1: string;
  ssn: string;
  governmentIssuedId: string;
  dateOfBirth: string;
  eftAccountId: string;
  primaryCardIndicator: 'Y' | 'N';
  ficoScore: number;
}
```

## 📋 Reglas de negocio

1. **Account ID** debe ser un número de 11 dígitos, no puede ser `00000000000` y se valida en la UI antes de hacer llamadas al backend (`AccountViewScreen`, `AccountUpdateScreen`, `useAccountView` y `useAccountUpdate`).  
2. **Status** solo acepta `'Y'` o `'N'`; solo las cuentas activas (`'Y'`) pueden generar transacciones.  
3. **Credito disponible** se calcula implicitamente como `creditLimit - currentBalance` y se refleja en la tarjeta financiera con colores rojo/verde (`getBalanceColor`).  
4. **Datos sensibles** (SSN, número de tarjeta) se muestran enmascarados por defecto y solo se revelan al apretar `Show Sensitive Data`.  
5. **Actualizaciones son atómicas**: al hacer `PUT /accounts/{id}`, el servidor devuelve errores si `activeStatus` no es `Y/N`, `zipCode` no cumple regex `^\d{5}(-\d{4})?$`, FICO está fuera de 300-850 o si el límite de crédito es negativo.  
6. **Verificación de existencia** recorre `CardXrefRecord → Account → Customer` (emulado en `accountHandlers`) y responde con `Account not found in Cross ref file` si no se localiza el registro.  
7. **InputValid/AccountFilterValid** llegan desde el backend para activar alertas y chips en la UI; la página muestra mensajes `errorMessage` o `infoMessage` según el payload.

## 🎯 Historias de Usuario y complejidad

- **Simple (1-2 pts)**: “Como representante de servicio, quiero buscar una cuenta por su ID para ver el saldo y el grupo, con los datos enmascarados por defecto.” Criterios: la búsqueda solo acepta 11 dígitos, muestra chips `Active/Inactive`, y respeta F3/Escape.  
- **Medio (3-5 pts)**: “Como administrador, quiero editar el límite de crédito y datos de contacto, y validar que el ZIP sea correcto antes de guardar.” Criterios: se habilita el modo de edición, el formulario valida `zipCode` con regex, y el diálogo de confirmación aparece antes del `PUT`.  
- **Complejo (5-8 pts)**: “Como oficial de cumplimiento, quiero integrarme con un servicio externo de scoring para reflejar cambios de FICO en el formulario de actualización y auditar el historial de cambios.” Criterios: se extiende `useAccountUpdate` para consultar scoring externo y se habilita un historial de auditoría (aún no implementado).

## ⚡ Factores de aceleración de desarrollo

- **`useAccountView` y `useAccountUpdate`** encapsulan toda la lógica de llamadas REST, loaders, errores y detección de cambios, por lo que nuevos formularios pueden reutilizar este patrón.  
- **Componentes del módulo** (`AccountViewScreen`, `AccountUpdateScreen`) usan solo MUI sin base components externos; el patrón de tarjetas + `Stack` provee un layout replicable.  
- **`LoadingSpinner` y `SystemHeader`** ya están disponibles para cualquier nuevo módulo que requiera un header transaccional.  
- **MSW `accountHandlers`/`accountUpdateHandlers`** proveen datos realistas con delays (800ms/1200ms) y validaciones, reduciendo el tiempo de integración con backend real.  
- **`apiClient`** normaliza responses, aborta peticiones viejas y agrega tokens de auth automáticamente, por lo que nuevos endpoints comparten el mismo `COMMON_HEADERS`.

## 🔧 Fundamento técnico

### Patrones de formularios y listas

- Las pantallas usan formularios “full page” (no modales) con `TextField`, `Select`, `Switch` y `Button` de Material-UI.  
- Validaciones “inline” se hacen en el `handleFieldChange` de `AccountUpdateScreen` (números válidos, status Y/N, ZIP).  
- No hay componentes genéricos como `BaseForm`; en cambio se reutiliza el patrón `Stack + Card` y hooks por pantalla.  
- La lista de cuentas no existe en este módulo; cada búsqueda resuelve un `Account ID` único y despliega varios `<Card>` con secciones (Account, Financial, Customer).

### Interacciones clave

- Teclas: F3/Escape salen del módulo; F5 dispara el guardado si hay cambios (`hasChanges`), y F12 resetea el formulario.  
- El botón “Show Sensitive Data” alterna `showSensitiveData` y controla `formatSSN`/`formatCardNumber`.  
- El listado de test accounts solo aparece en `import.meta.env.DEV` y sirve como fallback para QA.  
- `setHasChanges` compara `JSON.stringify(accountData)` vs. `originalData`, habilitando botones `Update`/`Reset` solo cuando hay diferencia real.

## 🌐 Internacionalización

Actualmente el módulo **no implementa i18n**; todos los textos ya están hardcodeados (inglés). La documentación central recomienda introducir `react-i18next` antes de agregar nuevas funcionalidades multi-idioma.

## 🧪 Testing y mocking

- **MSW handlers** (`app/mocks/accountHandlers.ts`, `app/mocks/accountUpdateHandlers.ts`) simulan el backend con cuentas ficticias (111... a 999...), manteniendo congruencia con `AccountViewResponse` y `AccountUpdateData`.  
- Se simulan delays (`setTimeout` de 800ms y 1200ms) para reflejar latencia real en pruebas de UI.  
- Las validaciones en los handlers replican reglas de negocio (ZIP, FICO, active status, account exists), lo que permite escribir tests e2e sin backend.

## ⚡ Presupuestos de performance

- Tiempo de respuesta de búsqueda: **< 500ms (P95)** cuando el backend responde correctamente.  
- Throughput objetivo: **100 búsquedas concurrentes por segundo** (fabricado como objetivo para dimensionar la UI).  
- Uso de memoria por sesión: **< 50MB** en la pestaña, ya que los datos más pesados (cards, customer info) solo se guardan en el estado.  
- Caché: no hay caching ni Redis hasta nuevo aviso; las consultas van directo a PostgreSQL/Microservicio.  
- Índices: `accountId`, `customerId` y `cardNumber` deben tener índices para mantener la ventana de 500ms.

## 🚨 Riesgos y mitigaciones

1. **Riesgo:** La búsqueda recorre tres tablas (`CardXrefRecord`, `Account`, `Customer`) y puede degradarse ante volúmenes altos.  
   **Mitigación:** Añadir índices compuestos y considerar caching en Redis para cuentas frecuentes.  
2. **Riesgo:** No hay internacionalización → mensajes en inglés.  
   **Mitigación:** Implementar `react-i18next` antes de cualquier nueva historia que requiera español.  
3. **Riesgo:** Falta auditoría de cambios.  
   **Mitigación:** Planear un Audit Trail con Spring Data Envers o un table de log (5 pts estimados).  
4. **Riesgo:** Validaciones COBOL comentadas en el código heredado.  
   **Mitigación:** Revisar cada validación y reemplazar con funciones modernas (regex `^\d{11}$`, `FICO 300-850`, etc.).

## ✅ Lista de tareas

### Completadas
- [x] **ACCOUNT-001**: Implementar consulta de cuenta y presentación de tarjetas.  
- [x] **ACCOUNT-002**: Implementar actualización de cuenta con validaciones de negocio.

### Pendientes
- [ ] **ACCOUNT-011**: Evaluar auditoría de cambios para actualizaciones críticas.  
- [ ] **ACCOUNT-012**: Diseñar buffer de caching para búsquedas frecuentes.

## 📈 Métricas de éxito

- **Funcionales:** 0 cuentas mal identificadas por búsqueda; 100% de `infoMessage`/`errorMessage` cubiertos.  
- **Técnicas:** Carga de la pantalla en < 2s, login simultáneo y búsqueda en < 3s.  
- **Negocio:** Tiempo promedio de solución de incidentes < 5 minutos; renovación de datos (+5% accuracy en FICO).

## 🔄 Diagrama de arquitectura

```mermaid
graph TD
  AccountViewScreen --> useAccountView
  useAccountView --> apiClient
  AccountUpdateScreen --> useAccountUpdate
  useAccountUpdate --> apiClient
  apiClient --> AccountViewEndpoint[GET /account-view?accountId]
  apiClient --> AccountInitEndpoint[GET /account-view/initialize]
  apiClient --> AccountDetailEndpoint[GET /accounts/{id}]
  apiClient --> AccountUpdateEndpoint[PUT /accounts/{id}]
  AccountViewScreen -->|muestra| AccountViewCards[Cards: Account, Financial, Customer]
  AccountUpdateScreen -->|edita| UpdateCardSections[Card layout with validation]
  accountHandlers -->|mock| AccountViewEndpoint
  accountUpdateHandlers -->|mock| AccountDetailEndpoint
  AuthModule -->|token| apiClient
```

## 🧭 Readiness & Secuencia recomendada

1. **Prerequisito:** `AUTH` debe entregar tokens JWT y `apiClient` debe tener `Authorization` en `localStorage`.  
2. **Orden recomendado:** primero implementar servicios REST (`/account-view`, `/accounts/{id}`), luego replicar el mock MSW, finalmente construir las pantallas `AccountViewScreen` y `AccountUpdateScreen`.  
3. **Dependencias externas:** Material-UI 5, Redux Toolkit (opcional), React Router para proteger las rutas con `ProtectedRoute`.  

## 📚 Material complementario

- `docs/site/modules/accounts/index.html`: guía visual de user stories, criterios de aceptación y checklist rápido.  
- `docs/system-overview.md#-account---gestión-de-cuentas`: catálogo maestro con datos de dominio, APIs y reglas de negocio.  

**Precisión del codebase:** > 95%
