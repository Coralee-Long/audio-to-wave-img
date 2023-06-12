<script>

  let mediaRecorder;
  let audioChunks = [];
  let audioBlob;
  let audioUrl;
  let isRecording = false;
  let canvas;
  let canvasContext;
  let startMarker = 0;
  let endMarker = 500; // canvas width
  let draggingMarker = null;
  let enableTrimming = false;

  async function startRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    mediaRecorder = new MediaRecorder(stream);
    mediaRecorder.ondataavailable = event => {
      audioChunks.push(event.data);
    };
    mediaRecorder.onstop = () => {
      audioBlob = new Blob(audioChunks, { type: "audio/wav" });
      audioUrl = URL.createObjectURL(audioBlob);
    };
    mediaRecorder.start();
    isRecording = true;
  }

  function stopRecording() {
    if (mediaRecorder) {
      mediaRecorder.stop();
      isRecording = false;
    }
  }

  function deleteRecording() {
    audioChunks = [];
    audioBlob = null;
    audioUrl = null;
  }

async function createWaveImage() {
    const audioContext = new (window.AudioContext || window.webkitAudioContext)();
    const arrayBuffer = await audioBlob.arrayBuffer();
    const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
    
    // Trim the silence from the audio
    const trimmedAudioBuffer = trimSilence(audioContext, audioBuffer);
    
    // Draw the waveform
    drawWaveform(trimmedAudioBuffer);
}

function trimSilence(audioContext, audioBuffer) {
    const data = audioBuffer.getChannelData(0);
    const threshold = 0.01; // Silence threshold (you may need to adjust this)
    const sampleRate = audioContext.sampleRate;

    let startIndex = null;
    let endIndex = null;

    // Find the start index
    for (let i = 0; i < data.length; i++) {
        if (Math.abs(data[i]) > threshold) {
            startIndex = i;
            break;
        }
    }

    // Find the end index
    for (let i = data.length - 1; i >= 0; i--) {
        if (Math.abs(data[i]) > threshold) {
            endIndex = i;
            break;
        }
    }

    if (startIndex === null || endIndex === null) {
        return audioBuffer; // return original buffer if silence not found
    }

    // Create a new buffer with the trimmed audio data
    const trimmedLength = endIndex - startIndex + 1;
    const trimmedAudioBuffer = audioContext.createBuffer(1, trimmedLength, sampleRate);
    trimmedAudioBuffer.copyToChannel(data.subarray(startIndex, endIndex + 1), 0, 0);

    return trimmedAudioBuffer;
}



  function drawWaveform(trimmedAudioBuffer) {
    canvasContext = canvas.getContext('2d');
    const data = trimmedAudioBuffer.getChannelData(0);
    const step = Math.ceil(data.length / canvas.width);
    const amp = canvas.height / 2;
    canvasContext.fillStyle = "black";
    canvasContext.clearRect(0, 0, canvas.width, canvas.height);
    for (let i = 0; i < canvas.width; i++) {
      let min = 1.0;
      let max = -1.0;
      for (let j = 0; j < step; j++) {
        const datum = data[(i * step) + j];
        if (datum < min) min = datum;
        if (datum > max) max = datum;
      }
      canvasContext.fillRect(i, (1 + min) * amp, 1, Math.max(1, (max - min) * amp));
    }
  }
</script>

<style>
 canvas {
  border: 1px solid black;
  margin-top: 10px;
}
</style>

<div>
  {#if isRecording}
    <button on:click={stopRecording}>Stop Recording</button>
  {:else if audioUrl}
    <audio controls let:src={audioUrl}></audio>
    <button on:click={deleteRecording}>Delete Recording</button>
    <button on:click={createWaveImage}>Submit</button>
    <button on:click={toggleTrimming}>
      {#if enableTrimming}Disable Trimming{else}Trim Audio Clip{/if}
    </button>
  {/if}
  <button on:click={startRecording} disabled={isRecording}>Start Recording</button>
</div>
<canvas
  width="500"
  height="200"
  bind:this={canvas}
  on:mousedown={mouseDown}
  on:mousemove={mouseMove}
  on:mouseup={mouseUp}
  on:mouseleave={mouseUp}
></canvas>



