# Ascend App — Voice Recording Handoff
Last updated: 2026-10-07

## Task
Fix Farsi voice recording pipeline:
**Tap mic → speak Farsi → Whisper transcribes → auto-translate to English**

## File
`~/Desktop/ascend/App.tsx` — single file, all edits here via device_bash + Python3 read/modify/write ONLY.

## What currently works (verified)
- ✅ Whisper API responds correctly to Farsi audio (browser-tested twice: MP3 and WAV)
- ✅ `FileSystem.uploadAsync` successfully sends audio to Whisper
- ✅ Translation (Google Translate) works
- ✅ `expo-speech-recognition` is installed and imported (v57.1.0)
- ✅ Brace balance verified: {=1893, }=1893, diff=0 — file is valid TypeScript

## CURRENT CODE STATE (after engine swap — NOT yet built/tested)

### Recording engine: expo-speech-recognition (NOT expo-audio)
The entire recording engine was swapped from `expo-audio`'s `useAudioRecorder` to `expo-speech-recognition`'s `ExpoSpeechRecognitionModule` because expo-audio's AVAudioRecorder produces silent audio on iOS (AVAudioRecorder.record() return value not checked in Swift — silent failure).

### startRecording
```typescript
const startRecording = async () => {
  isRecordingRef.current = true;
  isTranscribingRef.current = true;
  speechTranscriptRef.current = '';
  Keyboard.dismiss();
  setTrInput('');
  setTrOutput('');
  setTrLoading(false);
  try {
    const perm = await ExpoSpeechRecognitionModule.requestPermissionsAsync();
    if (!perm.granted) { setTrOutput('Permission denied'); isRecordingRef.current = false; isTranscribingRef.current = false; return; }
    ExpoSpeechRecognitionModule.start({
      lang: trSourceLang === 'fa' ? 'fa-IR' : trSourceLang,
      interimResults: true,
      continuous: true,
      recordingOptions: { persist: true, outputFileName: 'speech.wav', outputSampleRate: 16000, outputEncoding: 'pcmFormatInt16' },
    });
    recordingStartRef.current = Date.now();
    setIsRecording(true);
  } catch (e: any) {
    setTrOutput('Mic error: ' + (e?.message ?? String(e)).slice(0, 100));
    setIsRecording(false);
    isRecordingRef.current = false;
    isTranscribingRef.current = false;
  }
};
```

### stopAndTranscribe
```typescript
const stopAndTranscribe = async () => {
  isRecordingRef.current = false;
  setIsRecording(false);
  setTrLoading(true);
  setTrOutput('Processing…');
  ExpoSpeechRecognitionModule.stop();
};
```

### audioend handler — Whisper upload path
```typescript
useSpeechRecognitionEvent('audioend', async (event: any) => {
  if (!isTranscribingRef.current) return;
  isTranscribingRef.current = false;
  const uri = event?.uri;
  if (!uri) {
    const t = speechTranscriptRef.current.trim();
    if (!t) { setTrOutput('No speech detected'); setTrLoading(false); return; }
    setTrInput(t); doTranslate(t); setTrLoading(false); return;
  }
  try {
    const params: Record<string, string> = { model: 'whisper-1', language: trSourceLang, response_format: 'verbose_json', temperature: '0', prompt: 'این یک مکالمه فارسی است.' };
    const res = await FileSystem.uploadAsync('https://api.openai.com/v1/audio/transcriptions', uri,
      { httpMethod: 'POST', uploadType: FileSystem.FileSystemUploadType.MULTIPART, fieldName: 'file', mimeType: 'audio/wav', parameters: params, headers: { Authorization: `Bearer ${OPENAI_API_KEY}` } });
    const data: any = JSON.parse(res.body);
    if (data.error) { setTrOutput('Transcription error'); setTrLoading(false); return; }
    const segs: any[] = data.segments ?? [];
    const allSilent = segs.length > 0 && segs.every((s: any) => s.no_speech_prob > 0.6);
    if (allSilent) { setTrOutput('No speech detected'); setTrLoading(false); return; }
    const text = (data.text ?? '').trim();
    if (!text) { setTrOutput('No speech detected'); setTrLoading(false); return; }
    setTrInput(text);
    await doTranslate(text);
  } catch (e: any) {
    setTrOutput('Error: ' + (e?.message ?? String(e)).slice(0, 80));
  }
  setTrLoading(false);
});
```

## Full Failure History (chronological)

| # | What was tried | Error |
|---|---|---|
| 1 | `prepareToRecordAsync()` with no args | Corrupted audio / silence |
| 2 | Added `RecordingPresets.HIGH_QUALITY` | "No speech detected" (allSilent=true) |
| 3 | Fixed `trSourceLang` default 'en'→'fa' | "No speech detected" still |
| 4 | `fetch + FormData` with `{ uri, type, name }` | "Error: Unsupported FormDataPart implementation" — RN New Arch |
| 5 | Bad Python script deleted startRecording + stopAndTranscribe | Functions missing, app broken |
| 6 | Restored functions + WAV/LINEARPCM via expo-audio | Root cause still expo-audio |
| 7 | ENGINE SWAP: expo-speech-recognition, persist:true WAV | NOT YET BUILT/TESTED |

## Root Cause
expo-audio's AVAudioRecorder.record() return value NOT checked in Swift.
If it returns false, recording silently fails — no error thrown, no audio.

## Why expo-speech-recognition should work
- Uses SFSpeechRecognizer — completely different iOS audio stack
- persist:true saves real PCM audio to WAV file
- audioend event gives URI to upload via FileSystem.uploadAsync
- Bypasses broken AVAudioRecorder entirely

## If expo-speech-recognition still fails — next steps
1. Check if audioend fires with null URI (persist failed) → fall back to SFSpeechRecognizer interim text
2. Install expo-av: `npx expo install expo-av` — more battle-tested
3. Try react-native-audio-recorder-player as last resort

## Rules (NEVER break)
- NO diagnostics inside app UI — ever
- Edit App.tsx via device_bash + Python3 ONLY
- Path: os.path.expanduser('~') + '/mnt/Desktop/ascend/App.tsx'
- Never use device_commit_files
- Verify every edit with grep BEFORE telling Shaz to build
- Never redesign/resize/recolor visuals without explicit instruction
- Short replies, one action at a time (Shaz has eye strain)
- Shaz tested 80+ times — NO MORE USER TESTING until fix is verified

## Key facts
- Expo SDK 57 / React Native 0.86.3 / New Architecture (JSI)
- expo-audio v57.0.5 — ABANDONED (silent recording bug in Swift)
- expo-speech-recognition v57.1.0 — NOW THE RECORDING ENGINE
- OpenAI API key: in App.tsx line ~45
- Shaz: Laguna Hills CA, speaks clearly, quiet room
