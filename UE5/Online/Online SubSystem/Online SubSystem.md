Keywords:
[[player state]]

##### Important:
- The new Multiplayer standard is ***EOS***. However:
> ***Gameplay framework lifecycle is still relevant.*** 
> Mostly the foundation (lifecycle + async/delegate model + separation from Gameplay Framework) is the same. What changes with EOS (and the newer “Online Services” direction in UE) is which interfaces you talk to, how identity/session/voice are modeled, and what must be true before an operation is valid (e.g., “logged in to EOS” vs “platform user logged in”, “in an EOS lobby” vs “in a UE session”, “joined a voice room” vs “registered talkers”). That difference is enough to feel like the old flow is being replaced—***but the core mental model remains***.

---

This is a life‑cycle / topology walkthrough of Unreal Engine’s Online Subsystem (OSS)** as it typically behaves in UE5, with special attention to **when things exist, when they’re valid, what survives travel. 
OSS will be described as a set of _layers_, then a _timeline_ (startup → login → session → gameplay → travel → teardown), then _common crash patterns_ and _safe patterns_.

> “Online Subsystem” in UE refers to the plugin architecture (`OnlineSubsystem`, `OnlineSubsystemSteam`, EOS, etc.) that provides services like Identity, Sessions, Voice, Friends, Presence, etc. **[[Gameplay Framework]]** (GameInstance/GameMode/PlayerController/PlayerState/etc.) is separate, but OSS integrates through it.

---

## 1) Mental model: the topology (what talks to what)

### 1.1 Core objects & modules (high level)

- **Engine process lifetime**
    - `UEngine` and modules load/unload
    - `OnlineSubsystem` module loads (depending on config)
      
- **Per-application lifetime**
    - **UGameInstance** (one per running game process)  
        Owns the long-lived managers you should prefer for online logic.
    - **UGameInstanceSubsystems** (excellent place for OSS wrappers)
    - **Online Subsystem instance** (one per named subsystem: `NULL`, `Steam`, `EOS`, etc.)
        - Accessed via `IOnlineSubsystem::Get()` or `Online::GetSubsystem(World)`
          
- **Per-local-user lifetime**
    - **ULocalPlayer** (one per local player/controller on that machine)
    - **Identity “local user index”** (`0..n`)
    - **UniqueNetId** for that user (becomes valid after login)
      
- **Per-connection / per-remote-player lifetime (networked match)**
    - Server: `APlayerController` per connection
    - Server & clients: `APlayerState` per player (replicated)
    - Usually: pawn(s) come/go

### 1.2 Online Subsystem interfaces (what you actually call)

An OSS instance exposes interfaces (some optional):

- `IOnlineIdentity` – login/logout, get UniqueNetId
- `IOnlineSession` – create/find/join/destroy session, register players
- `IOnlineVoice` – Voice Chat – voice capture/playback and routing
- `IOnlineFriends`, `IOnlinePresence`, `IOnlineUser`, `IOnlineParty`, etc.

**Key point:** each interface has its own internal state machine and async callbacks (delegates).

### 1.3 Delegate-driven async model (how it “flows”)

Most OSS operations:

1. You call a method (`Login`, `CreateSession`, `FindSessions`, `JoinSession`, etc.)
2. OSS starts async work
3. Later, on the **game thread**, it fires delegates:
    - `OnLoginComplete`
    - `OnCreateSessionComplete`
    - `OnFindSessionsComplete`
    - `OnJoinSessionComplete`
    - `OnDestroySessionComplete`
    - plus voice events / talker events depending on implementation

**Crashes often come from:**

- Delegates firing after the object that bound them is destroyed (stale `this`)
- Delegates firing during/after map travel when world changes and your world pointers become invalid
- Doing OSS calls from gameplay actors that do not persist (Pawn/GameState) and then expecting callbacks after travel

### 1.4 Two “worlds”: engine lifetime vs map lifetime

A huge source of confusion:

- **Online Subsystem + GameInstance** live across map loads.
- **World + GameMode + GameState + PlayerController/Pawn** may be recreated on travel (seamless travel keeps some actors, but not all).

This is why **online code belongs in GameInstance / GameInstanceSubsystem** most of the time.

---

## 2) The “who exists when” cheat sheet (Gameplay Framework vs OSS)

### 2.1 Persistency across travel (esp. seamless)

Typical (server seamless travel):

- **UGameInstance**: persists
- **OnlineSubsystem / interfaces**: persist (module-level; not map-bound)
- **PlayerController**: usually persists (seamless travel)
- **PlayerState**: usually persists (seamless travel)
- **GameMode**: recreated
- **GameState**: recreated
- **Pawn**: usually recreated (unless you explicitly preserve)

Non-seamless travel (open level):

- Most things are torn down and recreated; controllers on clients are recreated.

### 2.2 “World context” dependency

Some OSS helpers require a `UWorld` context (`Online::GetSubsystem(World)`), but the subsystem itself is not world-bound; it’s per game instance / per platform service.

**During travel, the old world is being destroyed while the new world is being created.**  
Anything that caches `World` pointers is risky if it spans travel.

---

## 3) The full life cycle, step-by-step (verbose timeline)

I’ll describe a fairly typical flow for a networked game with sessions and voice.

### Phase A — Process startup (before any map gameplay)

1. **Modules load**

- `OnlineSubsystem` module loads.
- Your specific subsystem plugin loads based on config:
    - `DefaultPlatformService=Steam` (or EOS, NULL, etc.)
- The subsystem instance may not be fully initialized until first access, depending on plugin.

2. **GameInstance created**

- `UGameInstance::Init()` is called.
- This is an ideal place to:
    - Acquire and store pointers/references to OSS interfaces (Identity/Session/etc.)
    - Bind global delegates (but do it carefully; see “safe binding” section)
    - Create your own “OnlineManager” subsystem

**Best practice:** create a `UGameInstanceSubsystem` like `UOnlineManagerSubsystem` and do your initialization in `Initialize(FSubsystemCollectionBase&)`.

3. **LocalPlayer(s) created**

- There will be at least one `ULocalPlayer` for local user index 0 (if not dedicated server).
- However, at this stage the player may not be logged in yet.

---

### Phase B — Identity/Login (getting a UniqueNetId)

4. **Call Login**

- Typically from UI or boot flow:
    - `Identity->Login(LocalUserNum, AccountCredentials)`
- Or you might do “auto login” if supported.

5. **OnLoginComplete delegate fires**

- This is a critical transition:
    - The local user now has a **valid UniqueNetId** (if success).
    - Presence/friends may begin to become meaningful.
    - Voice may require identity or user id mapping.

**Common mistakes:**

- Assuming `GetUniquePlayerId(0)` is valid before login.
- Triggering session/voice actions in `BeginPlay` of a pawn before login is complete.

**Safer pattern:**

- Gate everything online behind a “logged in” state in your GameInstance subsystem.
- Broadcast your own event `OnLocalUserReady`.

---

### Phase C — Sessions (hosting/joining a match)

6. **Host flow: CreateSession**

- Call `Session->CreateSession(LocalUserNum, SessionName, Settings)`
- You bind:
    - `OnCreateSessionComplete`
    - then `OnStartSessionComplete` (depending on usage)
- After create completes, host usually does a server travel:
    - `GetWorld()->ServerTravel(MapURL, /*bAbsolute*/false, /*bShouldSkipGameNotify*/false);`
    - with seamless travel enabled if desired.

7. **Join flow: FindSessions / JoinSession**

- `FindSessions` returns a list asynchronously.
- `JoinSession` attempts to join.
- On join success:
    - You ask for the connect string: `GetResolvedConnectString`
    - Then the client does `ClientTravel(Address, TRAVEL_Absolute);`

**Important:** session delegates can fire while you’re mid-travel or UI teardown. Don’t bind them on ephemeral actors.

---

### Phase D — Network connection & spawning (PostLogin, PlayerState, etc.)

#### On the **server**:

8. Client connects → `APlayerController` created (per connection)
9. `AGameModeBase::PreLogin` / `Login` / `PostLogin` flow
10. `GameMode::PostLogin(APlayerController*)`

- Server may:
    - Assign team, initialize PlayerState values
    - Register player with session: `Session->RegisterPlayer(...)` (depending on plugin/flow)
    - (Optionally) do voice registration/talker setup

11. Pawn spawn / possession

- `GameMode::RestartPlayer` / `SpawnDefaultPawnFor` / `Possess`
- `BeginPlay` order (simplified):
    - World actors begin play
    - GameMode BeginPlay (server only)
    - GameState BeginPlay (server & clients)
    - PlayerController BeginPlay
    - PlayerState BeginPlay
    - Pawn BeginPlay

#### On the **client**:

12. Client loads map → creates World
13. Replication brings down PlayerController/PlayerState/GameState
14. `APlayerController::BeginPlay` happens, but:

- PlayerState may not be replicated immediately.
- Session/Identity may still be in flux for late joiners.

**Common crash sources here:**

- Doing voice “register remote talker” in PlayerState `BeginPlay` before it has a valid UniqueNetId, or before voice system is initialized.
- Assuming `PlayerController->PlayerState` is non-null in `BeginPlay` on client.
- Binding to OSS delegates in `BeginPlay` of PC/PS and forgetting to unbind on travel.

---

### Phase D.2 —  Voice / VoIP (how [[Voice Chat]] fits into this lifecycle)

#### What voice systems typically require

Voice routing usually needs:

- A **local user identity** (who is speaking)
- A **channel/session context** (who should hear whom)
- **remote talker IDs** (UniqueNetIds)
- Audio device/capture resources (which are global-ish and sensitive to teardown)

#### When should initialize voice

- Prefer doing “voice manager init” in **GameInstance** or a **GameInstanceSubsystem**:
    - Acquire voice interface
    - Bind voice events
    - Start/stop voice capture based on match state

#### When should attach voice to gameplay objects

- Don’t make voice _owned by_ Pawn or GameState.
- Use PlayerState only as an _identifier_ source, not as the owner of voice resources.
- It’s OK for PlayerController to _request_ voice changes, but the manager should live in GameInstance.

#### The two critical transitions for voice

- **Local login complete** (UniqueNetId becomes valid)
- **Session join/leave / travel** (membership changes; remote talkers change)

So your voice flow usually aligns with:

- OnLoginComplete → “local talker can exist”
- OnJoinSessionComplete / PostLogin / OnRep_PlayerState → “remote talkers can be registered”
- OnSeamlessTravel / OnDestroySession → “unregister/cleanup”

---

### Phase E — Seamless travel (danger zone)

Seamless travel has **two worlds** briefly:

- Old world is still partially alive while new world loads.
- Some actors are “carried over” (PC/PS), others are recreated.

#### What happens conceptually

1. Server triggers seamless travel.
2. Engine creates a transition world / loads destination.
3. Certain actors are moved/preserved.
4. New GameMode/GameState spawn.
5. Players re-associate; pawns usually respawn.
6. Old world actors get EndPlay / Destroyed.

**Seamless travel safe points**

- **GameInstance** persists and can observe travel events:
    - `FCoreUObjectDelegates::PreLoadMap`
    - `FCoreUObjectDelegates::PostLoadMapWithWorld`
    - `UGameInstance::OnStart` / `LoadComplete`-style hooks
- **GameMode::PostSeamlessTravel** (server)
- **APlayerController::SeamlessTravelTo / SeamlessTravelFrom** (advanced)
- `AGameModeBase::HandleSeamlessTravelPlayer` (server)

---

### Phase F — Match end / leaving / teardown

Leaving a session

- Call `Session->EndSession` or `DestroySession`
- On completion:
    - For clients: `ClientTravel` back to main menu
    - For server: `ServerTravel` to lobby or shutdown

Logout / shutdown

- Unregister delegates
- Voice system
	- Stop voice capture
	- Release voice/chat resources
- OSS module unloads when process exits

**Dedicated server note:** voice capture usually irrelevant, but talker registration may still exist depending on backend.

---

## 4) Delegate & lifetime management ( #1 crash cause)

### 4.1 Typical crash pattern

- You bind a delegate with a raw `this` pointer (or bind in Blueprint) from an actor:
    - `GameState`, `PlayerState`, `Pawn`, sometimes `PlayerController`
- You travel / disconnect
- Actor is destroyed, but OSS operation completes later
- Delegate fires → calls into freed memory → crash

### 4.2 Safe binding rules

- Bind OSS delegates in **GameInstance** / **GameInstanceSubsystem** whenever possible.
- If you must bind from an Actor:
    - Use weak binding patterns and unbind in `EndPlay`
    - Store `FDelegateHandle`s and clear them deterministically
- Never assume “delegate won’t fire after travel”. It often will.

### 4.3 Gate everything by “current world / net driver validity”

When a callback fires, check:

- Is the subsystem still valid?
- Is the game instance still in the same “online state”?
- Is the world you care about current, not the old one?
- Are you in teardown (leaving session, traveling, quitting)?

Make a simple internal state machine in your manager:

- `EOnlineFlowState::Boot`
- `LoggingIn`
- `LoggedIn`
- `CreatingSession`
- `InSession`
- `Traveling`
- `LeavingSession`
- etc.

Then ignore callbacks that arrive in an unexpected state.

---

## 5) Where to put what (a practical architecture)

### Recommended “online stack” in your project

**UGameInstanceSubsystem: UOnlineFacadeSubsystem**

- Owns pointers to:
    - `IOnlineIdentity`
    - `IOnlineSession`
    - voice/voicechat interface (whatever you use)
- Owns all delegate handles
- Exposes clean events to gameplay/UI:
    - `OnAuthStateChanged`
    - `OnSessionStateChanged`
    - `OnVoiceStateChanged`
- Holds “current” values:
    - Local UniqueNetId
    - Current session name/settings
    - Who is registered as talker, etc.

**UI layer (menus)**

- Calls into OnlineFacade only.
- Does not touch OSS interfaces directly.

**Gameplay layer**

- PlayerController/PlayerState can _query_ OnlineFacade
- They do not bind OSS delegates directly (or only in rare, tightly-managed cases)

### Why this reduces crashes

- GameInstanceSubsystem persists across travel.
- It has a stable lifetime for delegate binding.
- You can centralize travel handling and cleanup.

---

## 6) How PlayerState / PostLogin / BeginPlay should interact with online

### Server: `PostLogin` is not “online ready”

`PostLogin` means: the connection exists and a PlayerController is created on server.

It does **not** guarantee:

- Session registration is done
- Client finished loading
- PlayerState has everything populated
- Voice channel is ready

Use `PostLogin` to:

- Initialize server-side gameplay state
- Optionally call `RegisterPlayer` for sessions
- Schedule voice registration only when you have the needed IDs

### Client: `BeginPlay` is not “replication ready”

On client, `BeginPlay` can run before:

- PlayerState is valid
- UniqueNetId replication is complete
- Session join callback is finished
- Voice system is initialized

So instead of `BeginPlay`, rely on:

- `APlayerController::OnRep_PlayerState`
- `APlayerState::OnRep_UniqueId` (if you use it / have access)
- Your OnlineFacade events (Login complete, session joined)

---

## 7) Seamless travel specifics: what to re-init vs preserve

During seamless travel:

- Your **OnlineFacadeSubsystem persists** and still has delegate bindings.
- **PlayerControllers persist**, but they may go through transitions.
- **PlayerStates persist**, but replicated properties may re-sync.
- **GameState is new**, so anything that you stored there as “online state” is gone.

Therefore:

- Session membership: usually persists at OSS level (you’re still in same session unless you explicitly leave).
- Voice: depending on backend, you might still be in same “voice room” or you may need to re-join/re-register talkers.
- Your manager should treat seamless travel as:
    - “world changed; rebind any world-specific references; keep online identity/session handles.”

---

## 8) Concrete “event timeline” you can instrument (highly recommended)

Add logging (or even an on-screen debug overlay) for these events in one place (GameInstanceSubsystem):

### Engine/map/travel events

- PreLoadMap(MapName)
- PostLoadMapWithWorld(NewWorld)
- OnWorldBeginPlay
- OnSeamlessTravelStart / PostSeamlessTravel (server hooks)
- OnNetworkFailure / OnTravelFailure

### Identity events

- OnLoginComplete
- OnLogoutComplete
- OnLoginStatusChanged (if supported)

### Session events

- OnCreateSessionComplete
- OnStartSessionComplete
- OnFindSessionsComplete
- OnJoinSessionComplete
- OnDestroySessionComplete
- OnSessionParticipantsChanged (if your backend supports it)

### Voice events (varies)

- Local voice started/stopped
- Remote talker registered/unregistered
- Player muted/unmuted
- Voice connection/channel joined/left (VoiceChat systems)

Once you see the real order in your game logs, 80% of the confusion disappears.

---

## 9) Common VoIP-related crash scenarios (not VoIP-only, but frequently triggered there)

1. **Registering talkers with invalid UniqueNetId**

- PlayerState exists but UniqueId not replicated yet
- Fix: register talker only after ID is valid; retry on replication event

2. **Voice manager owned by Pawn**

- Pawn destroyed on travel/respawn → voice capture object freed mid-stream
- Fix: voice capture manager in GameInstance; pawn just toggles speaking state

3. **Delegate callback after EndPlay**

- You call `JoinSession`, then travel away, then callback fires to an actor that’s gone
- Fix: bind in GameInstanceSubsystem; or clear delegate handles in `EndPlay`

4. **Using GameState to store online pointers**

- GameState recreated on travel → you lose interface pointers and state
- Fix: store in GameInstanceSubsystem

5. **Calling OSS from wrong thread / wrong timing**

- Most OSS calls should be from game thread; avoid calling during shutdown phases
- Fix: guard with “is tearing down / traveling” state, defer until stable

---

## 10) Practical rules of thumb (“don’t crash” checklist)

- Put **all OSS interface access** behind a **GameInstanceSubsystem**.
- Never bind OSS delegates in **Pawn**. Rarely in GameState. Be careful in PC/PS.
- Treat these as **not safe** for “online is ready”:
    - `BeginPlay` (client)
    - `PostLogin` (server)
- Treat these as better gates:
    - `OnLoginComplete` (identity ready)
    - `OnJoinSessionComplete` + successful travel (session + world ready)
    - Replication events for PlayerState UniqueId (remote talker ready)
- On travel:
    - Set state = Traveling
    - Stop/pause voice capture if backend/device is sensitive
    - Clear world-bound references
    - After travel, re-assert voice/session invariants

---

## 11) Quick questions that would let me tailor this to your exact crash points

If you answer these, I can map your exact lifecycle and suggest the safest hook points for your project:

1. Which OSS backend: **Steam / EOS / NULL / PSN / Xbox / custom**?
2. Are you using **legacy `IOnlineVoice`** or the **VoiceChat** system (or both)?
3. Dedicated server or listen server?
4. Are crashes happening **after seamless travel**, **on client connect**, or **on disconnect**?
5. Are you binding OSS delegates in **PlayerState / GameState / Pawn** currently?

If you paste one representative crash callstack (even partial) and say which event it happens in (`BeginPlay`, `EndPlay`, `PostLogin`, `PostSeamlessTravel`, etc.), I can pinpoint the likely invalid object / ordering issue and propose a robust refactor pattern.