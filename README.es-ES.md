

<div align="center">

![AmpliPost Banner](./banner.svg)

<br>

[![License](https://img.shields.io/badge/License-MIT-6366f1?style=flat-square)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.9+-06b6d4?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Go](https://img.shields.io/badge/Go-1.21+-00acd7?style=flat-square&logo=go&logoColor=white)](https://go.dev)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Multi--Agent-8b5cf6?style=flat-square)](https://docs.anthropic.com/claude-code)
[![MCP](https://img.shields.io/badge/MCP-xiaohongshu--mcp-00acd7?style=flat-square)](https://github.com/xpzouying/xiaohongshu-mcp)
[![Playwright](https://img.shields.io/badge/Playwright-Latest-10b981?style=flat-square&logo=playwright&logoColor=white)](https://playwright.dev)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-f43f5e?style=flat-square)](CONTRIBUTING.md)

<br>

**AmpliPost** es una plataforma central de marketing inteligente construida sobre la **arquitectura Multi-Agent de Claude Code**. Tres agentes colaborativos (`content-coordinator`, `content-reviewer`, `publish-guard`) tienen roles claramente definidos y, junto con el sistema de Hooks y la memoria a largo plazo, transforman un «comando de una sola línea» en la producción y distribución totalmente automatizada de contenido multiplataforma. La capa de publicación de Xiaohongshu es impulsada por **xiaohongshu-mcp** (Go + go-rod + CDP). —Sin confirmación manual, sin operaciones manuales, decisión completamente autónoma.

<br>

[Inicio rápido](#-快速开始) · [Arquitectura Multi-Agent](#-multi-agent-架构) · [Pipeline de publicación](#-发布流水线) · [Matriz de plataformas](#-平台矩阵) · [Sistema de Hooks](#-hooks-系统) · [Memoria a largo plazo](#-长期记忆)

</div>

---

## ¿Por qué Multi-Agent + MCP

El cuello de botella de los scripts de automatización tradicionales no está en la ejecución, sino en el juicio y el control de riesgos. ¿Es buena la calidad del contenido? ¿Esta publicación activará el control de riesgos? Si un solo Agente se hace y responde estas preguntas, existe un sesgo natural. Al mismo tiempo, Playwright utiliza el protocolo WebDriver, y la característica `navigator.webdriver=true` es detectada por plataformas como Xiaohongshu, lo que provoca el bloqueo continuo de cuentas.

La solución de AmpliPost es una doble separación: **separación de responsabilidades** (`content-coordinator` genera, `content-reviewer` evalúa, `publish-guard` controla riesgos; los tres se comunican mediante JSON estructurado sin interferirse) + **separación tecnológica** (la capa de publicación de Xiaohongshu se independiza como el servidor MCP `xiaohongshu-mcp`, Go + go-rod se conecta directamente a CDP, sin características de WebDriver, comportamiento humanizado, persistencia de cookies, el autor original lo ha usado durante un año sin bloqueos).

---

## 🤖 Arquitectura Multi-Agent

![Architecture](./architecture.svg)

### Límites de responsabilidad de los tres Agentes + Servidor MCP

<table>
<tr>
<th width="22%">Componente</th>
<th width="38%">Responsabilidad</th>
<th width="40%">Qué NO hace</th>
</tr>
<tr>
<td><strong>content-coordinator</strong><br><sub>Agente principal</sub></td>
<td>Analiza la entrada del usuario, genera contenido para cada plataforma, orquesta subagentes, ejecuta la publicación, informa resultados y actualiza la memoria</td>
<td>No evalúa la calidad del contenido por sí mismo, no toma decisiones de control de riesgos</td>
</tr>
<tr>
<td><strong>content-reviewer</strong><br><sub>Evaluación de calidad</sub></td>
<td>Evalúa independientemente en cinco dimensiones: fuerza del gancho, densidad de información, autenticidad, adaptación a la plataforma y diversidad, ofreciendo sugerencias de modificación específicas</td>
<td>No genera contenido, no toma decisiones de publicación</td>
</tr>
<tr>
<td><strong>publish-guard</strong><br><sub>Guardián de riesgos</sub></td>
<td>Evalúa la frecuencia de publicación, la regularidad de los intervalos, la diversidad del contenido y las características de comportamiento de la cuenta, emitiendo una decisión de tres estados: allow / delay / block</td>
<td>No califica la calidad del contenido, no modifica el contenido</td>
</tr>
<tr>
<td><strong>xiaohongshu-mcp</strong><br><sub>Servidor MCP · Go</sub></td>
<td>Capa de ejecución subyacente para publicación en Xiaohongshu, escucha en localhost:18060, se conecta directamente a Chromium mediante go-rod + CDP, proporciona 13 herramientas</td>
<td>No genera contenido, no toma ningún tipo de decisión</td>
</tr>
</table>

### Límites de decisión autónoma

El Agente **manea automáticamente sin consultar al usuario** en los siguientes casos: cuando no se especifica la plataforma, infiere la plataforma objetivo según el tipo de contenido; si Douyin carece de imágenes, invoca automáticamente `generate_images.py` para generar infografías; cuando el contenido contiene palabras prohibidas, las reemplaza automáticamente; si la calidad del contenido no cumple los estándares, lo entrega a `reviewer` para evaluación y lo reescribe según las sugerencias (máximo 2 veces); cuando la evaluación de riesgos es `delay`, espera un retraso aleatorio antes de publicar; si el servicio `xiaohongshu-mcp` no se está ejecutando, lo invoca automáticamente en segundo plano (incluyendo instalación automática); si la publicación falla y es reparable, reintenta automáticamente (máximo 3 veces).

**Solo se detendrá para consultar al usuario en dos situaciones**: estado de inicio de sesión inválido (requiere escaneo manual de código, limitación física); instrucción completamente ambigua (tema vacío, no se puede inferir).

---

## 🚀 Inicio rápido

### Instalar dependencias

```bash
# Playwright 浏览器自动化（闲鱼 / B站 / 抖音）
npm install -g agent-browser
agent-browser install

# Scrapling 反爬增强
pip install "scrapling[all]>=0.4.3"
scrapling install --force
```

### Instalación de un solo comando de xiaohongshu-mcp (capa de publicación Xiaohongshu)

```bash
# 自动 clone、编译、写入环境变量（需要 Go 1.21+）
bash scripts/setup-xhs-mcp.sh

# 首次登录小红书（扫码，只需一次）
bash scripts/start-xhs-mcp.sh --login
```

> Ya no es necesario administrar el servicio manualmente. Cuando `content-coordinator` publique en Xiaohongshu, detectará y invocará automáticamente el Servidor MCP.

### Configurar API Key

```bash
cp keys.example.txt keys.txt
# 编辑 keys.txt，填入各平台所需的 API Key
```

### Disparar toda la cadena con una línea

```bash
# 内容类型自动推断平台（干货 → 小红书 + 抖音 + B站）
"帮我发：2025年最值得入手的5款AI工具"

# 指定平台
"发小红书和抖音：职场效率提升的3个反直觉技巧"

# 闲鱼商品（自动识别为商品类 → 闲鱼 + 小红书）
"帮我发闲鱼：iPhone 15 Pro Max 256G，5999元，95新"

# 深度技术文章（→ B站 + 小红书）
"写一篇关于 Scrapling 反爬原理的深度文章"
```

### Notas sobre el estado de inicio de sesión

```bash
# 各平台首次使用需手动扫码（90秒窗口期），之后 Cookie 自动复用
# 闲鱼:  ~/.openclaw/browser_profiles/xianyu_default/
# 小红书: $XHS_MCP_DIR/cookies.json  （由 xiaohongshu-mcp 管理）
# B站:   ~/.catpaw/bilibili_browser_profile/
# 抖音:  ~/.catpaw/douyin_browser_profile/
```

---

## 🔄 Pipeline de publicación

![Pipeline](./scrapling-flow.svg)

El pipeline completo consta de 8 fases, ejecutadas automáticamente de principio a fin:

**Fase 0** Lee `memory.md`, extrae direcciones de contenido históricas efectivas, preferencias del usuario, huellas digitales de contenido y registros de control de riesgos, proporcionando referencia para la generación posterior.

**Fases 1–2** Analiza la entrada del usuario, infiere la plataforma objetivo, busca la ruta de los scripts Skill de cada plataforma (para Xiaohongshu verifica si el Servidor MCP está en ejecución; si no, lo invoca automáticamente en segundo plano), procesa el árbol de decisiones de imágenes (genera automáticamente infografías si Douyin no tiene imágenes, utiliza el modo de imagen con texto si Xiaohongshu no tiene imágenes).

**Fase 3** Genera contenido de forma independiente para cada plataforma objetivo, cumpliendo estrictamente las especificaciones de longitud, estructura y estilo de cada plataforma, prohibiendo la copia cruzada de contenido entre plataformas.

**Fase 3.5** Invoca al subagente `content-reviewer` para una evaluación de calidad independiente; si la puntuación total es inferior a 70, se reescribe según las sugerencias específicas, máximo 2 reescrituras.

**Fase 4.5** Invoca al subagente `publish-guard` para la evaluación de riesgos; en caso de decisión `delay`, espera un retraso aleatorio; en caso de `block`, omite la publicación y explica el motivo en el informe.

**Fases 5–8** Ejecuta la publicación secuencialmente (Xiaohongshu mediante llamada HTTP POST a `xiaohongshu-mcp`, otras plataformas invocan scripts Skill en Python, con un intervalo de 15 segundos entre plataformas), verifica los indicadores de éxito, genera un informe en tabla y escribe el registro de publicación y la huella digital del contenido en `memory.md`.

---

## 🎯 Matriz de plataformas

<table>
<tr>
<th align="center" width="25%">🐟 Xianyu</th>
<th align="center" width="25%">📕 Xiaohongshu</th>
<th align="center" width="25%">📺 Bilibili Column</th>
<th align="center" width="25%">🎵 Douyin图文</th>
</tr>
<tr>
<td>

Título 10–30 caracteres<br>
【Estado nuevo/usado】Nombre del producto Especificación<br>
Reemplazo automático de palabras prohibidas<br>
Imagen AI opcional<br>
Python · Playwright

</td>
<td>

Título ≤20 caracteres<br>
Cuerpo 200–300 caracteres<br>
Estructura de cuatro partes: Dolor → Contenido útil → Cierre → Interacción<br>
**xiaohongshu-mcp · Go+CDP**<br>
Sin WebDriver · Verificación anti-riesgos ✅

</td>
<td>

Título ≤40 caracteres<br>
Cuerpo 800–1500 caracteres<br>
Estructura de cinco partes: Introducción → Análisis → Contenido útil → Errores comunes → Interacción<br>
3–5 etiquetas de tema<br>
Python · Playwright

</td>
<td>

Título ≤30 caracteres<br>
Cuerpo 150–500 caracteres<br>
Debe tener imágenes (genera automáticamente infografías si no hay imágenes)<br>
3–5 etiquetas de tema #<br>
Python · Playwright

</td>
</tr>
</table>

### Reglas de inferencia inteligente de plataformas

| Tipo de contenido | Publicación automática en |
|---------|-----------|
| Venta de productos de segunda mano | Xianyu + Xiaohongshu |
| Contenido útil / Experiencia compartida | Xiaohongshu + Douyin + Bilibili |
| Promoción de productos / Marketing | Xiaohongshu + Douyin |
| Artículo técnico en profundidad | Bilibili + Xiaohongshu |

---

## ⚡ Sistema de Hooks

AmpliPost establece una interceptación antes y después de cada comportamiento de publicación mediante el mecanismo de Hooks de Claude Code:

**Hook PreToolUse** Se activa antes de cada invocación de script de publicación (Xianyu / Bilibili / Douyin), verificando palabras prohibidas de Xianyu (alta imitación/falsificación/precio más bajo de la web/compra surrogada), emojis de Bilibili y Douyin, y palabras extremas (mejor de la web/en la historia/absolutamente mejor). Si se detectan palabras prohibidas o emojis, `exit 2` detiene la ejecución; las palabras extremas solo generan una advertencia sin bloquear. Xiaohongshu utiliza llamadas HTTP MCP, y la verificación de cumplimiento se completa en la fase de generación de contenido, sin pasar por este Hook.

**Hook PostToolUse** Se activa de forma asíncrona después de que finalice la ejecución del script de publicación, detecta indicadores de éxito según la plataforma y escribe los resultados en `~/.amplipost/logs/publish_YYYYMMDD.log`.

```json
{
  "hooks": {
    "PreToolUse":  [{ "matcher": "Bash", "command": "python3 .claude/hooks/pre-publish-check.py" }],
    "PostToolUse": [{ "matcher": "Bash", "command": "python3 .claude/hooks/post-publish-verify.py", "async": true }]
  }
}
```

---

## 💾 Memoria a largo plazo

`memory.md` es la capa de memoria persistente de AmpliPost, mantenida automáticamente por `content-coordinator` al finalizar cada tarea, y leída activamente por `content-reviewer` y `publish-guard` durante sus evaluaciones.

La capa de memoria contiene seis secciones: **Evolución de direcciones de publicación** (direcciones efectivas y por evitar), **Acumulación de experiencia por plataforma** (patrones de títulos/introducciones efectivos por plataforma), **Registros de publicación** (hora, plataforma, tema y estado de cada publicación), **Notas de iteración autónoma del Agente** (patrones y ideas de mejora descubiertos por el Agente), **Registros de preferencias del usuario** (requisitos de estilo expresados explícitamente), **Registros de control de riesgos** (decisiones y puntuaciones de riesgo de cada evaluación).

A medida que aumenta el uso, el Agente acumulará gradualmente comprensión sobre las preferencias del usuario y los patrones de las plataformas, mejorando continuamente la calidad del contenido y la seguridad ante riesgos.

---

## 📁 Estructura del proyecto

```
Amplipost/
├── CLAUDE.md                          # Instrucciones del proyecto Claude Code (chuleta)
├── AGENTS.md                          # Descripción de la arquitectura Multi-Agent
├── SPEC.md                            # Especificaciones completas del sistema
├── memory.md                          # Memoria a largo plazo (mantenimiento automático)
├── keys.example.txt                   # Plantilla de configuración de API Key
│
├── scripts/
│   ├── setup-xhs-mcp.sh              # Instalación de un comando de xiaohongshu-mcp (clone+compilación+variables de entorno)
│   └── start-xhs-mcp.sh              # Iniciar/invocar automáticamente el Servidor MCP
│
├── publishers/                        # Scripts Skill (solo lectura, solo ejecutan publicación)
│   ├── xianyu-publisher/
│   │   └── scripts/
│   │       ├── xianyu_publish.py
│   │       └── xianyu_publish_scrapling.py
│   ├── xhs-publisher/
│   │   └── SKILL.md                  # Normas de publicación de Xiaohongshu (método de llamada MCP)
│   ├── bilibili-publisher/
│   └── douyin-publisher/
│       └── scripts/
│           └── generate_images.py    # Generación de infografías AI para Douyin
│
└── .claude/
    ├── settings.json                  # Registro de Hooks + configuración de permisos
    ├── agents/
    │   ├── content-coordinator.md    # Agente principal
    │   ├── content-reviewer.md       # Subagente de evaluación de calidad
    │   └── publish-guard.md          # Subagente de guardia de riesgos
    └── hooks/
        ├── pre-publish-check.py      # Hook PreToolUse
        └── post-publish-verify.py    # Hook PostToolUse
```

---

## 🔧 Solución de problemas

**Servidor MCP de Xiaohongshu no iniciado**

`content-coordinator` invocará automáticamente `scripts/start-xhs-mcp.sh --bg` para levantar el servicio en segundo plano. Si la invocación automática falla (generalmente por falta de inicio de sesión inicial), mostrará: `bash scripts/start-xhs-mcp.sh --login`.

**Despliegue en una computadora nueva**

```bash
bash scripts/setup-xhs-mcp.sh        # 自动 clone + 编译 + 写环境变量
bash scripts/start-xhs-mcp.sh --login  # 首次扫码登录
```

**Evaluación de contenido sin aprobación continua**

Cuando `content-reviewer` falla consecutivamente 2 veces, se omitirá esa plataforma, y el informe final indicará «Evaluación de contenido no aprobada» junto con el motivo específico. Puede ajustar la descripción de entrada y volver a disparar.

**Publicación bloqueada por control de riesgos**

Cuando `publish-guard` emite la decisión `block`, el informe explicará el motivo (generalmente frecuencia de publicación demasiado alta ese día o repetición de contenido excesiva). Se recomienda esperar 24–48 horas antes de reintentar o ajustar el tema del contenido.

**Texto chino ilegible en imágenes (macOS)**

```bash
brew install font-morisawa
cp $(find /usr/fonts -name "*.ttc" | head -1) ~/Library/Fonts/
```

---

## 🤝 Contribuciones

```bash
git checkout -b feature/your-feature
git commit -m 'feat: add your feature'
git push origin feature/your-feature
# 提交 Pull Request
```

**Nota:** `publishers/*/scripts/*.py` y `publishers/*/SKILL.md` son archivos de solo lectura, y `.claude/settings.json` está protegido mediante reglas `deny`. Por favor, no modifique estas rutas en los PR.

---

## 📄 Licencia

[MIT License](LICENSE) · © 2025 Alan Song & Roxy Li

---

## 🙏 Agradecimientos

La capacidad de publicación de Xiaohongshu de este proyecto se basa en [xiaohongshu-mcp](https://github.com/xpzouying/xiaohongshu-mcp). ¡Gracias por el código abierto!

---

<div align="center">

[![Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-8b5cf6?style=for-the-badge)](https://docs.anthropic.com/claude-code)
[![xiaohongshu-mcp](https://img.shields.io/badge/XHS-xiaohongshu--mcp-00acd7?style=for-the-badge&logo=go)](https://github.com/xpzouying/xiaohongshu-mcp)
[![Playwright](https://img.shields.io/badge/Browser-Playwright-06b6d4?style=for-the-badge&logo=playwright)](https://playwright.dev)

*Una frase entra · Cuatro plataformas salen · Cero intervención humana*

</div>
