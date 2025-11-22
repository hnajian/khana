# Text-to-Speech Integration in Annotation System

**Added**: January 2025 (commits 07b04b82, 74021412)

## Overview

The Text-to-Speech (TTS) Integration in the annotation system allows users to listen to selected text using either Web Speech API or Microsoft Edge TTS service. This feature is accessible directly from the annotation toolbar.

## Key Components

**Primary Files:**
- **`src/app/reader/components/annotator/Annotator.tsx:339-343,358`** - "Speak" button handler
- **`src/services/tts/TTSController.ts`** - TTS orchestration controller
- **`src/services/tts/WebSpeechClient.ts`** - Web Speech API client
- **`src/services/tts/EdgeTTSClient.ts`** - Edge TTS service client
- **`src/utils/event.ts`** - Event dispatcher for TTS commands
- **`src/utils/ssml.ts`** - SSML generation for TTS

## Annotation Toolbar Integration

**"Speak" Button** (8th tool in annotator toolbar):

```typescript
// From Annotator.tsx:358
{ tooltipText: _('Speak'), Icon: FaHeadphones, onClick: handleSpeakText }
```

**Handler** (Annotator.tsx:339-343):
```typescript
const handleSpeakText = async () => {
  if (!selection || !selection.text) return;
  setShowAnnotPopup(false);
  eventDispatcher.dispatch('tts-speak', { bookKey, range: selection.range });
};
```

**Flow:**
1. User **selects text** in book
2. Annotation popup appears with 8 tools
3. User clicks **"Speak"** button (headphones icon)
4. `handleSpeakText()` dispatches `'tts-speak'` event
5. TTS controller receives event and starts speaking

## TTS Architecture

**Two-tier TTS Backend:**

1. **Web Speech API** (browser native):
   - Free, built-in browser TTS
   - Lower quality voices
   - Limited voice options
   - Works offline

2. **Edge TTS** (Microsoft cloud service):
   - High-quality neural voices
   - Many voice options per language
   - Requires internet connection
   - Free (uses Microsoft Edge TTS API)

**Dynamic Backend Selection:**
- User selects voice in TTS panel
- Controller automatically switches backend based on voice
- Web Speech voices → WebSpeechClient
- Edge TTS voices → EdgeTTSClient

## Event-Driven Communication

**TTS Events** (via `eventDispatcher`):

| Event | Payload | Purpose |
|-------|---------|---------|
| `tts-speak` | `{ bookKey, range }` | Start speaking selected text |
| `tts-play` | - | Resume playback |
| `tts-pause` | - | Pause playback |
| `tts-stop` | - | Stop playback |

**Listening for Events:**
```typescript
// In TTSController or TTS components
eventDispatcher.on('tts-speak', handleTTSSpeak);
```

## Text Processing

**SSML Generation** (`src/utils/ssml.ts`):
- Converts plain text to SSML (Speech Synthesis Markup Language)
- Handles emphasis, pauses, pronunciation
- Example:
  ```xml
  <speak>
    <s>Hello, world!</s>
    <break time="500ms"/>
    <s>This is text to speech.</s>
  </speak>
  ```

## Integration with Reading Flow

**Coordinated with Reader:**
- TTS can speak selected text (from annotation)
- TTS can also speak entire sections (from reader controls)
- Both use same TTS infrastructure
- Selection-based speech is one-shot (doesn't continue to next section)

## Modification Guidelines for AI Agents

### Adding Custom TTS Backend

To add a new TTS service (e.g., Google TTS):

1. **Create new client** `src/services/tts/GoogleTTSClient.ts`:
   ```typescript
   import { TTSClient } from './TTSClient';

   export class GoogleTTSClient implements TTSClient {
     async speak(text: string, voice: string): Promise<void> {
       // Implement Google TTS API call
     }

     async getVoices(): Promise<Voice[]> {
       // Fetch available voices
     }
   }
   ```

2. **Register in TTSController**:
   ```typescript
   // In TTSController.ts
   const googleClient = new GoogleTTSClient();

   const selectBackend = (voice: string) => {
     if (voice.startsWith('google-')) return googleClient;
     if (voice.startsWith('edge-')) return edgeClient;
     return webSpeechClient;
   };
   ```

3. **Add voice selection UI** in `TTSPanel.tsx`

### Customizing Speak Button Behavior

To change what happens when "Speak" is clicked:

**Example: Speak and highlight simultaneously**
```typescript
const handleSpeakText = async () => {
  if (!selection || !selection.text) return;

  // Highlight the text
  handleHighlight(true);

  // Start speaking
  setShowAnnotPopup(false);
  eventDispatcher.dispatch('tts-speak', {
    bookKey,
    range: selection.range,
    highlightWhileSpeaking: true  // Custom option
  });
};
```

### Adding Speak to Other Popups

To add "Speak" button to Wikipedia/Wiktionary popups:

1. **Edit popup component** (e.g., `WikipediaPopup.tsx`):
   ```typescript
   const handleSpeak = () => {
     eventDispatcher.dispatch('tts-speak', {
       bookKey,
       text: wikiContent  // Speak wiki content, not book text
     });
   };

   return (
     <Popup>
       {/* ... existing content ... */}
       <button onClick={handleSpeak}>
         <FaHeadphones /> Speak
       </button>
     </Popup>
   );
   ```

## Toolbar Button Order

**Current order** (Annotator.tsx:346-359):
1. **Copy** - Copy text to clipboard and notebook
2. **Highlight/Delete** - Toggle highlight on selected text
3. **Annotate** - Open note editor
4. **Search** - Search for text in book
5. **Dictionary** - Wiktionary lookup
6. **Wikipedia** - Wikipedia lookup
7. **Translate** - DeepL translation
8. **Speak** - Text-to-speech (NEW in Jan 2025)

**Modification:**
To change button order, reorder the `buttons` array in `Annotator.tsx:346-359`.

## Performance Considerations

**TTS-specific optimizations:**
- **Cache audio** for frequently read passages
- **Preload voices** on app startup
- **Debounce speak events** if user rapidly clicks
- **Cancel previous speech** when starting new

**Implementation:**
```typescript
let currentSpeech: Promise<void> | null = null;

const handleSpeakText = async () => {
  // Cancel ongoing speech
  if (currentSpeech) {
    eventDispatcher.dispatch('tts-stop');
  }

  currentSpeech = eventDispatcher.dispatch('tts-speak', {
    bookKey,
    range: selection.range
  });
};
```

## Common Issues

**Issue: "Speak" button not working**
- Check TTS controller is initialized
- Verify `eventDispatcher` is imported in Annotator
- Inspect browser console for TTS API errors
- Test if Web Speech API is supported (check browser)

**Issue: No voices available**
- Web Speech API may not support the language
- Edge TTS requires internet connection
- Check `TTSController.getVoices()` returns voices

**Issue: Poor voice quality**
- Switch to Edge TTS backend (select Edge voice)
- Adjust speech rate/pitch in TTS panel
- Some languages have limited Web Speech quality

**Issue: Speech interrupted when scrolling**
- TTS continues independently of scroll position
- Consider pausing TTS on scroll events
- Or implement visual indicator of speaking position

## Related Files

| File | Purpose |
|------|---------|
| `Annotator.tsx:339-343` | handleSpeakText handler |
| `Annotator.tsx:358` | Speak button definition |
| `TTSController.ts` | Main TTS orchestration |
| `WebSpeechClient.ts` | Browser TTS implementation |
| `EdgeTTSClient.ts` | Microsoft Edge TTS |
| `ssml.ts` | SSML generation utilities |
| `event.ts` | Event dispatcher for TTS |

---

**Related**:
- [index.md](./index.md) (Main annotation system documentation)
- [../text-to-speech/index.md](../text-to-speech/index.md) (Full TTS feature documentation)
