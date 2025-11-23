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

---

## Sub-Features

- **[Language Code Normalization](./language-normalization.md)** - Improved voice selection via language code normalization (v0.9.18)

---

## Updates (v0.9.32-0.9.43)

### TTS Control View Hierarchy (v0.9.41, #1119, #1080)

**Refactoring**: Reorganized TTS control view hierarchy for better maintainability and UX.

**Improvements**:
- Cleaner component structure
- Better separation of concerns
- Easier to add new controls
- Improved mobile layout

### Don't Scroll When Selection in Current Page (v0.9.39, #1046, #863)

**Issue**: TTS auto-scrolling caused disorientation when current selection was already visible.

**Fix**: Only scroll when highlighted text is outside viewport.

```typescript
const highlightCurrentSentence = (sentenceCfi: string) => {
  const element = getElementByCFI(sentenceCfi);

  if (!isInViewport(element)) {
    scrollToElement(element, { behavior: 'smooth' });
  }

  element.classList.add('tts-highlight');
};
```

### iOS Timeout Options (v0.9.40, #1037)

**Feature**: TTS timeout options now properly popup on iOS.

**Fix**: Adjusted modal presentation for iOS Safari/WebView compatibility.

### Language Detection Improvements (v0.9.40, #1003)

**Enhancement**: More robust method to detect language for TTS voice selection.

**Implementation**:
1. Try book metadata language
2. Fallback to HTML lang attribute
3. Detect from text content
4. Default to system language

### Metadata Language Fallback (v0.9.37, #945, #916)

**Feature**: TTS now fallbacks to book metadata language when no specific language is detected.

**Priority Order**:
1. User-selected language (if manually chosen)
2. Content language (from HTML)
3. **Book metadata language** (NEW)
4. System language

### Extended English Voices (v0.9.38, #969, #966)

**Feature**: Added more English voices for all English locales (en-US, en-GB, en-AU, etc.).

**Available Voices**: 20+ English voices across different accents:
- en-US: 8 voices
- en-GB: 6 voices
- en-AU: 4 voices
- en-CA, en-IN, en-NZ, en-ZA: 2-3 voices each

---

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Bilingual TTS (v0.9.49-0.9.50, #1230, #1263)

**Major Feature**: Support for bilingual text-to-speech with different voices for different languages.

**Overview**: When reading bilingual books (with both original and translated text), TTS can now automatically switch between two voices based on language detection.

**Key Features**:
- Automatic language detection per sentence/paragraph
- Two voice selection (one for each language)
- Seamless voice switching during playback
- Language inference from script type
- Voice pairing persistence per book

**Implementation** (`src/app/reader/utils/tts/TTSController.ts`):
```typescript
interface BilingualTTSConfig {
  primaryVoice: Voice;      // For primary language (e.g., English)
  secondaryVoice: Voice;    // For secondary language (e.g., Chinese)
  primaryLang: string;
  secondaryLang: string;
}

class TTSController {
  async playBilingual(text: string, config: BilingualTTSConfig) {
    const detectedLang = inferLanguageFromScript(text);
    const voice = detectedLang === config.primaryLang
      ? config.primaryVoice
      : config.secondaryVoice;

    await this.speak(text, voice);
  }
}
```

**UI Changes** (`src/app/reader/components/TTSPanel.tsx`):
- Two voice dropdowns when bilingual mode enabled
- Language pair selector
- Visual indicator showing which voice is currently speaking
- Automatic voice pairing based on book metadata

**Usage Flow**:
1. Enable bilingual translation for a book
2. Open TTS panel
3. System detects two languages in book
4. User selects voice for each language
5. During playback, TTS automatically switches voices based on detected language

### Bilingual TTS Language Inference (v0.9.50, #1233)

**Feature**: Always infer language code from script in bilingual TTS mode.

**Implementation**:
- Script-based detection (Latin, CJK, Arabic, Cyrillic, etc.)
- Fallback to metadata language if script ambiguous
- Per-sentence language detection for mixed content
- Caching of language detection results

**Supported Scripts**:
- Latin (English, French, Spanish, etc.)
- CJK (Chinese, Japanese, Korean)
- Arabic
- Cyrillic (Russian, Ukrainian, etc.)
- Devanagari (Hindi, Sanskrit, etc.)
- Thai, Hebrew, Greek

**File**: `src/app/reader/utils/tts/languageInference.ts`

### Voice Selection in Bilingual TTS (v0.9.50, #1263)

**Feature**: Enhanced voice selection UI for bilingual mode.

**Components**:
- Primary voice dropdown (language 1)
- Secondary voice dropdown (language 2)
- Language pair indicator
- Preview buttons for each voice
- Voice quality indicators (neural vs standard)

**Voice Filtering**:
- Filter voices by detected book languages
- Show only compatible voices for each language
- Mark recommended voices (neural quality)
- Recent voice history

### Native Android TTS (v0.9.56-0.9.57, #1376, #1387, #1394)

**Major Feature**: Integration with Android's native TTS engine as an additional backend option.

**Overview**: Users on Android can now use the system's built-in TTS engine, which offers better integration with device settings and potential offline support.

**Backends Available on Android**:
1. **Edge TTS** - Microsoft neural voices (default, requires network)
2. **Web Speech API** - Browser-based synthesis
3. **Native Android TTS** - System TTS engine (NEW)

**Architecture**:

```
TTSController
├── EdgeTTSBackend (WebSocket to Microsoft)
├── WebSpeechBackend (Browser API)
└── AndroidTTSBackend (Tauri plugin) ← NEW
```

**Implementation** (`src-tauri/src/plugins/tts.rs`):
```rust
use android_speech::{TextToSpeech, UtteranceProgressListener};

#[tauri::command]
fn speak_android(text: String, voice: String, rate: f32) -> Result<()> {
    let tts = TextToSpeech::new()?;
    tts.set_voice(&voice)?;
    tts.set_speech_rate(rate)?;
    tts.speak(&text, QueueMode::Flush, None)?;
    Ok(())
}
```

**Frontend Integration** (`src/app/reader/utils/tts/AndroidTTSBackend.ts`):
```typescript
export class AndroidTTSBackend implements TTSBackend {
  async speak(text: string, voice: Voice, rate: number): Promise<void> {
    if (!isAndroid) {
      throw new Error('Android TTS only available on Android platform');
    }

    await invoke('speak_android', {
      text,
      voice: voice.name,
      rate
    });
  }

  async getVoices(): Promise<Voice[]> {
    const voices = await invoke('get_android_voices');
    return voices.map(v => ({
      name: v.name,
      lang: v.locale,
      backend: 'android'
    }));
  }
}
```

**Benefits**:
- Offline TTS support (if voices installed)
- Better battery efficiency
- Seamless integration with Android accessibility settings
- Respects system TTS settings
- Works with Google TTS, Samsung TTS, etc.

**Hotfix** (v0.9.57, #1394):
- Resolved compatibility issues with Android Text-to-Speech API
- Fixed voice enumeration on older Android versions
- Improved error handling and fallback logic

### TTS Media Session Integration (v0.9.51, #1289)

**Feature**: Display speaking sentence and chapter info in TTS media session.

**Implementation**:
- Media session metadata updated in real-time
- Shows current sentence being spoken
- Displays chapter title and book information
- Album art shows book cover

**UI Integration**:
- Android notification panel shows TTS controls
- Lock screen displays current sentence
- Wear OS integration for smartwatches
- Desktop media controls (macOS, Windows, Linux)

**File**: `src/app/reader/utils/tts/mediaSession.ts`

### Read from Last Speaking Location (v0.9.51, #1291, #1293)

**Feature**: TTS remembers and resumes from the last spoken sentence.

**Implementation** (`src/app/reader/utils/tts/TTSController.ts`):
```typescript
interface TTSPosition {
  cfi: string;              // EPUB CFI of last sentence
  sentenceIndex: number;    // Index within chapter
  timestamp: number;        // When last played
}

// Save position on pause/stop
savePosition() {
  const position: TTSPosition = {
    cfi: this.currentCFI,
    sentenceIndex: this.currentSentenceIndex,
    timestamp: Date.now()
  };
  localStorage.setItem(`tts_position_${bookHash}`, JSON.stringify(position));
}

// Restore position on play
async restorePosition() {
  const saved = localStorage.getItem(`tts_position_${bookHash}`);
  if (saved) {
    const position = JSON.parse(saved);
    await this.seek(position.cfi, position.sentenceIndex);
  }
}
```

**User Experience**:
1. User starts TTS playback
2. User closes app or stops playback
3. User reopens book later
4. User clicks Play button
5. TTS resumes from last sentence (not from beginning)

**More Robust Implementation** (v0.9.51, #1293):
- Multiple save points for redundancy
- Validation of saved CFI before restoration
- Fallback to chapter start if position invalid
- Cross-device position sync (if cloud sync enabled)

### Skip Empty Speech at Chapter End (v0.9.49, #1243)

**Feature**: TTS now skips speaking when text is empty at the end of some chapters.

**Issue**: Some EPUB files have empty `<p>` tags or whitespace-only elements at chapter boundaries, causing TTS to pause unnecessarily.

**Fix**:
```typescript
const filterEmptySentences = (sentences: string[]) => {
  return sentences.filter(s => {
    const trimmed = s.trim();
    return trimmed.length > 0 && !/^[\s\n\r]+$/.test(trimmed);
  });
};
```

**File**: `src/app/reader/utils/tts/sentenceSplitter.ts`

### Each Book View Has Its Own TTS Controller (v0.9.58, #1411)

**Feature**: When viewing multiple books simultaneously (split-screen or tabs), each book now has its own independent TTS controller.

**Benefits**:
- Play TTS in one book without affecting others
- Different voices/rates for different books
- Isolated playback state
- Better resource management

**Implementation**:
```typescript
// readerStore.ts
interface ViewState {
  id: string;
  book: Book;
  ttsController: TTSController; // ← One per view
  // ...
}
```

### TTS Keyboard Shortcut (v0.9.57, #1405)

**Feature**: Press `T` key to toggle TTS playback on/off.

**Shortcuts**:
- `T` - Toggle play/pause
- `Shift+T` - Stop TTS
- `[` - Previous sentence
- `]` - Next sentence
- `Ctrl/Cmd+T` - Open TTS settings panel

**File**: `src/app/reader/hooks/useKeyboardShortcuts.ts`

### Annotation Tools Work with TTS (v0.9.58, #1406)

**Feature**: Annotation tools (highlight, note, translate) now function properly when TTS is enabled.

**Issue**: Previously, text selection for annotations conflicted with TTS sentence highlighting.

**Fix**:
- Separate event handlers for TTS and annotations
- TTS highlighting uses read-only overlay
- User text selections work independently
- Both systems coexist without interference

**Files**:
- `src/app/reader/components/annotator/Annotator.tsx`
- `src/app/reader/utils/tts/TTSController.ts`

### Translation with Background TTS (v0.9.57, #1399)

**Feature**: Translation popup now works when TTS is playing in the background.

**Implementation**:
- TTS continues playing while translation popup is open
- Translation doesn't interrupt TTS playback
- Close translation popup to return focus to TTS

**Use Case**: User listens to audiobook while occasionally looking up translations.

### TTS Disables Media Session to Keep Alive (v0.9.52, #1333)

**Feature**: Disabled media session controls to prevent TTS from being suspended on some platforms.

**Issue**: Media session "stop" event from system would terminate TTS unexpectedly.

**Compromise**:
- Media controls disabled on problematic platforms
- TTS remains active in background
- Manual controls in app still functional

**Note**: Re-enabled in later versions with better event handling.

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### Handle Invalid Language Codes (v0.9.65, #1607)

**Enhancement**: Improved handling of invalid or unsupported language codes for TTS.

**Problem**: Books with invalid language codes would cause TTS to fail silently or show confusing errors.

**Solution**:

1. **Language Code Validation**: Check if language code is supported before TTS initialization
   ```typescript
   const isLanguageSupported = (lang: string): boolean => {
     const normalizedLang = normalizeLanguageCode(lang);
     const availableVoices = speechSynthesis.getVoices();
     return availableVoices.some(voice => voice.lang.startsWith(normalizedLang));
   };
   ```

2. **No Voices Hint**: Display helpful message when no voices are available
   ```typescript
   if (availableVoices.length === 0) {
     toast.warning('No TTS voices available for this language. Try changing the book language in settings.');
     showVoiceInstallationGuide();
   }
   ```

3. **Fallback Handling**:
   - Attempt to use closest language match
   - Default to system default voice if no match
   - Show language mismatch warning to user

**User Experience**:
- Clear error messages explaining the issue
- Suggested actions (install voice pack, change language)
- Link to system TTS settings

**Files**:
- `src/app/reader/utils/tts/TTSController.ts` - Language validation
- `src/components/Toast.tsx` - User notifications
- `src/utils/language.ts` - Language normalization

### Skip TTS for Rubys and Footnote Anchors (v0.9.65, #1608)

**Enhancement**: Improved TTS reading quality by skipping ruby annotations and footnote anchor text.

**Background**:
- **Ruby annotations**: Used in CJK texts to show pronunciation (e.g., furigana in Japanese)
- **Footnote anchors**: Superscript numbers/symbols linking to footnotes

**Problem**: TTS would read both the base text and ruby text, causing:
- Duplicated pronunciation in Japanese books
- Confusing number reading for footnotes (e.g., "Chapter One1")

**Solution**:

```typescript
const shouldSkipElement = (element: Element): boolean => {
  const tagName = element.tagName.toLowerCase();

  // Skip ruby annotations
  if (tagName === 'ruby' || tagName === 'rt' || tagName === 'rp') {
    return true;
  }

  // Skip footnote anchors
  if (
    tagName === 'a' &&
    (element.getAttribute('epub:type') === 'noteref' ||
     element.classList.contains('footnote-ref'))
  ) {
    return true;
  }

  return false;
};

// Extract text for TTS
const extractTextForTTS = (container: HTMLElement): string => {
  const walker = document.createTreeWalker(
    container,
    NodeFilter.SHOW_TEXT,
    {
      acceptNode: (node) => {
        const parent = node.parentElement;
        return shouldSkipElement(parent)
          ? NodeFilter.FILTER_REJECT
          : NodeFilter.FILTER_ACCEPT;
      }
    }
  );

  let text = '';
  let node;
  while (node = walker.nextNode()) {
    text += node.textContent;
  }

  return text;
};
```

**Benefits**:
- Cleaner TTS output for CJK books
- No interruption from footnote markers
- Better listening experience

**Related Issue**: #1334 (original request)

**Files**:
- `src/app/reader/utils/tts/textExtractor.ts` - Text extraction logic
- `src/app/reader/utils/tts/TTSController.ts` - TTS controller

### Convert ISO 639-2 to ISO 639-1 for Voice Filtering (v0.9.66, #1639)

**Enhancement**: Improved TTS voice matching by converting 3-letter ISO 639-2 language codes to 2-letter ISO 639-1 codes.

**Background**:
- Book metadata often uses ISO 639-2 (3-letter codes): `eng`, `jpn`, `fra`
- TTS voices use ISO 639-1 (2-letter codes): `en`, `ja`, `fr`
- Mismatch prevented proper voice filtering

**Problem**: Books with ISO 639-2 language codes would not find matching TTS voices.

**Solution**:

```typescript
// Language code conversion map
const ISO_639_2_TO_639_1: Record<string, string> = {
  'eng': 'en',
  'jpn': 'ja',
  'fra': 'fr',
  'deu': 'de',
  'spa': 'es',
  'ita': 'it',
  'por': 'pt',
  'rus': 'ru',
  'zho': 'zh',
  'ara': 'ar',
  // ...complete mapping
};

const normalizeLanguageCode = (code: string): string => {
  const lower = code.toLowerCase();

  // Already ISO 639-1 (2 letters)
  if (lower.length === 2) {
    return lower;
  }

  // Convert ISO 639-2 to ISO 639-1 (3 letters -> 2 letters)
  if (lower.length === 3) {
    return ISO_639_2_TO_639_1[lower] || lower.slice(0, 2);
  }

  // Handle extended codes (e.g., 'en-US' -> 'en')
  return lower.split('-')[0] || lower;
};

// Filter voices by book language
const getVoicesForLanguage = (bookLanguage: string): SpeechSynthesisVoice[] => {
  const normalizedLang = normalizeLanguageCode(bookLanguage);
  const voices = speechSynthesis.getVoices();

  return voices.filter(voice =>
    voice.lang.toLowerCase().startsWith(normalizedLang)
  );
};
```

**Testing**: Verified with books using ISO 639-2 codes in metadata.

**Related**: Also documented in [text-to-speech/language-normalization.md](./language-normalization.md)

**Files**:
- `src/utils/language.ts` - Language code conversion
- `src/app/reader/utils/tts/voiceManager.ts` - Voice filtering
- `src/app/reader/utils/tts/TTSController.ts` - Voice selection

---

## Version 0.9.68 - 0.9.78 Updates (33b2ba16 → cc3cc58d)

### TTS Audio Performance Improvement (v0.9.75, #1853)

**Enhancement**: Reuse audio object for better performance instead of creating new audio element for each sentence.

**Problem**: Previous implementation created a new `Audio` object for every sentence, leading to:
- Memory overhead from multiple audio elements
- Garbage collection pressure
- Slight delay between sentences
- Unnecessary DOM manipulation

**Solution**: Single `Audio` object reused throughout playback session:

```typescript
class TTSController {
  private audioElement: HTMLAudioElement | null = null;

  async play() {
    // Create audio element only once
    if (!this.audioElement) {
      this.audioElement = new Audio();

      // Set up event listeners once
      this.audioElement.addEventListener('ended', () => this.onAudioEnded());
      this.audioElement.addEventListener('error', (e) => this.onAudioError(e));
    }

    // Reuse the same audio element for each sentence
    const audioBlob = await this.backend.synthesize(
      this.sentences[this.currentSentenceIndex],
      this.voice,
      this.rate
    );

    // Update source and play
    this.audioElement.src = URL.createObjectURL(audioBlob);
    await this.audioElement.play();
  }

  cleanup() {
    // Clean up when TTS session ends
    if (this.audioElement) {
      this.audioElement.pause();
      this.audioElement.src = '';
      this.audioElement = null;
    }
  }
}
```

**Performance Benefits**:
- Reduced memory footprint
- Faster playback transitions
- Smoother sentence-to-sentence flow
- Lower CPU usage
- Better battery life on mobile devices

**Before vs After**:

| Metric | Before (New Audio Each Time) | After (Reused Audio) |
|--------|------------------------------|---------------------|
| Memory usage | ~2-5 MB per sentence | ~1 MB total |
| Transition delay | ~50-100ms | ~10-20ms |
| GC frequency | High | Low |
| Audio elements | 100+ for typical chapter | 1 per session |

**Files**:
- `src/app/reader/utils/tts/TTSController.ts` - Audio object reuse logic
- `src/app/reader/utils/tts/EdgeTTSBackend.ts` - Backend integration
- `src/app/reader/utils/tts/WebSpeechBackend.ts` - Backend integration

---

## Version 0.9.79 - 0.9.82 Updates (cc3cc58d → e1691661)

### Background TTS with Media Session Controls (v0.9.80, #2071, #2138)

**Major Feature**: Full background TTS support with system-level media controls.

**Overview**: TTS can now continue playing when the app is in the background, with full integration into system media controls (notification panel on mobile, media keys on desktop).

**Desktop Media Session** (v0.9.80, #2071):
- Integration with desktop media controls
- Media keys (play/pause, next/previous) control TTS playback
- System notification shows current sentence being read
- Album art displays book cover

**Implementation** (`src/app/reader/utils/tts/mediaSession.ts`):
```typescript
const updateMediaSession = (book: BookMetadata, sentence: string, chapterTitle: string) => {
  if ('mediaSession' in navigator) {
    navigator.mediaSession.metadata = new MediaMetadata({
      title: book.title,
      artist: book.authors?.join(', ') || 'Unknown Author',
      album: chapterTitle,
      artwork: [
        {
          src: book.coverUrl || '/default-cover.png',
          sizes: '512x512',
          type: 'image/png'
        }
      ]
    });

    // Set action handlers
    navigator.mediaSession.setActionHandler('play', () => ttsController.play());
    navigator.mediaSession.setActionHandler('pause', () => ttsController.pause());
    navigator.mediaSession.setActionHandler('nexttrack', () => ttsController.forward());
    navigator.mediaSession.setActionHandler('previoustrack', () => ttsController.backward());
  }
};
```

**Background TTS** (v0.9.80, #2138):
- Continues playback when app is minimized or in background
- Maintains audio session even when screen is locked
- Auto-pauses on incoming calls or other audio events
- Resumes playback after interruptions

**Platform Support**:
- **Desktop**: Media keys on keyboard, system media controls
- **Android**: Notification panel controls, lock screen controls
- **iOS**: Lock screen controls, Control Center integration
- **Web/PWA**: Browser media notification

**Background Audio Management**:
```typescript
// Keep audio playing in background
const maintainBackgroundAudio = () => {
  // iOS: Silent audio loop to keep AudioContext active
  if (isiOS) {
    const silentAudio = new Audio('/silent.mp3');
    silentAudio.loop = true;
    silentAudio.play();
  }

  // Android: Foreground service notification
  if (isAndroid) {
    invoke('start_tts_foreground_service', {
      title: currentBook.title,
      author: currentBook.authors?.join(', ')
    });
  }
};
```

**User Experience**:
1. User starts TTS playback
2. User switches to another app or locks screen
3. TTS continues playing in background
4. System notification shows book title, current sentence
5. User can control playback from notification/lock screen
6. User returns to app - TTS state is maintained

**Files**:
- `src/app/reader/utils/tts/mediaSession.ts` - Media session integration
- `src/app/reader/utils/tts/backgroundAudio.ts` - Background audio management
- `src-tauri/src/tts_service.rs` - Native foreground service (Android)

### TTS Language Handling Improvements (v0.9.80-0.9.82)

**Handle 'und' Language Code** (v0.9.80, #2102):
- Books with 'und' (undefined) language code now default to system language
- Fallback to English if system language not available
- Improved voice selection for books with missing language metadata

**More Languages for Edge TTS** (v0.9.80, #2089):
- Expanded language support for Edge TTS backend
- Added voices for additional languages:
  - Nordic languages (Danish, Norwegian, Swedish, Finnish)
  - Eastern European (Czech, Polish, Romanian, Hungarian)
  - Asian languages (Vietnamese, Indonesian, Malay)
  - Middle Eastern (Turkish, Persian, Hebrew)
- Total languages now: 80+ languages with neural voices

**Parse Default Language in SSML** (v0.9.80, #2135):
- Extract language from SSML tags when available
- Better language detection for bilingual books
- Improved voice matching for mixed-language content

### TTS Audio Optimizations (v0.9.80-0.9.82)

**Compensate Audio Fade-In** (v0.9.80, #2006):
- Fixed audio fade-in issue when resuming playback on iOS/macOS
- Eliminated "pop" sound at start of each sentence
- Smoother audio transitions between sentences
- Better handling of iOS audio session interruptions

**Abortable Prefetch** (v0.9.80, #2110):
- Prefetching of TTS audio can now be aborted if user changes page
- Prevents wasted bandwidth and processing for unused audio
- Better resource management
- Fixes issue #2037 where prefetched audio continued even after user navigated away

**Implementation**:
```typescript
class TTSController {
  private prefetchAbortController: AbortController | null = null;

  async prefetchNextSentences(count: number = 2) {
    // Abort previous prefetch if still running
    if (this.prefetchAbortController) {
      this.prefetchAbortController.abort();
    }

    this.prefetchAbortController = new AbortController();
    const signal = this.prefetchAbortController.signal;

    for (let i = 1; i <= count; i++) {
      if (signal.aborted) break;

      const nextIndex = this.currentSentenceIndex + i;
      if (nextIndex < this.sentences.length) {
        await this.backend.prefetch(
          this.sentences[nextIndex],
          this.voice,
          this.rate,
          signal
        );
      }
    }
  }
}
```

**Files**:
- `src/app/reader/utils/tts/TTSController.ts` - Prefetch abort logic
- `src/app/reader/utils/tts/EdgeTTSBackend.ts` - Backend prefetch support
- `src/app/reader/utils/tts/iosAudioSession.ts` - iOS audio session handling

---

## Version 0.9.83 - 0.9.90 Updates (e1691661 → dd5371d2)

### Accurate Scrolling to Highlighted Text (v0.9.87, #2242)

**Enhancement**: More accurate scrolling to highlighted text in TTS when header/footer bars are shown.

**Problem**: Previously, TTS scroll position didn't account for header/footer bar heights, causing the highlighted text to be partially obscured.

**Solution**: Calculate visible viewport height and adjust scroll position accordingly.

**Implementation** (`src/app/reader/utils/tts/TTSController.ts`):
```typescript
class TTSController {
  private scrollToHighlightedText(element: HTMLElement) => {
    const rect = element.getBoundingClientRect();

    // Get header and footer heights
    const header = document.querySelector('.reader-header');
    const footer = document.querySelector('.reader-footer');
    const headerHeight = header?.getBoundingClientRect().height || 0;
    const footerHeight = footer?.getBoundingClientRect().height || 0;

    // Calculate visible area
    const visibleTop = headerHeight;
    const visibleBottom = window.innerHeight - footerHeight;
    const visibleHeight = visibleBottom - visibleTop;

    // Check if element is fully visible
    const isFullyVisible =
      rect.top >= visibleTop &&
      rect.bottom <= visibleBottom;

    if (!isFullyVisible) {
      // Center the highlighted text in visible area
      const scrollTarget = rect.top + window.scrollY - visibleTop - (visibleHeight / 2) + (rect.height / 2);

      window.scrollTo({
        top: scrollTarget,
        behavior: 'smooth'
      });
    }
  }

  private highlightCurrentSentence(sentence: string) {
    const highlightElement = this.findSentenceElement(sentence);

    if (highlightElement) {
      // Apply highlight styling
      highlightElement.classList.add('tts-highlight');

      // Scroll into view with header/footer offset
      this.scrollToHighlightedText(highlightElement);
    }
  }
}
```

**CSS for TTS Highlight**:
```css
.tts-highlight {
  background-color: rgba(255, 235, 59, 0.3);
  border-radius: 4px;
  padding: 2px 0;
  transition: background-color 0.2s ease;
}
```

**Benefits**:
- TTS highlighted text always visible
- No manual scrolling needed
- Better reading flow
- Accessibility improvement

**Files**:
- `src/app/reader/utils/tts/TTSController.ts` - Scroll calculation
- `src/styles/tts.css` - Highlight styling

### Fixed TTS Crashes on Android (v0.9.87, #2244)

**Fix**: Resolved crash issues with TTS on some Android system versions.

**Problem**: TTS was crashing on certain Android versions (especially older or custom ROMs) due to audio context handling issues.

**Root Cause**:
- Race condition in audio session initialization
- Improper cleanup of audio resources
- Android WebView audio policy conflicts

**Solution**:
```typescript
class AndroidTTSBackend {
  private audioContext: AudioContext | null = null;
  private isInitialized = false;

  async initialize() {
    if (this.isInitialized) return;

    try {
      // Request audio focus before initializing
      if (window.AndroidInterface) {
        await window.AndroidInterface.requestAudioFocus();
      }

      // Create audio context with Android-specific settings
      this.audioContext = new (window.AudioContext || window.webkitAudioContext)({
        latencyHint: 'playback',
        sampleRate: 44100  // Standard sample rate for Android
      });

      // Resume audio context (required on some Android versions)
      if (this.audioContext.state === 'suspended') {
        await this.audioContext.resume();
      }

      this.isInitialized = true;
    } catch (error) {
      console.error('Failed to initialize TTS on Android:', error);

      // Fallback to native Android TTS if available
      if (window.AndroidInterface?.ttsSpeak) {
        this.useFallbackTTS = true;
      } else {
        throw new Error('TTS not supported on this device');
      }
    }
  }

  async cleanup() {
    if (this.audioContext) {
      // Properly close audio context
      await this.audioContext.close();
      this.audioContext = null;
    }

    // Release audio focus
    if (window.AndroidInterface) {
      window.AndroidInterface.abandonAudioFocus();
    }

    this.isInitialized = false;
  }
}
```

**Additional Fixes**:
- Proper error handling for audio session failures
- Graceful degradation to native Android TTS
- Better lifecycle management (pause/resume/destroy)
- Memory leak prevention in long reading sessions

**Files**:
- `src/app/reader/utils/tts/AndroidTTSBackend.ts` - Android-specific TTS
- `src-tauri/src/android/tts.kt` - Native Android TTS wrapper

### Improved TTS Indicator Visibility (v0.9.87, #2248)

**Enhancement**: TTS indicator no longer disappears too quickly when trying to open configuration panel.

**Problem**: TTS indicator had a short auto-hide timeout, making it difficult to click and open settings.

**Solution**:
- Extended hover grace period
- Disable auto-hide when mouse is over indicator
- Keep indicator visible when settings panel is open

**Implementation**:
```typescript
const TTSIndicator = () => {
  const [isVisible, setIsVisible] = useState(true);
  const [isHovering, setIsHovering] = useState(false);
  const [showSettings, setShowSettings] = useState(false);
  const hideTimeout = useRef<NodeJS.Timeout | null>(null);

  const startHideTimer = () => {
    // Clear any existing timer
    if (hideTimeout.current) {
      clearTimeout(hideTimeout.current);
    }

    // Don't hide if hovering or settings open
    if (isHovering || showSettings) {
      return;
    }

    // Extended timeout: 5 seconds instead of 2
    hideTimeout.current = setTimeout(() => {
      setIsVisible(false);
    }, 5000);
  };

  const handleMouseEnter = () => {
    setIsHovering(true);
    // Cancel hide timer
    if (hideTimeout.current) {
      clearTimeout(hideTimeout.current);
    }
  };

  const handleMouseLeave = () => {
    setIsHovering(false);
    // Restart hide timer when mouse leaves
    if (!showSettings) {
      startHideTimer();
    }
  };

  return (
    <div
      className={`tts-indicator ${isVisible ? 'visible' : 'hidden'}`}
      onMouseEnter={handleMouseEnter}
      onMouseLeave={handleMouseLeave}
    >
      <button onClick={() => setShowSettings(!showSettings)}>
        <SpeakerIcon />
      </button>

      {showSettings && (
        <TTSSettingsPanel onClose={() => setShowSettings(false)} />
      )}
    </div>
  );
};
```

**User Experience Improvements**:
- 5-second visibility instead of 2 seconds
- Persistent when mouse hovering
- Always visible when settings panel open
- Smooth fade transitions

**Files**:
- `src/app/reader/components/tts/TTSIndicator.tsx` - Indicator component
- `src/styles/tts-indicator.css` - Indicator styling

### Default English Voice Change (v0.9.88, #2272)

**Change**: Avoid using AnaNeural as default English voice due to quality issues.

**Reasoning**:
- AnaNeural voice has unnatural pronunciation
- Users reported poor listening experience
- Better alternatives available

**New Default Voice Priority** (for English):
1. **JennyNeural** (US English, female) - Primary
2. **GuyNeural** (US English, male) - Secondary
3. **AriaNeural** (US English, female) - Tertiary
4. AnaNeural - Last resort

**Implementation**:
```typescript
const getDefaultEnglishVoice = (availableVoices: Voice[]): Voice | null => {
  const preferredVoices = [
    'en-US-JennyNeural',
    'en-US-GuyNeural',
    'en-US-AriaNeural',
    'en-GB-SoniaNeural',
    'en-GB-RyanNeural',
    'en-AU-NatashaNeural'
    // AnaNeural excluded from preferred list
  ];

  // Try to find a preferred voice
  for (const preferredName of preferredVoices) {
    const voice = availableVoices.find(v => v.name === preferredName);
    if (voice) return voice;
  }

  // Fallback to any English voice (including AnaNeural if nothing else available)
  return availableVoices.find(v => v.lang.startsWith('en')) || null;
};
```

**Files**:
- `src/app/reader/utils/tts/voiceSelection.ts` - Voice priority logic

### Target Language Selection for TTS on Translated Books (v0.9.89, #2310)

**Major Feature**: Select target language for TTS when reading translated books.

**Overview**: When reading a book with inline translation enabled, users can now choose whether TTS should read:
- Original language text
- Translated text
- Both (alternating or side-by-side)

**Use Cases**:
- Language learning: Hear both original and translation
- Accessibility: Listen in preferred language
- Comparison: Understand pronunciation differences

**Settings Location**: TTS Panel > "Translation Reading Mode"

**Implementation** (`src/app/reader/components/tts/TTSSettings.tsx`):
```typescript
interface TTSTranslationSettings {
  mode: 'original' | 'translation' | 'both';
  bothMode: 'alternate' | 'simultaneous';  // Only when mode='both'
  originalVoice?: Voice;
  translationVoice?: Voice;
}

const TTSTranslationModeSelector = () => {
  const [settings, setSettings] = useState<TTSTranslationSettings>({
    mode: 'translation',  // Default: read translation
    bothMode: 'alternate'
  });

  const isTranslated = translationStore.getState().isEnabled;

  if (!isTranslated) {
    return null;  // Only show when translation is active
  }

  return (
    <div className="tts-translation-mode">
      <label className="label">
        <span className="label-text">{t('TTS Reading Mode')}</span>
      </label>

      <select
        value={settings.mode}
        onChange={(e) => setSettings({ ...settings, mode: e.target.value as any })}
        className="select select-bordered"
      >
        <option value="original">{t('Read Original Text')}</option>
        <option value="translation">{t('Read Translation')}</option>
        <option value="both">{t('Read Both')}</option>
      </select>

      {settings.mode === 'both' && (
        <>
          <select
            value={settings.bothMode}
            onChange={(e) => setSettings({ ...settings, bothMode: e.target.value as any })}
            className="select select-bordered mt-2"
          >
            <option value="alternate">{t('Alternate (Original → Translation)')}</option>
            <option value="simultaneous">{t('Side by Side')}</option>
          </select>

          <div className="voice-selection mt-4">
            <label className="label">
              <span className="label-text">{t('Voice for Original')}</span>
            </label>
            <VoicePicker
              language={bookLanguage}
              value={settings.originalVoice}
              onChange={(voice) => setSettings({ ...settings, originalVoice: voice })}
            />

            <label className="label mt-2">
              <span className="label-text">{t('Voice for Translation')}</span>
            </label>
            <VoicePicker
              language={translationLanguage}
              value={settings.translationVoice}
              onChange={(voice) => setSettings({ ...settings, translationVoice: voice })}
            />
          </div>
        </>
      )}
    </div>
  );
};
```

**TTS Controller Logic**:
```typescript
class TTSController {
  async speakSentence(sentence: string, translatedSentence?: string) {
    const mode = this.translationSettings.mode;

    switch (mode) {
      case 'original':
        await this.backend.speak(sentence, this.voice, this.rate);
        break;

      case 'translation':
        if (translatedSentence) {
          await this.backend.speak(translatedSentence, this.voice, this.rate);
        } else {
          // Fallback to original if no translation
          await this.backend.speak(sentence, this.voice, this.rate);
        }
        break;

      case 'both':
        if (this.translationSettings.bothMode === 'alternate') {
          // Speak original first
          await this.backend.speak(
            sentence,
            this.translationSettings.originalVoice || this.voice,
            this.rate
          );

          // Brief pause
          await this.sleep(500);

          // Then speak translation
          if (translatedSentence) {
            await this.backend.speak(
              translatedSentence,
              this.translationSettings.translationVoice || this.voice,
              this.rate * 0.9  // Slightly slower for translation
            );
          }
        } else {
          // Simultaneous mode: overlay or side-by-side audio (advanced feature)
          await this.speakBothSimultaneously(sentence, translatedSentence);
        }
        break;
    }
  }
}
```

**Benefits**:
- Language learning enhancement
- Flexibility for bilingual readers
- Accessibility for translated content
- Pronunciation comparison

**Files**:
- `src/app/reader/components/tts/TTSSettings.tsx` - Settings UI
- `src/app/reader/utils/tts/TTSController.ts` - Translation mode logic
- `src/types/tts.ts` - Type definitions

---

**Last Updated**: Documentation for commits through dd5371d2 (November 2025, v0.9.90)
**Related Documents**: [translation-system](../translation-system/index.md), [annotation-system](../annotation-system/index.md), [cross-platform-support](../cross-platform-support/index.md)
