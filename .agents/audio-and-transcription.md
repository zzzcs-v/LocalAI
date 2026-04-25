# Audio and Transcription

This guide covers how audio transcription (speech-to-text) and text-to-speech (TTS) are handled in LocalAI, including backend selection, file handling, and response formatting.

## Overview

LocalAI supports two audio-related capabilities:
- **Transcription (STT)**: Convert audio files to text (OpenAI `/v1/audio/transcriptions` compatible)
- **Text-to-Speech (TTS)**: Convert text to audio files (OpenAI `/v1/audio/speech` compatible)

Backends that support these include `whisper`, `piper`, `bark`, and `transformers-musicgen`.

## Transcription Flow

### Endpoint Handler

```go
func TranscriptionEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) func(c *fiber.Ctx) error {
    return func(c *fiber.Ctx) error {
        input := new(schema.AudioRequest)

        // Parse multipart form — audio is uploaded as a file
        if err := c.BodyParser(input); err != nil {
            return fiber.ErrBadRequest
        }

        // Get the uploaded file from the multipart form
        file, err := c.FormFile("file")
        if err != nil {
            return fiber.NewError(fiber.StatusBadRequest, "audio file is required")
        }

        // Save to a temp file so the backend can read it from disk
        tmpFile, err := os.CreateTemp("", "localai-audio-*.wav")
        if err != nil {
            return err
        }
        defer os.Remove(tmpFile.Name())

        if err := c.SaveFile(file, tmpFile.Name()); err != nil {
            return err
        }

        // Load config for the requested model
        cfg, err := cl.LoadBackendConfigFileByName(input.Model, appConfig.ModelPath)
        if err != nil {
            return err
        }

        // Run inference
        result, err := backend.ModelTranscription(tmpFile.Name(), input.Language, ml, cfg, appConfig)
        if err != nil {
            return err
        }

        return c.JSON(schema.TranscriptionResult{
            Text: result.Text,
        })
    }
}
```

### Key Points

- Audio is received as `multipart/form-data`, not JSON
- The file must be saved to disk before passing to the backend — most backends (e.g. whisper.cpp) expect a file path, not a byte slice
- `Language` is optional; if empty, whisper will auto-detect
- Always clean up temp files with `defer os.Remove(...)`

## Text-to-Speech Flow

### Endpoint Handler

```go
func TTSEndpoint(cl *config.BackendConfigLoader, ml *model.ModelLoader, appConfig *config.ApplicationConfig) func(c *fiber.Ctx) error {
    return func(c *fiber.Ctx) error {
        input := new(schema.TTSRequest)
        if err := c.BodyParser(input); err != nil {
            return fiber.ErrBadRequest
        }

        cfg, err := cl.LoadBackendConfigFileByName(input.Model, appConfig.ModelPath)
        if err != nil {
            return err
        }

        // TTS backends write audio to a file, then we stream it back
        outputFile, err := os.CreateTemp("", "localai-tts-*.wav")
        if err != nil {
            return err
        }
        defer os.Remove(outputFile.Name())
        outputFile.Close()

        if err := backend.ModelTTS(input.Input, input.Voice, outputFile.Name(), ml, cfg, appConfig); err != nil {
            return err
        }

        // Stream the audio file back to the client
        c.Set("Content-Type", "audio/wav")
        return c.SendFile(outputFile.Name())
    }
}
```

### Key Points

- TTS backends write to a file path you provide — they do not return bytes directly
- The `Voice` field maps to a speaker/voice model file (e.g. a `.onnx` file for piper)
- Set `Content-Type: audio/wav` (or `audio/mpeg` for mp3 output) before sending
- Use `c.SendFile()` to efficiently stream the file without loading it fully into memory

## Schema Types

```go
// AudioRequest is used for transcription (STT)
type AudioRequest struct {
    Model    string `form:"model"`
    Language string `form:"language"`
    Prompt   string `form:"prompt"`   // optional hint for whisper
    Format   string `form:"response_format"` // json, text, srt, vtt
}

// TTSRequest is used for text-to-speech
type TTSRequest struct {
    Model  string  `json:"model"`
    Input  string  `json:"input"`
    Voice  string  `json:"voice"`
    Speed  float64 `json:"speed"`
}

type TranscriptionResult struct {
    Text string `json:"text"`
}
```

## Backend Configuration

For whisper models, a typical `model.yaml` looks like:

```yaml
name: whisper-1
backend: whisper
parameters:
  model: whisper-base.en.bin
```

For piper TTS:

```yaml
name: tts-1
backend: piper
parameters:
  model: en_US-lessac-medium.onnx
```

## Testing Transcription

```go
func TestTranscriptionEndpoint(t *testing.T) {
    // Create a minimal WAV file for testing
    audioData := createSilentWAV(t, 1*time.Second)

    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)
    part, _ := writer.CreateFormFile("file", "test.wav")
    part.Write(audioData)
    writer.WriteField("model", "whisper-1")
    writer.Close()

    req := httptest.NewRequest(http.MethodPost, "/v1/audio/transcriptions", body)
    req.Header.Set("Content-Type", writer.FormDataContentType())

    // ... assert response contains `text` field
}
```

## Common Pitfalls

- **Don't forget to close the temp file** before passing its path to the backend — some backends will fail to open a file that's still held open
- **WAV vs MP3**: whisper.cpp expects WAV (16kHz mono). If the user uploads MP3, you may need to convert it first using ffmpeg or a Go audio library
- **Piper voice files** must be co-located with their `.onnx.json` config file — ensure both are present in the model directory
- **Large files**: For long audio, transcription can take significant time. Consider the request timeout settings in your fiber app config
