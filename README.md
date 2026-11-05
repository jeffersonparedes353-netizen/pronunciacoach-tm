# PronunciaCoach TM

Entrenador de pronunciación en inglés en tiempo real que usa un modelo
entrenado por el propio usuario en **Google Teachable Machine (Audio
Project)**. 100% client-side, sin servidor, sin que el audio salga del
navegador.

> Proyecto académico — Programación Avanzada
> Institución Universitaria Pascual Bravo · 2026

---

## Demo

- **App live:** [https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/](https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/) (activar GitHub Pages — ver abajo)
- **Modo demo** incluido: no necesita micrófono ni modelo, útil para la exposición si falla el equipo.

---

## ¿Qué problema resuelve?

Los hispanohablantes aprendiendo inglés practican pronunciación sin saber
si lo están haciendo bien. Las apps genéricas (Duolingo y similares) marcan
"correcto" casi todo con reconocimiento de voz general, sin discriminar los
sonidos realmente difíciles para hispanohablantes (ship/sheep, think/sink,
very/berry, road/load...).

**PronunciaCoach TM** te deja entrenar tu propio modelo con los pares
mínimos que más se te complican, usando tu propia voz. Después, en tiempo
real, te dice qué palabra escuchó. Si dijiste "ship" pero el modelo oyó
"sheep" — ese es el feedback de oro que te muestra dónde está fallando
tu boca.

---

## Características

- Carga modelos TM Audio desde URL pública (TensorFlow.js + speech-commands).
- Visualización del nivel de micrófono en vivo.
- **Modo práctica**: la app te propone una palabra, tú la dices, recibes feedback inmediato con guía fonética.
- **Modo libre**: habla lo que quieras, ve qué está detectando el modelo.
- Puntaje, racha, precisión global, palabras más difíciles.
- Exportación de reporte JSON con historial completo + recomendaciones personalizadas.
- **Modo demo** con predicciones simuladas (sin modelo, sin micrófono).
- Un solo archivo HTML, sin dependencias de build.

---

## Cómo usar

### 1. Entrenar tu modelo (una vez, ~45 min)

Ver guía completa en [`docs/guion-exposicion.md`](docs/guion-exposicion.md).
Resumen:

1. Ve a [teachablemachine.withgoogle.com](https://teachablemachine.withgoogle.com) → New Project → **Audio Project**.
2. Graba 20 segundos para la clase **Background Noise** que viene por defecto.
3. Crea 3–6 clases con nombres de las palabras en inglés (`ship`, `sheep`, `think`, `sink`...).
4. Graba **mínimo 30 muestras por palabra en al menos 3 tomas separadas** (importante — evita data leakage temporal).
5. Click **Train Model** → espera 1–2 min.
6. **Export Model** → tab **Tensorflow.js** → **Upload (shareable link)** → **Upload my model** → copia la URL.

### 2. Usar el entrenador

1. Abre `index.html` localmente o visita la URL de GitHub Pages.
2. Pega la URL de tu modelo en el campo.
3. Click **Iniciar entrenamiento** y acepta el permiso del micrófono.
4. En **Modo práctica**, di la palabra que se te muestre en pantalla.
5. Al terminar, click **Exportar JSON** para guardar el reporte.

---

## Arquitectura

```
Teachable Machine (entrenamiento, una vez)
    └── Transfer learning sobre MobileNet adaptado para audio
            └── Export TensorFlow.js
                    └── URL pública (model.json + weights.bin + metadata.json)

Navegador del usuario (ejecución)
    ├── index.html carga TF.js + speech-commands desde CDN
    ├── getUserMedia({audio: true}) → stream de micrófono
    ├── speech-commands.listen() → ventanas de ~1s con BROWSER_FFT
    │     ├── Short-Time Fourier Transform
    │     ├── Modelo predict() → vector de probabilidades por clase
    │     └── Callback con los scores
    ├── Lógica: debounce 1.2s + threshold 0.6
    ├── Modo práctica: compara detección con palabra objetivo
    └── Export JSON con historial + resumen
```

**Stack:** HTML + CSS + JavaScript vanilla · TensorFlow.js 1.3 · `@tensorflow-models/speech-commands` 0.4 · BROWSER_FFT · Web Audio API.

---

## Privacidad

- **Audio nunca sale del dispositivo.** El stream vive en el `AudioContext` del navegador.
- Solo viaja por red la descarga inicial del modelo desde el CDN de Teachable Machine.
- El JSON del reporte contiene únicamente etiquetas de palabra, probabilidades y timestamps — **nunca audio**.
- Verificable abriendo **DevTools → Network** durante el uso: después de cargar, cero peticiones.

Compatible con la **Ley 1581 de 2012 (Habeas Data)** colombiana por diseño:
no hay tratamiento de dato biométrico (voz) en el sentido legal porque nada
se transmite ni almacena externamente.

---

## Limitaciones

- Precisión depende de la calidad del entrenamiento (cantidad y diversidad de muestras).
- No evalúa pronunciación "nativa" — solo si tu producción es suficientemente distinguible entre los pares entrenados.
- Vocabulario limitado a las palabras que entrenaste (3–6 típicamente).
- Ruido de fondo alto reduce la precisión.
- Algunos navegadores suspenden Web Audio si la pestaña pierde foco.

---

## Mejoras futuras

- Persistir progreso en IndexedDB (ver evolución semanal por palabra).
- Visualización del espectrograma en vivo para que el usuario vea las diferencias.
- Modo "grabar nativo" para comparar visualmente tu espectrograma con una referencia.
- Baterías de sonidos predefinidas (vocales largas/cortas, fricativas, etc.).
- Convertir a PWA instalable.
- Integración con flashcards de vocabulario para practicar las palabras del día.

---

## Estructura del repositorio

```
pronunciacoach-tm/
├── index.html                    MVP completo (HTML + CSS + JS en un solo archivo)
├── README.md                     Este archivo
├── LICENSE                       MIT
├── .gitignore
└── docs/
    └── guion-exposicion.md       Guion completo + respuestas a preguntas del docente
```

---

## Activar GitHub Pages

Para que `index.html` quede servido como URL pública:

1. En GitHub, ve a **Settings** del repositorio.
2. Barra lateral izquierda → **Pages**.
3. **Source:** Deploy from a branch → **Branch:** `main` / `(root)` → **Save**.
4. En 1–2 minutos estará disponible en `https://jeffersonparedes353-netizen.github.io/pronunciacoach-tm/`.

---

## Licencia

MIT — ver [LICENSE](LICENSE).

## Autor

**Jefferson Paredes** — Ingeniería de Software, Pascual Bravo (2026)
