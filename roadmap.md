# Roadmap — Herramienta Digital de Educación Financiera

**Proyecto:** APLICACIONMATEFI
**Cliente (caso):** banco colombiano — programa de educación financiera
**Entregable:** prototipo funcional web con tres secciones (Educación Financiera, Simulación, Interacciones)
**Stack decidido:** HTML + CSS + JavaScript en **un solo archivo autocontenido** (`index.html`), sin dependencias ni build
**Persistencia:** `localStorage` del navegador (progreso FSRS)
**IA declarada:** Claude (Anthropic) — Cowork
**Fecha de creación del roadmap:** 2026-08-19

---

## 0. Reglas de oro del proyecto (leer antes de escribir una sola línea)

Estas cinco reglas se violan con facilidad y son las que hacen perder puntos:

1. **La herramienta NO convierte tasas entre periodicidades.** Si la tasa es bimestral, el tiempo se mide en bimestres. Nunca calcular una tasa efectiva anual a partir de una bimestral. Esta restricción es deliberada.
2. **Contar periodos SÍ es válido y obligatorio.** 2 años y 4 meses = 28 meses = **14 bimestres**. Eso es conteo de periodos, no conversión de tasa.
3. **Interés continuo: el tiempo SIEMPRE se expresa en años.** Es la convención del curso. La UI debe forzarlo y explicarlo.
4. **Formato numérico colombiano:** punto para miles, coma para decimales (`1.234.567,89`). Tanto en salida como en la lectura de lo que escribe el usuario.
5. **Claridad nivel "niño de 5 años"** en la sección educativa, sin sacrificar precisión técnica. Si la explicación solo la entiende quien ya sabe el tema, no cumple.

---

## 1. Sistema de diseño

### 1.1 Paleta obligatoria

| Token | Hex | Uso |
|---|---|---|
| `--verde-claro` | `#A3B18A` | acentos suaves, badges, hover, barras de progreso |
| `--hueso` | `#DAD7CD` | fondos de sección alternos, bordes, separadores |
| `--verde-medio` | `#588157` | botones primarios, enlaces activos, iconografía |
| `--verde-oscuro` | `#3A5A40` | sidebar, encabezados de tabla, botón activo |
| `--verde-profundo` | `#344E41` | fondo del sidebar, footer, sombras profundas |
| `--blanco` | `#FFFFFF` | fondo principal del área de contenido, tarjetas |
| `--negro` | `#000000` | todo el texto |

Reglas de aplicación:

- **Sidebar lateral fijo:** fondo `--verde-profundo`, ítem activo `--verde-oscuro` con borde izquierdo `--verde-claro`.
- **Área de contenido:** fondo `--blanco`; tarjetas con borde `--hueso`.
- **Botones primarios:** fondo `--verde-medio`, texto blanco; hover `--verde-oscuro`.
- **Botones secundarios:** fondo transparente, borde `--verde-medio`, texto negro.
- **Gráficos y elementos gráficos:** exclusivamente la escala de 5 verdes (de `--verde-claro` a `--verde-profundo`).
- **Texto:** negro `#000000` sobre fondos claros; blanco solo sobre `--verde-oscuro`/`--verde-profundo`.
- Prohibido introducir colores fuera de la paleta (nada de rojos/azules para errores: usar `--verde-profundo` + icono + texto).

### 1.2 Tipografía y layout

- Fuente del sistema (`system-ui, -apple-system, "Segoe UI", Roboto, sans-serif`) — cero dependencias externas.
- Layout: `grid` de dos columnas → sidebar `260px` fijo + contenido `1fr`. En < 900px el sidebar colapsa a barra superior.
- Espaciado en múltiplos de `8px`. Radio de esquina `10px`. Sombras sutiles y en tono verde.
- Interfaz **limpia y ordenada**: máximo 2 niveles de jerarquía visual por pantalla, mucho aire en blanco.

- [ ] Definir todas las variables CSS en `:root`
- [ ] Construir el shell (sidebar + contenido + router por hash)
- [ ] Verificar contraste de texto negro sobre cada color de la paleta

---

## 2. Arquitectura de archivos

Aunque el entregable es un solo archivo, **internamente se organiza en módulos claramente separados** por comentarios-bandera, en este orden. Esto mantiene el archivo navegable y facilita extraerlo a archivos separados si más adelante se quiere.

```
C:\Proyectos\APLICACIONMATEFI\
├── roadmap.md              ← este documento
├── index.html              ← ENTREGABLE ÚNICO (autocontenido)
└── docs\
    ├── interacciones.md    ← registro fuente de prompts (se copia a la sección Interacciones)
    └── casos-prueba.md     ← resultados esperados de la batería de pruebas
```

Estructura interna de `index.html`:

```
<style>
  ├── [1] TOKENS         variables CSS de la paleta
  ├── [2] BASE           reset, tipografía, utilidades
  ├── [3] LAYOUT         sidebar, contenido, responsive
  └── [4] COMPONENTES    botones, inputs, tarjetas, tablas, flashcards, timeline
</style>

<body>
  ├── <aside id="sidebar">        navegación entre las 3 secciones
  └── <main id="contenido">
        ├── <section id="educacion">
        ├── <section id="simulacion">
        └── <section id="interacciones">

<script>
  ├── [M1] fmt.js        formato/parsing numérico colombiano y monedas
  ├── [M2] tiempo.js     años+meses ↔ número de periodos de la tasa
  ├── [M3] finanzas.js   factores de los 3 regímenes + ecuación de valor
  ├── [M4] solver.js     despeje algebraico + bisección/Newton-Raphson
  ├── [M5] fsrs.js       algoritmo FSRS-4.5 + persistencia localStorage
  ├── [M6] contenido.js  textos educativos, mazo de tarjetas, prompts
  ├── [M7] ui.js         render de secciones, tabla de flujos, timeline
  └── [M8] app.js        router, eventos, arranque
</script>
```

- [ ] Crear el esqueleto de `index.html` con los 8 módulos vacíos y sus banderas
- [ ] Crear `docs\interacciones.md` y `docs\casos-prueba.md`

---

## 3. Núcleo financiero (el corazón del proyecto)

### 3.1 Módulo M2 — Tiempo y periodicidad (capacidades **a** y **b**)

Cada periodicidad se representa por `m` = número de meses que dura un periodo de la tasa:

| Periodicidad | `m` (meses) |
|---|---|
| Mensual | 1 |
| Bimestral | 2 |
| Trimestral | 3 |
| Cuatrimestral | 4 |
| Semestral | 6 |
| Anual | 12 |
| **Cada N meses** (p. ej. quintumestral) | `N` (entrada libre del usuario, entero ≥ 1) |

Conversión de entrada (años + meses → número de periodos):

```
mesesTotales = años * 12 + meses
n            = mesesTotales / m
```

Conversión de salida (cuando la incógnita es el tiempo):

```
mesesTotales = n * m
años         = trunc(mesesTotales / 12)
meses        = mesesTotales - años * 12       // se muestra con decimales si los tiene
```

> **Excepción interés continuo:** `m = 12` siempre. El tiempo se convierte a **años** (`t = mesesTotales / 12`) y el resultado se devuelve en años y meses. La UI deshabilita el selector de periodicidad y muestra el aviso: *"En interés continuo la tasa es anual y el tiempo se mide en años (convención del curso)."*

- [ ] Implementar `mesesAPeriodos(años, meses, m)` y `periodosAAñosMeses(n, m)`
- [ ] Implementar el modo "cada N meses" con validación `N ≥ 1` entero
- [ ] Bloquear periodicidad cuando el régimen es continuo

### 3.2 Módulo M3 — Los tres regímenes (capacidad **h**)

Sea `VP` valor presente, `VF` valor futuro, `i` tasa por periodo (decimal), `n` número de periodos, `r` tasa nominal anual continua, `t` tiempo en años.

**Interés simple** — los intereses se calculan siempre sobre el capital inicial; no se reinvierten.

```
VF = VP · (1 + i·n)
VP = VF / (1 + i·n)
i  = (VF/VP − 1) / n
n  = (VF/VP − 1) / i
I  = VP · i · n
```

**Interés compuesto** — los intereses se suman al capital y a partir de ahí también generan intereses.

```
VF = VP · (1 + i)^n
VP = VF · (1 + i)^(−n)
i  = (VF/VP)^(1/n) − 1
n  = ln(VF/VP) / ln(1 + i)
I  = VF − VP
```

**Interés continuo** — la capitalización ocurre en todo instante; es el límite del compuesto cuando el número de capitalizaciones por año tiende a infinito.

```
VF = VP · e^(r·t)          (t en AÑOS)
VP = VF · e^(−r·t)
r  = ln(VF/VP) / t
t  = ln(VF/VP) / r
I  = VF − VP
```

**Factor de traslado unificado.** Todo el motor se apoya en una sola función que mueve un flujo desde su momento `tk` hasta la fecha focal `tf`:

```
factor(tk, tf, régimen):
  Δ = tf − tk
  simple:     Δ ≥ 0 → (1 + i·Δ)          Δ < 0 → 1 / (1 − i·Δ)
  compuesto:  (1 + i)^Δ
  continuo:   e^(r·Δ)
```

> Nota sobre interés simple: el traslado hacia adelante **capitaliza** y hacia atrás **descuenta** (no son operaciones inversas exactas, a diferencia del compuesto). Por eso en interés simple la **fecha focal es un dato del ejercicio** y la UI la pide explícitamente.

- [ ] Implementar `factor(tk, tf, regimen, tasa)`
- [ ] Cubrir el caso degenerado `Δ = 0` (factor = 1)
- [ ] Validar `1 + i·Δ > 0` en simple (evitar división por cero / negativos)

### 3.3 Módulo M3 — Ecuación de valor con múltiples flujos (capacidades **c** y **f**)

Cada flujo de la tabla es un objeto:

```js
{
  id,
  etiqueta,      // "Cuota 1", "Desembolso inicial"...
  signo,         // +1 = entra (ingreso/abono)   −1 = sale (egreso/pago)
  coef,          // COEFICIENTE (capacidad f), por defecto 1
  valor,         // número, o null si es la incógnita
  tAños, tMeses, // momento del flujo (se convierte a periodos)
  incognitaValor,   // true si el valor es lo que se busca
  incognitaMomento  // true si el momento es lo que se busca
}
```

**Ecuación de valor en la fecha focal `tf`:**

```
Σ_k  signo_k · coef_k · valor_k · factor(t_k, tf) = 0
```

El coeficiente resuelve directamente enunciados como *"el primer desembolso fue 1,4 veces el segundo"*: se define `X` como el flujo desconocido, el segundo desembolso lleva `coef = 1`, el primero lleva `coef = 1,4`, y el usuario **no hace álgebra**.

- [ ] Tabla dinámica de flujos: agregar / eliminar / editar fila
- [ ] Campo `coef` por fila, valor por defecto `1`
- [ ] Selector de signo con etiquetas humanas ("Recibo" / "Pago")
- [ ] Selector de fecha focal (por defecto `t = 0`)
- [ ] Validar: exactamente **una** incógnita en todo el ejercicio

### 3.4 Módulo M4 — Solver (capacidad **d**)

Seis incógnitas seleccionables. Todas se reducen a dos casos:

**Caso A — la incógnita entra linealmente** (valor futuro, valor presente, valor de un flujo desconocido).
La ecuación de valor tiene la forma `A + B·X = 0` con

```
A = Σ_{k ≠ X}  signo_k · coef_k · valor_k · factor(t_k, tf)
B = Σ_{k ∈ X}  signo_k · coef_k · factor(t_k, tf)      // suma sobre todos los flujos que son múltiplo de X

X = −A / B
```

Se resuelve **de forma exacta, sin iterar**. Esto cubre de un solo golpe VP, VF y "valor de un flujo desconocido", incluyendo el caso de varios flujos proporcionales a la misma incógnita (coeficientes).

**Caso B — la incógnita entra de forma no lineal** (tasa de interés, tiempo, momento en que ocurre un flujo).
Se define `g(x) = Σ_k signo_k · coef_k · valor_k · factor_x(t_k, tf)` y se busca `g(x) = 0`:

1. **Bracketing:** barrer el dominio buscando cambio de signo.
   - tasa: `x ∈ (0, 5]` (0 % a 500 % por periodo), paso `0,0005`
   - tiempo / momento: `x ∈ [0, 600]` periodos, paso `0,25`
2. **Bisección** hasta `|b − a| < 1e−10` (garantiza convergencia).
3. **Pulido con Newton-Raphson** (derivada numérica central, `h = 1e−7`), máx. 60 iteraciones, tolerancia `|g(x)| < 1e−8`.
4. Si no hay cambio de signo → mensaje claro: *"Con los datos ingresados no existe una solución en el rango razonable. Revisa signos y montos."*

**Atajo de precisión:** para los casos de dos flujos (VP↔VF) se usa la fórmula cerrada del apartado 3.2 en lugar del iterativo — es exacta y más rápida.

- [ ] Implementar `resolverLineal(flujos, tf)` (Caso A)
- [ ] Implementar `bisección(g, a, b)` + `newton(g, x0)` (Caso B)
- [ ] Implementar detección automática de caso según la incógnita elegida
- [ ] Mensajes de error humanos para cada modo de fallo

### 3.5 Módulo M3 — Ganancia neta (capacidad **e**)

Además del valor total, siempre se reporta **únicamente los intereses**:

```
capitalMovido  = Σ  coef_k · valor_k        de los flujos de SALIDA (signo −1)
retornoMovido  = Σ  coef_k · valor_k        de los flujos de ENTRADA (signo +1)
gananciaNeta   = |retornoMovido| − |capitalMovido|
```

En el caso simple de dos flujos esto colapsa a `I = VF − VP` (compuesto/continuo) y a `I = VP·i·n` (simple), que se usa como verificación cruzada.

La tarjeta de resultado muestra tres cifras separadas:
`Valor total` · `Capital` · **`Ganancia neta (solo intereses)`**

- [ ] Implementar `gananciaNeta(flujos)`
- [ ] Verificación cruzada contra la fórmula cerrada; si difieren > 0,01 → advertencia en consola

### 3.6 Módulo M1 — Formato numérico y monedas (capacidad **g**)

**Salida:**

```js
new Intl.NumberFormat('es-CO', {minimumFractionDigits:2, maximumFractionDigits:2}).format(x)
// 7520000  →  "7.520.000,00"
```

**Entrada:** el usuario puede escribir `7.520.000,00`, `7520000,00` o `7520000.00`. El parser normaliza:
quitar espacios → quitar puntos de miles → cambiar coma decimal por punto → `parseFloat`.

**Monedas:** `COP` (`$`), `EUR` (`€`), `USD` (`US$`). Es solo **etiquetado y presentación**: todos los flujos de una misma simulación deben estar en la misma moneda. La herramienta **no convierte entre monedas** (igual que no convierte tasas). Aviso explícito en la UI.

Tasas: se ingresan en **porcentaje** (`3,5`) y se convierten internamente a decimal (`0,035`). Se muestran con 4 decimales.

- [ ] `formatearMoneda(x, moneda)` y `parsearNumero(str)`
- [ ] Formateo en vivo del input mientras se escribe (sin romper el cursor)
- [ ] Selector de moneda global de la simulación + aviso de no conversión

---

## 4. Sección 1 — Educación Financiera (con FSRS)

### 4.1 Contenido expositivo

Tres bloques (Simple / Compuesto / Continuo). Cada bloque, **en este orden**:

1. **La idea en una frase** — nivel niño de 5 años.
2. **¿Qué es?**
3. **¿Cómo funciona?**
4. **¿En qué casos se usa?** (compuesto: *¿por qué es diferente al simple?* · continuo: *¿en qué contextos se usa?* y *¿cuál es la diferencia frente al compuesto?*)
5. **Fórmula** — presentada en grande, con cada símbolo etiquetado debajo.
6. **Explicación sencilla** de la fórmula, término por término.
7. **Un ejemplo práctico resuelto**, paso a paso, con números colombianos reales.

Analogías guía (mantener el mismo hilo narrativo en los tres bloques para que se note la progresión):

- **Simple:** *"Prestas tu matera. Cada mes te dan una moneda por prestarla. Siempre la misma moneda, porque siempre es la misma matera."*
- **Compuesto:** *"Las monedas que te dan también se convierten en materas pequeñitas que empiezan a producir sus propias monedas. Cada mes tienes más materas."*
- **Continuo:** *"No esperas a fin de mes: la matera está produciendo monedas todo el tiempo, sin parar ni un segundo. Es el máximo posible de crecimiento con esa velocidad."*

Ejemplos resueltos sugeridos (ver §7 para los valores exactos verificados):

- Simple: $5.000.000 al 1,8 % mensual durante 2 años y 4 meses.
- Compuesto: $5.000.000 al 3 % bimestral durante 2 años y 4 meses.
- Continuo: $5.000.000 al 12 % anual continuo durante 2 años y 4 meses.

Los tres usan el mismo capital y el mismo plazo **a propósito**: al ponerlos lado a lado se ve la diferencia entre regímenes. Cerrar la sección con una tabla comparativa de los tres resultados y un mini-gráfico de barras (paleta verde).

- [ ] Redactar los 3 bloques completos
- [ ] Resolver y verificar los 3 ejemplos
- [ ] Tabla comparativa + gráfico de barras en SVG inline
- [ ] Botón "Probar este ejemplo en el simulador" que precarga los datos en la Sección 2

### 4.2 Repaso espaciado con FSRS (Free Spaced Repetition Scheduler)

Debajo del contenido expositivo va un **mazo de tarjetas** que el usuario repasa. Cada tarjeta se programa con **FSRS-4.5**.

**Estado por tarjeta:**

```js
{ id, S,          // stability (días)
      D,          // difficulty ∈ [1, 10]
      due,        // timestamp de próximo repaso
      lastReview, // timestamp
      reps, lapses, estado }   // estado: 'nueva' | 'aprendiendo' | 'repaso'
```

**Constantes:**

```
F = 19/81 ≈ 0,234568        DECAY = −0,5        R_objetivo = 0,90
w = [0.4872, 1.4003, 3.7145, 13.8206, 5.1618, 1.2298, 0.8975, 0.0310,
     1.6474, 0.1367, 1.0461, 2.1072, 0.0793, 0.3246, 1.5870, 0.2272, 2.8755]
```

**Fórmulas (G = calificación: 1 Otra vez · 2 Difícil · 3 Bien · 4 Fácil):**

```
Retrievability:      R(t, S) = (1 + F·t/S)^DECAY                 t = días desde el último repaso
Intervalo:           I(S)    = (S/F) · (R_objetivo^(1/DECAY) − 1)

Primera vez:
  S₀(G) = w[G−1]
  D₀(G) = w₄ − e^(w₅·(G−1)) + 1                                  clamp a [1, 10]

Repasos siguientes:
  ΔD  = −w₆ · (G − 3)
  D'  = w₇·D₀(4) + (1 − w₇)·(D + ΔD)                             clamp a [1, 10]

  Si G ≥ 2 (recordó):
    S' = S · (1 + e^(w₈) · (11 − D) · S^(−w₉) · (e^(w₁₀·(1−R)) − 1) · h · b)
         con h = w₁₅ si G = 2 (Difícil), si no 1
             b = w₁₆ si G = 4 (Fácil),   si no 1

  Si G = 1 (olvidó → lapse):
    S' = w₁₁ · D^(−w₁₂) · ((S + 1)^w₁₃ − 1) · e^(w₁₄·(1−R))
    S' = min(S', S)          // la estabilidad nunca sube tras un olvido
```

**Mazo mínimo (12 tarjetas, 4 por régimen):** definición · fórmula · caso de uso · una de cálculo mental. Ejemplos:

- *"¿Sobre qué capital se calculan los intereses en interés simple?"* → *"Siempre sobre el capital inicial."*
- *"¿Por qué el interés compuesto crece más rápido que el simple?"* → *"Porque los intereses ganados también generan intereses."*
- *"En interés continuo, ¿en qué unidad se mide siempre el tiempo?"* → *"En años."*
- *"Tasa bimestral y plazo de 2 años y 4 meses: ¿cuántos periodos?"* → *"14 bimestres."*

**UI del repaso:**

- Tarjeta grande centrada, fondo blanco, borde `--hueso`.
- Botón "Mostrar respuesta" → revela y aparecen 4 botones con la escala de verdes:
  `Otra vez` (`--verde-profundo`) · `Difícil` (`--verde-oscuro`) · `Bien` (`--verde-medio`) · `Fácil` (`--verde-claro`).
- Debajo de cada botón, el **intervalo que resultaría** ("en 3 días", "en 2 semanas") calculado en vivo con `I(S')`.
- Barra de progreso: tarjetas pendientes hoy / dominadas / totales.
- Botón "Reiniciar progreso" con confirmación.

**Persistencia:** `localStorage` bajo la clave `matefi.fsrs.v1`, serializando el mapa `{cardId: estado}`. Al cargar, se rehidrata y se filtran las tarjetas con `due ≤ ahora`.

- [ ] Implementar `fsrs.calificar(estado, G, ahora)` → nuevo estado
- [ ] Implementar `fsrs.previsualizarIntervalos(estado)` → 4 intervalos
- [ ] Escribir las 12 tarjetas
- [ ] Persistencia + rehidratación + reinicio
- [ ] Probar: calificar `Otra vez` nunca debe aumentar `S`

---

## 5. Sección 2 — Simulación / Cotización

### 5.1 Flujo de la interfaz (de arriba hacia abajo)

```
[1] Tipo de operación      ( ) Crédito      ( ) Inversión
[2] Tipo de interés        ( ) Simple  ( ) Compuesto  ( ) Continuo
[3] Moneda                 [ COP ▾ ]        (no hay conversión entre monedas)
[4] Periodicidad de la tasa[ Bimestral ▾ ]  → si "Cada N meses": [ N = 5 ]
                            (deshabilitado y fijo en ANUAL si el interés es continuo)
[5] Variable a calcular    [ Valor futuro ▾ ]
        · Valor futuro
        · Valor presente
        · Tasa de interés
        · Tiempo
        · Valor de un flujo desconocido
        · Momento en que ocurre un flujo
[6] Modo                   ( ) Simple: un VP y un VF
                           ( ) Ecuación de valor: varios flujos
[7] Datos según lo anterior
[8] Tabla de flujos (modo ecuación de valor)
[9] [ Calcular ]  [ Limpiar ]  [ Cargar ejemplo ]
[10] RESULTADO
```

**Tabla de flujos:**

| Etiqueta | Tipo | Coef. | Valor | Años | Meses | ¿Incógnita? | 🗑 |
|---|---|---|---|---|---|---|---|
| Desembolso 1 | Pago | 1,4 | *(X)* | 0 | 6 | Valor | |
| Desembolso 2 | Pago | 1 | *(X)* | 1 | 0 | — | |
| Préstamo | Recibo | 1 | 10.000.000 | 0 | 0 | — | |

Debajo, un **timeline horizontal en SVG** con la paleta verde: flechas hacia arriba = ingresos, hacia abajo = egresos, altura proporcional al monto, y una línea vertical punteada marcando la fecha focal. Esto es lo que convierte la herramienta de "calculadora" en "herramienta educativa".

### 5.2 Tarjeta de resultado

```
┌──────────────────────────────────────────────────────┐
│  VALOR FUTURO                                        │
│  $ 7.562.948,62  COP                                 │
│                                                      │
│  Capital                         $ 5.000.000,00      │
│  Ganancia neta (solo intereses)  $ 2.562.948,62      │
│                                                      │
│  ── Cómo se calculó ──────────────────────────────   │
│  Régimen: interés compuesto                          │
│  Tasa: 3,0000 % bimestral                            │
│  Plazo: 2 años y 4 meses = 28 meses = 14 bimestres   │
│  VF = 5.000.000 × (1 + 0,03)^14                      │
│  VF = 5.000.000 × 1,5125897                          │
│  ─────────────────────────────────────────────────   │
│  [ Ver desarrollo paso a paso ▾ ]                    │
└──────────────────────────────────────────────────────┘
```

Cuando la incógnita es **el tiempo** o **el momento de un flujo**, el resultado se devuelve en **años y meses** además de en número de periodos (capacidad **b**).

- [ ] Construir el formulario completo con habilitado/deshabilitado condicional
- [ ] Tabla de flujos dinámica
- [ ] Timeline SVG
- [ ] Tarjeta de resultado + desarrollo paso a paso desplegable
- [ ] Validaciones: una sola incógnita, montos > 0, `n > 0`, `N` entero ≥ 1
- [ ] Botón "Cargar ejemplo" con los 10 casos de §7

---

## 6. Sección 3 — Interacciones

Sección **visible dentro de la herramienta** (no es anexo del PDF), accesible desde el sidebar.

**Encabezado obligatorio:**

> **Herramienta de IA utilizada:** Claude (Anthropic), a través de Cowork.
> Toda la herramienta —lógica financiera, algoritmo FSRS, interfaz y contenido educativo— se construyó mediante las interacciones documentadas a continuación.

**Formato de cada tarjeta de interacción** (mínimo 3, se irán agregando):

```
┌──────────────────────────────────────────────────────┐
│  Interacción #01                    [ Planeación ]   │
│  Fecha: 19/08/2026                                   │
│  Herramienta: Claude (Anthropic) — Cowork            │
│                                                      │
│  PROMPT                                              │
│  ┌────────────────────────────────────────────────┐  │
│  │ (texto literal del prompt, monoespaciado)      │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  QUÉ SE OBTUVO                                       │
│  (resumen breve del resultado)                       │
│                                                      │
│  QUÉ SE AJUSTÓ MANUALMENTE                           │
│  (decisiones propias del equipo sobre la respuesta)  │
└──────────────────────────────────────────────────────┘
```

Categorías (badge en verde): `Planeación` · `Lógica financiera` · `Algoritmo FSRS` · `Interfaz` · `Contenido educativo` · `Depuración`.

**Interacción #01 — ya registrada.** El prompt inicial completo del proyecto (el enunciado del caso del banco + las capacidades mínimas a–h + la paleta + la instrucción de usar FSRS + la instrucción de generar `roadmap.md`) va literal en la primera tarjeta, categoría `Planeación`.

Las interacciones #02 en adelante se agregan **a medida que el usuario las indique**. Los prompts viven en `contenido.js` como un arreglo `INTERACCIONES = [...]`, así agregar una es añadir un objeto al arreglo y nada más.

- [ ] Maquetar la sección y el componente tarjeta
- [ ] Cargar la Interacción #01 con el prompt literal
- [ ] Dejar el arreglo `INTERACCIONES` listo para crecer
- [ ] Declarar visiblemente la herramienta de IA usada

---

## 7. Batería de casos de prueba (valores verificados)

Ejecutar todos antes de dar el prototipo por terminado. Resultados calculados y verificados el 2026-08-19.

| # | Escenario | Entradas | Resultado esperado |
|---|---|---|---|
| C1 | Simple, VF | VP 5.000.000 · 1,8 % mensual · 2a 4m (28 meses) | **VF = 7.520.000,00** · Interés = 2.520.000,00 |
| C2 | Compuesto, VF | VP 5.000.000 · 3 % bimestral · 2a 4m (**14 bimestres**) | **VF = 7.562.948,62** · Interés = 2.562.948,62 |
| C3 | Continuo, VF | VP 5.000.000 · 12 % anual continuo · 2a 4m (2,3333 años) | **VF = 6.615.649,06** · Interés = 1.615.649,06 |
| C4 | Compuesto, "cada N meses" | VP 3.000.000 · 8 % **quintumestral** (N = 5) · 3a 4m (40 meses = **8 periodos**) | **VF = 5.552.790,63** |
| C5 | Compuesto, tasa | VP 2.000.000 → VF 3.000.000 en 10 periodos | **i = 4,1380 % por periodo** |
| C6 | Compuesto, tiempo | VP 2.000.000 → VF 3.000.000 · 2,5 % mensual | **n = 16,4205 meses ≈ 1 año y 4,42 meses** |
| C7 | Ecuación de valor + **coeficiente** | Préstamo 10.000.000 hoy, 2,0 % mensual compuesto, se paga con 2 cuotas en el mes 6 y el mes 12; **la primera es 1,4 veces la segunda**; focal t = 0 | **Segunda cuota X = 4.922.100,11** · **Primera = 1,4·X = 6.890.940,15** |
| C8 | Momento de un flujo | 10.000.000 hoy se cancela con un único pago de 13.000.000 · 2 % mensual compuesto | **t = 13,2490 meses ≈ 1 año y 1,25 meses** |
| C9 | Simple, tasa | VP 4.000.000 → VF 5.200.000 en 10 periodos | **i = 3,0000 % por periodo** |
| C10 | Continuo, tiempo | VP 8.000.000 → VF 12.000.000 · 15 % anual continuo | **t = 2,7031 años ≈ 2 años y 8,44 meses** |

Pruebas adicionales no numéricas:

- [ ] Formato: `1234567.891` se muestra como `1.234.567,89`; el usuario escribe `1.234.567,89` y se parsea a `1234567.891`
- [ ] Al elegir interés continuo, el selector de periodicidad queda bloqueado en anual
- [ ] Dos incógnitas marcadas → error claro, no cálculo silencioso
- [ ] FSRS: calificar `Otra vez` nunca aumenta `S`; recargar la página conserva el progreso
- [ ] La app funciona **sin conexión a internet** (verdadero test de "autocontenido")
- [ ] Ningún color fuera de la paleta (buscar hex en el CSS y contrastarlos con la tabla de §1.1)

---

## 8. Trazabilidad de las capacidades mínimas obligatorias

| Cap. | Requisito | Dónde se cumple | Fase |
|---|---|---|---|
| **a** | Periodicidad configurable (mensual → anual) + "cada N meses" | M2 `tiempo.js` · selector [4] de la Sección 2 | §3.1 · §5.1 |
| **b** | Tiempo en años y meses ↔ periodos; si la incógnita es el tiempo, salida en años y meses | M2 `periodosAAñosMeses()` · tarjeta de resultado | §3.1 · §5.2 |
| **c** | Varios flujos en momentos distintos (ecuación de valor) | M3 `ecuacionDeValor()` · tabla de flujos + timeline | §3.3 · §5.1 |
| **d** | Incógnita seleccionable entre 6 opciones | M4 `solver.js` (Caso A lineal + Caso B iterativo) · selector [5] | §3.4 · §5.1 |
| **e** | Reportar la ganancia neta (solo intereses) además del total | M3 `gananciaNeta()` · tarjeta de resultado | §3.5 · §5.2 |
| **f** | Coeficiente configurable por flujo ("1,4 veces el otro") | campo `coef` de cada flujo · término `B` del solver lineal | §3.3 · §3.4 |
| **g** | Tres monedas: COP, EUR, USD | M1 `fmt.js` · selector [3] de la Sección 2 | §3.6 · §5.1 |
| **h** | Los tres regímenes; en continuo el tiempo siempre en años | M3 `factor()` · bloqueo de periodicidad en continuo | §3.2 · §3.1 |

---

## 9. Fases de ejecución (checklist maestro)

### Fase 0 — Preparación

- [ ] Crear `index.html` con el esqueleto de los 8 módulos
- [ ] Definir tokens CSS de la paleta y las utilidades base
- [ ] Montar sidebar + router por hash entre las 3 secciones

### Fase 1 — Núcleo financiero (sin UI, probado en consola)

- [ ] M1 `fmt.js` — formato/parsing colombiano y monedas
- [ ] M2 `tiempo.js` — conversión años+meses ↔ periodos, incluido "cada N meses"
- [ ] M3 `finanzas.js` — `factor()`, `ecuacionDeValor()`, `gananciaNeta()`
- [ ] M4 `solver.js` — Caso A lineal y Caso B bisección + Newton
- [ ] Pasar C1–C10 de §7 desde la consola del navegador

### Fase 2 — Sección de Simulación

- [ ] Formulario completo con habilitados condicionales
- [ ] Tabla de flujos dinámica con coeficiente y marcaje de incógnita
- [ ] Timeline SVG
- [ ] Tarjeta de resultado + desarrollo paso a paso
- [ ] Validaciones y mensajes de error humanos
- [ ] Botón "Cargar ejemplo" con los 10 casos

### Fase 3 — Sección de Educación Financiera

- [ ] Redactar los 3 bloques con el estándar "niño de 5 años"
- [ ] Resolver y verificar los 3 ejemplos prácticos
- [ ] Tabla comparativa + gráfico de barras SVG
- [ ] Enlace "Probar este ejemplo en el simulador"

### Fase 4 — FSRS

- [ ] Implementar el algoritmo completo (§4.2)
- [ ] Escribir las 12 tarjetas del mazo
- [ ] UI de repaso con previsualización de intervalos
- [ ] Persistencia en `localStorage` + reinicio de progreso

### Fase 5 — Sección de Interacciones

- [ ] Componente tarjeta + arreglo `INTERACCIONES`
- [ ] Cargar la Interacción #01 (prompt inicial literal)
- [ ] Declaración visible de la herramienta de IA (Claude — Cowork)
- [ ] Ir agregando #02, #03… conforme el usuario lo indique (mínimo 3 al entregar)

### Fase 6 — Pulido y verificación final

- [ ] Auditoría de paleta: cero colores fuera de la tabla de §1.1
- [ ] Responsive a 1366×768, 1920×1080 y móvil
- [ ] Prueba sin internet
- [ ] Repasar la batería completa de §7
- [ ] Revisar la tabla de trazabilidad de §8 capacidad por capacidad
- [ ] Entregar `index.html` en `C:\Proyectos\APLICACIONMATEFI\`

---

## 10. Riesgos conocidos y cómo evitarlos

| Riesgo | Mitigación |
|---|---|
| Convertir tasas entre periodicidades "por ayudar" | Prohibido por diseño. No escribir ninguna función que eleve una tasa a `m` o `12/m`. Revisión explícita en Fase 6. |
| Confundir conteo de periodos con conversión de tasa | Caso C4 (quintumestral) y C2 (bimestral) están en la batería justo para detectarlo. |
| Interés continuo con tiempo en meses | El módulo M2 fuerza `m = 12` cuando el régimen es continuo; la UI lo bloquea y lo explica. |
| Formato numérico invertido (`,` miles / `.` decimales) | Usar `Intl.NumberFormat('es-CO')` y probar explícitamente el parsing inverso. |
| El solver no converge y devuelve `NaN` sin avisar | Bracketing obligatorio antes de iterar + mensaje explícito cuando no hay cambio de signo. |
| Explicaciones educativas demasiado técnicas | Regla: cada concepto abre con una frase de analogía concreta antes de cualquier símbolo. |
| Menos de 3 interacciones documentadas al entregar | Registrar cada prompt en `docs\interacciones.md` en el momento, no al final. |
| Interés simple: capitalizar y descontar no son inversos | La fecha focal es un dato explícito del ejercicio en régimen simple; documentarlo en la UI. |
