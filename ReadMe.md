# Introduction

An AI voice agent combines **speech capabilities** with the **reasoning power of Foundation Models (LLMs)** to enable real-time, human-like conversation.

* **Key Use Cases:** Education (mock interviews), Business (customer service/booking), and Accessibility (hands-free health logging).

### **2. The Voice Agent Stack**

There are two main architectural approaches:

1. **Speech-to-Speech / Real-time API:** Simpler to implement but offers less flexibility and control.
2. **The Pipeline Approach (Course Focus):** Composed of three distinct layers giving developers fine-grained control.

#### **The Core Pipeline Components**

**STT / ASR** - Speech-to-Text Transcribes raw audio waveforms into text  
**LLM** - Large Language Model The "brain" that generates a response based on the text  
**TTS** - Text-to-Speech Synthesizes the text response into human-like audio

#### **Essential Pre-Processing Tasks**

* **VAD (Voice Activity Detection):** Determines if human speech is present (filtering out silence or background noise).
* **End-of-Turn Detection:** Identifies when a speaker has finished talking. This is complex due to natural pauses in human speech.

### **3. The Challenge: Latency**

The biggest hurdle in voice agents is timing. If the system lags, the illusion of conversation breaks.

* **Human Expectation:** Humans expect a response within **~236ms** (though this varies by language and speaker).
* **Pipeline Reality:** A typical optimized pipeline takes **~540ms** to **1.5s+**, heavily depending on the API providers used.

**Mitigation Strategies:**

* **Infrastructure:** Use Real-Time Peer-to-Peer communication (**WebRTC**) via platforms like **LiveKit** to bypass intermediary servers.
* **Model Selection:** Use faster inference providers (e.g., Groq, Cerebras) or smaller quantized models for the LLM layer.
* **Prompting:** Instruct the LLM to generate shorter initial replies to reduce perceived lag.


### **4. Building the Agent**

Developers do not need to build from scratch; they can mix and match APIs based on use-case needs (e.g., prioritizing high-accuracy STT for medical apps vs. high-reasoning LLMs for business logic).

**The "Minimal Example" in Python:**
Building a LiveKit agent requires three main code components:

1. **Agent Definition:** Setting the instructions and prompts.
2. **Session Management:** Linking the STT, LLM, and TTS providers.
3. **Entrypoint Function:** The main function executed when a user connects.

**Demo Configuration:**
The lesson concludes with a demo of an avatar (using Andrew Ng’s voice) capable of handling interruptions.

* **STT:** OpenAI
* **TTS:** ElevenLabs (Custom Voice Clone)
* **Frontend:** LiveKit Playground (for testing without building a UI)


### **5. Unique Challenges**

* **Disfluencies:** Filler words like "um" or long pauses can confuse the transcription and End-of-Turn detection.
* **Multilingual Support:** Non-English ASR models generally underperform compared to English models, affecting latency and accuracy.

# Arhitecture

For a voice agent to feel "conversational," speed (latency) is more critical than perfect reliability. The system must minimize the time between the user speaking and the agent responding.

### 1. The Transport Layer: TCP vs. UDP

The transcript compares the two fundamental internet protocols regarding audio streaming:

* **TCP (Transmission Control Protocol):** Prioritizes reliability. If a packet (Packet 2) is lost, TCP pauses the stream (blocking Packets 1 and 3) until Packet 2 is resent and received.
* *Issue:* This causes "Head of Line Blocking," leading to stuttering and freezing in audio playback.


* **UDP (User Datagram Protocol):** Prioritizes speed. It delivers packets immediately as they arrive. If a packet is lost, the application can skip it or interpolate.
* *Verdict:* **UDP is essential** for real-time voice agents to avoid lag.

### 2. High-Level Protocols

The lesson evaluates three common web protocols for carrying voice data:

* **HTTP (State-based on TCP):**
* *Mechanism:* Connect  Send Data  Disconnect.
* *Cons:* High latency due to connection overhead; no native audio optimization. Used by early assistants (old Siri/Alexa) which felt slow and transactional, not conversational.


* **WebSockets (Persistent on TCP):**
* *Mechanism:* Keeps a connection open (Stateful).
* *Cons:* Still built on TCP, so it suffers from the same blocking/lag issues when network packets are dropped.


* **WebRTC (Built on UDP):**
* *Mechanism:* Designed specifically for audio/video (used by Zoom, Google Meet).
* *Pros:*
* **Congestion Control:** Measures network health to pace packets.
* **Compression:** Compresses audio significantly (down to ~3% of the size compared to raw HTTP audio).
* **Timestamping:** Makes handling user interruptions (barge-ins) trivial.

### 3. The Implementation Solution: LiveKit

While WebRTC is the correct technical choice, it is difficult to implement (complex stack) and hard to scale (standard WebRTC is Peer-to-Peer, which is slow over public internet distances).

The transcript introduces **LiveKit** as the solution:

* **Infrastructure:** An open-source framework that manages WebRTC complexity.
* **Optimization:** Uses a "private tunnel" network (LiveKit Cloud) rather than the public internet, reducing latency by **20–50%**.
* **Real-world Use:** This architecture is currently used by OpenAI for **ChatGPT’s Advanced Voice Mode**.

# WebRTC connection (which ensures <50ms latency)

A voice agent is defined as a **stateful program** that manages a persistent connection for every user, handling input streams, context, and external API calls (DB, RAG, etc.).

### **1. The Core Pipeline (Cascaded Architecture)**

The agent processes data in a specific, ordered sequence to "listen, think, and speak."

1. **Input (STT):** User audio streams to the agent and is relayed to a **Speech-to-Text** model. Real-time transcriptions are returned to the agent.
2. **Reasoning (LLM):** Once the user finishes speaking, the full transcript is sent to the **Large Language Model**. The LLM streams generated tokens back to the agent.
3. **Synthesis (TTS):** The agent aggregates LLM tokens into sentences. Each sentence is sent to the **Text-to-Speech** model immediately.
4. **Output:** TTS audio bytes are streamed back to the user via the WebRTC connection.

### **2. The "Hard Problem": Turn Detection**

The system must determine when the user has actually finished speaking versus just pausing for breath. This requires a hybrid approach:

#### **A. Signal Processing (VAD)**

* **Voice Activity Detection:** A small binary classifier neural network that detects the presence or absence of human speech.
* **Mechanism:** When speech stops, a timer starts. If the timer expires without new speech, the turn is considered over.

#### **B. Semantic Processing (Semantic Turn Detector)**

* **The Logic Check:** A transformer model analyzes the *text* of what the user just said (plus context from previous turns).
* **Conflict Resolution:** If VAD detects silence (timer starts), but the Semantic model predicts the user's sentence is incomplete (e.g., "I want to buy a..."), the system **delays the timer**, waiting for the user to finish their thought.


### **3. Interruption Handling**

To mimic natural conversation, users must be able to interrupt the agent.

* **Trigger:** The VAD detects the **presence** of human speech while the agent is speaking.
* **Action:** The pipeline is immediately **flushed**.
* LLM inference is stopped.
* TTS generation is cancelled.
* Audio playback is halted.


### **4. Context Management**

The agent maintains the state of the conversation to ensure the LLM has the full history.

* **Standard Context:** The LLM receives the full transcript history of the session.
* **Interruption Syncing:** If an interruption occurs, the LiveKit SDK uses timestamps to determine exactly what the user heard before they interrupted. The conversation history is **truncated/aligned** to that exact moment so the LLM doesn't "remember" saying things the user never actually heard.
