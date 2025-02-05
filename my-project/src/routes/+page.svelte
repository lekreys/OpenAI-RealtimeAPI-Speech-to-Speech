<!-- src/routes/+page.svelte -->
<script>
  import { onMount } from 'svelte';
  import Toastify from 'toastify-js';
  import "toastify-js/src/toastify.css";

  const clientId = crypto.randomUUID();
  let ws;
  let messages = [];
  let isConnected = false;
  let isRecording = false;
  let mediaRecorder;
  let audioContext;
  let audioChunks = [];
  let audioQueue = [];
  let isPlaying = false;

  onMount(() => {
    connectWebSocket();
    return () => {
      if (audioContext) {
        audioContext.close();
      }
      if (mediaRecorder) {
        mediaRecorder.stop();
      }
      if (ws) {
        ws.close();
      }
    };
  });

  async function processAudioQueue() {
    if (isPlaying || audioQueue.length === 0) return;
    
    try {
      while (audioQueue.length > 0) {
        isPlaying = true;
        const audioData = audioQueue[0];
        await playAudioFromBase64(audioData);
        audioQueue.shift(); // Hapus audio yang sudah diputar
      }
    } catch (error) {
      console.error('Error processing audio queue:', error);
      showToast("Error playing audio queue", "error");
    } finally {
      isPlaying = false;
    }
  }

  function showToast(message, type = "success") {
    Toastify({
      text: message,
      duration: 3000,
      gravity: "top",
      position: "right",
      style: {
        background: type === "error" ? "#EF4444" : "#10B981",
        borderRadius: "10px",
        padding: "12px 24px",
        boxShadow: "0 4px 6px -1px rgba(0, 0, 0, 0.1)",
      }
    }).showToast();
  }

  async function playAudioFromBase64(base64Data) {
    return new Promise((resolve, reject) => {
      try {
        if (!base64Data) {
          throw new Error('Invalid audio data');
        }

        const binaryString = atob(base64Data);
        const len = binaryString.length;
        const bytes = new Int16Array(len/2);

        for (let i = 0; i < len; i+=2) {
          bytes[i/2] = (binaryString.charCodeAt(i) & 0xff) | ((binaryString.charCodeAt(i+1) & 0xff) << 8);
        }

        if (!audioContext || audioContext.state === 'closed') {
          audioContext = new (window.AudioContext || window.webkitAudioContext)({
            sampleRate: 24000,
          });
        }

        if (audioContext.state === 'suspended') {
          audioContext.resume();
        }

        const audioBuffer = audioContext.createBuffer(1, bytes.length, 24000);
        const channelData = audioBuffer.getChannelData(0);

        for (let i = 0; i < bytes.length; i++) {
          channelData[i] = bytes[i] / 32768.0;
        }

        const source = audioContext.createBufferSource();
        source.buffer = audioBuffer;
        source.connect(audioContext.destination);
        
        source.onended = () => {
          resolve();
        };

        source.onerror = (error) => {
          reject(error);
        };

        source.start(0);
        showToast("Playing audio response");
      } catch (error) {
        console.error('Error playing audio:', error);
        showToast("Error playing audio response", "error");
        reject(error);
      }
    });
  }

  async function copyToClipboard(text, message) {
    try {
      const textToCopy = message?.fullData ? 
        JSON.stringify({
          type: "audio",
          data: message.fullData
        }, null, 2) : 
        text;

      await navigator.clipboard.writeText(textToCopy);
      showToast("Copied to clipboard!");
    } catch (err) {
      console.error('Failed to copy text: ', err);
      showToast("Failed to copy text", "error");
    }
  }

  async function startRecording() {
    try {
      audioChunks = [];
      const stream = await navigator.mediaDevices.getUserMedia({ 
        audio: {
          channelCount: 1,
          sampleRate: 24000,
          sampleSize: 16
        } 
      });
      
      audioContext = new AudioContext({
        sampleRate: 24000,
      });
      
      const source = audioContext.createMediaStreamSource(stream);
      const processor = audioContext.createScriptProcessor(512, 1, 1);
      
      processor.onaudioprocess = (e) => {
        const inputData = e.inputBuffer.getChannelData(0);
        const pcmData = new Int16Array(inputData.length);
        
        for (let i = 0; i < inputData.length; i++) {
          const s = Math.max(-1, Math.min(1, inputData[i]));
          pcmData[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
        }
        
        audioChunks.push(pcmData);
      };

      source.connect(processor);
      processor.connect(audioContext.destination);

      isRecording = true;
      window.stream = stream;
      window.source = source;
      window.processor = processor;

      showToast("Recording started");
    } catch (error) {
      console.error('Recording error:', error);
      showToast("Error starting recording", "error");
    }
  }

  async function stopRecording() {
    if (!isRecording) return;

    try {
      isRecording = false;

      if (window.stream) {
        window.stream.getTracks().forEach(track => track.stop());
      }
      if (window.source) {
        window.source.disconnect();
      }
      if (window.processor) {
        window.processor.disconnect();
      }

      const totalLength = audioChunks.reduce((acc, chunk) => acc + chunk.length, 0);
      const combinedPcmData = new Int16Array(totalLength);
      let offset = 0;
      
      for (const chunk of audioChunks) {
        combinedPcmData.set(chunk, offset);
        offset += chunk.length;
      }
      
      const base64Data = btoa(String.fromCharCode(...new Uint8Array(combinedPcmData.buffer)));
      
      if (ws?.readyState === WebSocket.OPEN) {
        const message = {
          type: "audio",
          data: base64Data
        };
        
        ws.send(JSON.stringify(message));
        addMessage('User', {
          type: "audio",
          data: base64Data,
        });
        
        showToast("Audio sent to server");
      }

    } catch (error) {
      console.error('Error processing audio:', error);
      showToast("Error processing audio", "error");
    }
  }

  function connectWebSocket() {
    ws = new WebSocket(`ws://localhost:8000/ws/${clientId}`);

    ws.onopen = () => {
      isConnected = true;
      addMessage('System', 'Connected to server');
    };

    ws.onclose = () => {
      isConnected = false;
      addMessage('System', 'Disconnected from server');
    };

    ws.onmessage = async (event) => {
      try {
        let response;
        if (event.data instanceof Blob) {
          const arrayBuffer = await event.data.arrayBuffer();
          const base64Data = btoa(
            new Uint8Array(arrayBuffer)
              .reduce((data, byte) => data + String.fromCharCode(byte), '')
          );
          response = {
            type: "audio",
            data: base64Data
          };
        } else {
          response = JSON.parse(event.data);
        }

        addMessage('OpenAI', response);

        if (response.type === "audio" && response.data) {
          audioQueue.push(response.data);
          processAudioQueue();
        }
      } catch (error) {
        console.error('Error processing message:', error);
        addMessage('Error', 'Failed to process message');
      }
    };

    ws.onerror = (error) => {
      addMessage('Error', 'WebSocket error occurred');
      console.error('WebSocket error:', error);
    };
  }

  function addMessage(sender, content) {
    if (content && typeof content === 'object' && content.data) {
      const displayContent = {
        ...content,
        data: content.data.substring(0, 100) + '...'
      };
      messages = [...messages, { 
        sender, 
        content: displayContent, 
        timestamp: new Date().toLocaleTimeString(),
        fullData: content.data
      }];
    } else {
      messages = [...messages, { 
        sender, 
        content, 
        timestamp: new Date().toLocaleTimeString()
      }];
    }
  }
</script>

<svelte:head>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
</svelte:head>

<div class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-50 p-4">
  <div class="container mx-auto max-w-4xl">
    <!-- Header Section -->
    <div class="bg-white rounded-2xl shadow-xl p-6 mb-6 transform hover:scale-[1.01] transition-all">
      <h1 class="text-3xl font-bold bg-gradient-to-r from-blue-600 to-indigo-600 bg-clip-text text-transparent">
        Openai Realtime API 
      </h1>
      
      <!-- Status Badge -->
      <div class="mt-4 inline-flex items-center px-4 py-2 rounded-full bg-gray-100">
        <div class={`w-2 h-2 rounded-full mr-2 ${isConnected ? 'bg-green-500' : 'bg-red-500'} animate-pulse`}></div>
        <span class="text-sm font-medium text-gray-700">
          {isConnected ? 'Connected' : 'Disconnected'}
        </span>
      </div>
    </div>

    <!-- Controls Section -->
    <div class="bg-white rounded-2xl shadow-xl p-6 mb-6">
      <div class="flex space-x-4">
        <button 
          on:click={startRecording}
          disabled={isRecording || !isConnected}
          class="flex-1 px-6 py-3 bg-gradient-to-r from-blue-500 to-blue-600 text-white rounded-xl hover:from-blue-600 hover:to-blue-700 disabled:opacity-50 disabled:cursor-not-allowed transform hover:scale-105 transition-all duration-200 shadow-lg hover:shadow-xl flex items-center justify-center gap-3"
        >
          <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M7 4a3 3 0 016 0v4a3 3 0 11-6 0V4zm4 10.93A7.001 7.001 0 0017 8a1 1 0 10-2 0A5 5 0 015 8a1 1 0 00-2 0 7.001 7.001 0 006 6.93V17H6a1 1 0 100 2h8a1 1 0 100-2h-3v-2.07z" clip-rule="evenodd" />
          </svg>
          Start Recording
        </button>
        
        <button 
          on:click={stopRecording}
          disabled={!isRecording || !isConnected}
          class="flex-1 px-6 py-3 bg-gradient-to-r from-red-500 to-red-600 text-white rounded-xl hover:from-red-600 hover:to-red-700 disabled:opacity-50 disabled:cursor-not-allowed transform hover:scale-105 transition-all duration-200 shadow-lg hover:shadow-xl flex items-center justify-center gap-3"
        >
          <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8 7a1 1 0 00-1 1v4a1 1 0 002 0V8a1 1 0 00-1-1zm4 0a1 1 0 00-1 1v4a1 1 0 002 0V8a1 1 0 00-1-1z" clip-rule="evenodd" />
          </svg>
          Stop Recording
        </button>
      </div>

      {#if isRecording}
        <div class="mt-4 flex items-center justify-center bg-red-50 p-3 rounded-xl">
          <div class="w-3 h-3 bg-red-500 rounded-full animate-pulse mr-3"></div>
          <span class="text-red-600 font-medium">Recording in progress...</span>
        </div>
      {/if}
    </div>

    <!-- Messages Section -->
    <div class="bg-white rounded-2xl shadow-xl overflow-hidden">
      <div class="p-4 border-b bg-gray-50">
        <h2 class="text-lg font-semibold text-gray-700">Conversation History</h2>
      </div>
      
      <div class="p-4 h-[500px] overflow-y-auto space-y-4" style="scrollbar-width: thin;">
        {#each messages as message}
          <div class="transform hover:scale-[1.01] transition-all duration-200">
            <div class="flex items-start gap-3 p-4 rounded-xl {message.sender === 'User' ? 'bg-blue-50' : message.sender === 'System' ? 'bg-gray-50' : 'bg-green-50'} border border-opacity-50 {message.sender === 'User' ? 'border-blue-200' : message.sender === 'System' ? 'border-gray-200' : 'border-green-200'}">
              <!-- Avatar -->
              <div class="w-8 h-8 rounded-full flex items-center justify-center {message.sender === 'User' ? 'bg-blue-500' : message.sender === 'System' ? 'bg-gray-500' : 'bg-green-500'} text-white font-bold text-sm">
                {message.sender[0]}
              </div>

              <div class="flex-1">
                <div class="flex justify-between items-start mb-2">
                  <div>
                    <span class="font-medium text-gray-900">{message.sender}</span>
                    <span class="ml-2 text-sm text-gray-500">{message.timestamp}</span>
                  </div>
                  
                  {#if message.sender === 'OpenAI' || (message.sender === 'User' && message.content.data)}
                    <button
                      on:click={() => copyToClipboard(JSON.stringify(message.content, null, 2), message)}
                      class="p-2 hover:bg-white rounded-lg transition-all duration-200 group"
                    >
                    .
                      <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 text-gray-400 group-hover:text-gray-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
                      </svg>
                    </button>
                  {/if}
                </div>

                <div class="text-gray-700 font-mono text-sm bg-white bg-opacity-50 rounded-lg p-3">
                  {#if typeof message.content === 'object'}
                    <pre class="whitespace-pre-wrap">
                      {JSON.stringify(message.content, null, 2)}
                    </pre>
                  {:else}
                    <pre class="whitespace-pre-wrap">
                      {message.content}
                    </pre>
                  {/if}
                </div>
              </div>
            </div>
          </div>
        {/each}
      </div>
    </div>
  </div>

  <!-- Status Indicators -->
  {#if isPlaying}
    <div class="fixed bottom-6 right-6 bg-gradient-to-r from-green-500 to-green-600 text-white px-6 py-4 rounded-xl shadow-xl flex items-center gap-3 animate-fade-in-up">
      <div class="relative">
        <div class="w-3 h-3 bg-white rounded-full animate-ping absolute"></div>
        <div class="w-3 h-3 bg-white rounded-full relative"></div>
      </div>
      <span class="font-medium">
        Playing audio response... ({audioQueue.length} remaining)
      </span>
    </div>
  {/if}
</div>

<style>
  :global(body) {
    font-family: 'Inter', sans-serif;
  }

  .animate-fade-in-up {
    animation: fadeInUp 0.3s ease-out;
  }

  @keyframes fadeInUp {
    from {
      opacity: 0;
      transform: translateY(20px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  div::-webkit-scrollbar {
    width: 8px;
  }

  div::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-radius: 4px;
  }

  div::-webkit-scrollbar-thumb {
    background: #888;
    border-radius: 4px;
  }

  div::-webkit-scrollbar-thumb:hover {
    background: #666;
  }

  button {
    outline: none !important;
  }

  .shadow-hover {
    transition: all 0.2s ease;
  }

  .shadow-hover:hover {
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  }
</style>