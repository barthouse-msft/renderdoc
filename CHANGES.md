# Changes

This file documents modifications made to this `renderdoc`/`renderdoccmd` fork
(`barthouse-msft/renderdoc`, branch `etw-frame-instrumentation`) as part of the GTA5
native-vs-D3D11On12 performance investigation.

## Unreleased / in progress

### ETW frame-boundary instrumentation in `renderdoccmd`

**File**: `renderdoccmd/renderdoccmd_win32.cpp`

Added a TraceLogging ETW provider (`Microsoft.Windows.Graphics.RenderDocCmd`, GUID
`8b7055ab-e765-4f16-814e-ca1e8051b007`) to `renderdoccmd`'s replay preview loop
(`DisplayRendererPreview`), emitting four events per loop iteration, each tagged with the
current `LoopCount`:

- `RenderDocCmd_ReplayStart` / `RenderDocCmd_ReplayEnd` — bracket `renderer->SetFrameEvent(...)`,
  marking the start/end of this frame's API replay (CPU-side re-issuing of the captured D3D
  commands into the replay device).
- `RenderDocCmd_PresentWaitStart` / `RenderDocCmd_PresentWaitEnd` — bracket `out->Display()`,
  marking the start/end of waiting for the frame's render (present/output blit) to complete.

**Purpose**: lets an ETW trace precisely delimit individual looped-replay frame boundaries
(via `LoopCount`), so other ETW instrumentation (e.g. D3D11On12 DDI-entry-point timing) can be
bucketed into exact per-frame breakdowns instead of only aggregate/whole-capture totals.

**Status**: implemented, verified working (frame boundaries correctly correlated with
D3D11On12 ETW events in analysis scripts). Not yet committed — more changes expected before
the first commit.

### ETW D3D11 API-call instrumentation in the core `renderdoc` D3D11 replay driver

**File**: `renderdoc/driver/d3d11/d3d11_context.cpp`

Added a new TraceLogging ETW provider (`Microsoft.Windows.Graphics.D3D11Replay`, GUID
`2f5a7c3e-6b8d-4e9a-9c1f-7a3d5e8b2c41`), separate from `renderdoccmd`'s `RenderDocCmd`
provider because this code runs inside the core `renderdoc` module (`renderdoc.dll`), not the
`renderdoccmd` executable, so the providers can't share a translation unit/registration.

`WrappedID3D11DeviceContext::ProcessChunk` is the single dispatch point that every captured
D3D11 API call passes through during replay: it switches on the chunk's `D3D11Chunk` type and
calls the matching `Serialise_<ApiName>` function, which deserializes the call's parameters and
(when replaying) re-issues the real D3D11 API call via `m_pRealContext`. The entire `switch`
statement is now bracketed with `QueryThreadCycleTime` calls, and after it an event
`D3D11Replay_ApiCall` is emitted with:

- `ApiName` (string) — `ToStr(chunk)`, the chunk/API name (e.g. `DrawIndexedInstanced`).
- `ThreadCycles` (uint64) — thread CPU cycles consumed while deserializing and replaying that
  one API call.

**Purpose**: gives a one-event-per-D3D11-API-call ETW trace of capture playback, analogous to
the D3D11On12 DDI-entry-point instrumentation, but one layer up the stack (the D3D11 calls
RenderDoc re-issues against the real device/context, which the D3D11On12 runtime then turns
into DDI calls). Lets per-API-type CPU cost be broken down and cross-checked against the DDI
and frame-boundary ETW data.

Emission is **off by default** and controlled by a new `renderdoccmd replay` option:

```
renderdoccmd replay --etw-api-calls <capture.rdc>
```

(short form `-e`). `renderdoccmd.cpp`'s `ReplayCommand` sets the process environment variable
`RENDERDOC_ETW_D3D11_API_CALLS=1` before opening/replaying the capture when the flag is passed;
`d3d11_context.cpp` reads this once (cached in a static local) and skips both the
`QueryThreadCycleTime` calls and the `TraceLoggingWrite` entirely when the flag isn't set, so
there's no overhead unless explicitly requested.

**File**: `renderdoccmd/renderdoccmd.cpp`

**Status**: implemented, builds cleanly in both x64/Development and x64/Release
(`renderdoc_d3d11` and `renderdoccmd` projects). Verified against
`C:\temp\captures\GTA5.rdc`: with `--etw-api-calls`, an ETW trace of the
`Microsoft.Windows.Graphics.D3D11Replay` provider captured tens of thousands of
`D3D11Replay_ApiCall` events with correctly populated `ApiName`/`ThreadCycles` fields (e.g.
95,593 events over 2 loops in the Release build); without the flag, the same provider produced
zero events. Not yet committed.

**Build note**: a one-off x64/Release build in this environment hit an internal compiler error
(C1001) in `driver/vulkan/wrappers/vk_wsi_funcs.cpp`, unrelated to any change in this fork —
caused by stale/corrupted incremental build state (likely the shared Vulkan driver `.pch` or
MSBuild `.tlog` files), not a missing dependency or a real compiler defect. A full
`msbuild renderdoc\renderdoc.vcxproj /t:Rebuild /p:Configuration=Release /p:Platform=x64` cleared
it, and subsequent incremental/parallel (`/m`) Release builds have been clean since. If this
recurs, delete `x64\Release\obj\vulkan` and rebuild.
