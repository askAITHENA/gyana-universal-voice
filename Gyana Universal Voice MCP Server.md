# Gyana Universal Voice MCP Server

A unified WebSocket-based MCP (Model Context Protocol) server for end-to-end voice processing: Speech-to-Text → AI Processing (with child safety) → Text-to-Speech through a single endpoint with secure access, usage tracking, and multi-provider support.

## 🚀 Quick Start

**Endpoint:** `wss://api.askaithena.com/mcpserver/voice`

**WebSocket Connection:**
websocat wss://api.askaithena.com/mcpserver/voice

## 📋 Table of Contents

- [Overview](#-overview)
- [Claude Desktop Integration](#-claude-desktop-integration)
- [Direct WebSocket Integration](#-direct-websocket-integration)
- [Voice Processing Pipeline](#-voice-processing-pipeline)
- [Available Providers](#-available-providers)
- [Supported API Methods](#-supported-api-methods)
- [Authentication & Usage](#-authentication--usage)
- [Integration Examples](#-integration-examples)
- [Child Safety Features](#-child-safety-features)
- [Key Benefits](#-key-benefits)
- [Error Handling](#-error-handling)
- [Use Cases](#-use-cases)
- [Getting Started](#-getting-started)
- [Pricing](#-pricing)

## 🎙️ Overview

Universal Voice MCP Server provides complete voice interaction capabilities through a single API:

- **Input:** Base64-encoded audio (WAV, OGG, MP3)
- **Processing:** Automatic transcription → AI response → Speech synthesis
- **Output:** Base64-encoded audio response
- **Safety:** Built-in child safety filters at every stage

Perfect for building voice assistants, educational tools, customer service bots, and accessibility applications.

## 🖥️ Claude Desktop Integration

### macOS Setup

1. **Install websocat:**

brew install websocat


2. **Configure Claude Desktop:**

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "gyana-universal-voice": {
      "command": "websocat",
      "args": ["wss://api.askaithena.com/mcpserver/voice"]
    }
  }
}
```

3. **Restart Claude Desktop** and look for Gyana Voice tools in new conversations

**Note:** Claude Desktop does not support audio playback. For full voice functionality, use direct WebSocket integration.

### Windows Setup

1. Download websocat from the [releases page](https://github.com/vi/websocat/releases)
2. Configure Claude Desktop config file with the same JSON configuration
3. Restart Claude Desktop

## 🔌 Direct WebSocket Integration

### Connection Details

- **Protocol:** WebSocket Secure (WSS)
- **URL:** `wss://api.askaithena.com/mcpserver/voice`
- **Format:** JSON-RPC 2.0
- **Max Message Size:** 10MB (for audio data)
- **Audio Formats:** WAV, OGG, MP3
- **Encoding:** Base64 for audio transmission

### Python

```python
import asyncio
import websockets
import json
import base64

async def connect_gyana_voice():
    uri = "wss://api.askaithena.com/mcpserver/voice"
    
    async with websockets.connect(uri, max_size=10*1024*1024) as websocket:
        # Initialize connection
        init_message = {
            "jsonrpc": "2.0",
            "method": "initialize",
            "params": {
                "protocolVersion": "2024-11-05",
                "capabilities": {},
                "clientInfo": {
                    "name": "your-app",
                    "version": "1.0.0"
                }
            },
            "id": 1
        }
        
        await websocket.send(json.dumps(init_message))
        response = await websocket.recv()
        print("Initialized:", json.loads(response))
        
        # Read audio file and encode
        with open("input.wav", "rb") as f:
            audio_b64 = base64.b64encode(f.read()).decode()
        
        # Process voice
        voice_message = {
            "jsonrpc": "2.0",
            "method": "tools/call",
            "params": {
                "name": "process_voice",
                "arguments": {
                    "access_key": "uai_your_key_here",
                    "audio_file": audio_b64,
                    "chat_id": "user123"
                }
            },
            "id": 2
        }
        
        await websocket.send(json.dumps(voice_message))
        result = await websocket.recv()
        data = json.loads(result)
        
        # Decode output audio
        response_data = json.loads(data["result"]["content"][0]["text"])
        output_audio = base64.b64decode(response_data["output_audio_base64"])
        with open("output.wav", "wb") as f:
            f.write(output_audio)
        
        print("Transcribed:", response_data["transcribed_text"])
        print("AI Response:", response_data["ai_response"])

asyncio.run(connect_gyana_voice())
```

### JavaScript/Node.js

```javascript
const WebSocket = require('ws');
const fs = require('fs');

const ws = new WebSocket('wss://api.askaithena.com/mcpserver/voice', {
    maxPayload: 10 * 1024 * 1024
});

ws.on('open', () => {
    // Initialize
    ws.send(JSON.stringify({
        jsonrpc: "2.0",
        method: "initialize",
        params: {
            protocolVersion: "2024-11-05",
            capabilities: {},
            clientInfo: { name: "my-app", version: "1.0.0" }
        },
        id: 1
    }));
});

ws.on('message', (data) => {
    const response = JSON.parse(data);
    
    if (response.id === 1) {
        // Initialized, now send audio
        const audioBuffer = fs.readFileSync('input.wav');
        const audioB64 = audioBuffer.toString('base64');
        
        ws.send(JSON.stringify({
            jsonrpc: "2.0",
            method: "tools/call",
            params: {
                name: "process_voice",
                arguments: {
                    access_key: "uai_your_key_here",
                    audio_file: audioB64,
                    chat_id: "user123"
                }
            },
            id: 2
        }));
    } else if (response.id === 2) {
        // Process response
        const result = JSON.parse(response.result.content[0].text);
        console.log('Transcribed:', result.transcribed_text);
        console.log('AI Response:', result.ai_response);
        
        // Save output audio
        const outputAudio = Buffer.from(result.output_audio_base64, 'base64');
        fs.writeFileSync('output.wav', outputAudio);
    }
});
```

### Browser

```javascript
const ws = new WebSocket('wss://api.askaithena.com/mcpserver/voice');

ws.onopen = () => {
    ws.send(JSON.stringify({
        jsonrpc: "2.0",
        method: "initialize",
        params: {
            protocolVersion: "2024-11-05",
            capabilities: {},
            clientInfo: { name: "browser-app", version: "1.0.0" }
        },
        id: 1
    }));
};

ws.onmessage = async (event) => {
    const response = JSON.parse(event.data);
    
    if (response.id === 1) {
        // Initialized, capture audio from microphone
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const mediaRecorder = new MediaRecorder(stream);
        const chunks = [];
        
        mediaRecorder.ondataavailable = (e) => chunks.push(e.data);
        mediaRecorder.onstop = async () => {
            const blob = new Blob(chunks, { type: 'audio/wav' });
            const reader = new FileReader();
            reader.onloadend = () => {
                const audioB64 = reader.result.split(',')[1];
                ws.send(JSON.stringify({
                    jsonrpc: "2.0",
                    method: "tools/call",
                    params: {
                        name: "process_voice",
                        arguments: {
                            access_key: "uai_your_key_here",
                            audio_file: audioB64,
                            chat_id: "browser-user"
                        }
                    },
                    id: 2
                }));
            };
            reader.readAsDataURL(blob);
        };
        
        mediaRecorder.start();
        setTimeout(() => mediaRecorder.stop(), 5000); // Record 5 seconds
    } else if (response.id === 2) {
        // Play response audio
        const result = JSON.parse(response.result.content[0].text);
        const audioData = atob(result.output_audio_base64);
        const audioArray = new Uint8Array(audioData.length);
        for (let i = 0; i < audioData.length; i++) {
            audioArray[i] = audioData.charCodeAt(i);
        }
        const audioBlob = new Blob([audioArray], { type: 'audio/wav' });
        const audioUrl = URL.createObjectURL(audioBlob);
        const audio = new Audio(audioUrl);
        audio.play();
    }
};
```

## 🎵 Voice Processing Pipeline

```
Input Audio (Base64)
       ↓
Speech-to-Text (STT)
       ↓
Content Safety Check
       ↓
AI Processing with Safety
       ↓
Content Safety Check
       ↓
Text-to-Speech (TTS)
       ↓
Output Audio (Base64)
```

**Processing Time:** 5-30 seconds depending on audio length and provider selection

## 🎤 Available Providers (all paid subscriptions by the provider)

### Speech-to-Text (STT)
- **OpenAI Whisper** - High accuracy, multi-language support
- **AssemblyAI** - Fast transcription with speaker detection
- **Google Speech** - Enterprise-grade recognition

### AI Processing
- **OpenAI** - GPT-4o, GPT-4, GPT-3.5-turbo
- **Anthropic** - Claude 3.5 Sonnet, Claude 3 Opus, Claude 3 Haiku
- **Google** - Gemini Pro

### Text-to-Speech (TTS)
- **OpenAI TTS** - Natural-sounding voices, multiple languages
- **Google** - Ultra-realistic voice synthesis

## 🛠️ Supported API Methods

### `process_voice`
Complete end-to-end voice processing pipeline

**Parameters:**
- `access_key` (required) - Your Universal Voice access key
- `audio_file` (required) - Base64-encoded audio data
- `chat_id` (optional) - Session identifier for conversation continuity
- `safety_level` (optional) - "strict", "moderate", or "permissive" (default: "strict")
- `base_prompt` (optional) - Custom system prompt for AI
- `stt_provider` (optional) - Override default STT provider
- `ai_provider` (optional) - Override default AI provider
- `tts_provider` (optional) - Override default TTS provider

**Response:**
```json
{
  "success": true,
  "transcribed_text": "User's spoken words",
  "ai_response": "AI's text response",
  "output_audio_base64": "Base64 encoded audio response",
  "blocked": false,
  "safety_reason": "Passed safety check",
  "providers": {
    "stt": "openai",
    "ai": "anthropic",
    "tts": "openai"
  },
  "chat_id": "user123"
}
```

### `get_available_providers`
View available STT, AI, and TTS providers

**Parameters:**
- None required

### `check_voice_usage`
Monitor your voice processing usage and remaining calls

**Parameters:**
- `access_key` (required) - Your Universal Voice access key

### `get_base_prompts`
Retrieve available system prompts for different use cases

**Parameters:**
- None required

## 🔐 Authentication & Usage

### Access Keys
All API calls require a valid Universal Voice access key (format: `uai_[key]`)

### Usage Limits
- **Free Tier:** 20 voice processing calls per day
- **Professional:** 1,000 calls per day
- **Enterprise:** Custom limits with dedicated support

### Usage Tracking
Each voice processing request (successful or failed) counts toward your daily limit. Use the `check_voice_usage` tool to monitor remaining calls.

## 📝 Integration Examples

### Basic Voice Processing
```python
message = {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
        "name": "process_voice",
        "arguments": {
            "access_key": "uai_your_key_here",
            "audio_file": "BASE64_ENCODED_AUDIO",
            "chat_id": "user123",
            "safety_level": "strict"
        }
    },
    "id": 1
}
```

### Custom Provider Selection
```python
custom_providers = {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
        "name": "process_voice",
        "arguments": {
            "access_key": "uai_your_key_here",
            "audio_file": "BASE64_ENCODED_AUDIO",
            "stt_provider": "assemblyai",
            "ai_provider": "anthropic",
            "tts_provider": "elevenlabs"
        }
    },
    "id": 2
}
```

### With Custom System Prompt
```python
custom_prompt = {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
        "name": "process_voice",
        "arguments": {
            "access_key": "uai_your_key_here",
            "audio_file": "BASE64_ENCODED_AUDIO",
            "base_prompt": "You are a helpful financial advisor"
        }
    },
    "id": 3
}
```

### Check Voice Usage
```python
usage_check = {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
        "name": "check_voice_usage",
        "arguments": {
            "access_key": "uai_your_key_here"
        }
    },
    "id": 4
}
```

### List Available Providers
```python
list_providers = {
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
        "name": "get_available_providers",
        "arguments": {}
    },
    "id": 5
}
```

## 🛡️ Child Safety Features

Built-in three-level safety system:

**Strict (Default):**
- Maximum content filtering for child-appropriate responses
- Blocks all potentially inappropriate content
- Recommended for applications targeting children

**Moderate:**
- Balanced safety filtering for general audiences
- Blocks explicit and harmful content
- Recommended for consumer applications

**Permissive:**
- Minimal content filtering for adult-appropriate responses
- Blocks only clearly harmful content
- Recommended for professional/enterprise use

Safety checks applied at:
1. **Input Safety:** Transcribed speech checked before AI processing
2. **AI Response Safety:** AI output filtered before speech synthesis
3. **Blocked Content:** Inappropriate requests receive safe alternative responses

## ✨ Key Benefits

✅ Complete voice pipeline in one API call  
✅ Built-in child safety at every stage  
✅ Multi-provider support with dynamic routing  
✅ Conversation continuity with chat_id  
✅ Usage tracking and tier management built-in  
✅ No need to manage multiple API keys for different providers  
✅ WebSocket-based for real-time communication  
✅ MCP protocol compliance for seamless integration  
✅ Base64 encoding/decoding handled automatically  
✅ Support for WAV, OGG, MP3 formats  
✅ Customizable system prompts  
✅ Provider-agnostic - switch between providers without code changes

## ⚠️ Error Handling

The server provides detailed error responses for:
- Invalid access keys
- Usage limit exceeded
- Audio format errors
- Base64 decoding failures
- Provider-specific errors
- Safety violations
- Network connectivity issues
- Rate limit violations
- Audio size exceeding limits (10MB max)

### Example Error Response
```json
{
    "jsonrpc": "2.0",
    "error": {
        "code": -32001,
        "message": "Daily limit of 20 calls exceeded. Resets at midnight UTC."
    },
    "id": 1
}
```

## 🎯 Use Cases

**Voice Assistants** - Build Telegram bots, mobile apps, or web applications with natural voice interactions

**Educational Tools** - Create child-safe learning assistants with built-in parental controls

**Customer Service** - Automate voice support with multi-language capabilities and conversation history

**Accessibility** - Enable voice-first interfaces for users with visual impairments

**Interactive Voice Response (IVR)** - Build intelligent phone systems with natural language understanding

**Voice-Enabled IoT** - Add voice control to smart home devices and IoT applications

## 🎯 Getting Started

1. Sign up for a Gyana Universal Voice account
2. Generate your access key from the dashboard
3. Configure your integration method (MCP client, Python, JavaScript, or browser)
4. Encode your audio input as Base64
5. Send to the voice processing API
6. Receive and decode the audio response
7. Monitor your usage through the `check_voice_usage` API method

## 💰 Pricing

### Free Tier
- 20 voice processing calls per day
- All providers available
- All safety levels included
- Community support
- Basic usage analytics

### Professional - Coming soon!
- 1,000 voice processing calls per day
- Priority provider routing
- 99.9% uptime SLA
- Email support
- Usage analytics dashboard

### Enterprise - Call for discussion!
- Custom usage limits (10,000+ calls/day)
- Dedicated support team
- SLA guarantees with penalties
- Volume discounts
- White-label options
- Dedicated account manager
- Custom provider configurations
- On-premise deployment available

Contact founder@askaithena.com for Enterprise pricing

## 📞 Contact

- **Website:** https://www.askaithena.com/universalvoicemcpserver
- **Support:** support@askaithena.com
- **Sales:** sales@askaithena.com

## 📚 Resources

Coming soon!
- Integration examples and documentation available
- Enterprise support for high-volume applications

---

**By askAITHENA** (brand owned by Consulting Nirmana 53462120A registered in Singapore)

*Complete voice interaction pipeline through a single MCP-compatible WebSocket endpoint*

© 2025 Gyana. All rights reserved.
