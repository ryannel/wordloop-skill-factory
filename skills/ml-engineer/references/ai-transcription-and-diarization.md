# Transcription and Voice Diarization Pipeline

The `wordloop-ml` handles intensive audio pipelines. Standardize all audio transcript formatting immediately to the explicitly defined JSON structures, ensuring unified machine-readable tracking metrics cleanly.

## 2026 Standard Transcript JSON Data Structure
When transcribing, ensure output data directly conforms to the following format natively. Use `Pydantic` schemas to strictly validate this boundary:

```json
{
  "metadata": {
    "duration": 120.5,
    "language": "en-US",
    "created": "2026-04-12T09:00:00Z",
    "speakers": ["Speaker 1", "Speaker 2"]
  },
  "segments": [
    {
      "id": 1,
      "speaker": "Speaker 1",
      "start": 0.0,
      "end": 3.2,
      "text": "Welcome to today's podcast.",
      "confidence": 0.98
    }
  ]
}
```
*   **Rule:** Timestamps MUST be stored as `float` variables representing exact seconds mathematically. Never store strings like `"00:03:00"` at the database/domain boundary. Converts occur only at the Frontend presentation boundary natively.

## Pipeline Architecture
1.  **Phonetic Alignment:** Use `WhisperX` models to correctly establish extremely accurate word-level alignment across sequences efficiently.
2.  **Voice Cluster Diarization:** Implement clustering tools utilizing `Pyannote 3.1` (for heavy cloud inference) or `Picovoice Falcon` (for high-speed isolated on-device or massive-batch computing) to confidently separate distinct voices.
3.  **Authentication Embeddings:** For identity verification scenarios, securely extract embeddings mapping `ECAPA-TDNN` formats tracking cosine similarities natively explicitly cleanly.
