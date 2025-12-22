keyword:
[[VOIP]]

---
# Voice Chatting system(s)

#### Voice in Unreal historically has two “worlds”

1. **Legacy OSS Voice** (often exposed as `IOnlineVoice` and “talkers”)
2. **VoiceChat system** (Epic’s newer voice chat abstraction; may be used by OSS plugins)

Depending on your project/plugin, you might be using:

- OSS Voice (Steam voice, NULL voice, etc.)
- EOS Voice Chat / Vivox-like integration (varies)
- The “Online Voice” features integrated with sessions

#### What voice systems typically require

Voice routing usually needs:

- A **local user identity** (who is speaking)
- A **channel/session context** (who should hear whom)
- **remote talker IDs** (UniqueNetIds)
- Audio device/capture resources (which are global-ish and sensitive to teardown)

#### When you should initialize voice

- Prefer doing “voice manager init” in **GameInstance** or a **GameInstanceSubsystem**:
    - Acquire voice interface
    - Bind voice events
    - Start/stop voice capture based on match state

#### When you should attach voice to gameplay objects

- Don’t make voice _owned by_ Pawn or GameState.
- Use PlayerState only as an _identifier_ source, not as the owner of voice resources.
- It’s OK for PlayerController to _request_ voice changes, but the manager should live in GameInstance.

#### The two critical transitions for voice

- **Local login complete** (UniqueNetId becomes valid)
- **Session join/leave / travel** (membership changes; remote talkers change)

So your voice flow usually aligns with:

- OnLoginComplete → “local talker can exist”
- OnJoinSessionComplete / PostLogin / OnRep_PlayerState → “remote talkers can be registered”
- OnSeamlessTravel / OnDestroySession → “unregister/cleanup”

# Seamless travel and Voice chat
####  this breaks online/voice code

- If you bound OSS delegates in GameState or Pawn:
    - Those actors are destroyed → callbacks fire later → crash.
- If you cached pointers to World subsystems, audio components, net driver, etc.
    - They may be invalid during the transition.
- If you register talkers tied to PlayerState objects that persist, but you assumed GameState persists:
    - Your “voice channel membership” logic might get reset unexpectedly.

Voice actions you typically do:

- Before travel: stop capture / leave channel (optional depending on backend)
- After travel: re-assert session/channel membership and re-register remote talkers as needed