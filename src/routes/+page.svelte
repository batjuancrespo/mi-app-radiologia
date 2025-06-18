<script lang="ts">
  import { onMount } from 'svelte';
  import { pipeline, type Pipeline } from '@xenova/transformers';

  // --- Constantes de Configuración ---
  const CHUNK_LENGTH_SECONDS = 5; // Procesar audio cada 5 segundos. Puedes ajustarlo a 3 para más reactividad.

  // --- Variables de Estado de la UI ---
  let status: string = 'Iniciando...';
  let transcript: string = '';
  let isRecording: boolean = false;
  let modelLoadingProgress: number = 0;

  // --- Variables Técnicas ---
  let transcriber: Pipeline | null = null;
  let mediaRecorder: MediaRecorder | null = null;
  
  // --- Variables para Streaming ---
  let audioQueue: Blob[] = [];
  let isProcessing: boolean = false;

  // --- Lógica Principal ---

  onMount(async () => {
    try {
      status = 'Cargando modelo optimizado de IA...';
      transcriber = await pipeline('automatic-speech-recognition', 'Xenova/whisper-small', {
        quantized: true,
        progress_callback: (p: any) => { 
          if (p.status === 'progress') {
            modelLoadingProgress = p.progress; 
            status = `Cargando modelo... ${Math.round(p.progress)}%`;
          }
        },
      });
      status = 'Modelo cargado. Listo para dictar.';
    } catch (error) {
      console.error('Error al cargar el modelo:', error);
      status = 'Error al cargar el modelo. Revisa la consola (F12).';
    }
  });

  async function processQueue() {
    if (isProcessing || audioQueue.length === 0) {
      return;
    }

    isProcessing = true;
    
    // Si no estamos grabando, actualizamos el estado para el último procesamiento
    if (!isRecording) {
      status = `Procesando ${audioQueue.length} fragmento(s) restante(s)...`;
    }
    
    const audioBlob = audioQueue.shift()!;

    try {
      const arrayBuffer = await audioBlob.arrayBuffer();
      const audioContext = new AudioContext({ sampleRate: 16000 });
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
      const audioData = audioBuffer.getChannelData(0);

      const output = await transcriber(audioData, {
        language: 'spanish',
        task: 'transcribe',
      });

      if (output && typeof output.text === 'string') {
        const processedText = processTranscript(output.text);
        transcript += processedText;
      }

    } catch (error) {
      console.error("Error procesando fragmento de audio:", error);
    } finally {
      isProcessing = false;
      // Vuelve a llamar a la función para procesar el siguiente trozo si lo hay
      processQueue();
    }
  }

  async function startRecording() {
    if (!transcriber) return;
    
    audioQueue = [];
    isRecording = true;
    status = 'Grabando... El texto aparecerá en tiempo real.';

    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      mediaRecorder = new MediaRecorder(stream);

      mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          audioQueue.push(event.data);
          processQueue();
        }
      };

      mediaRecorder.onstop = () => {
        isRecording = false;
        // Al parar, la cola se seguirá procesando. Cuando termine, actualizamos el estado.
        const checkQueueEmpty = setInterval(() => {
          if (audioQueue.length === 0 && !isProcessing) {
            status = 'Listo para dictar.';
            clearInterval(checkQueueEmpty);
          }
        }, 500);
      };

      mediaRecorder.start(CHUNK_LENGTH_SECONDS * 1000);

    } catch (error) {
      console.error('Error al iniciar la grabación:', error);
      status = 'Error al acceder al micrófono.';
      isRecording = false;
    }
  }

  function stopRecording() {
    if (mediaRecorder && mediaRecorder.state === 'recording') {
      mediaRecorder.stop();
    }
  }

  function handleRecordClick() {
    if (isRecording) {
      stopRecording();
    } else {
      startRecording();
    }
  }

  function processTranscript(text: string): string {
    const trimmedText = text.trim();
    if (!trimmedText) return '';

    const replacements: { [key: string]: string } = {
        'coma': ',',
        'punto': '.',
        'punto y seguido': '.',
        'punto y aparte': '.\n\n',
        'dos puntos': ':',
        'punto y coma': ';',
        'abrir interrogación': '¿',
        'cerrar interrogación': '?',
        'abrir exclamación': '¡',
        'cerrar exclamación': '!',
        'nueva línea': '\n',
        'nuevo párrafo': '\n\n',
    };

    let processedText = ` ${trimmedText.toLowerCase()} `;
    
    for (const command in replacements) {
        const regex = new RegExp(` ${command} `, 'gi');
        processedText = processedText.replace(regex, `${replacements[command]} `);
    }
    
    processedText = processedText.trim();
    processedText = processedText.replace(/\s+([,.?!:;])/g, '$1');
    processedText = processedText.replace(/([.!?]\s*|^)(\w)/g, (match, p1, p2) => p1 + p2.toUpperCase());
    
    return ' ' + processedText;
  }

  function copyToClipboard() {
    if (!transcript) return;
    navigator.clipboard.writeText(transcript.trim()).then(() => {
      const originalStatus = status;
      status = '¡Informe copiado al portapapeles!';
      setTimeout(() => { status = originalStatus }, 2000);
    });
  }

  function clearText() {
    transcript = '';
    status = 'Área de texto limpiada. Listo para dictar.';
  }

  function downloadText() {
    if (!transcript) return;
    const blob = new Blob([transcript.trim()], { type: 'text/plain;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `informe-${new Date().toISOString().split('T')[0]}.txt`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    status = 'Informe descargado como .txt';
  }
</script>

<!-- PARTE VISUAL (HTML) -->
<svelte:head>
  <title>Asistente de Dictado Radiológico</title>
</svelte:head>

<div class="container">
  <header>
    <svg xmlns="http://www.w3.org/2000/svg" width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"></path><path d="M19 10v2a7 7 0 0 1-14 0v-2"></path><line x1="12" y1="19" x2="12" y2="22"></line></svg>
    <h1>Asistente de Dictado Radiológico</h1>
  </header>

  <div class="status-bar">
    <div class="status-text">{status}</div>
    {#if status.startsWith('Cargando modelo')}
      <progress value={modelLoadingProgress} max="100"></progress>
    {/if}
  </div>

  <div class="main-controls">
    <button on:click={handleRecordClick} class:recording={isRecording} disabled={!transcriber}>
      {isRecording ? 'Detener Dictado' : 'Iniciar Dictado'}
    </button>
  </div>
  
  <textarea
    bind:value={transcript}
    placeholder="Aquí aparecerá la transcripción de su dictado en tiempo real..."
    rows="20"
  ></textarea>

  <div class="action-buttons">
    <button on:click={copyToClipboard} disabled={!transcript}>Copiar Informe</button>
    <button on:click={downloadText} disabled={!transcript}>Descargar (.txt)</button>
    <button on:click={clearText} class="danger" disabled={!transcript}>Limpiar Todo</button>
  </div>

  <footer>
    <p>Motor de IA: OpenAI Whisper (ejecución 100% local en su navegador)</p>
  </footer>
</div>

<!-- ESTILOS (CSS) -->
<style>
  :root {
    --primary-color: #007bff;
    --primary-hover: #0056b3;
    --danger-color: #dc3545;
    --danger-hover: #c82333;
    --light-gray: #f8f9fa;
    --gray: #6c757d;
    --border-color: #dee2e6;
  }

  .container {
    max-width: 900px;
    margin: 0 auto;
    padding: 2rem;
    font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    color: #333;
  }

  header {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1.5rem;
    color: var(--primary-color);
  }

  header h1 {
    font-size: 2rem;
    margin: 0;
  }

  .status-bar {
    background-color: var(--light-gray);
    border-radius: 8px;
    padding: 0.75rem 1rem;
    margin-bottom: 1.5rem;
    border: 1px solid var(--border-color);
  }
  
  .status-text {
    font-weight: 500;
  }

  progress {
    width: 100%;
    margin-top: 0.5rem;
  }

  .main-controls button {
    width: 100%;
    padding: 1.25rem;
    font-size: 1.5rem;
    font-weight: bold;
    cursor: pointer;
    border: none;
    border-radius: 8px;
    background-color: var(--primary-color);
    color: white;
    transition: background-color 0.2s;
  }

  .main-controls button:hover:not(:disabled) {
    background-color: var(--primary-hover);
  }
  
  .main-controls button.recording {
    background-color: var(--danger-color);
  }
  
  .main-controls button.recording:hover {
    background-color: var(--danger-hover);
  }

  .main-controls button:disabled {
    background-color: #a0c3e6;
    cursor: not-allowed;
  }

  textarea {
    width: 100%;
    margin-top: 1.5rem;
    padding: 1rem;
    font-size: 1.1rem;
    line-height: 1.6;
    border: 1px solid var(--border-color);
    border-radius: 8px;
    box-sizing: border-box;
    resize: vertical;
  }

  .action-buttons {
    display: flex;
    gap: 1rem;
    margin-top: 1rem;
  }

  .action-buttons button {
    flex-grow: 1;
    padding: 0.75rem;
    font-size: 1rem;
    cursor: pointer;
    border-radius: 8px;
    border: 1px solid var(--primary-color);
    background-color: white;
    color: var(--primary-color);
    transition: all 0.2s;
  }

  .action-buttons button:hover:not(:disabled) {
    background-color: var(--primary-color);
    color: white;
  }

  .action-buttons button.danger {
    border-color: var(--danger-color);
    color: var(--danger-color);
  }

  .action-buttons button.danger:hover:not(:disabled) {
    background-color: var(--danger-color);
    color: white;
  }

  .action-buttons button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  footer {
    text-align: center;
    margin-top: 3rem;
    color: var(--gray);
    font-size: 0.9rem;
  }
</style>