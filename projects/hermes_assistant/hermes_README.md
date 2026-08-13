# Hermes Voice Assistant

Hermes is a local, privacy-focused, zero-cloud voice assistant built for Linux. It combines real-time speech recognition, intelligent tool execution via local Large Language Models (Ollama), hands-free continuous dictation, and low-latency neural text-to-speech synthesis.

---

## Evolution of Hermes

```
[V1.0 Initial Concept] ──► [V1.1 Optimization] ──► [V1.2 Bug Fixes] ──► [V2.0 Architectural Overhaul]
 (Monolithic PyAudio)      (Streaming & Security)     (LLM Loop Fix)       (JACK, Client-Server WS, Dual-STT,
                                                                             Barge-in, Dictation Engine)

```

### **Version 1.0 — Monolithic Prototype**

* **Architecture:** A single-process monolithic script running PyAudio, `openWakeWord`, `RealtimeSTT` (`faster-whisper`), Ollama (`qwen2.5:7b`), and Piper TTS sequentially.
* **Limitations:** Audio blocking issues between PyAudio and wake-word streams; high voice latency (waiting for full LLM token completion before speaking); basic string-prefix path checks.

### **Version 1.1 — Performance & Security Upgrade**

* **Security Sandboxing:** Replaced primitive path checks with strict `target_path.relative_to(WORKSPACE_DIR)` resolution.


* **Sentence-Level TTS Streaming:** Implemented sentence-boundary buffering (`.`, `!`, `?`). Speech synthesis starts while the LLM is still generating subsequent tokens.


* **Direct Tool Result Fast-Path:** Allowed Hermes to speak tool execution outputs directly, skipping redundant second LLM passes.
* **Audio Threading:** Switched to a single-worker background thread for STT timeout watching and lowered VAD post-speech silence from `1.5s` to `1.2s`.

### **Version 1.2 — LLM Loop & Tool Execution Fix**

* **Context Corruption Fix:** Resolved a cascade failure where unconditional appending of assistant messages caused Ollama to receive empty context turns (`{"role": "assistant", "content": ""}`).


* **Two-Path Execution:** Separated processing into **Path 1 (Tool Calls)** and **Path 2 (Standard Conversation)**, preventing double-prompting and infinite silence loops.



---

## What’s New in Version 2.0 (V2)

Version 2 represents a complete architectural rewrite. Key changes include:

| Feature | V1.x | V2.0 |
| --- | --- | --- |
| **System Architecture** | Monolithic local script | Decoupled Client-Server over WebSockets (`ws://localhost:8765`)

 |
| **Audio Routing Engine** | PyAudio / PortAudio | **JACK Audio Connection Kit** client (`jack.Client`)

 |
| **STT Engine Strategy** | Single `base.en` Whisper model | **Dual-STT Engine**: `base.en` (fast commands) + `small.en` (dictation)

 |
| **Interruption / Barge-in** | Non-interruptible speech | **Active Barge-In**: Real-time acoustic echo rejection & verbal interruption

 |
| **Dictation Engine** | None / basic note creation | **Continuous Dictation Loop** with live terminal streaming & voice overrides

 |
| **Local Command Interception** | All text sent to LLM | **0 ms LLM Latency Local Catch** for exit/cancel regex phrases

 |

---

## V2 Deep-Dive Architecture & Data Pipeline

V2 splits responsibilities between two decoupled components:

1. **The Audio Client (`hermes_audio_integration.py`)**: Handles JACK audio I/O, VAD, wake-word detection, STT transcription, Piper TTS, barge-in logic, and dictation file operations.


2. **The Brain Server (`main.py`)**: Runs an asynchronous WebSocket server connected to Ollama (`llama3.2:3b` / `qwen2.5:7b`), controlling tool selection, context management, and response streaming.



```
               +-------------------------------------------------------+
               |                  HERMES SYSTEM ARCHITECTURE           |
               +-------------------------------------------------------+

 +-----------------------------------------------------------------------------------+
 |                             AUDIO CLIENT (JACK Client)                            |
 |                                                                                   |
 |  [ Microphones ]                                                                  |
 |         │                                                                         |
 |         ▼                                                                         |
 |  (JACK Inport) ──► jack_process_callback() ──► audio_in_queue (Resampled PCM)      |
 |                                                       │                           |
 |         +---------------------------------------------+                           |
 |         │                                                                         |
 |         ▼                                                                         |
 |  [ openWakeWord ] ──(Confidence > 0.55)──► Triggers Active Session                |
 |                                                   │                               |
 |                                                   ▼                               |
 |                              [ Dual-Engine RealtimeSTT Recorder ]                 |
 |                              • Standby/Command: faster-whisper base.en             |
 |                              • Dictation Mode:  faster-whisper small.en            |
 |                                                   │                               |
 |                                                   ▼                               |
 |                                         Transcribed Text String                   |
 +---------------------------------------------------|-------------------------------+
                                                     │
                                            JSON WebSocket Frame
                                            ("type": "user_input")
                                                     │
                                                     ▼
 +-----------------------------------------------------------------------------------+
 |                             BRAIN SERVER (ws://localhost:8765)                    |
 |                                                                                   |
 |  handle_client_connection()                                                       |
 |         │                                                                         |
 |         ▼                                                                         |
 |  [ Ollama API ] (Messages + Context + Available Tools)                            |
 |         │                                                                         |
 |         ├───► [ Tool Call Detected ] ──► Execute hermes_tools.py                    |
 |         │                                      │                                  |
 |         │                                      ├──► Dictation Trigger?            |
 |         │                                      │       └──► Send dictation_trigger |
 |         │                                      └──► Summarize Tool Result         |
 |         │                                                                         |
 |         └───► [ Standard Conversation ] ──► Stream Sentence Chunks                |
 |                                                       │                           |
 +-------------------------------------------------------|---------------------------+
                                                         │
                                               JSON WebSocket Frames
                                               ("type": "tts_chunk")
                                                         │
                                                         ▼
 +-----------------------------------------------------------------------------------+
 |                             TTS & PLAYBACK PIPELINE                               |
 |                                                                                   |
 |  Synthesize via Piper Voice ──► audio_out_queue ──► JACK Outports ──► [ Speakers ] |
 |                                        │                                          |
 |                         [ Barge-in Interruption Active ]                          |
 |                         (Cancels playback if valid speech detected)               |
 +-----------------------------------------------------------------------------------+

```

---

## Detailed Step-by-Step Execution Pipeline

### 1. Audio Capture & JACK Resampling

* The system registers a C-native JACK client (`HermesAudioClient`) with input (`input_1`) and output ports (`output_1`, `output_2`).


* `jack_process_callback()` processes raw audio frames in real time. If the system sample rate is 48 kHz or 96 kHz, it downsamples the input stream to 16 kHz before pushing raw 16-bit PCM bytes into `audio_in_queue`.



### 2. Wake-Word Activation & STT Routing

* **Standby Listening:** `wait_for_wakeword()` feeds audio chunks into `openWakeWord` checking against `hey_hermes.onnx`. When confidence exceeds `0.55`, the audio queue flushes and the active voice turn begins.


* **Dual-Model Strategy:**
* **Command Mode (`base.en`):** Optimized for low latency (74M parameters, `compute_type="int8"`, `silero_sensitivity=0.35`).


* **Dictation Mode (`small.en`):** Swapped automatically when editing or dictating (244M parameters). It provides higher accuracy and fires real-time transcription updates directly to the terminal via `on_realtime_dictation_update()`.





### 3. Brain Server Reasoning & Tool Execution

When `main.py` receives user input over WebSockets, it passes the payload to Ollama:

1. **Tool Inspection:** Python's `inspect` module dynamically discovers all public functions inside `hermes_tools.py` and converts them into JSON schema formats.


2. **Execution via Reflection:** If the LLM generates a `tool_calls` request:
```python
func_to_call = getattr(hermes_tools, func_name)
result = func_to_call(**kwargs)

```


`main.py` runs the function locally. The string result is appended back to `conversation_history` under the `"tool"` role.


3. **Summarization Pass:** A second fast inference pass converts raw tool outputs (e.g., list of files) into short, natural spoken summaries.



### 4. Streaming TTS & Acoustic Barge-In

* Response text is streamed into `stream_sentence_chunks()` which splits on clause/sentence boundaries (`.`, `!`, `?`, `;`, `,`) and strips tags like `[LISTEN]`.


* Chunks are sent over WebSockets as `tts_chunk` frames and synthesized by Piper TTS into PCM float32 arrays.


* **Barge-In Interruption Verification:** While speech plays through the speakers, `synthesize_and_play(..., enable_bargein=True)` activates the microphone. If the user speaks during playback, `_is_valid_interruption()` evaluates the user's audio:


* Checks for explicit stop keywords (`STOP_COMMANDS`) or `"override"`.


* Calculates Levenshtein string similarity (`difflib.SequenceMatcher`) between spoken text and currently playing TTS text.


* Rejects speaker feedback echoes (if $>30\%$ of spoken words match the playing sentence).


* If valid speech is confirmed, TTS output queues wipe instantly and the assistant switches to process the user's new command.





---

## Dictation Mode Workflow

Dictation Mode provides continuous, hands-free voice typing directly into workspace files.

```
                 +--------------------------------------------+
                 |            USER SAYS DICTATION             |
                 | "Start dictating in my meeting_notes file" |
                 +--------------------------------------------+
                                       │
                                       ▼
                 +--------------------------------------------+
                 |            LLM TRIGGERS TOOL               |
                 | enter_dictation_mode("meeting_notes.md")   |
                 +--------------------------------------------+
                                       │
                                       ▼
                 +--------------------------------------------+
                 |          WEBSOCKET MESSAGE SENT            |
                 |  {"type": "dictation_trigger",             |
                 |   "file_name": "meeting_notes.md"}         |
                 +--------------------------------------------+
                                       │
                                       ▼
                 +--------------------------------------------+
                 |       CLIENT ENTERS DICTATION LOOP         |
                 |   • Swaps to high-accuracy small.en STT    |
                 |   • Live terminal feedback active          |
                 +--------------------------------------------+
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
        [ Dictating Content ]                  [ Spoken Voice Override ]
        "We need to review the                 "override create a summary
         Q3 roadmap tomorrow"                   section at the top"
                    │                                     │
                    ▼                                     ▼
        Appended directly to                   Splits at "override":
        workspace/meeting_notes.md             1. Saves text before "override"
                                               2. Sends command to LLM Brain
                                               3. Returns to dictation

```

### **Hands-Free Override Commands**

To issue commands without leaving Dictation Mode, speak the word **`"override"`**:

* **Inline Command:** *"We are done with section one override exit"*
* Text before `"override"` (*"We are done with section one"*) is appended to the file.


* Text after `"override"` (*"exit"*) is parsed as a system command.




* **Pause Command:** *"override"* (spoken alone)
* System asks: *"Command?"* and opens a 4-second listening window.




* **Local Zero-Latency Exit Catch:**
If the override command matches `EXIT_COMMANDS` (`exit`, `quit`, `stop dictation`, `cancel dictation`, `close dictation`), it bypasses the LLM network call entirely, speaks *"Exiting dictation mode"*, restores the `base.en` model, and returns to standby mode with **0 ms latency**.



---

## File Security & Workspace Sandboxing

All file tools inside `hermes_tools.py` are strictly anchored to `./workspace/`.

### Path Resolution Rules

1. **Strict Subpath Verification:** `get_safe_path()` resolves target paths using `pathlib.Path.resolve()` and enforces boundaries using `relative_to(WORKSPACE_DIR)` inside a `try/except ValueError` block.


2. **Automatic Naming:** Filenames missing extensions default to `.md`. Spaces are automatically converted to underscores and letters normalized to lowercase.


3. **Linux XDG Trash Integration:** `move_to_bin()` moves files to `~/.local/share/Trash/files/` and writes standard `.trashinfo` metadata files containing deletion timestamps and original URI paths.



---

## WebSocket Event Protocol Reference

Communication between `hermes_audio_integration.py` and `main.py` uses JSON frames:

### 1. `user_input` (Client $\rightarrow$ Server)

```json
{
  "type": "user_input",
  "text": "List the files in my workspace.",
  "turn_id": 1
}

```

### 2. `tts_chunk` (Server $\rightarrow$ Client)

```json
{
  "type": "tts_chunk",
  "turn_id": 1,
  "text_token": "In your workspace, you have notes.md and project_ideas.md."
}

```

### 3. `dictation_trigger` (Server $\rightarrow$ Client)

```json
{
  "type": "dictation_trigger",
  "turn_id": 2,
  "file_name": "notes.md"
}

```

### 4. `turn_complete` (Server $\rightarrow$ Client)

```json
{
  "type": "turn_complete",
  "turn_id": 1,
  "listen_followup": false
}

```

---

## Installation & Setup
---
## Automatic:
1. Clone the hermes_assistant repository onto your machine (strictly speaking the python scripts, custom_wakeword, .local_models, pyproject.toml, install.sh and run.sh scripts)
2. Run the install.sh script followed by the run.sh script

---
---

## Manual installation and execution:
### Requirements

* **OS:** Linux (Ubuntu/Debian/Arch)
* **Python:** 3.10+
* **Dependencies:** JACK Audio Connection Kit (`jackd2` or `pipewire-jack`), Ollama, Piper TTS voices

### 1. System Dependencies

```bash
sudo apt update
sudo apt install portaudio19-dev jackd2 libjack-jackd2-dev

```

### 2. Python Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install websockets numpy jack-client openwakeword RealtimeSTT piper-tts ollama

```

### 3. Running Hermes

Start the Brain Server first, then launch the Audio Integration Client:

```bash
# Terminal 1: Brain Server
python3 main.py

# Terminal 2: Audio Integration Client
python3 hermes_audio_integration.py

```