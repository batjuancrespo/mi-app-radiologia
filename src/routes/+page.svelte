<script lang="ts">
  import { onMount } from 'svelte';
  // Importamos directamente de la librería correcta
  import { pipeline, type Pipeline } from '@xenova/transformers';

  let status: string = 'Iniciando... Por favor, espera.';
  let transcript: string = '';
  let isRecording: boolean = false;
  
  // La variable para el transcriptor ahora es de tipo Pipeline
  let transcriber: Pipeline;
  let mediaRecorder: MediaRecorder | null = null;
  let audioChunks: Blob[] = [];

  onMount(async () => {
    status = 'Cargando modelo de IA (esto puede tardar la primera vez)...';
    
    // Creamos un "pipeline" de transcripción. 
    // "automatic-speech-recognition" es la tarea.
    // "Xenova/whisper-base" es el modelo. Es multilingüe y tiene un buen balance de velocidad/precisión.
    // Puedes cambiarlo por "Xenova/whisper-tiny" para que sea más rápido.
    transcriber = await pipeline('automatic-speech-recognition', 'Xenova/whisper-base');
    
    status = 'Modelo cargado. Listo para grabar.';
  });

  async function startRecording() {
    try {
      // Pedimos acceso al micrófono
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      mediaRecorder = new MediaRecorder(stream);
      
      mediaRecorder.ondataavailable = (event) => {
        audioChunks.push(event.data);
      };

      mediaRecorder.onstop = async () => {
        status = 'Grabación detenida. Transcribiendo...';
        const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
        const audioUrl = URL.createObjectURL(audioBlob);
        
        // Transcribimos el audio usando la URL del Blob
        const output = await transcriber(audioUrl, {
          chunk_length_s: 30,
          stride_length_s: 5,
        });

        // 'output.text' puede no existir si la transcripción es nula
        if (output && typeof output.text === 'string') {
            // Añadimos la transcripción y un espacio para la siguiente frase
            transcript += output.text.trim() + ' '; 
        }

        status = 'Transcripción completada. Listo para una nueva grabación.';
        audioChunks = []; // Limpiamos los chunks para la próxima grabación
      };

      audioChunks = [];
      mediaRecorder.start();
      isRecording = true;
      status = 'Grabando... Habla ahora. Haz clic en "Detener" para finalizar.';

    } catch (error) {
      console.error('Error al acceder al micrófono:', error);
      status = 'Error: No se pudo acceder al micrófono. Revisa los permisos.';
    }
  }

  function stopRecording() {
    if (mediaRecorder) {
      mediaRecorder.stop();
      isRecording = false;
    }
  }

  function handleRecordClick() {
    if (isRecording) {
      stopRecording();
    } else {
      startRecording();
    }
  }
</script>

<!-- EL RESTO DEL CÓDIGO (main, style) SE MANTIENE EXACTAMENTE IGUAL -->
<main>
  <h1>Asistente de Informes Radiológicos</h1>
  <p class="status"><strong>Estado:</strong> {status}</p>
  
  <button on:click={handleRecordClick} class:recording={isRecording}>
    {isRecording ? 'Detener Grabación' : 'Iniciar Grabación'}
  </button>
  
  <textarea
    bind:value={transcript}
    placeholder="Aquí aparecerá la transcripción..."
    rows="15"
  ></textarea>
</main>

<style>
  main {
    max-width: 800px;
    margin: 2rem auto;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
  .status {
    padding: 0.5rem;
    background-color: #f0f0f0;
    border-radius: 5px;
  }
  button {
    padding: 1rem;
    font-size: 1.2rem;
    cursor: pointer;
    border: 2px solid #007bff;
    background-color: white;
    color: #007bff;
    border-radius: 5px;
    transition: all 0.2s;
  }
  button:hover {
    background-color: #007bff;
    color: white;
  }
  button.recording {
    background-color: #dc3545;
    border-color: #dc3545;
    color: white;
  }
  textarea {
    width: 100%;
    padding: 0.5rem;
    font-size: 1.1rem;
    border: 1px solid #ccc;
    border-radius: 5px;
    box-sizing: border-box; /* Asegura que el padding no desborde */
  }
</style>