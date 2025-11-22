# Text-to-Speech (TTS) Feature

**Feature Category**: Accessibility, Audio, Reading Enhancement
**Added**: January 2025 (commits 07b04b82, 74021412, and 40+ related commits)
**Dependencies**: Web Speech API, Edge TTS (Microsoft), AsyncQueue

## Overview

The Text-to-Speech (TTS) feature enables users to listen to ebooks being read aloud with natural-sounding voices. Readest implements a dual-backend architecture supporting both browser-native Web Speech API and Microsoft's Edge TTS service, offering 100+ voices across 50+ languages.

### Key Capabilities

- **Dual TTS Backends**: Web Speech API (browser fallback) + Edge TTS (neural voices)
- **100+ Voices**: Support for 50+ languages with male/female options
- **Speech Rate Control**: Adjustable speed from 0.2x to 3.0x
- **Audio Preloading**: Next 2 sentences pre-loaded for seamless playback
- **Platform Support**: iOS (audio unblocking), Android, desktop media controls
- **Smart Navigation**: Forward/backward by sentence
- **State Persistence**: Remembers voice and rate preferences per book

## Architecture

### Dual Backend System

| Backend | Purpose | Voices | Quality | Availability |
|---------|---------|--------|---------|--------------|
| **Web Speech API** | Browser fallback | ~20-50 (OS-dependent) | Good | Always available |
| **Edge TTS** | Premium voices | 100+ neural voices | Excellent | Network required |

**Backend Selection**: Automatic switching based on selected voice. Edge TTS voices use WebSocket streaming; Web Speech API uses browser's native synthesis.

### Component Structure

```
src/app/reader/
├── components/
│   ├── TTSIcon.tsx (54 lines)          # Animated play/pause button
│   ├── TTSPanel.tsx (164 lines)        # Voice & rate controls
│   └── TTSControl.tsx                  # Floating TTS button
├── hooks/
│   └── useTTSController.ts             # React hook wrapper
└── utils/
    └── tts/
        ├── TTSController.ts (248 lines) # Main orchestrator
        ├── EdgeTTSBackend.ts           # Edge TTS implementation
        ├── WebSpeechBackend.ts         # Web Speech implementation
        ├── AsyncQueue.ts               # Sequential processing
        ├── ssmlParse.ts                # SSML parsing
        └── audioCache.ts               # LRU cache (200 entries)
```

## Key Components

### 1. TTSController (`src/app/reader/utils/tts/TTSController.ts`)

**Purpose**: Orchestrates TTS playback with 7-state state machine.

**States**:
- `idle`: Not initialized
- `loading`: Initializing backend
- `loaded`: Ready to play
- `playing`: Currently speaking
- `paused`: Playback paused
- `stopped`: User stopped playback
- `error`: Error occurred

**Key Methods**:
```typescript
class TTSController {
  async init(backend: 'edge' | 'webspeech'): Promise<void>
  async play(): Promise<void>
  async pause(): Promise<void>
  async stop(): Promise<void>
  async forward(): Promise<void>  // Next sentence
  async backward(): Promise<void> // Previous sentence
  setRate(rate: number): void     // 0.2 - 3.0
  setVoice(voice: Voice): void
}
```

### 2. TTSPanel (`src/app/reader/components/TTSPanel.tsx`)

**Purpose**: UI controls for voice selection and speech rate.

**Components**:
- **Voice Dropdown**: Searchable list of 100+ voices filtered by language
- **Rate Slider**: 0.2x to 3.0x with 0.1 increments
- **Backend Indicator**: Shows active backend (Edge TTS vs Web Speech)

**Voice Filtering**:
```typescript
// Filters voices by detected book language
const bookLang = getBookLanguage(metadata);
const filteredVoices = voices.filter(v => 
  v.lang.startsWith(bookLang) || v.lang === 'en-US'
);
```

### 3. Edge TTS Backend (`src/app/reader/utils/tts/EdgeTTSBackend.ts`)

**Purpose**: Microsoft neural voice synthesis via WebSocket.

**Features**:
- WebSocket streaming for low latency
- SSML support for pronunciation control
- Automatic voice switching based on language
- MD5 caching of audio segments

**Voice Quality**: Neural voices with natural prosody, emotion, and multi-lingual support.

**API Endpoint**: `wss://speech.platform.bing.com/consumer/speech/synthesize/readaloud/edge/v1`

### 4. Audio Preloading (`src/app/reader/utils/tts/audioCache.ts`)

**Purpose**: LRU cache with MD5 hashing for instant playback.

**Configuration**:
- **Cache Size**: 200 audio segments
- **Preload Count**: Next 2 sentences
- **Hash Algorithm**: MD5 of text + voice + rate
- **Eviction**: Least Recently Used (LRU)

**Performance**: Eliminates loading delays between sentences.

## Usage Workflow

### Basic Playback

1. User opens book in reader
2. Clicks TTS icon (speaker button) in footer
3. TTSPanel appears with voice/rate controls
4. User selects voice (auto-switches backend if needed)
5. User clicks Play button
6. TTS begins reading from current location
7. Audio for next 2 sentences pre-loads in background

### Voice Selection

```
User opens voice dropdown
  ↓
Voices filtered by book language (e.g., 'en' for English)
  ↓
User selects "Microsoft David - English (United States)"
  ↓
Backend switches to Edge TTS
  ↓
Voice preference saved to book config
```

### Speech Rate Adjustment

```
User drags rate slider to 1.5x
  ↓
Current audio stops
  ↓
TTS restarts with new rate
  ↓
Rate saved to book config
```

## Platform-Specific Features

### iOS Support

**Challenge**: iOS requires user interaction to enable WebAudio.

**Solution** (`src/app/reader/utils/tts/iosAudioUnblock.ts`):
```typescript
// Plays silent audio on first user interaction
function unblockAudio() {
  const silent = new Audio('data:audio/wav;base64,UklGRigAAABXQVZFZm10...');
  silent.loop = true;
  silent.play();
}
```

**Mute Switch Handling**: Loops silent audio to keep AudioContext active even when mute toggle is ON.

### Android Support

- Native TTS engine integration (optional)
- Media session controls for notification panel
- Background playback support

### Desktop Support

- Media session API for keyboard media keys
- System notification with current sentence
- Dock/taskbar controls (macOS/Windows/Linux)

## State Management

### TTSStore (`src/store/ttsStore.ts`)

```typescript
interface TTSStore {
  isOpen: boolean;              // Panel visibility
  isPlaying: boolean;           // Playback state
  rate: number;                 // Speech rate (0.2-3.0)
  voice: Voice | null;          // Selected voice
  backend: 'edge' | 'webspeech';// Active backend
  
  setIsOpen: (open: boolean) => void;
  setIsPlaying: (playing: boolean) => void;
  setRate: (rate: number) => void;
  setVoice: (voice: Voice) => void;
}
```

**Persistence**: Voice and rate saved per book in `BookConfig.ttsSettings`.

## AI Agent Modification Guidelines

### Adding a New TTS Backend

**Example: Google Cloud TTS**

1. **Create backend implementation**:

```typescript
// src/app/reader/utils/tts/GoogleTTSBackend.ts
export class GoogleTTSBackend implements TTSBackend {
  async speak(text: string, voice: Voice, rate: number): Promise<void> {
    const response = await fetch('https://texttospeech.googleapis.com/v1/text:synthesize', {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${apiKey}` },
      body: JSON.stringify({
        input: { text },
        voice: { languageCode: voice.lang, name: voice.name },
        audioConfig: { speakingRate: rate },
      }),
    });
    const { audioContent } = await response.json();
    // Decode and play audio
  }
}
```

2. **Register in TTSController**:

```typescript
// TTSController.ts
async init(backend: 'edge' | 'webspeech' | 'google') {
  switch (backend) {
    case 'google':
      this.backend = new GoogleTTSBackend();
      break;
    // ... other cases
  }
}
```

3. **Add to voice list**:

```typescript
const googleVoices = await fetchGoogleVoices();
allVoices.push(...googleVoices.map(v => ({ ...v, backend: 'google' })));
```

### Customizing Speech Rate Range

To extend speech rate beyond 3.0x:

1. **Update slider in TTSPanel.tsx**:

```typescript
<input
  type="range"
  min="0.2"
  max="5.0"  // Changed from 3.0
  step="0.1"
  value={rate}
  onChange={(e) => setRate(parseFloat(e.target.value))}
/>
```

2. **Update backend constraints**:

```typescript
// EdgeTTSBackend.ts
const ssml = `<speak><prosody rate="${Math.max(0.2, Math.min(5.0, rate))}">${text}</prosody></speak>`;
```

### Adding Sentence Navigation UI

To add Previous/Next sentence buttons:

1. **Add to TTSPanel.tsx**:

```typescript
<button onClick={() => controller.backward()}>
  <MdSkipPrevious />
</button>
<button onClick={() => controller.forward()}>
  <MdSkipNext />
</button>
```

2. **Update TTSController** (already implemented):

```typescript
async forward() {
  this.currentSentenceIndex++;
  await this.play();
}

async backward() {
  this.currentSentenceIndex = Math.max(0, this.currentSentenceIndex - 1);
  await this.play();
}
```

## Entry Points for Common Tasks

| Task | Primary Files | Steps |
|------|---------------|-------|
| **Enable/disable TTS** | `TTSIcon.tsx`, `TTSPanel.tsx` | Toggle `isOpen` in TTSStore |
| **Change voice** | `TTSPanel.tsx`, `TTSController.ts` | Select from voice dropdown → `setVoice()` |
| **Adjust speed** | `TTSPanel.tsx`, `TTSController.ts` | Drag rate slider → `setRate()` |
| **Add new backend** | `TTSController.ts`, new backend file | Implement `TTSBackend` interface |
| **Customize preloading** | `audioCache.ts`, `TTSController.ts` | Adjust `PRELOAD_COUNT` constant |
| **Debug playback issues** | Browser DevTools, `TTSController.ts` | Check state transitions, audio errors |
| **Add voice filtering** | `TTSPanel.tsx` | Modify `filteredVoices` logic |

## Common Issues and Debugging

### Issue: TTS not playing on iOS

**Symptoms**: Play button clicks but no audio

**Debug steps**:
1. Check if audio context is blocked (Safari security)
2. Verify `iosAudioUnblock.ts` was called on user interaction
3. Check mute switch setting (silent mode blocks audio)
4. Inspect console for WebAudio errors

**Solution**: Ensure silent audio loop is active before TTS initialization.

### Issue: Voice list empty

**Symptoms**: Voice dropdown shows no voices

**Debug steps**:
1. Check network connectivity (Edge TTS requires internet)
2. Verify Web Speech API support: `'speechSynthesis' in window`
3. Check browser compatibility (Safari, Chrome, Firefox)
4. Inspect console for API errors

**Solution**: Fallback to Web Speech API if Edge TTS unavailable.

### Issue: Choppy playback

**Symptoms**: Audio stutters or pauses between sentences

**Debug steps**:
1. Check preloading status (next 2 sentences should be cached)
2. Verify network speed for Edge TTS streaming
3. Check CPU usage (high usage can delay audio processing)
4. Inspect audio cache hit rate

**Solution**: Increase `PRELOAD_COUNT` or reduce speech rate.

### Issue: Wrong language voice

**Symptoms**: Book in English but voice speaks in wrong language

**Debug steps**:
1. Check book metadata language: `bookDoc.metadata.language`
2. Verify voice filtering logic in `TTSPanel.tsx`
3. Check default voice selection

**Solution**: Manually select correct voice or fix book metadata.

## Performance Considerations

### Audio Preloading

- **Tradeoff**: Memory vs latency
- **Current**: Preload next 2 sentences (optimal for most use cases)
- **High memory**: Reduce `PRELOAD_COUNT`
- **High latency**: Increase `PRELOAD_COUNT` to 5+

### Cache Size

- **200 entries**: ~2-5 MB memory (varies by audio length)
- **LRU eviction**: Automatically removes least recently used
- **Tuning**: Adjust `MAX_CACHE_SIZE` based on device memory

### Backend Selection

- **Edge TTS**: Better quality, requires network, ~200ms latency
- **Web Speech API**: Instant, offline, lower quality
- **Recommendation**: Default to Edge TTS, fallback to Web Speech

## Security Considerations

### API Keys

- **Edge TTS**: Uses public Microsoft endpoint (no auth required)
- **Future backends**: Store API keys securely, never in client code
- **Recommendation**: Use environment variables for third-party keys

### SSML Injection

- **Risk**: Malicious book content could inject SSML tags
- **Mitigation**: `ssmlParse.ts` sanitizes text before synthesis
- **Validation**: Strips dangerous tags, escapes special characters

### Audio Caching

- **Privacy**: Audio cache stored in memory (not persisted to disk)
- **Cleanup**: Cache cleared when book closes
- **Recommendation**: No sensitive data leaked across sessions

## Testing TTS Functionality

### Manual Testing Checklist

- [ ] Play/pause/stop controls work
- [ ] Voice selection changes voice
- [ ] Speech rate slider adjusts speed
- [ ] Forward/backward navigate sentences
- [ ] Preloading reduces gaps between sentences
- [ ] iOS audio unblocking works on first interaction
- [ ] Media controls appear in notification panel (mobile)
- [ ] Voice and rate persist after closing book
- [ ] Edge TTS fallback to Web Speech works offline
- [ ] TTS stops when navigating away from book

### Automated Tests

```typescript
// test/tts.test.ts
describe('TTS Controller', () => {
  it('initializes Edge TTS backend', async () => {
    const controller = new TTSController();
    await controller.init('edge');
    expect(controller.state).toBe('loaded');
  });

  it('plays audio', async () => {
    await controller.play();
    expect(controller.state).toBe('playing');
  });

  it('respects speech rate', async () => {
    controller.setRate(1.5);
    expect(controller.rate).toBe(1.5);
  });
});
```

## Future Enhancements

### Planned Features

1. **Bilingual TTS**: Different voices for different languages in same book
2. **Emotional prosody**: Adjust tone based on punctuation (excitement, sadness)
3. **SSML editor**: Advanced users can fine-tune pronunciation
4. **Voice cloning**: Custom voice training
5. **Offline mode**: Download voices for offline use
6. **Speed reading**: Highlight current word while speaking
7. **Smart pauses**: Longer pauses at paragraph breaks

### Integration Opportunities

1. **Screen readers**: Integration with NVDA, JAWS, VoiceOver
2. **Translation**: Speak translated text while showing original
3. **Annotation**: Add voice notes to highlights
4. **Audiobook sync**: Sync with professional audiobook narration

---

## Language Code Normalization (Added v0.9.18)

### Overview
Commit #457 improved TTS voice selection by normalizing language codes across different metadata formats.

### Problem Solved
Books may have language metadata in various formats:
- ISO 639-1: `en`, `zh`, `ja`
- ISO 639-2: `eng`, `chi`, `jpn`
- BCP 47: `en-US`, `zh-CN`, `ja-JP`
- Regional variants: `en-GB`, `pt-BR`, `zh-TW`

TTS voices use BCP 47 codes, leading to mismatches when book metadata uses different formats.

### Normalization Logic

**Location**: `src/utils/language.ts`

```typescript
function normalizeLangCode(code: string): string {
  if (!code) return 'en-US';

  // Convert ISO 639-2 to ISO 639-1
  const iso639_2to1 = {
    'eng': 'en',
    'chi': 'zh',
    'jpn': 'ja',
    'fra': 'fr',
    'deu': 'de',
    'spa': 'es',
    // ... full mapping
  };

  const normalized = iso639_2to1[code] || code;

  // Ensure BCP 47 format
  if (normalized.length === 2) {
    const regionMap = {
      'en': 'en-US',
      'zh': 'zh-CN',
      'ja': 'ja-JP',
      'fr': 'fr-FR',
      'de': 'de-DE',
      'es': 'es-ES',
      // ... default regions
    };
    return regionMap[normalized] || `${normalized}-${normalized.toUpperCase()}`;
  }

  return normalized;
}
```

### Voice Matching Algorithm
1. Book language code normalized to BCP 47 format
2. Filter TTS voices by normalized code
3. Fallback to base language (e.g., `en` for `en-GB`)
4. Final fallback to default voice (`en-US`)

**Example Flow**:
```typescript
// Book has metadata: language="eng" (ISO 639-2)
normalizeLangCode("eng") → "en-US"
// Finds Edge TTS voice: "en-US-AvaNeural"

// Book has metadata: language="zh" (ISO 639-1)
normalizeLangCode("zh") → "zh-CN"
// Finds Edge TTS voice: "zh-CN-XiaoxiaoNeural"

// Book has metadata: language="pt-BR" (BCP 47 regional)
normalizeLangCode("pt-BR") → "pt-BR"
// Finds Edge TTS voice: "pt-BR-FranciscaNeural"
```

### Impact
- Better automatic voice selection for books with varied metadata
- Supports more book metadata formats
- Consistent voice matching across platforms
- Reduced need for manual voice selection

### AI Modification Guidelines

**To add a new language code mapping**:

1. Add ISO 639-2 to ISO 639-1 mapping in `iso639_2to1`:
   ```typescript
   'rus': 'ru',  // Russian
   'ara': 'ar',  // Arabic
   ```

2. Add default region mapping in `regionMap`:
   ```typescript
   'ru': 'ru-RU',
   'ar': 'ar-SA',
   ```

3. Test with books that have the language code in metadata

---

**Last Updated**: Documentation for commits through cab757257 (Feb 2025)
**Related Documents**: [feature-cross-platform-support.md](./feature-cross-platform-support.md), [feature-settings-system.md](./feature-settings-system.md), [feature-internationalization.md](./feature-internationalization.md)
