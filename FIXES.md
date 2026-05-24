# OpenTurbine ECU — Prioritised fix backlog

Bug count is 113. This document orders them by what to fix first, with an effort
estimate and a patch sketch where the right shape is obvious.

Effort sizing:
- **S** -- single function or single check; under 10 lines, no design choice.
- **M** -- multiple sites or a small refactor; tens of lines, one design choice.
- **L** -- needs a cross-cutting design decision or a structural change.

Engine-safety risk drives priority. Fixes that close a Critical finding are P0
even when they are S effort.

---

## P0 — fix before the next power-on (16 items)

Each one of these can damage hardware, panic the ECU, or bypass a safety limiter.

### Fuel / actuator default state

- **ECU-ACT-01 / ECU-BOOT-01 / ECU-BOOT-02** -- Actuator GPIOs float from reset.
  Effort: **M**.
  Fix: emit `pinMode(pin, OUTPUT)` + `digitalWrite(pin, safe_level)` for every
  relay/MOSFET actuator pin as the first thing in `setup()`, before LittleFS or
  any configuration loads. Drive the safe level (LOW for active-high relays,
  HIGH for active-low relays). Move OT_FUEL_SOL_PIN off GPIO 12 (MTDI strapping
  pin) in `hardware_profile.h`, or document a hard-pulled external resistor.
  Add a `Hardware::makeSafe()` that any path can call without state.

- **ECU-ACT-02** -- `allOff()` does not zero EngineData demand fields.
  Effort: **S**.
  Fix: in `Hardware::allOff()`, also set `ed.fuelSolOpen=false; ed.igniterOn=false;
  ed.igniter2On=false; ed.starterDemand=0; ed.starterEnabled=false; ed.abSolOpen=false;
  ed.abPumpDemand=0; ed.fuelPump2Demand=0; ed.bleedValveOpen=false;
  ed.airstarterOpen=false; ed.oilScavengeOn=false;` so the next call to
  `updateActuators()` re-issues the safe state from a clean baseline.

- **ECU-ACT-03** -- Coil-igniter `s_coilPhaseStart` uninitialised.
  Effort: **S**.
  Fix: zero-initialise `static unsigned long s_coilPhaseStart = 0;` and reset it
  in `Hardware::initActuators()` so the first invocation runs the dwell math
  from a known phase.

### Controllers / runaway protection

- **ECU-CTL-01** -- ThrottleSlew safety pullback disabled when sensor unhealthy.
  Effort: **S**.
  Fix: when `n1Healthy==false` or `totHealthy==false`, freeze the demand at
  `_current` for the affected axis (never increase). Replace `if (ed.n1Healthy &&
  ed.n1Rpm > rpmSoftLimit ...)` with explicit branches for healthy / unhealthy.

- **ECU-CTL-02** -- DynamicIdle runaway when `idleUseN2=true` but no N2 fitted.
  Effort: **S**.
  Fix two things: (1) flip the `EngineData.h:61` default `n2Healthy = false` so
  "unfitted" reads as fault; (2) cross-check `HardwareConfig::hasN2Rpm` in
  `DynamicIdle::tick` and refuse to engage if `idleUseN2` is selected but the
  sensor is absent.

- **ECU-CTL-03** -- OilPressureLoop failsafe always pushes 60% duty regardless of
  fault direction.
  Effort: **M**.
  Fix: when transitioning to failsafe, latch `_outputPct = max(_outputPct,
  failsafePct)` as a floor, never as an unconditional set. Rate-limit the change
  to one slew step per tick. On `oilHealthy=true` recovery, seed `_outputPct =
  failsafePct` so the P-loop resumes smoothly (this fixes ECU-CTL-04 in the same
  patch).

### Safety monitor / fault path

- **ECU-LIM-01** -- NULL deref in `enterFaultShutdown`.
  Effort: **S**.
  Fix: `const char* fault = g_safety.lastFault(); if (!fault) fault = "UNKNOWN";`
  before the `snprintf` and `strcmp` chain at `main.cpp:961-975`. Same idiom in
  any other caller of `lastFault()`.

- **ECU-LIM-02** -- Cooldown-skip has no TOT guard.
  Effort: **S**.
  Fix: in `checkCooldownSkip`, before calling `enterStandby()`, add
  `if (ed.totHealthy && ed.tot > Config::totCooldownTarget) { ... refuse and
  beep; }`. Conservative default for `totCooldownTarget` if not configured: 150 °C.

### Web / OTA exposure

- **ECU-WEB-01** -- Unauthenticated remote engine start.
  Effort: **M**.
  Fix: gate `POST /api/start` (and ideally every non-GET endpoint) on a token
  the operator sets in `Config::webApiToken` and supplies as
  `Authorization: Bearer …` or `?token=…`. As an interim, hard-disable
  `/api/start` and require the physical START switch when no token is set.
  Make AP password mandatory (no default-empty).

- **ECU-WEB-02** -- `TOGGLE_DEV_MODE` reachable from `/api/command` with no gate.
  Effort: **S**.
  Fix: in `handleCommand(OTCommand::TOGGLE_DEV_MODE)` and
  `TOGGLE_SAFETY_CHECKS`, require either a compile-time `OT_DEV_MODE` flag or
  the physical START+STOP held during STANDBY; reject the toggle in any other
  context. Removing the web exposure of these two commands is the cleanest fix.

### Persistence / fault recording

- **ECU-RTOS-01** -- `FlightRecorder::clear()` on Core 1 uses `portMAX_DELAY`.
  Effort: **S**.
  Fix: replace `portMAX_DELAY` with a bounded wait (e.g. `pdMS_TO_TICKS(50)`)
  and return failure on timeout. Caller in `handleCommand(CLEAR_LOG)` should
  defer to Core 0 via the same `_savePending` pattern Config uses.

- **ECU-LOG-01** -- Eviction mutex leak on open failure.
  Effort: **S**.
  Fix: in `FlightRecorder::runEviction()` (cpp:265-283), use RAII or an explicit
  `xSemaphoreGive(_mutex);` on every error path. Also do NOT clear
  `_evictionPending` until the eviction actually succeeded; on failure, leave it
  set so the next Core 0 tick retries.

- **ECU-LOG-02** -- `/api/log/raw` streams without mutex; collides with eviction.
  Effort: **M**.
  Fix: snapshot the file under the mutex to a tmp file (e.g. `/logs/events.snap`),
  release the mutex, then `AsyncFileResponse` the snapshot. Delete snapshot in
  the `onSendComplete` callback. Same pattern for `/api/log/csv`.

### Sensors

- **ECU-SEN-01** -- Flame sensor `isHealthy()` always returns true.
  Effort: **S**.
  Fix: implement `AnalogThSensor::isHealthy()` to return false if `_rawLast == 0`
  or `_rawLast >= 4095` (ADC saturation) sustained for N consecutive reads.

### Comms

- **ECU-COM-01** -- `mavlinkIntervalMs=0` floods UART and stalls main loop.
  Effort: **S**.
  Fix: in `MAVLinkOutput::tick()`, clamp `mavlinkIntervalMs` to a minimum of
  100 ms. Reject zero in `Config::fromJson`. Also non-block the UART write: use
  `Serial2.availableForWrite()` and skip the frame if it would block.

---

## P1 — fix in the next release (37 items, mostly High)

Group by theme for easier batching.

### Theme A: Cross-core and config concurrency  (M-L total)

- **ECU-CTL-05** -- DynamicIdle reads `Config::idleIGain/idleIMax/idleUseN2`
  unsynchronised.
- **ECU-RTOS-03** -- `Hardware::applyConfig` called from both cores.
- **ECU-RTOS-09** -- EngineData char arrays torn-read across cores.
- **ECU-CAL-04** -- `_savePending` not atomic.
- **ECU-CAL-06** -- `HardwareConfig::fromJson` no isLocked guard.
- **ECU-CAL-11** -- `POST /api/config` TOCTOU on isLocked.
- **ECU-LIM-10** -- `skipSafetyChecks` racy.

  Fix: introduce a single "control-state" mutex (or `xSemaphoreCreateMutex`)
  taken at the top of the Core-1 loop body and around every config writer on
  Core 0. Don't hold it across LittleFS I/O. For char-array fields,
  double-buffer (writer fills B, swaps a pointer atomically). For Config gain
  fields, mirror into controller instances at `applyConfig` and require
  `applyConfig` to be deferred to STANDBY (consistent with how blocks are
  already handled at `main.cpp:1326`).

### Theme B: Web hardening  (M each)

- **ECU-WEB-03/10** -- Shared 8 KB RX buffer with no mutex, off-by-one.
  Fix: allocate per-request `std::unique_ptr<uint8_t[]>` body buffer in the
  upload callback; size-cap via `setBodyLimit`. Forbid concurrent body uploads
  with a single global "upload-in-flight" flag.

- **ECU-WEB-04** -- OTA STANDBY guard only on first chunk.
  Fix: re-check mode on every chunk. On a mode change mid-upload, call
  `Update.abort()` and respond 503.

- **ECU-WEB-05** -- No firmware size cap / no rate-limit on `/update`.
  Fix: `Update.begin(MAX_FIRMWARE_BYTES)` with a configured ceiling (e.g.
  1.5 MB per the OTA partition). Track last upload timestamp; reject `/update`
  within 10 s of a previous attempt unless explicitly bypassed.

- **ECU-WEB-06** -- `PATCH /api/config` accepted in FAULT mode.
  Fix: extend `Config::isLocked()` to return true for FAULT in addition to
  STARTUP/RUNNING/SHUTDOWN. Add an explicit "unlock" command that requires
  STANDBY entry first.

### Theme C: Calibration safety  (S each)

- **ECU-CAL-01** -- One-byte overflow in `g_webTxBuf`.
  Fix: serialise into `g_webTxBuf, sizeof(g_webTxBuf)-1` and assert remaining
  space before append.

- **ECU-CAL-02 / ECU-CAL-03** -- HardwareConfig save not atomic; rename window.
  Fix: copy `Config::save`'s tmp+rename pattern. For the rename gap, accept
  load from `.tmp` if the primary file is missing.

- **ECU-CAL-05** -- No bounds check on safety floats.
  Fix: in `Config::fromJson`, clamp every safety-critical field to a sensible
  range. Reject (not clamp) values outside hard bounds and report via
  `ed.faultDescription`. Recommended ranges:
  - `rpmLimit`: [10000, 200000]
  - `totLimit`: [200, 1200]
  - `minRpm`: [500, rpmLimit/2]
  - `oilMinBar`: [0.1, 20.0]
  - `governorKp`: [0, 0.01]
  - `pitchRampSec`: [1, 60]
  - `safetyCheckIntervalMs`: [10, 1000]

- **ECU-CAL-07** -- Oil polynomial defaults all zero.
  Fix: set safe defaults in `Config::_applyDefaults`: at minimum, a linear
  default that returns 1.0 bar per 100 raw counts so first-boot reads non-zero.

### Theme D: Limit-monitor correctness  (S each)

- **ECU-LIM-03** -- RulesEngine runs in SHUTDOWN/FAULT.
  Fix: in `RulesEngine::evaluate()`, return early if mode is SHUTDOWN or FAULT
  in addition to the existing STANDBY/benchMode suppression.

- **ECU-LIM-04** -- ThrottleSlew pullback not suppressed by skipSafetyChecks.
  Fix: in `ThrottleSlew::tick`, read `ed.skipSafetyChecks` and bypass the
  pullback branches when true. Treat as consistency with SafetyMonitor.

- **ECU-LIM-05** -- Limp cap can't override DynamicIdle floor.
  Fix: in the limp clamp at `main.cpp:1638-1645`, also clamp `_idleFloor`
  inside DynamicIdle, or evaluate the limp cap before DynamicIdle runs.
  Document tuning constraint: `limpMaxThrottlePct >= idleMinFloor`.

### Theme E: RTOS hardening  (S each, ~6 items)

- **ECU-RTOS-02** -- webTask spawned before Watchdog. Reorder: Watchdog::begin
  first, then webTask.
- **ECU-RTOS-05** -- RCInput ISR float work without FPU save. Switch RCInput
  ISR to capture micros() only; do all float conversion in `tick()` on the
  task.
- **ECU-RTOS-06/07** -- Unchecked `xTaskCreate*` and `xQueueCreate` returns.
  Add `if (!handle) panic("...");` with a Serial.println and a safe halt.
- **ECU-RTOS-08** -- FlightRecorder mutex priority inversion. Enable
  `xSemaphoreCreateMutex` (already PI-capable) but reduce web-side hold time
  by snapshot-then-release (same fix as ECU-LOG-02).

### Theme F: Sequencer correctness  (S each, 4 items)

- **ECU-SEQ-01** -- Double `onExit()` on abort-then-shutdown.
  Fix: in `SequenceEngine::startSequence`, set `_running=false` before calling
  any callback; check `_running` before calling `onExit` to avoid the second
  call.
- **ECU-SEQ-02** -- `SafetyHold` final gate silently skipped when unhealthy.
  Fix: treat unhealthy as Abort, not as "skip the check".
- **ECU-SEQ-03** -- `FuelPulse::onExit()` empty.
  Fix: explicitly `ed.fuelSolOpen = false;` in `FuelPulse::onExit()`.
- **ECU-SEQ-04** -- `_abTriggerPrev` stale across restart.
  Fix: reset `_abTriggerPrev = false` in `enterRunning()` and on every
  transition out of RUNNING.

### Theme G: Sensor robustness  (S each)

- **ECU-SEN-02** -- PCNTRpmSensor SATURATED false-positive on decel.
  Fix: require N consecutive saturated reads, or compare against last-known
  count direction.
- **ECU-SEN-03** -- RC ISR window vs runtime `rcMinUs/rcMaxUs`.
  Fix: read configured bounds in `tick()` after the ISR-captured pulse width;
  let the ISR accept the widest envelope.
- **ECU-SEN-04** -- PCNT `_ppr == 0` divide-by-zero.
  Fix: reject `setPpr(0)` and clamp `_ppr = max(1, _ppr)` in `getRpm()`.

### Theme H: Comms reliability  (S each)

- **ECU-COM-02** -- Web STOP/AB_STOP silent drop.
  Fix: check `CommandQueue::push` return value; on failure, retry with
  `xQueueOverwrite` for STOP/AB_STOP specifically (these are higher priority
  than tool tests); also return HTTP 503 to the client.
- **ECU-COM-03** -- MAVLink 303-byte burst exceeds 256-byte TX buffer.
  Fix: split MAVLink frame into < 200-byte chunks per `tick()`, or grow the
  UART TX buffer via `setTxBufferSize(512)` before `Serial2.begin()`.

### Theme I: Controller dt math  (S, single touch)

- **ECU-CTL-07 / ECU-CTL-08** -- dt=0 and unbounded dt.
  Fix: in every controller `tick()`, clamp `dt = constrain(dt, 0.001f, 0.050f);`
  consistent with what `PowerTurbineGovernor` already does. Move the clamp
  into a shared `IController::computeDt(now, _lastMs)` helper.

- **ECU-CTL-09** -- `enterRunning()` zeroes `throttleDemand` before
  `ThrottleSlew::begin()` snapshots it.
  Fix: remove the `ed.throttleDemand = 0` line at `main.cpp:927`; let
  `ThrottleSlew::begin()` actually carry forward the Spool exit demand.

---

## P2 — fix when convenient (38 items, mostly Medium)

These are scoped to specific subsystems. Group by file to do them in clusters:

- `main.cpp` (limp cap, applyConfig timing, throttle zero on enter) — pair with P1
  Theme A.
- `Hardware.h` actuator path (oil pump scale, multiple writers, LEDC re-attach,
  applyConfig re-entry, ServoActuator overflow, LEDC rounding) — single review
  pass through `updateActuators()`.
- `Config.h`/`.cpp` (savePending atomic, isLocked, format-on-fail, password
  redaction in GET, totalRunSeconds overflow, legacy migration order) — single
  review pass.
- `SafetyMonitor.h` (flameoutShutdownMs cast, safetyCheckIntervalMs cast,
  externalFault pointer, surge buffer reset, relight callback re-entry,
  minRpm vs rpmLimit cross-check) — single review pass.
- `EngineData.h` (default n2Healthy=false; string-field double-buffer) — pair
  with P0 (ECU-CTL-02) and P1 Theme A.
- `SessionLogger.cpp` (file-handle race, drainQueue scheduling, eviction
  capacity) — single review pass.
- `WebServer.cpp` WS path (full-frame drop, static JsonDocument, no body limit,
  password optional warning) — single review pass after the P1 web hardening.
- `partitions.csv` -- change `spiffs` to `littlefs` (subtype `0x82` -> `0x83`).
  Effort: **S**, but test on the target hardware first; some IDF versions
  accept the lax mapping.

---

## P3 — hygiene (21 Low + 1 Info)

Most of these are read-only changes: stale comments, dead code, off-by-one in
truncation that does not affect correctness, header defaults vs Config defaults
diverging. Walk through in a single PR per subsystem when the team next touches
that file. Suggested batches:

- One PR for `engine/controllers/*` comments and `Hardware.h` controller mirror
  cleanup.
- One PR for `engine/sequencer/blocks/*` minor cleanups (FlameConfirm waitReason,
  ABIgnite int->ulong, OilPrime onExit on abort, WaitForInput timeout=0 guard).
- One PR for boot-init hygiene (`PlatformInit.h` ordering, NVS Preferences
  return checks, ClusterSerial delay() removal).
- One PR for web-server hygiene (security headers, captive portal cleanup,
  serveStatic explicit allowlist).
- One PR for sensors (NTCSensor health flag, AnalogLinear extrapolation guard,
  MAX6675 ring buffer reset on fault, MAX31856 SPI mode comment, RCInput
  compiler barrier).

---

## Sequencing recommendation

If you have one working session:

1. **30 minutes** — patch the four P0 actuator/boot fixes (ECU-ACT-01/02/03,
   ECU-BOOT-01/02). These are the only items that can damage hardware on a
   single power cycle.
2. **30 minutes** — patch the three P0 controller fixes (ECU-CTL-01/02/03)
   and the two safety-monitor fixes (ECU-LIM-01/02). Pure C++ logic, no UI.
3. **30 minutes** — patch the three P0 web fixes (ECU-WEB-01/02 +
   ECU-COM-01). Adds an `OT_REQUIRE_TOKEN` build flag and a token in Config.
4. **15 minutes** — patch the three P0 persistence fixes (ECU-RTOS-01,
   ECU-LOG-01/02). Bounded mutex waits, snapshot for log read.
5. **15 minutes** — patch the one P0 sensor fix (ECU-SEN-01). One function.

That's all 16 Criticals in roughly two hours. P1 themes can then be tackled
one at a time in subsequent sessions.

Do NOT skip the actuator default-state fix even if it looks low-impact; ESP32
GPIO behaviour is documented but it varies per silicon revision and per
strapping configuration, and the failure mode is "fuel solenoid clicks open
during every boot".
