# # 🤖 SmartRecruit AI

Sistema Inteligente de Matching y Selección de Talento desarrollado con **Streamlit**, **n8n**, **Google Gemini** y **Google Sheets**.

---

## 📋 Descripción General

SmartRecruit AI es una aplicación web que permite a reclutadores:

1. **Analizar vacantes** y obtener un ranking automático de los mejores candidatos de una base de datos.
2. **Consultar la base de datos** de candidatos mediante un chat inteligente con IA.
3. **Exportar el historial** del chat por correo electrónico en formato HTML.

La aplicación frontend está construida en **Streamlit** (Python) y se comunica con un flujo de automatización en **n8n** mediante un webhook HTTP. La inteligencia artificial es provista por **Google Gemini** y los datos de candidatos se almacenan en **Google Sheets**.

---

## 🏗️ Arquitectura General

```
Streamlit (Frontend)
        |
        | HTTP POST (JSON)
        ↓
   Webhook1 (n8n)
        |
        ↓
   Switch (enrutador)
   /         |         \
  /          |          \
Rama 1    Rama 2      Rama 3
Vacante    Chat      Export Chat
```

---

## 📁 Estructura del Proyecto

```
PROYECTO_TAM/
│
├── app.py                          # Aplicación Streamlit (frontend)
├── README.md                       # Este archivo
│
└── n8n/
    ├── Proyecto.json               # Flujo principal n8n (3 ramas)
    └── My_Sub-Workflow_1.json      # Sub-flujo de estadísticas
```

---

## 🔧 Tecnologías Utilizadas

| Tecnología | Uso |
|---|---|
| Python + Streamlit | Interfaz web del usuario |
| n8n (Cloud) | Automatización y orquestación del flujo |
| Google Gemini 2.0 Flash | Modelo de IA para chat y extracción |
| Google Sheets | Base de datos de 100 candidatos |
| Gmail OAuth2 | Envío de correos con resumen del chat |

---

## 🗄️ Base de Datos de Candidatos

La base de datos está almacenada en Google Sheets con **100 candidatos** y las siguientes columnas:

| Columna | Descripción |
|---|---|
| `nombre` | Nombre completo del candidato |
| `profesion` | Título profesional |
| `experiencia_anios` | Años de experiencia laboral |
| `habilidades` | Skills técnicas separadas por coma |
| `universidad` | Universidad de graduación |
| `ingles` | Nivel de inglés (A1, A2, B1, B2, C1, C2) |
| `certificaciones` | Certificaciones obtenidas |
| `expectativa_salarial` | Salario esperado en pesos colombianos |
| `modalidad` | Presencial / Híbrido / Remoto |

---

## 🌐 Frontend — app.py (Streamlit)

### Descripción
La interfaz web permite al usuario interactuar con el sistema a través de dos módulos: **Vacante** y **Chat ATS**.

### Configuración
- El usuario ingresa la URL del webhook de n8n en el sidebar.
- Selecciona el módulo que desea usar.

### Módulo 1 — Vacante
El usuario ingresa la descripción de una vacante en un área de texto y presiona "Analizar Candidatos". Streamlit envía un `POST` al webhook con:
```json
{ "action": "vacante", "vacante": "descripción de la vacante" }
```
Recibe como respuesta el ranking de los 10 mejores candidatos y los muestra con métricas, gráfica de barras y tabla completa.

### Módulo 2 — Chat ATS
El usuario escribe preguntas sobre la base de datos (ej: "¿cuántos ingenieros hay?"). Streamlit envía:
```json
{ "action": "chat", "pregunta": "pregunta del usuario" }
```
Recibe `{ "answer": "respuesta del agente" }` y lo muestra en el chat.

### Exportar Chat por Correo
El usuario activa el checkbox de exportación, ingresa su correo y presiona "Enviar resumen". Streamlit envía:
```json
{
  "action": "export_chat",
  "email": "correo@ejemplo.com",
  "chat_history": [ {"role": "user", "content": "..."}, ... ]
}
```

---

## ⚙️ Flujo Principal n8n — Proyecto.json

El flujo principal contiene **3 ramas** activadas por un webhook único. Cada rama es activada según el valor del campo `action` enviado desde Streamlit.

---

### 🔗 Nodo 1 — Webhook1

**Tipo:** `n8n-nodes-base.webhook`

**¿Qué hace?**
Es el punto de entrada del sistema. Recibe todas las peticiones HTTP POST desde Streamlit y las pasa al Switch para enrutarlas.

**Configuración:**
- **Método HTTP:** POST
- **Path:** `vacante-ai`
- **Response Mode:** Response Node (espera que un nodo posterior responda)

**URL de producción:**
```
https://[instancia].app.n8n.cloud/webhook/vacante-ai
```

---

### 🔀 Nodo 2 — Switch (Enrutador principal)

**Tipo:** `n8n-nodes-base.switch`

**¿Qué hace?**
Lee el campo `action` del cuerpo de la petición y decide a cuál de las 3 ramas enviar el flujo.

**Configuración:**

| Regla | Expresión | Valor | Destino |
|---|---|---|---|
| Routing Rule 1 | `{{$json.body.action}}` | `vacante` | Rama 1 |
| Routing Rule 2 | `{{$json.body.action}}` | `chat` | Rama 2 |
| Routing Rule 3 | `{{$json.body.action}}` | `export_chat` | Rama 3 |

**Modo:** Rules (Expresión evaluada en tiempo de ejecución)

---

## 🟢 RAMA 1 — Análisis de Vacante y Ranking

**Flujo:** `Switch → AI Agent4 → Get row(s) in sheet → Code in JavaScript1 → Respond to Webhook`

---

### Nodo 3 — AI Agent4

**Tipo:** `@n8n/n8n-nodes-langchain.agent`

**¿Qué hace?**
Recibe la descripción de la vacante en texto libre y la convierte en un JSON estructurado con los requisitos del cargo: nombre del cargo, skills requeridas, años de experiencia, nivel de inglés y modalidad.

**Configuración:**
- **Modelo:** Google Gemini 2.0 Flash Lite
- **Temperatura:** 0 (respuestas deterministas)
- **Prompt del sistema:** Instrucciones estrictas para devolver únicamente JSON válido con el formato:
```json
{
  "cargo": "string",
  "skills": ["string"],
  "experiencia": number,
  "ingles": "A1|A2|B1|B2|C1|C2",
  "modalidad": "Presencial|Remoto|Híbrido",
  "salario": number
}
```

---

### Nodo 4 — Get row(s) in sheet (Rama 1)

**Tipo:** `n8n-nodes-base.googleSheets`

**¿Qué hace?**
Lee todos los registros de la hoja `candidatos` del Google Sheets con los 100 candidatos y los pasa al nodo de código para calcular el ranking.

**Configuración:**
- **Documento:** SmartRecruitAI_100_Candidatos
- **Hoja:** candidatos
- **Operación:** Get Row(s) — sin filtros (trae todos los registros)
- **Credencial:** Google Sheets OAuth2 API

---

### Nodo 5 — Code in JavaScript1 (Ranking)

**Tipo:** `n8n-nodes-base.code`

**¿Qué hace?**
Es el núcleo del sistema de matching. Recibe el JSON de la vacante y los 100 candidatos, calcula un puntaje para cada candidato según 6 criterios ponderados y devuelve el Top 10.

**Sistema de puntuación (100 puntos total):**

| Criterio | Peso | Descripción |
|---|---|---|
| Skills / Habilidades | 35 pts | Coincidencias entre skills del candidato y los requeridos |
| Experiencia | 25 pts | Proporcional a los años requeridos |
| Inglés | 10 pts | Comparación de nivel requerido vs nivel del candidato |
| Universidad | 5 pts | Puntaje según ranking de universidades colombianas |
| Profesión | 20 pts | Coincidencia entre profesión del candidato y cargo solicitado |
| Certificaciones | 5 pts | Bonus si tiene alguna certificación válida |

**Código completo comentado:**

```javascript
// ======================
// PASO 1: LIMPIAR EL JSON QUE DEVUELVE GEMINI
// El agente a veces devuelve texto adicional junto al JSON,
// por eso se limpia antes de parsearlo
// ======================

// Obtiene el output del AI Agent4
let raw = $('AI Agent4').first().json.output || "";

// Elimina la palabra "json" si aparece (markdown de Gemini)
// Elimina todo lo que esté antes del primer {
// Elimina todo lo que esté después del último }
raw = raw
  .replace(/json/gi, "")
  .replace(/somePattern/g, "")
  .replace(/^[^\{]*/, "")
  .replace(/[^\}]*$/, "")
  .trim();

// Intenta convertir el texto limpio en un objeto JavaScript
let vacante;
try {
  vacante = JSON.parse(raw);
} catch (e) {
  // Si el JSON es inválido, devuelve un error claro
  return [{ json: { error: true, mensaje: "JSON inválido", raw: raw } }];
}

// ======================
// PASO 2: NORMALIZAR LA VACANTE
// Estandariza los campos para evitar errores de comparación
// ======================

vacante = {
  cargo: String(vacante.cargo || vacante.carga || "").trim(),
  skills: Array.isArray(vacante.skills)
    ? vacante.skills.map(s => normalizarSkill(s))
    : [],
  experiencia: Number(vacante.experiencia || 0),
  ingles: String(vacante.ingles || "").toUpperCase().trim(),
  modalidad: String(vacante.modalidad || "").trim(),
  salario: Number(vacante.salario || vacante.sueldo || 0)
};

// ======================
// PASO 3: FUNCIONES AUXILIARES
// ======================

// Tabla de equivalencia de niveles de inglés a números
// Para poder comparar niveles: C2 > C1 > B2 > B1 > A2 > A1
const niveles = { A1: 1, A2: 2, B1: 3, B2: 4, C1: 5, C2: 6 };

// Normaliza un skill: quita tildes, pasa a minúscula
// y resuelve alias (ej: "piton" → "python")
function normalizarSkill(skill) {
  skill = String(skill)
    .toLowerCase()
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .trim();

  const alias = {
    "piton": "python", "python3": "python", "py": "python",
    "esp-32": "esp32",
    "internet of things": "iot",
    "machine learning": "ml", "aprendizaje automatico": "ml",
    "inteligencia artificial": "ia", "artificial intelligence": "ia"
  };
  return alias[skill] || skill;
}

// Convierte texto separado por comas en array de skills normalizados
function normalizarSkills(texto) {
  return String(texto || "")
    .split(",")
    .map(s => normalizarSkill(s))
    .filter(s => s.length > 0);
}

// ======================
// PASO 4: TABLA DE PUNTAJES POR UNIVERSIDAD
// Universidades colombianas ordenadas por reputación académica
// Escala del 70 al 100
// ======================

const puntajesUniversidad = {
  "Universidad Nacional de Colombia": 100,
  "Universidad de los Andes": 98,
  "Universidad de Antioquia": 95,
  "Pontificia Universidad Javeriana": 94,
  "Universidad del Valle": 93,
  "Universidad EAFIT": 92,
  "Universidad Industrial de Santander": 90,
  "Universidad del Norte": 88,
  "Universidad Distrital Francisco José de Caldas": 86,
  "Universidad Tecnológica de Pereira": 85,
  "Universidad de Nariño": 84,
  "Universidad de Caldas": 82,
  "Universidad del Cauca": 80,
  "Universidad de Cartagena": 79,
  "Universidad Pedagógica Nacional": 78,
  "Universidad Libre": 75,
  "Universidad Cooperativa de Colombia": 73,
  "Universidad Santo Tomás": 72,
  "Universidad Central": 70
};

// ======================
// PASO 5: FUNCIÓN SCORE PROFESIÓN (máx 20 puntos)
// Compara la profesión del candidato con el cargo de la vacante
// Si coinciden exactamente en área: 20 pts
// Si hay relación parcial en software/backend/frontend: 15 pts
// Si no hay relación: 0 pts
// ======================

function scoreProfesion(profesion, cargo) {
  // Normaliza ambos textos para comparar sin tildes ni mayúsculas
  profesion = String(profesion || "").toLowerCase()
    .normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  cargo = String(cargo || "").toLowerCase()
    .normalize("NFD").replace(/[\u0300-\u036f]/g, "");

  if (profesion.includes("electron") && cargo.includes("electron")) return 20;
  if (profesion.includes("telecom") && cargo.includes("telecom")) return 20;
  if (profesion.includes("sistemas") && cargo.includes("sistemas")) return 20;
  if (profesion.includes("software") && cargo.includes("software")) return 20;
  if (profesion.includes("ia") && cargo.includes("ia")) return 20;
  if ((profesion.includes("datos") || profesion.includes("cientifico")) && cargo.includes("datos")) return 20;
  if (profesion.includes("industrial") && cargo.includes("industrial")) return 20;
  if ((profesion.includes("software") || profesion.includes("backend") || profesion.includes("frontend")) &&
      (cargo.includes("software") || cargo.includes("backend") || cargo.includes("frontend"))) return 15;
  return 0;
}

// ======================
// PASO 6: CALCULAR RANKING
// Itera sobre los 100 candidatos y calcula el puntaje de cada uno
// ======================

const candidatos = $input.all(); // Array con todos los candidatos de Sheets
const ranking = [];

for (const item of candidatos) {
  const c = item.json; // Datos del candidato actual
  let score = 0;

  // --- CRITERIO 1: SKILLS (35 puntos) ---
  const skillsCand = normalizarSkills(c.habilidades); // Skills del candidato
  const skillsVac = vacante.skills;                    // Skills requeridos
  let coincidencias = 0;
  for (const s of skillsCand) {
    if (skillsVac.includes(s)) coincidencias++;        // Cuenta coincidencias
  }
  if (skillsVac.length > 0) {
    score += (coincidencias / skillsVac.length) * 35;  // Porcentaje de coincidencia × 35
  }

  // --- CRITERIO 2: EXPERIENCIA (25 puntos) ---
  const exp = Number(c.experiencia_anios || 0);
  const expReq = Number(vacante.experiencia || 0);
  if (expReq > 0) {
    score += Math.min(exp / expReq, 1) * 25; // Máximo 25 pts, proporcional
  } else {
    score += 25; // Si no se requiere experiencia, todos obtienen los 25 pts
  }

  // --- CRITERIO 3: INGLÉS (10 puntos) ---
  const nivCand = niveles[String(c.ingles || "").toUpperCase()] || 1;
  const nivReq = niveles[String(vacante.ingles || "").toUpperCase()] || 1;
  score += vacante.ingles
    ? Math.min(nivCand / nivReq, 1) * 10  // Proporcional si se requiere inglés
    : 10;                                   // 10 pts si no se requiere nivel

  // --- CRITERIO 4: UNIVERSIDAD (5 puntos) ---
  const puntUni = puntajesUniversidad[c.universidad] || 70; // Default 70 si no está en tabla
  score += (puntUni / 100) * 5; // Proporcional al puntaje de la universidad

  // --- CRITERIO 5: PROFESIÓN (20 puntos) ---
  score += scoreProfesion(c.profesion, vacante.cargo);

  // --- CRITERIO 6: CERTIFICACIONES (5 puntos) ---
  if (c.certificaciones && String(c.certificaciones).toLowerCase() !== "ninguna") {
    score += 5; // Bonus fijo por tener certificación
  }

  // Agrega el candidato al ranking con su puntaje final
  ranking.push({
    nombre: c.nombre,
    profesion: c.profesion,
    experiencia: exp,
    ingles: c.ingles,
    universidad: c.universidad,
    habilidades: c.habilidades,
    score_profesion: scoreProfesion(c.profesion, vacante.cargo),
    puntaje: Number(score.toFixed(2)) // Redondea a 2 decimales
  });
}

// ======================
// PASO 7: ORDENAR Y DEVOLVER TOP 10
// ======================

ranking.sort((a, b) => b.puntaje - a.puntaje); // Orden descendente por puntaje
const top10 = ranking.slice(0, 10);             // Toma los primeros 10

// Devuelve la vacante normalizada, el total de candidatos evaluados y el Top 10
return [{ json: { vacante, total_candidatos: ranking.length, top10 } }];
```

---

### Nodo 6 — Respond to Webhook (Rama 1)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Qué hace?**
Devuelve la respuesta final al Streamlit con el resultado del ranking.

**Configuración:**
- **Respond With:** First Incoming Item (devuelve el JSON del nodo anterior directamente)

---

## 🔵 RAMA 2 — Chat ATS con IA

**Flujo:** `Switch → AI Agent → Respond to Webhook1`

El agente tiene conectado como herramienta (Tool) el sub-flujo de estadísticas y una memoria de conversación.

---

### Nodo 7 — AI Agent (Chat)

**Tipo:** `@n8n/n8n-nodes-langchain.agent`

**¿Qué hace?**
Es el cerebro del chat. Recibe la pregunta del usuario, consulta el tool de estadísticas y responde en español con información real de la base de datos.

**Configuración:**
- **Modelo:** Google Gemini 2.0 Flash (`models/gemini-2.0-flash`)
- **Temperatura:** 0.4
- **Source for Prompt:** Define below
- **Prompt (User Message):** `={{ $json.body.pregunta }}` — Expression
- **System Message (Fixed):**

```
Eres un analista de talento de SmartRecruit AI.

Tu única fuente de información es el tool "Call n8n Workflow Tool" que contiene 
las estadísticas completas de la base de 100 candidatos.

REGLAS OBLIGATORIAS:
* Llama SIEMPRE al tool antes de responder cualquier pregunta.
* Nunca inventes datos.
* Si no encuentras el dato en el tool, responde: "No tengo esa información".
* Responde siempre en español.
* Sé breve, claro y profesional.

EL TOOL CONTIENE:
- Total de candidatos
- Profesiones y cantidad por cada una
- Universidades y candidatos por universidad
- Experiencia: promedio, mayores y menores de 5 y 10 años
- Niveles de inglés (A1, A2, B1, B2, C1, C2)
- Modalidades (Presencial, Híbrido, Remoto)
- Top 10 habilidades más comunes
- Certificaciones
- Salarios: promedio, mínimo y máximo
```

**Tools conectados:**
- `Call n8n Workflow Tool` → Sub-flujo de estadísticas

**Memory conectada:**
- `Simple Memory (Window Buffer Memory)` → Session Key: `smartrecruit_session`, Context Window: 10 turnos

---

### Nodo 8 — Call n8n Workflow Tool

**Tipo:** `@n8n/n8n-nodes-langchain.toolWorkflow`

**¿Qué hace?**
Es el tool que el agente llama cuando necesita consultar datos. Ejecuta el sub-flujo `My Sub-Workflow 1` y devuelve las estadísticas completas de la base de datos.

**Configuración:**
- **Description:** "Llama a este tool SIEMPRE que el usuario pregunte cualquier cosa sobre candidatos, profesiones, universidades, habilidades, experiencia, inglés, salarios o modalidades. Contiene estadísticas completas de los 100 candidatos."
- **Source:** Database
- **Workflow:** My Sub-Workflow 1

---

### Nodo 9 — Simple Memory (Window Buffer Memory)

**Tipo:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`

**¿Qué hace?**
Almacena el historial de la conversación para que el agente recuerde las preguntas anteriores dentro de la misma sesión.

**Configuración:**
- **Session ID:** Define below — Fixed
- **Session Key:** `smartrecruit_session`
- **Context Window Length:** 10 (recuerda los últimos 10 turnos de conversación)

---

### Nodo 10 — Respond to Webhook1 (Rama 2)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Qué hace?**
Devuelve la respuesta del agente a Streamlit en el formato que espera el frontend.

**Configuración:**
- **Respond With:** JSON
- **Body (Expression):**
```javascript
={{ {"answer": $json.output ?? $json.text ?? $json.response} }}
```
El operador `??` (nullish coalescing) prueba los tres posibles campos de salida del agente hasta encontrar el que tenga valor.

---

## 🟡 RAMA 3 — Exportar Chat por Correo

**Flujo:** `Switch → Code in JavaScript → Send a message2 → Respond to Webhook2`

---

### Nodo 11 — Code in JavaScript (Export Chat)

**Tipo:** `n8n-nodes-base.code`

**¿Qué hace?**
Recibe el historial de chat desde Streamlit y lo convierte en un correo HTML formateado con colores diferentes para el usuario y el asistente.

**Código completo comentado:**

```javascript
// ======================
// PASO 1: EXTRAER DATOS DEL BODY
// Los datos vienen en $input.first().json.body
// porque el webhook encapsula el cuerpo en .body
// ======================

const data = $input.first().json.body; // Cuerpo completo de la petición
const email = data.email;              // Correo destino ingresado por el usuario
const historial = data.chat_history || []; // Array de mensajes del chat

// ======================
// PASO 2: MANEJAR HISTORIAL VACÍO
// Si no hay conversación, se avisa pero se envía igual
// con el email correcto para no perder el destinatario
// ======================

if (historial.length === 0) {
  return [{ json: { 
    email: email,  // IMPORTANTE: siempre incluir el email
    answer: "<p>No hay conversación para exportar.</p>" 
  }}];
}

// ======================
// PASO 3: CONSTRUIR EL HTML DEL CORREO
// Cada mensaje se convierte en un div con color de fondo diferente:
// - Azul claro (#e8f0fe) para mensajes del usuario
// - Gris claro (#f1f3f4) para mensajes del asistente
// ======================

let resumen = "";
for (const msg of historial) {
  // Determina el rol y el color según el remitente
  const rol = msg.role === "user" ? "👤 Usuario" : "🤖 Asistente";
  const bg = msg.role === "user" ? "#e8f0fe" : "#f1f3f4";
  
  // Crea un div estilizado para cada mensaje
  resumen += `<div style="background:${bg};padding:10px;margin-bottom:8px;border-radius:6px">
    <strong>${rol}:</strong><br>${msg.content}
  </div>`;
}

// ======================
// PASO 4: ARMAR EL HTML FINAL
// Se agrega un título y el cuerpo con todos los mensajes
// ======================

const html = `<h2>Resumen del Chat - SmartRecruit AI</h2>${resumen}`;

// ======================
// PASO 5: DEVOLVER LOS DATOS AL NODO GMAIL
// Devuelve email y answer para que Gmail los use
// ======================

return [{ json: { email: email, answer: html } }];
```

---

### Nodo 12 — Send a message2 (Gmail)

**Tipo:** `n8n-nodes-base.gmail`

**¿Qué hace?**
Envía el correo HTML con el resumen del chat al destinatario ingresado en Streamlit.

**Configuración:**
- **Credencial:** Gmail OAuth2 API (autenticado con Google)
- **Resource:** Message
- **Operation:** Send
- **To:** `{{$json.email}}` — Expression
- **Subject:** `Resumen del chat - SmartRecruit AI` — Fixed
- **Email Type:** HTML
- **Message:** `{{$json.answer}}` — Expression

---

### Nodo 13 — Respond to Webhook2 (Rama 3)

**Tipo:** `n8n-nodes-base.respondToWebhook`

**¿Qué hace?**
Confirma a Streamlit que el correo fue enviado exitosamente devolviendo un status 200 con JSON.

**Configuración:**
- **Respond With:** JSON — Fixed
- **Body:** `{ "status": "ok" }`

Streamlit recibe este `200` y muestra el mensaje verde "Resumen enviado a [correo] ✅".

---

## 🔄 Sub-Flujo — My Sub-Workflow 1

Este flujo secundario es llamado por el agente de chat como herramienta. Su función es leer la base de datos y calcular estadísticas completas.

**Flujo:** `When Executed by Another Workflow → Get row(s) in sheet → Code in JavaScript`

---

### Nodo S1 — When Executed by Another Workflow

**Tipo:** `n8n-nodes-base.executeWorkflowTrigger`

**¿Qué hace?**
Es el trigger que activa el sub-flujo cuando el agente principal lo llama como tool.

**Configuración:**
- **Input data mode:** Accept All Data (acepta cualquier dato sin validación de esquema)

---

### Nodo S2 — Get row(s) in sheet (Sub-flujo)

**Tipo:** `n8n-nodes-base.googleSheets`

**¿Qué hace?**
Lee todos los 100 candidatos de la hoja `candidatos` del Google Sheets.

**Configuración:**
- **Documento:** base datos
- **Hoja:** candidatos
- **Operación:** Get Row(s) — sin filtros
- **Credencial:** Google Sheets OAuth2 API

---

### Nodo S3 — Code in JavaScript (Estadísticas)

**Tipo:** `n8n-nodes-base.code`

**¿Qué hace?**
Procesa los 100 candidatos y calcula estadísticas completas agrupadas por categoría. Este resumen es lo que el agente de chat usa para responder preguntas.

**Código completo comentado:**

```javascript
// ======================
// PASO 1: OBTENER TODOS LOS CANDIDATOS
// $input.all() devuelve un array con los 100 registros de Sheets
// .map(i => i.json) extrae solo el objeto JSON de cada registro
// ======================

const candidatos = $input.all().map(i => i.json);
const total = candidatos.length; // Total de candidatos (100)

// ======================
// PASO 2: CONTAR POR PROFESIÓN
// Crea dos objetos:
// - profesiones: conteo exacto por título ("Ingeniero de Software": 5)
// - tiposProfesion: conteo agrupado por área ("Ingeniería de Software": 14)
// ======================

const profesiones = {};
for (const c of candidatos) {
  const prof = (c.profesion || "SIN_PROFESION").trim();
  profesiones[prof] = (profesiones[prof] || 0) + 1; // Incrementa contador
}

const tiposProfesion = {};
for (const c of candidatos) {
  // Normaliza el texto: minúsculas y sin tildes para comparar
  let tipo = (c.profesion || "").toLowerCase()
    .normalize("NFD").replace(/[\u0300-\u036f]/g, "").trim();

  // Clasifica en categorías grandes según palabras clave
  if (tipo.includes("software")) tipo = "Ingeniería de Software";
  else if (tipo.includes("datos") || tipo.includes("data")) tipo = "Datos / Ciencia de Datos";
  else if (tipo.includes("ia") || tipo.includes("inteligencia")) tipo = "Inteligencia Artificial";
  else if (tipo.includes("electroni")) tipo = "Ingeniería Electrónica";
  else if (tipo.includes("sistemas")) tipo = "Ingeniería de Sistemas";
  else if (tipo.includes("telecom")) tipo = "Telecomunicaciones";
  else if (tipo.includes("industrial")) tipo = "Ingeniería Industrial";
  else if (tipo.includes("backend")) tipo = "Desarrollo Backend";
  else if (tipo.includes("frontend")) tipo = "Desarrollo Frontend";
  else tipo = "Otra";

  tiposProfesion[tipo] = (tiposProfesion[tipo] || 0) + 1;
}

// ======================
// PASO 3: UNIVERSIDADES
// Cuenta cuántos candidatos hay de cada universidad
// ======================

const universidades = {};
for (const c of candidatos) {
  const uni = (c.universidad || "SIN_UNIVERSIDAD").trim();
  universidades[uni] = (universidades[uni] || 0) + 1;
}
const totalUniversidades = Object.keys(universidades).length; // Cuántas universidades diferentes hay

// ======================
// PASO 4: EXPERIENCIA
// Calcula estadísticas de años de experiencia
// ======================

let expMas10 = 0;    // Candidatos con 10+ años
let expMenos10 = 0;  // Candidatos con menos de 10 años
let expMas5 = 0;     // Candidatos con 5+ años
let expMenos5 = 0;   // Candidatos con menos de 5 años
let totalExp = 0;    // Suma total para calcular promedio

for (const c of candidatos) {
  const exp = Number(c.experiencia_anios || 0);
  totalExp += exp;
  if (exp >= 10) expMas10++; else expMenos10++;
  if (exp >= 5) expMas5++; else expMenos5++;
}

// Promedio de experiencia redondeado a 1 decimal
const promedioExperiencia = total > 0
  ? Number((totalExp / total).toFixed(1))
  : 0;

// ======================
// PASO 5: NIVELES DE INGLÉS
// Cuenta candidatos por nivel y agrupa en básico/medio/alto
// ======================

const niveles = {};
for (const c of candidatos) {
  const nivel = (c.ingles || "SIN_NIVEL").trim().toUpperCase();
  niveles[nivel] = (niveles[nivel] || 0) + 1;
}

// Agrupación por banda de nivel
const nivelesAltos = (niveles["C1"] || 0) + (niveles["C2"] || 0);   // Avanzado
const nivelesMedios = (niveles["B1"] || 0) + (niveles["B2"] || 0);  // Intermedio
const nivelesBajos = (niveles["A1"] || 0) + (niveles["A2"] || 0);   // Básico

// ======================
// PASO 6: MODALIDADES DE TRABAJO
// ======================

const modalidades = {};
for (const c of candidatos) {
  const mod = (c.modalidad || "SIN_MODALIDAD").trim();
  modalidades[mod] = (modalidades[mod] || 0) + 1;
}

// ======================
// PASO 7: HABILIDADES / SKILLS
// Cada candidato tiene múltiples habilidades separadas por coma
// Se cuenta cuántos candidatos tienen cada habilidad
// ======================

const habilidades = {};
for (const c of candidatos) {
  const skills = (c.habilidades || "").split(","); // Separa por coma
  for (const s of skills) {
    const skill = s.trim(); // Elimina espacios
    if (skill) {
      habilidades[skill] = (habilidades[skill] || 0) + 1;
    }
  }
}

// Ordena por frecuencia y toma solo el Top 10
const top10Habilidades = Object.entries(habilidades)
  .sort((a, b) => b[1] - a[1])   // Orden descendente
  .slice(0, 10)                    // Solo los primeros 10
  .reduce((acc, [k, v]) => { acc[k] = v; return acc; }, {}); // Convierte a objeto

// ======================
// PASO 8: CERTIFICACIONES
// Cuenta cuántos tienen y cuántos no tienen certificación
// ======================

let conCertificacion = 0;
let sinCertificacion = 0;
const tipoCertificaciones = {};

for (const c of candidatos) {
  const cert = (c.certificaciones || "").trim().toLowerCase();
  if (!cert || cert === "ninguna" || cert === "sin certificacion") {
    sinCertificacion++;
  } else {
    conCertificacion++;
    // Cuenta por tipo de certificación
    tipoCertificaciones[c.certificaciones.trim()] =
      (tipoCertificaciones[c.certificaciones.trim()] || 0) + 1;
  }
}

// ======================
// PASO 9: SALARIOS
// Calcula promedio, mínimo y máximo de expectativa salarial
// ======================

const salarios = candidatos
  .map(c => Number(c.expectativa_salarial || 0))
  .filter(s => s > 0); // Excluye valores en cero

const promedioSalario = salarios.length > 0
  ? Math.round(salarios.reduce((a, b) => a + b, 0) / salarios.length)
  : 0;

const salarioMin = salarios.length > 0 ? Math.min(...salarios) : 0;
const salarioMax = salarios.length > 0 ? Math.max(...salarios) : 0;

// ======================
// PASO 10: DEVOLVER TODAS LAS ESTADÍSTICAS
// Este objeto es el que recibe el agente de chat
// para responder las preguntas del usuario
// ======================

return [{
  json: {
    resumen_general: {
      total_candidatos: total,
      total_universidades_diferentes: totalUniversidades,
      promedio_experiencia_anios: promedioExperiencia
    },
    profesiones: {
      detalle_por_profesion: profesiones,         // Conteo exacto por título
      agrupado_por_tipo: tiposProfesion,          // Conteo agrupado por área
      total_profesiones_diferentes: Object.keys(profesiones).length
    },
    universidades: {
      total_universidades: totalUniversidades,
      candidatos_por_universidad: universidades   // Cuántos de cada universidad
    },
    experiencia: {
      con_10_o_mas_anios: expMas10,
      con_menos_de_10_anios: expMenos10,
      con_5_o_mas_anios: expMas5,
      con_menos_de_5_anios: expMenos5,
      promedio_anios: promedioExperiencia
    },
    ingles: {
      distribucion_por_nivel: niveles,            // A1, A2, B1, B2, C1, C2
      nivel_alto_C1_C2: nivelesAltos,
      nivel_medio_B1_B2: nivelesMedios,
      nivel_basico_A1_A2: nivelesBajos
    },
    modalidades: {
      distribucion: modalidades                   // Presencial, Híbrido, Remoto
    },
    habilidades: {
      top_10_habilidades: top10Habilidades,
      total_habilidades_diferentes: Object.keys(habilidades).length
    },
    certificaciones: {
      con_certificacion: conCertificacion,
      sin_certificacion: sinCertificacion,
      detalle_certificaciones: tipoCertificaciones
    },
    salarios: {
      promedio: promedioSalario,
      minimo: salarioMin,
      maximo: salarioMax
    }
  }
}];
```

---

## 🚀 Cómo Ejecutar el Proyecto

### Requisitos
- Cuenta en n8n Cloud
- Cuenta de Google (Sheets + Gmail)
- API Key de Google AI Studio (Gemini)
- Python 3.8+
- Google Colab o entorno local

### Pasos

1. **Importar flujos en n8n:**
   - Importar `Proyecto.json` como flujo principal
   - Importar `My_Sub-Workflow_1.json` como sub-flujo
   - Configurar credenciales de Google Sheets, Gmail y Gemini

2. **Preparar Google Sheets:**
   - Crear una hoja llamada `candidatos`
   - Agregar las columnas: `nombre`, `profesion`, `experiencia_anios`, `habilidades`, `universidad`, `ingles`, `certificaciones`, `expectativa_salarial`, `modalidad`

3. **Publicar el flujo en n8n:**
   - Hacer clic en "Publish" en el flujo principal
   - Copiar la URL de producción del webhook

4. **Ejecutar Streamlit:**
```bash
pip install streamlit plotly requests pandas openpyxl
streamlit run app.py
```

5. **Usar la aplicación:**
   - Pegar la URL del webhook en el sidebar
   - Seleccionar el módulo deseado
   - ¡Listo!

---

## 📊 Estadísticas de la Base de Datos Actual

| Métrica | Valor |
|---|---|
| Total candidatos | 100 |
| Universidades diferentes | 19 |
| Promedio de experiencia | 5.4 años |
| Salario promedio | $5.760.000 COP |
| Salario mínimo | $2.500.000 COP |
| Salario máximo | $10.000.000 COP |
| Con certificación | 90 |
| Sin certificación | 10 |
| Nivel inglés alto (C1/C2) | 34 |
| Nivel inglés medio (B1/B2) | 37 |
| Nivel inglés básico (A1/A2) | 29 |
| Modalidad Híbrido | 39 |
| Modalidad Presencial | 34 |
| Modalidad Remoto | 27 |

---

## 👨‍💻 Autor

Proyecto desarrollado como sistema inteligente de selección de talento humano usando automatización con IA.
