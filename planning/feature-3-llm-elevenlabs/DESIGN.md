# Feature 3 -- LLM + ElevenLabs Integrations

## Status Quo

### Existing NeoVNext Integrations
- `SettingsIntegrationsSection.tsx` already shows Venice AI and ComfyUI as integration providers
- `IntegrationsController.cs` exists but only checks env vars for configured status
- No LLM service, no ElevenLabs integration in NeoVNext

### NeoComfy Reference Code (Fully Working)

**ElevenLabs:**
- `ElevenLabsVoiceService.cs`: Syncs all voices from ElevenLabs API to local DB, lists/filters voices
- `VoiceCloneService.cs`: Full voice cloning pipeline -- silence detection with ffmpeg, segment extraction, multipart upload to `POST https://api.elevenlabs.io/v1/voices/add`, stores voice_id on media record
- `VoicesController.cs`: `GET /api/voices` (list), `POST /api/voices/sync` (refresh from ElevenLabs)

**LLM Providers (all working in NeoComfy):**
- `OpenAiController.cs`: Chat completions with vision support, cost tracking
- `AnthropicController.cs`: Claude messages API with vision, cost tracking
- `GeminiService.cs`: Full Gemini integration with vision (image analysis, motion prompt generation, transform classification)
- Venice AI: Used in `WorkflowTestController.cs` via OpenAI-compatible API at `https://api.venice.ai/api/v1/chat/completions`

**Key Venice AI details from NeoComfy:**
- Venice uses OpenAI-compatible API format (Bearer token auth, `/chat/completions` endpoint)
- Model names: `venice-uncensored`, plus the video models in `VeniceModelCatalog`
- API key via env var `VENICE_API_KEY`
- Venice is the preferred uncensored LLM provider

---

## Architecture: Unified ILlmService

Rather than NeoComfy's pattern of separate controllers per provider, NeoVNext should have a unified `ILlmService` interface with provider implementations, plus a routing layer.

### Interface Definition

```csharp
public interface ILlmService
{
    /// <summary>Text-only completion.</summary>
    Task<LlmResponse> GenerateTextAsync(LlmRequest request, CancellationToken ct = default);

    /// <summary>Vision: analyze an image and return text.</summary>
    Task<LlmResponse> AnalyzeImageAsync(LlmImageRequest request, CancellationToken ct = default);

    /// <summary>Convenience: describe a person from a face crop image.</summary>
    Task<string?> DescribePersonFromImageAsync(string imagePath, CancellationToken ct = default);

    /// <summary>Convenience: summarize a transcript.</summary>
    Task<string?> SummarizeTranscriptAsync(string transcript, CancellationToken ct = default);
}

public record LlmRequest(
    string Prompt,
    string? SystemPrompt = null,
    string? Provider = null,    // null = use default provider
    string? Model = null,       // null = use provider default
    int MaxTokens = 1024,
    float Temperature = 0.7f);

public record LlmImageRequest(
    string Prompt,
    string ImagePath,           // local file path -- service handles base64 encoding
    string? SystemPrompt = null,
    string? Provider = null,
    string? Model = null,
    int MaxTokens = 1024,
    float Temperature = 0.7f) : LlmRequest(Prompt, SystemPrompt, Provider, Model, MaxTokens, Temperature);

public record LlmResponse(
    string Text,
    string Provider,
    string Model,
    int InputTokens,
    int OutputTokens,
    decimal EstimatedCostUsd);
```

### Provider Implementations

Each provider is an `ILlmProvider`:

```csharp
public interface ILlmProvider
{
    string Name { get; }
    bool SupportsVision { get; }
    Task<LlmResponse> CompleteAsync(LlmRequest request, CancellationToken ct);
    Task<LlmResponse> CompleteWithImageAsync(LlmImageRequest request, CancellationToken ct);
}
```

Implementations:
1. **VeniceLlmProvider** -- OpenAI-compatible, `https://api.venice.ai/api/v1`, uncensored models
2. **OpenAiLlmProvider** -- `https://api.openai.com/v1`, GPT-4o family
3. **AnthropicLlmProvider** -- `https://api.anthropic.com/v1`, Claude family
4. **GeminiLlmProvider** -- `https://generativelanguage.googleapis.com/v1beta`, free tier
5. **GrokLlmProvider** -- xAI, `https://api.x.ai/v1` (OpenAI-compatible format)

### LlmService (Router)

The `LlmService : ILlmService` reads the user's configured default provider from settings, then delegates to the appropriate `ILlmProvider`. If no provider is specified in the request, it falls through: Venice -> OpenAI -> Anthropic -> Gemini -> Grok (based on what has a configured API key).

### Venice AI Research

Venice AI confirmed OpenAI-compatible:
- **Base URL:** `https://api.venice.ai/api/v1`
- **Auth:** `Authorization: Bearer {VENICE_API_KEY}`
- **Chat endpoint:** `POST /chat/completions` with `{ model, messages, max_tokens, temperature }`
- **Models:** `venice-uncensored` (main), plus specific model IDs from catalog
- **Response format:** Standard OpenAI `choices[0].message.content`
- **Special:** No content filtering on uncensored models

xAI Grok confirmed OpenAI-compatible:
- **Base URL:** `https://api.x.ai/v1`
- **Auth:** `Authorization: Bearer {XAI_API_KEY}`
- **Chat endpoint:** `POST /chat/completions`
- **Models:** `grok-3`, `grok-3-mini`, `grok-2`
- **Vision:** Supported via standard OpenAI image_url format

---

## ElevenLabs Voice Cloning

### Database Schema (New columns on persons table -- coordinate with Feature 1)

Feature 1's migration 010 adds `elevenlabs_voice_id TEXT` to persons. Feature 3 uses this column.

### API Endpoints

```
POST   /api/persons/{id}/clone-voice
  - Finds the person's best voice sample (voice_sample_path from diarization)
  - Extracts a clean segment using ffmpeg silence detection (ported from NeoComfy VoiceCloneService)
  - Submits to ElevenLabs POST /v1/voices/add
  - Stores returned voice_id in persons.elevenlabs_voice_id
  - Returns { success, voiceId, voiceName }

GET    /api/persons/{id}/voice-status
  - Returns { hasVoice: bool, voiceId?: string, voiceName?: string }
  - If voiceId exists, optionally validates it's still active in ElevenLabs

DELETE /api/persons/{id}/voice
  - Calls ElevenLabs DELETE /v1/voices/{voiceId}
  - Clears persons.elevenlabs_voice_id
  - Returns { success }

GET    /api/voices/elevenlabs
  - Lists all ElevenLabs voices (synced to local cache)
  - Ported from NeoComfy's ElevenLabsVoiceService

POST   /api/voices/elevenlabs/sync
  - Refreshes voice list from ElevenLabs API
```

### Voice Cloning Service

Port `VoiceCloneService.cs` from NeoComfy with adaptations:
- Use the person's `voice_sample_path` instead of `mp3_path`
- Voice name format: `{person.name ?? "Person"} (NeoVLab)` instead of performer name
- ffmpeg path: use `ffmpeg` from PATH instead of hardcoded `C:\ffmpeg\bin\ffmpeg.exe`
- Store voice_id on `persons` table instead of `media_files`

### ElevenLabs Voice Service

Port `ElevenLabsVoiceService.cs` from NeoComfy:
- Sync voices from ElevenLabs to a local `elevenlabs_voices` table
- Query cached voices by category (cloned, premade, etc.)

### New SQLite Migration

```sql
-- elevenlabs_voices cache table (local API DB)
CREATE TABLE IF NOT EXISTS elevenlabs_voices (
    voice_id    TEXT PRIMARY KEY NOT NULL,
    name        TEXT NOT NULL,
    category    TEXT,
    preview_url TEXT,
    labels_json TEXT,
    synced_at   TEXT NOT NULL
);
```

---

## LLM Settings UI

### Settings Page Enhancement

Extend the existing `SettingsPage.tsx` and `SettingsIntegrationsSection.tsx`:

Add a new **"AI Providers"** settings section with:
- Provider list: Venice, OpenAI, Anthropic, Gemini, xAI Grok, ElevenLabs
- Each provider shows: name, status (configured/unconfigured), API key input field (masked), "Test" button
- Default LLM provider selector dropdown
- Default LLM model selector (filtered by selected provider)

### API Key Storage

Store API keys in the local SQLite settings system. The existing `IntegrationsController` checks env vars -- extend it to also check the settings DB:

```
GET  /api/integrations/status       -- returns configured status per provider
PUT  /api/integrations/{provider}   -- set API key for a provider
DELETE /api/integrations/{provider} -- remove API key
POST /api/integrations/{provider}/test -- test connectivity with stored key
```

The settings DB approach (not env vars) is better for the desktop app because users configure keys through the UI.

---

## Implementation Plan

### Phase A: ILlmService Interface + Venice Provider (Day 1)

1. Define `ILlmService`, `ILlmProvider` interfaces
2. Implement `VeniceLlmProvider` (highest priority, OpenAI-compatible)
3. Implement `LlmService` router
4. Register in DI
5. Add `POST /api/llm/chat` endpoint for testing
6. Add `GET /api/llm/providers` to list available providers

### Phase B: Additional LLM Providers (Day 1-2)

7. `OpenAiLlmProvider` (port from NeoComfy)
8. `AnthropicLlmProvider` (port from NeoComfy)
9. `GeminiLlmProvider` (port from NeoComfy GeminiService)
10. `GrokLlmProvider` (new, OpenAI-compatible like Venice)

### Phase C: ElevenLabs Integration (Day 2)

11. Create `elevenlabs_voices` SQLite migration
12. Port `ElevenLabsVoiceService` from NeoComfy
13. Port `VoiceCloneService` from NeoComfy (adapt for persons table)
14. Add PersonsController voice endpoints (clone, status, delete)
15. Add VoicesController (list, sync)

### Phase D: Settings UI (Day 2-3)

16. Extend `SettingsIntegrationsSection` with all providers
17. Add API key management (masked input, save, test)
18. Add default provider/model selectors
19. Add ElevenLabs voice management panel

### Phase E: Integration Endpoints (Day 3)

20. `POST /api/persons/{id}/generate-description` -- calls ILlmService with person's face crop
21. `POST /api/llm/summarize-transcript` -- transcript summarization
22. Wire description generation into Feature 1's pipeline (optional automatic trigger)

---

## Files to Create/Modify

### New Files (C#)
- `src/backend/NeoVLab.Api/Services/Llm/ILlmService.cs`
- `src/backend/NeoVLab.Api/Services/Llm/ILlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/Llm/LlmService.cs`
- `src/backend/NeoVLab.Api/Services/Llm/VeniceLlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/Llm/OpenAiLlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/Llm/AnthropicLlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/Llm/GeminiLlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/Llm/GrokLlmProvider.cs`
- `src/backend/NeoVLab.Api/Services/ElevenLabsVoiceService.cs`
- `src/backend/NeoVLab.Api/Services/VoiceCloneService.cs`
- `src/backend/NeoVLab.Api/Controllers/LlmController.cs`
- `src/backend/NeoVLab.Api/Controllers/VoicesController.cs`

### Modified Files (C#)
- `src/backend/NeoVLab.Api/Controllers/PersonsController.cs` -- add voice clone/status/delete endpoints
- `src/backend/NeoVLab.Api/Controllers/IntegrationsController.cs` -- extend for API key CRUD
- `src/backend/NeoVLab.Api/Program.cs` -- register new services in DI

### New Migrations
- `src/data/migrations/002_elevenlabs_voices.sql` (SQLite)
- `src/data/migrations/003_api_keys.sql` (SQLite) -- if not using existing settings table

### Frontend
- `src/frontend/src/components/SettingsIntegrationsSection.tsx` -- extend with all providers
- `src/frontend/src/api/client.ts` -- new API calls (LLM, voices, integrations)
- NEW: `src/frontend/src/components/settings/LlmProviderCard.tsx`
- NEW: `src/frontend/src/components/settings/VoiceManagementPanel.tsx`

---

## Shared Infrastructure

- **ILlmService interface**: Defined here, consumed by Feature 1 for person description generation.
- **API key storage**: Used by Feature 2 for Venice video generation API key.
- **ElevenLabs voice_id**: Stored on `persons.elevenlabs_voice_id` (column added by Feature 1's migration).

## Venice AI API Reference

```
Base:     https://api.venice.ai/api/v1
Auth:     Authorization: Bearer {key}
Chat:     POST /chat/completions  { model, messages[], max_tokens, temperature }
Image:    POST /images/generations { model, prompt, n, size }
Video:    POST /video/generations  { model, prompt, ... }
Models:   GET /models?type=chat|image|video
```

Chat models: `venice-uncensored`, `llama-3.3-70b`, `deepseek-r1-671b`, `qwen-2.5-vl` (vision)
Image models: `flux-dev`, `fluently-xl`
Video models: See VeniceModelCatalog in Feature 2 design doc
