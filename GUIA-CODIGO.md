# Guía del código — PronunciaCoach TM

Guía para preparar la exposición. La meta es que **los tres** puedan explicar el
núcleo de la app (el Modo Personal con Teachable Machine) y responder preguntas
sobre el código. No hace falta memorizar todo `index.html`: concéntrense en la
sección 2.

> Consejo: en el editor, busquen `function nombre(` para saltar a cada función.

---

## 1. Cómo está organizado

Toda la app está en **un solo archivo**, [`index.html`](index.html), en tres partes:

| Parte | Qué contiene |
|---|---|
| `<head>` | Las librerías por CDN: **TensorFlow.js 1.3.1** y **speech-commands 0.4.0** |
| `<style>` | El diseño (colores, animaciones, versión para celular) |
| `<body>` | Los diálogos (consentimiento, fin de sesión), el panel para elegir modo y el panel de práctica |
| `<script>` | Toda la lógica, en JavaScript sin frameworks |

Variables globales importantes (al inicio del `<script>`):

| Variable | Qué guarda |
|---|---|
| `recognizer` | El modelo de Teachable Machine cargado |
| `classLabels` | Las palabras que el modelo sabe reconocer (sin "Background Noise") |
| `currentTarget` | La palabra que el usuario debe decir ahora |
| `session` | Todo lo que se exporta en el JSON: intentos, modo, consentimiento, resumen |
| `attempted`, `hits`, `streak` | Contadores que se ven en pantalla |
| `DETECTION_COOLDOWN_MS` | 1500 ms de espera entre intentos (debounce) |

---

## 2. Lo que deben dominar: el recorrido del Modo Personal

```
Clic en "Cargar modelo"
 └─ startMode('custom')
     ├─ askConsent('custom')     → aviso de privacidad (Ley 1581); si cancela, no sigue
     ├─ loadCustomModel()        → descarga el modelo de Teachable Machine
     ├─ setupMicVolume()         → abre el micrófono (para el visualizador)
     ├─ beginSession()           → reinicia contadores y elige la primera palabra
     └─ startListening()         → el modelo empieza a escuchar

Cada ~0,5 segundos (lo hace la librería):
 └─ callback de recognizer.listen(...)
     └─ handlePredictions(preds)
         └─ processAttempt(best)  → ¿acertó? feedback, contadores, guarda el intento

Clic en "Terminar sesión"
 └─ openEndDialog() → finishSession() → exportReport() y/o downloadPdfReport() → stopSession()
```

### 2.1 `loadCustomModel()`: cargar el modelo de Teachable Machine

```js
recognizer = speechCommands.create('BROWSER_FFT', undefined,
  base + 'model.json', base + 'metadata.json');
await recognizer.ensureModelLoaded();
labelsFull = recognizer.wordLabels();
classLabels = labelsFull.filter(l =>
  l !== '_background_noise_' && l !== 'Background Noise' && l !== '_unknown_');
if (classLabels.length < 2) throw new Error('El modelo debe tener al menos 2 palabras');
```

- **`model.json`**: la arquitectura de la red. **`metadata.json`**: los nombres de las clases (las palabras). Los pesos (`weights.bin`) se descargan solos.
- **`'BROWSER_FFT'`**: el navegador calcula la transformada de Fourier (con Web Audio API) y arma el **espectrograma**, igual que cuando se entrenó en Teachable Machine. Por eso usamos `speech-commands` y no TensorFlow.js solo: si no, habría que programar a mano ese preprocesamiento.
- Se quita **Background Noise** de la lista porque no es una palabra para practicar.
- Con menos de 2 palabras no hay nada que comparar, así que se lanza un error.

### 2.2 `startListening()`: el modelo escucha

```js
recognizer.listen(result => {
  const scores = Array.from(result.scores);
  const preds = labelsFull.map((label, i) => ({ label, score: scores[i] }))
    .filter(p => classLabels.includes(p.label));
  const sum = preds.reduce((a, p) => a + p.score, 0) || 1;
  preds.forEach(p => p.score = p.score / sum);
  handlePredictions(preds);
}, {
  includeSpectrogram: false,
  probabilityThreshold: 0.60,
  invokeCallbackOnNoiseAndUnknown: false,
  overlapFactor: 0.50
});
```

- `result.scores` trae **una probabilidad por clase**, en el mismo orden que `wordLabels()`. Salen de la capa **softmax** y suman 1.
- **`probabilityThreshold: 0.60`**: la librería solo nos avisa si la clase ganadora tiene al menos 60 %. Así se evitan detecciones dudosas.
- **`invokeCallbackOnNoiseAndUnknown: false`**: si lo que más se parece es ruido de fondo, no nos avisa.
- **`overlapFactor: 0.50`**: las ventanas de ~1 s se solapan a la mitad, así que hay una predicción cada ~0,5 s y ninguna palabra queda cortada entre dos ventanas.
- **Renormalización**: se quita el ruido de fondo y se reparten los porcentajes solo entre las palabras, para que las barras sumen 100 %.

### 2.3 `handlePredictions(preds)`: filtrar antes de contar

```js
if (isPaused || endDialogOpen) return;
preds.sort((a, b) => b.score - a.score);
renderPredictionBars(preds);
if (now - lastDetectedAt < DETECTION_COOLDOWN_MS) return;   // debounce 1,5 s
const best = preds[0];
if (!best || best.score < 0.4) return;
lastDetectedAt = now;
processAttempt(best);
```

- En pausa o con el diálogo de salida abierto no se cuenta nada.
- Siempre se actualizan las barras, aunque no se cuente un intento.
- **Debounce**: una palabra dura más que una ventana, así que el modelo la detecta varias veces seguidas. Esperar 1,5 s evita contar el mismo intento dos o tres veces.

### 2.4 `processAttempt(best)`: ¿acertó?

```js
const correct = best.label === currentTarget;
// actualiza hits, streak, bestStreak y muestra el feedback
session.practice_attempts.push({
  target: currentTarget, detected: best.label,
  confidence: +best.score.toFixed(3), correct, t: Date.now()
});
if (correct) setTimeout(() => nextWord(), 1500);
```

- Compara la palabra detectada con la objetivo. Si no coinciden, muestra *"Sonó más como X"*: ese es el feedback útil para los pares mínimos.
- Cada intento se guarda **sin audio**: solo la palabra pedida, la detectada, la confianza, si acertó y la hora.

### 2.5 Nota y reportes

- **`buildSummary()`**: totales, precisión, resultado por palabra, las 3 palabras más difíciles, mejor racha y una recomendación. Es el `summary` del JSON.
- **`gradeFromAccuracy(acc)`**: **nota = precisión × 5**, redondeada a un decimal (escala colombiana 0.0–5.0, se aprueba con 3.0). `gradeLevel(g)` da la etiqueta: Excelente (≥ 4.5), Muy bien (≥ 4.0), Bien (≥ 3.5), Aceptable (≥ 3.0), Por mejorar.
- **`topConfusions()`**: cuenta los pares "pedida → detectada" de los errores. Sirve para *"ship sonó como sheep (2 veces)"*.
- **`exportReport()`**: convierte `session` a JSON y lo descarga (un `Blob` y un enlace temporal).
- **`buildPdfReport()`**: arma el PDF con **jsPDF**, que se descarga de internet solo al entrar al Modo Personal (`loadJsPdf()`).
- **`openEndDialog()` → `finishSession()`**: al terminar, **nada se descarga sin preguntar**. El usuario elige PDF, JSON o salir sin descargar.

---

## 3. Los otros modos (deben poder explicarlos en una frase)

- **Modo Rápido** (`loadExpressModel`, `startSRLoop`): usa `webkitSpeechRecognition` del navegador en inglés (`en-US`). Cada resultado de texto se pasa a `handlePredictions` como si fuera una predicción. Cuando termina de escuchar, se reinicia solo (`onend`). **En Chrome el audio va a los servidores de Google**, por eso el aviso de consentimiento lo dice.
- **Modo Demo** (`loadDemoMode`, `startDemoLoop`): cada 2 s inventa probabilidades que favorecen a la palabra objetivo el 70 % de las veces. **No usa el micrófono ni el modelo.** Es solo un respaldo para mostrar el flujo.

---

## 4. Piezas de apoyo (no son el núcleo, pero pueden preguntar)

| Función | Qué hace |
|---|---|
| `askConsent()` / `resolveConsent()` | Diálogo de consentimiento antes de usar el micrófono. Se guarda en `session.consent` (fecha, versión del aviso y si el audio sale del dispositivo). |
| `loadTargetWordData()` | Muestra la palabra. En Modo Rápido y Demo trae fonética, significado y audio de la Dictionary API. En Modo Personal no consulta el diccionario. |
| `playReferenceAudio()` / `speakWithTTS()` | Reproduce el audio nativo. Si no hay, usa la voz del navegador (TTS): es el *fallback*. |
| `setupMicVolume()` / `startVisualizer()` | `AnalyserNode` de Web Audio (`fftSize` 256) y un `canvas` con barras que reaccionan a la voz. Solo visual: no afecta el reconocimiento. |
| `togglePause()` | Detiene y reanuda la escucha según el modo (`stopListening`, `abort`, o detener el temporizador de la demo). |
| `stopSession()` | Libera el micrófono, detiene el modelo y vuelve a la pantalla de modos. |

La presentación web ([`presentacion.html`](presentacion.html)) es aparte: diapositivas de lado a lado, barra lateral y un glosario que subraya los términos difíciles. No forma parte del MVP.

---

## 5. Preguntas probables y respuestas cortas

**¿Dónde se usa Teachable Machine en el código?**
En `loadCustomModel()`, que carga el modelo exportado, y en `startListening()`, que lo ejecuta en tiempo real.

**¿Por qué `speech-commands` y no solo TensorFlow.js?**
Porque el modelo de Teachable Machine Audio espera el espectrograma calculado exactamente como lo hace `speech-commands` (`BROWSER_FFT`). La librería además captura el micrófono y corta el audio en ventanas.

**¿Por qué TensorFlow.js 1.3.1 y speech-commands 0.4.0, si hay versiones más nuevas?**
Son las que usa el código oficial que entrega Teachable Machine al exportar. Así el modelo carga sin problemas de compatibilidad.

**¿Para qué sirve el umbral de 0.60?**
Para no contar predicciones dudosas. Más alto da menos errores pero detecta menos; más bajo detecta más pero con más falsos positivos.

**¿Qué pasa con el ruido de fondo?**
Teachable Machine obliga a entrenar la clase "Background Noise". Con `invokeCallbackOnNoiseAndUnknown: false` la app ignora los momentos en que el modelo cree que no estás hablando.

**¿Cómo se calcula la nota?**
Precisión × 5. Ejemplo: 9 aciertos de 14 intentos = 64 % → 3.2 / 5.0 (Aceptable).

**¿Qué se guarda? ¿Se guarda audio?**
Nunca audio. El JSON guarda palabras pedidas y detectadas, confianza, aciertos, horas, el resumen y el registro del consentimiento.

**¿Qué pasa sin internet?**
Para empezar hace falta red: TensorFlow.js, el modelo y jsPDF se descargan. Una vez cargado, el Modo Personal reconoce sin red. El audio nativo (Dictionary API) sí necesita red; si falla, se usa la voz del navegador.

**¿Por qué todo en un solo archivo HTML?**
Alcance de MVP: no hay paso de compilación ni servidor, se publica gratis en GitHub Pages y cualquiera lo ejecuta con un servidor local.

---

## 6. Ejercicio para estudiar (15 minutos)

1. Sirvan la app localmente (`python3 -m http.server 8000`) y ábranla en Chrome con **DevTools** (F12).
2. Entren al **Modo Demo**. En la pestaña **Sources**, busquen `processAttempt` y pongan un *breakpoint*.
3. Cuando se detenga, revisen `best`, `currentTarget` y `session.practice_attempts` en la consola.
4. Terminen la sesión, descarguen el JSON y ubiquen cada campo en el código (`processAttempt`, `buildSummary`, `beginSession`).
5. Con el modelo real: repitan en **Modo Personal** y miren en la pestaña **Network** que ninguna petición suba audio.
