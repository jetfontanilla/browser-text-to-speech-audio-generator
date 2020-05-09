<script>
    const BASE_PITCH = 1.0;
    const BASE_RATE = 1.0;

    let text = "";
    let pitch = BASE_PITCH;
    let rate = BASE_RATE;
    let selectedVoiceIndex = 0;

    let isTextToSpeechEnabled = "speechSynthesis" in window && "SpeechSynthesisUtterance" in window;
    let isOutputStreamNotSupported = !navigator.mediaDevices || !navigator.mediaDevices.enumerateDevices;
    let speechSynth = window.speechSynthesis;
    let availableVoices = [];

    let fileName = "test.webm";
    let selectedDeviceId = "";
    let availableOutputDevices = [];
    let mimeType = MediaRecorder.isTypeSupported("audio/webm; codecs=opus")
            ? "audio/webm; codecs=opus"
            : "audio/ogg; codecs=opus";

    async function initialize() {
        if (isOutputStreamNotSupported) {
            return;
        }
        await navigator.mediaDevices.getUserMedia({audio: true});
        let devices = await navigator.mediaDevices.enumerateDevices();
        availableOutputDevices = devices.filter(device => device.kind === "audiooutput");
        if (!availableOutputDevices || availableOutputDevices.length === 0) {
            isOutputStreamNotSupported = true;
            return;
        }

        selectedDeviceId = availableOutputDevices[0].deviceId;
    }

    initialize();

    function reset() {
        text = "";
        pitch = BASE_PITCH;
        rate = BASE_RATE;
        selectedVoiceIndex = 0;
    }

    async function recordUtterance() {
        if (!selectedDeviceId) {
            isOutputStreamNotSupported = true;
        }

        let chunks = [];
        const constraints = {audio: {deviceId: {exact: selectedDeviceId}}};
        console.log(constraints);
        let stream = await navigator.mediaDevices.getUserMedia(constraints);

        let mediaRecorder = new MediaRecorder(stream, {
            mimeType: mimeType,
            bitsPerSecond: 256 * 8 * 1024
        });

        mediaRecorder.ondataavailable = (event) => {
            if (event.data.size > 0) {
                chunks.push(event.data);
            }
        };
        mediaRecorder.onstop = () => {
            let blob = new Blob(
                    chunks,
                    {type: mimeType}
            );
            saveData(blob);
        };
        mediaRecorder.onerror = (error) => {

        };

        let utterance = new SpeechSynthesisUtterance(text);
        utterance.voice = speechSynth.getVoices()[selectedVoiceIndex];
        utterance.pitch = pitch;
        utterance.rate = rate;
        utterance.onend = () => {
            mediaRecorder.stop();
            stream.getTracks().forEach(track => track.stop());
        };

        mediaRecorder.start();
        speechSynth.speak(utterance);
    }

    function saveData(blob) {
        let blobUrl = URL.createObjectURL(blob);
        let a = document.createElement("a");
        document.body.appendChild(a);
        a.style = "display: none";
        a.href = blobUrl;
        a.download = fileName;
        //a.click();

        console.log(blobUrl);
    }

    function utterText() {
        let utterance = new SpeechSynthesisUtterance(text);
        utterance.voice = speechSynth.getVoices()[selectedVoiceIndex];
        utterance.pitch = pitch;
        utterance.rate = rate;
        speechSynth.speak(utterance);
    }

    // window.speechSynthesis.getVoices() is an async task. invoking it immediately returns an empty array
    function initializeSpeechSynth() {
        setTimeout(() => {
            availableVoices = speechSynth.getVoices();
            if (availableVoices.length === 0) {
                if (retries > 0) {
                    retries -= 1;
                    initializeSpeechSynth();
                } else {
                    isLoading = false;
                    isTextToSpeechEnabled = false;
                }
            } else {
                isLoading = false;
            }
        }, 1000);
    }

    let retries = 3;
    let isLoading = isTextToSpeechEnabled;
    if (isTextToSpeechEnabled) {
        initializeSpeechSynth();
    }

</script>


<div class="container">
    {#if isLoading}
        <div class="section align-items-center justify-content-center">
            <div class="lds-grid">
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
            </div>
            <div>Fetching available voices...</div>
        </div>
    {:else if isTextToSpeechEnabled}
        <div class="section">
            <textarea class="phrase" bind:value={text} placeholder="Enter a phrase..."></textarea>
        </div>

        <div class="section">
            <label>
                <span class="text-label mr-5">Pitch {pitch.toFixed(1)}</span>
                <input type="range" bind:value={pitch} min="0.0" max="2.0" step="0.1">
            </label>
        </div>
        <div class="section">
            <label>
                <span class="text-label mr-5">Rate: {rate.toFixed(1)}</span>
                <input type="range" bind:value={rate} min="0.1" max="10.0" step="0.1">
            </label>
        </div>

        <div class="section">
            <label>
                <span class="text-label mr-5">Select Voice</span>
                <select class="voices" bind:value={selectedVoiceIndex}>
                    {#each availableVoices as {name, lang}, index}
                        <option value={index}>
                            {name} - {lang}
                        </option>
                    {/each}
                </select>
            </label>
        </div>

        <div class="section">
            <label>
                <span class="text-label mr-5">Select Speaker</span>
                <select class="voices" bind:value={selectedDeviceId}>
                    {#each availableOutputDevices as {label, deviceId, kind}}
                        <option value={deviceId}>
                            {label} - {kind} - {deviceId}
                        </option>
                    {/each}
                </select>
            </label>
        </div>

        <div class="section justify-content-center confirm">
            <button class="secondary" on:click={reset}>
                <span>Reset</span>
            </button>

            <button class="primary" on:click={utterText}>
                <span>Play Phrase</span>
            </button>

            <button class="primary" on:click={recordUtterance}>
                <span>Download Phrase Audio</span>
            </button>
        </div>

    {:else}
        <h2>Text to speech is not supported by your browser</h2>
    {/if}
</div>




