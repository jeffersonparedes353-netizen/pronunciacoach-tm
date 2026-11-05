# PronunciaCoach TM — Guion de exposición
### Programación Avanzada · Pascual Bravo · Jefferson Paredes

---

## Resumen en una frase

Un entrenador de pronunciación en inglés 100% client-side que carga un modelo de **Google Teachable Machine (Audio)** entrenado por el propio usuario con pares mínimos (ship/sheep, think/sink, very/berry...), escucha por el micrófono y le dice si lo que pronunció efectivamente se parece más a la palabra objetivo o a la trampa.

---

## Problema real que resuelve

Los hispanohablantes aprendiendo inglés tienen un ciclo frustrante:
- Practican repitiendo palabras **sin saber si lo están haciendo bien**.
- Un profesor particular cuesta $40–80k/hora y no está disponible 24/7.
- Apps como Duolingo evalúan con reconocimiento de voz genérico que marca "bien" casi todo, sin discriminar los sonidos realmente difíciles.

**PronunciaCoach TM** permite entrenar un modelo con tu propia voz pronunciando bien y mal cada par mínimo, y después te dice en tiempo real cuál palabra sonó más cerca. Si dijiste "ship" pero el modelo escuchó "sheep", ese feedback es oro: te muestra dónde está fallando tu boca.

---

## Estructura sugerida (28–32 minutos)

1. **Problema (3 min)** — La trampa de las apps de inglés y el feedback genérico.
2. **Qué es Teachable Machine (4 min)** — Herramienta web gratuita de Google para entrenar clasificadores sin código, usando transfer learning.
3. **Por qué proyecto Audio en lugar de Image o Pose (2 min)** — Es el menos usado; el vocabulario de inglés es ideal para clasificación por espectrograma.
4. **Arquitectura de la solución (4 min)** — Diagrama + flujo de datos.
5. **Demo en vivo (12 min)** — Modo práctica + exportar JSON + modo libre.
6. **Decisiones técnicas y preguntas Requeridas (6 min)**.
7. **Limitaciones y futuro (3 min)**.
8. **Preguntas**.

---

## Respuestas a las preguntas REQUERIDAS

### 1. ¿Qué leyes aplican según la ley colombiana para el uso de datos personales?

- **Ley 1581 de 2012** (Habeas Data) — régimen general. La voz es un **dato biométrico** y entra en la categoría de **dato sensible** (Art. 5), por lo que requiere consentimiento expreso e informado.
- **Decreto 1377 de 2013** — reglamenta el consentimiento.
- **GDPR** (si hay usuarios UE) — Art. 9 clasifica biometría como categoría especial.

**Cómo cumple PronunciaCoach TM**: el audio **nunca sale del navegador del usuario**; el procesamiento ocurre completamente en el dispositivo con TensorFlow.js. No hay recolección ni tratamiento de dato biométrico en el sentido legal porque nada se transmite ni almacena en servidores.

### 2. Privacidad

- Procesamiento 100% local — el micrófono se conecta vía Web Audio API al modelo en memoria.
- El modelo se descarga una vez desde el CDN de Teachable Machine; después opera offline.
- El JSON del reporte solo contiene etiquetas de palabra, probabilidades y timestamps; **no incluye audio**.
- Cero cuentas, cero cookies, cero telemetría.
- **Privacy by Design** (Art. 25 GDPR).

### 3. Costo de infraestructura

| Componente | Costo mensual |
|------------|---------------|
| Entrenamiento (Teachable Machine) | $0 — gratis en Google |
| Hosting del modelo | $0 — Google Cloud Storage de TM |
| Hosting de la app | $0 — GitHub Pages o abrir `.html` local |
| Inferencia | $0 — corre en el dispositivo del usuario |
| **Total MVP** | **$0** |

Escalable a producto comercial: dominio ($12/año) + CDN opcional. El costo marginal por usuario adicional es cero.

### 4. Arquitectura

```
┌─────────────────────────────────────────────────────────┐
│  FASE 1 — ENTRENAMIENTO (una sola vez, por usuario)      │
│                                                           │
│  Teachable Machine (teachablemachine.withgoogle.com)     │
│    ├── Proyecto Audio                                     │
│    ├── Clase "ship"  ← tomas pronunciando correctamente  │
│    ├── Clase "sheep" ← tomas pronunciando correctamente  │
│    ├── Clase "think" ← ...                                │
│    └── (clase "Background Noise" se agrega sola)          │
│                        ↓                                  │
│  Transfer learning sobre ventanas de 1s + espectrograma  │
│    (Short-Time Fourier Transform → MobileNet audio)       │
│                        ↓                                  │
│  Export → TensorFlow.js  →  URL pública del modelo       │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  FASE 2 — INFERENCIA (navegador del usuario)              │
│                                                           │
│  index.html (PronunciaCoach TM)                           │
│    ├── Cargar modelo (TF.js + speech-commands lib)        │
│    ├── Capturar micrófono (getUserMedia + Web Audio API)  │
│    ├── speech-commands.listen() analiza ventanas de ~1s   │
│    ├── Para cada ventana → vector de probabilidades       │
│    ├── Modo práctica: compara con palabra objetivo        │
│    ├── Debounce 1.2s + threshold 0.6                       │
│    └── Exportar sesión como JSON                          │
└─────────────────────────────────────────────────────────┘
```

**Stack**: HTML + CSS + JS vanilla · TensorFlow.js 1.3 · `@tensorflow-models/speech-commands` 0.4 · BROWSER_FFT (Short-Time Fourier Transform en el navegador) · Web Audio API.

### 5. Problema / funcionalidad

- **Problema concreto**: feedback inexistente u objetivo pobre al practicar pronunciación de inglés.
- **Usuarios objetivo**: estudiantes de inglés hispanohablantes de nivel A2–B2, profesores particulares de inglés, instituciones.
- **Utilidad práctica**: minuto a minuto el usuario sabe si está produciendo sonidos suficientemente diferenciados; indicador objetivo del progreso.
- **Diferencial**: el modelo se entrena con la propia voz del usuario, no con hablantes nativos genéricos. Así detecta mejoras marginales que un sistema generalista no distinguiría.

### 6. Riesgos

| Riesgo | Mitigación |
|--------|------------|
| Modelo sesgado por entrenar con una sola voz | Ventaja aquí: el usuario entrena su propio modelo, es personal |
| Entrenar con pocas tomas → sobreajuste a la cabina/ruido de fondo | Guía: mínimo 30 tomas por palabra en al menos 2 sesiones |
| Reconocer "palabra incorrecta" no es lo mismo que "pronunciación realmente nativa" | Documentado como limitación; el objetivo es *producir distintivamente*, no imitar acento |
| Ruido de fondo intermitente confunde el modelo | Threshold 0.6 + clase "Background Noise" automática de TM |
| Falsos positivos repetidos desmotivan | Debounce de 1.2s y "modo libre" para experimentar sin puntaje |
| Uso para evaluar certificaciones | Advertencia explícita: es herramienta de práctica personal, no evaluación oficial |

### 7. ¿Qué consentimiento se necesita?

Antes de iniciar:

1. **Permiso del micrófono** — el navegador lo pide automáticamente vía `getUserMedia`.
2. **Aviso explícito en la app** — al cargar el modelo:
   > *"PronunciaCoach TM activa tu micrófono para analizar pronunciación. El audio no sale de tu dispositivo. Al continuar aceptas el procesamiento local de audio con fines de retroalimentación personal."*
3. **Finalidad explícita** — práctica de pronunciación personal; **nunca** para evaluar empleados, estudiantes sin su consentimiento, o emitir certificaciones.
4. **Revocación** — cerrar la pestaña detiene el procesamiento.

---

## Respuestas a las preguntas OPCIONALES

### ¿Correr modelos sin servidor?

Sí. TensorFlow.js ejecuta inferencia en el navegador con WebGL (GPU si está disponible) o WASM (CPU). La librería `@tensorflow-models/speech-commands` envuelve la captura de audio + FFT + predicción en una sola API. Ventajas: latencia local, privacidad, costo cero. Desventajas: la primera carga del modelo (~5 MB) consume bandwidth.

### ¿Cómo está compuesto el sistema?

Tres capas:
1. **Captura**: `navigator.mediaDevices.getUserMedia({audio: true})` → stream de audio → Web Audio API.
2. **Transformación + inferencia**: `speech-commands.listen()` toma ventanas de audio de ~1s, aplica Short-Time Fourier Transform, convierte a espectrograma, y lo alimenta a la red (MobileNet adaptada para audio).
3. **Lógica de negocio**: comparar con palabra objetivo, debounce, feedback, scoring, logging y export.

### ¿Qué hace el proyecto con el modelo entrenado?

Lo carga desde su URL pública (`model.json` + `metadata.json` + `weights.bin`), lo mantiene en memoria y le pasa cada ventana de audio capturada. El vector de probabilidades resultante alimenta las barras, el motor de práctica y el log de sesión.

### ¿Por qué la red está programada desde cero y no con TensorFlow.js?

**En este MVP sí usamos TensorFlow.js**, porque es la forma oficial de ejecutar modelos exportados de Teachable Machine. Programar la red desde cero (forward-pass manual) tendría solo valor didáctico; en producción perderíamos WebGL/WASM y compatibilidad. **Decisión**: TF.js por madurez, rendimiento y compatibilidad directa con TM.

### ¿Por qué se divide por "tomas" y no por cuadros al azar?

Porque los segmentos consecutivos de una misma toma de audio son casi idénticos (misma voz, mismo ambiente, misma respiración). Si mezclas al azar frames entre train y validation, el modelo "ve" en test lo que ya aprendió — es **data leakage temporal** y infla las métricas artificialmente.

Dividir por **tomas separadas** (cada una siendo una grabación distinta, con tu voz un poco diferente, distinto ángulo del mic, otro momento del día) garantiza que el modelo generalice a condiciones reales de uso.

### ¿El audio se envía a algún servidor?

**No.** El stream vive en Web Audio API dentro del navegador. Lo único que viaja es la descarga inicial del modelo desde el CDN de Teachable Machine. Verificable en **DevTools → Network**: después de cargar, cero peticiones.

### ¿Qué datos se guardan en el JSON del proyecto?

Dos niveles:

1. **JSON del modelo Teachable Machine** (lo generó TM):
   - `model.json`: arquitectura
   - `metadata.json`: nombres de clases (palabras), hiperparámetros
   - `weights.bin`: pesos de la red

2. **JSON del reporte de sesión** (lo genera PronunciaCoach TM):
   ```json
   {
     "session_id": "ses_1730000000000",
     "start": "2026-11-05T...",
     "end": "2026-11-05T...",
     "model_url": "https://teachablemachine.withgoogle.com/models/XXX/",
     "mode": "teachable_machine_audio",
     "classes": ["ship", "sheep", "think", "sink"],
     "practice_attempts": [
       {"target": "ship", "detected": "sheep", "confidence": 0.78, "correct": false, "t": ...}
     ],
     "free_detections": [{"word": "...", "confidence": ..., "t": ...}],
     "summary": {
       "total_attempts": 20,
       "correct": 14,
       "accuracy": 70.0,
       "by_word": {"ship": {"attempted": 5, "correct": 2}, ...},
       "hardest_words": [{"word": "ship", "accuracy": 0.4, "attempts": 5}],
       "recommendation": "Enfocate en la palabra ship (precision 40%)"
     }
   }
   ```
   **Nunca se guarda el audio capturado.**

### ¿Qué riesgos tiene un modelo entrenado con pocas personas?

- **Sesgo acústico**: entrenado con una sola voz, no reconoce bien a otros hablantes. Para esta app es una **ventaja** (modelo personal), no un defecto.
- **Sobreajuste al ambiente**: aprende el micrófono, la cabina, el ruido base, no solo la fonética.
- **Mitigación**: entrenar en diferentes sesiones, varios momentos del día, distintos niveles de ruido ambiente.
- Para una versión general multi-usuario se requeriría un dataset como *Common Voice* de Mozilla, con miles de hablantes diversos.

### ¿Qué pasaría si se usara para vigilar a personas en el trabajo?

Sería **gravemente problemático**:
- Violaría **finalidad** (Ley 1581): el modelo se entrenó para aprendizaje personal, no para evaluación o vigilancia.
- Sería **discriminatorio**: un modelo entrenado con pocos acentos marcaría negativamente a hablantes con acentos distintos, llevando a sesgo laboral.
- Grabar audio de empleados sin consentimiento informado es ilegal en Colombia.
- La asimetría empleador–empleado invalida el consentimiento.

Por eso el diseño:
- Es **personal** (cada usuario entrena su propio modelo).
- No tiene funcionalidad de reporte a terceros.
- El JSON lo descarga el propio usuario, no se sube automáticamente.

---

## Guía rápida de la demo en vivo

1. Abrir `https://teachablemachine.withgoogle.com` → New Project → **Audio Project**.
2. Crear 3–6 clases (por ejemplo: `ship`, `sheep`, `think`, `sink`).
3. Para la clase **Background Noise** (ya está) → grabar **20 segundos** de silencio con ruido ambiente.
4. Para cada palabra de inglés → grabar **mínimo 30 muestras**, cada una ~1 segundo, en **al menos 3 sesiones separadas** (tomas distintas).
5. **Train Model** → espera 1–2 minutos.
6. En **Preview**, probar cada palabra. Si confunde dos → agregar más muestras diferenciadas.
7. **Export Model** → **Tensorflow.js** → **Upload (shareable link)** → **Upload my model** → copiar URL.
8. Abrir PronunciaCoach TM → pegar URL → **Iniciar entrenamiento** → aceptar micrófono.
9. Modo práctica: la app muestra una palabra, el usuario la dice, recibe feedback inmediato.
10. Al terminar: **Exportar JSON** con historial y recomendaciones.
11. **Fallback**: si falla micrófono/red, usar **Modo demo** para mostrar flujo.

---

## Limitaciones honestas

- Depende mucho de la calidad y diversidad de las tomas de entrenamiento.
- No detecta pronunciación "nativa" — solo si tu producción es suficientemente distinguible entre pares.
- Vocabulario limitado a las palabras que entrenaste (típicamente 3–6).
- Ruido de fondo alto baja la precisión.
- La pestaña necesita estar activa (si pierde foco, Web Audio se suspende en algunos navegadores).

## Mejoras futuras

- Persistir progreso en IndexedDB (ver evolución semanal por palabra).
- Agregar visualización del espectrograma en vivo para que el usuario vea diferencias.
- Modo "grabar nativo" — comparar visualmente el espectrograma del usuario con una referencia.
- Modos por tipo de sonido (vocales largas vs cortas, consonantes fricativas, etc.).
- Convertir a PWA instalable.
- Integración con flashcards de vocabulario (Anki) para practicar las palabras del día.
