# Runtime Configuration

`RunConfig` defines runtime behavior and options for agents in the ADK. It
controls speech and streaming settings, function calling, artifact saving, and
limits on LLM calls.

When constructing an agent run, you can pass a `RunConfig` to customize how the
agent interacts with models, handles audio, and streams responses. By default,
no streaming is enabled and inputs aren’t retained as artifacts. Use `RunConfig`
to override these defaults.

---

## Class Definition

The `RunConfig` class is a Pydantic model that enforces strict validation of
configuration parameters.

```python
class RunConfig(BaseModel):
    """Configs for runtime behavior of agents."""

    model_config = ConfigDict(
        extra='forbid',
    )

    speech_config: Optional[types.SpeechConfig] = None
    response_modalities: Optional[list[str]] = None
    save_input_blobs_as_artifacts: bool = False
    support_cfc: bool = False
    streaming_mode: StreamingMode = StreamingMode.NONE
    output_audio_transcription: Optional[types.AudioTranscriptionConfig] = None
    max_llm_calls: int = 500
```

### Parameters

**`speech_config`**

**Type:** `Optional[types.SpeechConfig]` <br/>
**Default:** `None`

Speech configuration settings for live agents with audio capabilities. The
`SpeechConfig` class has the following structure:

```python
class SpeechConfig(_common.BaseModel):
    """The speech generation configuration."""

    voice_config: Optional[VoiceConfig] = Field(
        default=None,
        description="""The configuration for the speaker to use.""",
    )
    language_code: Optional[str] = Field(
        default=None,
        description="""Language code (ISO 639. e.g. en-US) for the speech synthesization.
        Only available for Live API.""",
    )
```

The `voice_config` parameter uses the `VoiceConfig` class:
```python
class VoiceConfig(_common.BaseModel):
    """The configuration for the voice to use."""

    prebuilt_voice_config: Optional[PrebuiltVoiceConfig] = Field(
        default=None,
        description="""The configuration for the speaker to use.""",
    )
```

And `PrebuiltVoiceConfig` has the following structure:

```python
class PrebuiltVoiceConfig(_common.BaseModel):
    """The configuration for the prebuilt speaker to use."""

    voice_name: Optional[str] = Field(
        default=None,
        description="""The name of the prebuilt voice to use.""",
    )
```

These nested configuration classes allow you to specify:

* `voice_config`: The name of the prebuilt voice to use (in the `PrebuiltVoiceConfig`)
* `language_code`: ISO 639 language code (e.g., "en-US") for speech synthesis

When implementing voice-enabled agents, configure these parameters to control
how your agent sounds when speaking.

**`response_modalities`**

**Type:**  `Optional[list[str]]` <br/>
**Default:** `None`

Defines the output modalities for the agent. If not set, defaults to AUDIO.
Response modalities determine how the agent communicates with users through
various channels (e.g., text, audio).


**`save_input_blobs_as_artifacts`**

**Type:** `bool` <br/>
**Default:** `False`

When enabled, input blobs will be saved as artifacts during agent execution.
This is useful for debugging and audit purposes, allowing developers to review
the exact data received by agents.

**`support_cfc`**

**Type:**  `bool` <br/>
**Default:** `False`

Enables Compositional Function Calling (CFC) support. Only applicable when using
StreamingMode.SSE. When enabled, the LIVE API will be invoked as only it
supports CFC functionality.

!!! warning

    This feature is experimental and its API or behavior may change in future
    releases.

**`streaming_mode`**

**Type:** `StreamingMode` (Enum) <br/>
**Default:** `StreamingMode.NONE`

Configures the streaming behavior of the agent. Possible values:

* `StreamingMode.NONE`: No streaming; responses delivered as complete units
* `StreamingMode.SSE`: Server-Sent Events streaming; one-way streaming from server to client
* `StreamingMode.BIDI`: Bidirectional streaming; simultaneous communication in both directions

Streaming modes affect both performance and user experience. SSE streaming lets users see partial responses as they're generated, while BIDI streaming enables real-time interactive experiences.

**`output_audio_transcription`**

**Type:** `Optional[types.AudioTranscriptionConfig]` <br/>
**Default:** `None`

Configuration for transcribing audio outputs from live agents with audio
response capability. This enables automatic transcription of audio responses for
accessibility, record-keeping, and multi-modal applications.

**`max_llm_calls`**

**Type:** `int` <br/>
**Default:** `500`

Sets a limit on the total number of LLM calls for a given agent run.

* Values greater than 0 and less than `sys.maxsize`: Enforces a bound on LLM calls
* Values less than or equal to 0: Allows unbounded LLM calls *(not recommended for production)*

This parameter prevents excessive API usage and potential runaway processes.
Since LLM calls often incur costs and consume resources, setting appropriate
limits is crucial.

---
## Validation Rules
The `RunConfig` class validates the `max_llm_calls` parameter:

1. If set to `sys.maxsize`, a `ValueError` is raised to prevent integer overflow issues
2. If less than or equal to 0, a warning is logged about potential unlimited LLM calls

---
## Examples

### Basic configuration
```python
from google.genai.adk import RunConfig, StreamingMode

config = RunConfig(
    streaming_mode=StreamingMode.NONE,
    max_llm_calls=100
)
```

This configuration creates a non-streaming agent with a limit of 100 LLM calls,
suitable for simple task-oriented agents where complete responses are
preferable.

### Enabling Streaming
```python
from google.genai.adk import RunConfig, StreamingMode

config = RunConfig(
    streaming_mode=StreamingMode.SSE,
    max_llm_calls=200
)
```
Using SSE streaming allows users to see responses as they're generated,
providing a more responsive feel for chatbots and assistants.

### With Speech Support
```python
from google.genai.adk import RunConfig, StreamingMode
from google.genai import types

config = RunConfig(
    speech_config=types.SpeechConfig(
        language_code="en-US",
        voice_config=types.VoiceConfig(
            prebuilt_voice_config=types.PrebuiltVoiceConfig(
                voice_name="Kore"
            )
        ),
    ),
    response_modalities=["AUDIO", "TEXT"],
    save_input_blobs_as_artifacts=True,
    support_cfc=True,
    streaming_mode=StreamingMode.SSE,
    max_llm_calls=1000,
)
```

This comprehensive example configures an agent with:

* Speech capabilities using the "Kore" voice with English US language
* Both audio and text output modalities
* Artifact saving for input blobs (useful for debugging)
* Experimental CFC support enabled
* SSE streaming for responsive interaction
* A limit of 1000 LLM calls

### With Experimental CFC Support

```python
from google.genai.adk import RunConfig, StreamingMode

config = RunConfig(
    streaming_mode=StreamingMode.SSE,
    support_cfc=True,
    max_llm_calls=150
)
```

Enabling Compositional Function Calling creates an agent that can dynamically
execute functions based on model outputs, powerful for applications requiring
complex workflows.
