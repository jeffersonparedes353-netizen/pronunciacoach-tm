# PronunciaCoach TM

Entrenador de pronunciación en inglés para hispanohablantes. Usa un modelo de
**Google Teachable Machine (Audio Project)** entrenado con la voz del propio
usuario para distinguir pares mínimos difíciles (ship/sheep, think/sink...) y
dar retroalimentación en tiempo real. Es una app web estática: no tiene
servidor propio.

> Proyecto académico — Programación Avanzada
> Institución Universitaria Pascual Bravo · 2026

---

## Enlaces

- **App:** https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/
- **Presentación interactiva** (con glosario de términos): https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/presentacion.html
- **Presentación en PowerPoint:** [`presentacion-pronunciacoach.pptx`](presentacion-pronunciacoach.pptx)
- **Guion de la exposición** (respuestas a las preguntas del docente): [`guion-exposicion.md`](guion-exposicion.md)
- **Guía del código** para estudiar: [`GUIA-CODIGO.md`](GUIA-CODIGO.md)

### Modelo del equipo

> **Pendiente:** cuando el equipo entrene y publique su modelo, pegar aquí su URL
> y el enlace directo para probarlo sin entrenar uno propio:
>
> - URL del modelo: `https://teachablemachine.withgoogle.com/models/XXXXXXX/`
> - Enlace directo: `https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/?model=https://teachablemachine.withgoogle.com/models/XXXXXXX/`
> - Palabras que reconoce: `ship`, `sheep`, `think`, `sink`

---

## ¿Qué problema resuelve?

Los hispanohablantes que aprenden inglés practican pronunciación sin saber si
lo están haciendo bien. Las apps genéricas usan reconocimiento de voz que
"adivina" la palabra más probable y marca bien casi todo, sin distinguir los
sonidos que más nos cuestan (ship/sheep, think/sink, very/berry...).

**PronunciaCoach TM** te deja entrenar tu propio modelo con los pares mínimos
que más se te complican, usando tu voz. Después, en tiempo real, te dice qué
palabra escuchó. Si dijiste "ship" pero el modelo oyó "sheep", ese es el
feedback que te muestra qué sonido corregir.

**Usuarios:** estudiantes de inglés hispanohablantes (niveles A2–B2) y docentes
que quieran dar a sus estudiantes una herramienta de práctica personal.

---

## Modos

| Modo | Cómo reconoce la voz | Privacidad | Para qué |
|---|---|---|---|
| **Personal** (el núcleo del proyecto) | Modelo propio de Teachable Machine con TensorFlow.js | La voz **no sale del dispositivo** | Practicar los pares mínimos entrenados con tu voz |
| **Rápido** | Web Speech API del navegador | En Chrome la voz **se envía a Google** | Probar sin entrenar, cualquier palabra en inglés |
| **Demo** | Predicciones simuladas | No usa el micrófono | Mostrar el flujo si falla el equipo o la red |

Antes de activar el micrófono, la app muestra un **aviso de consentimiento**
que explica qué pasa con la voz en ese modo. La aceptación queda registrada en
el reporte JSON.

## Funcionalidades

- Palabra objetivo con fonética, significado y audio nativo (Dictionary API; si no hay audio, voz del navegador).
- Probabilidades por palabra en vivo y visualizador de la voz.
- Feedback inmediato: correcto, o "sonó más como X".
- Intentos, aciertos, precisión, racha y botón de pausa.
- Al terminar, la app **pregunta** antes de descargar:
  - **Calificación en PDF** (solo Modo Personal): nota de 0.0 a 5.0, resultado por palabra, confusiones y recomendaciones.
  - **Reporte JSON** con todos los intentos (nunca audio).

---

## Cómo usar

### 1. Entrenar el modelo (una vez, ~45 min)

En la app, **Modo Personal → "Ir a entrenar en Teachable Machine"**, o directamente
en [teachablemachine.withgoogle.com/train/audio](https://teachablemachine.withgoogle.com/train/audio):

1. Graba ~20 segundos para la clase **Background Noise** (ruido ambiente sin hablar).
2. Crea una clase por palabra, con el nombre **en inglés y en minúscula** (`ship`, `sheep`, `think`, `sink`).
3. Graba **mínimo 30 muestras por palabra en al menos 3 sesiones separadas** (evita *data leakage* temporal).
4. **Train Model** y revisa **Advanced → Under the hood** (precisión por clase y matriz de confusión).
5. **Export Model → Tensorflow.js → Upload (shareable link) → Upload my model** y copia la URL.

### 2. Usar la app

1. Abre la app (enlace de arriba) o sírvela localmente (ver abajo).
2. **Modo Personal** → pega la URL del modelo → **Cargar modelo**.
3. Acepta el aviso de consentimiento y el permiso del micrófono.
4. Di la palabra que aparece en pantalla.
5. **Terminar sesión** → elige si descargar la calificación en PDF y/o el JSON.

Atajo: `index.html?model=URL_DEL_MODELO` abre el Modo Personal con la URL ya pegada.

### Ejecutar localmente

El micrófono solo funciona en `https` o en `localhost` (no abriendo el archivo con doble clic):

```bash
git clone https://github.com/jeffersonparedes353-netizen/pronunciacoach-tm.git
cd pronunciacoach-tm
python3 -m http.server 8000
# abrir http://localhost:8000
```

Navegadores: Chrome o Edge recomendados. El Modo Rápido no funciona en Firefox.

---

## Arquitectura

```
Teachable Machine (entrenamiento, una vez)
    └── Transfer learning sobre el modelo Speech Commands de Google (CNN)
            └── Export TensorFlow.js → URL pública (model.json + weights.bin + metadata.json)

Navegador del usuario (Modo Personal)
    ├── Aviso de consentimiento → getUserMedia (micrófono)
    ├── speech-commands.listen() → ventanas de ~1 s con BROWSER_FFT
    │     ├── Short-Time Fourier Transform → espectrograma
    │     ├── Modelo predict() → probabilidad por clase (softmax)
    │     └── Callback con los scores (umbral 0.60)
    ├── Lógica: debounce 1.5 s, compara con la palabra objetivo
    └── Reportes: JSON y calificación en PDF (jsPDF)
```

**Stack:** HTML + CSS + JavaScript vanilla · TensorFlow.js 1.3.1 ·
`@tensorflow-models/speech-commands` 0.4.0 (las mismas versiones que usa el código
oficial de Teachable Machine) · Web Audio API · Web Speech API (Modo Rápido) ·
Dictionary API · jsPDF 2.5.1 (se carga solo al entrar al Modo Personal).

**Costo de infraestructura:** $0 (Teachable Machine, hosting del modelo y GitHub Pages son gratuitos; la inferencia corre en el dispositivo).

---

## Privacidad

- **Modo Personal:** el audio nunca sale del navegador. Solo se descarga el modelo
  desde Teachable Machine. Verificable en **DevTools → Network**: ninguna petición sube audio.
- **Modo Rápido:** la Web Speech API de Chrome envía la voz a Google para reconocerla
  (ese tráfico no aparece en DevTools). La app lo avisa y pide consentimiento.
- Los reportes (JSON y PDF) solo contienen palabras, probabilidades, aciertos y horas. **Nunca audio.**
- Marco legal: Ley 1581 de 2012 (Habeas Data) y Decreto 1377 de 2013. La voz es un dato
  biométrico (dato sensible), por eso el consentimiento es previo, expreso e informado.

---

## Limitaciones

- La precisión depende de la calidad y diversidad de las muestras de entrenamiento.
- No evalúa "acento nativo": mide si tu pronunciación es distinguible entre las palabras entrenadas.
- Vocabulario limitado a las palabras entrenadas (normalmente 3–6) en Modo Personal.
- Un modelo entrenado con una sola voz funciona mal con otras personas.
- El ruido de fondo alto reduce la precisión.
- Es una herramienta de práctica personal, no una evaluación oficial.

## Mejoras futuras

- Guardar el progreso en IndexedDB y ver la evolución semanal por palabra.
- Espectrograma visual: comparar tu voz con la nativa.
- Paquetes de sonidos predefinidos (vocales largas/cortas, fricativas, diptongos).
- Convertir en PWA instalable.

---

## Estructura del repositorio

```
pronunciacoach-tm/
├── index.html                          App completa (HTML + CSS + JS en un solo archivo)
├── presentacion.html                   Presentación interactiva con glosario
├── presentacion-pronunciacoach.pptx    Presentación en PowerPoint
├── guion-exposicion.md                 Guion y respuestas a las preguntas del docente
├── GUIA-CODIGO.md                      Guía del código para estudiar
├── README.md
└── LICENSE                             MIT
```

## Activar GitHub Pages

**Settings → Pages → Source:** Deploy from a branch → **Branch:** `main` / `(root)` → **Save**.

---

## Licencia

MIT — ver [LICENSE](LICENSE).

## Equipo

Jefferson Paredes · Jhon Deyby Higuita · Jorman Santiago Cataño — Ingeniería de Software, Pascual Bravo (2026)
