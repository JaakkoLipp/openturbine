# OpenTurbine ECU — Aggregated bug report

Static code audit of `claude/esp32-ecu-audit-2nlXI`, 11 subsystems, source only,
no execution. See `CODEMAP.md` for subsystem layout and `.audit/<subsystem>/bugs.md`
for the full finding (description, trigger, evidence, suggested fix, confidence).

Engine-safety impact has been weighted into severity. A "small" bug that touches
the ignition or fuel path outranks a "large" bug in telemetry. Counts:

| Severity | Count |
|---|---:|
| Critical | 16 |
| High     | 37 |
| Medium   | 38 |
| Low      | 21 |
| Info     | 1  |
| **Total** | **113** |

Subsystem coverage:

| Subsystem | C | H | M | L | I | Total |
|---|---:|---:|---:|---:|---:|---:|
| controllers          | 3 | 6 | 4 | 3 | 0 | 16 |
| actuator-output      | 3 | 5 | 5 | 1 | 0 | 14 |
| calibration-storage  | 0 | 5 | 6 | 1 | 0 | 12 |
| limits-protection    | 2 | 3 | 7 | 2 | 0 | 14 |
| rtos-architecture    | 1 | 6 | 6 | 2 | 0 | 15 |
| sequencer-state      | 0 | 4 | 4 | 4 | 0 | 12 |
| persistence-logging  | 2 | 3 | 4 | 3 | 0 | 12 |
| wireless-web         | 2 | 4 | 5 | 3 | 0 | 14 |
| sensor-input         | 1 | 3 | 5 | 2 | 0 | 11 |
| boot-init            | 1 | 1 | 3 | 4 | 1 | 10 |
| comms-protocols      | 1 | 2 | 2 | 2 | 0 | 7  |

---

## Critical (16)

Items that can cause uncommanded ignition or fueling, runaway, stuck actuator,
bypass of a safety limiter, or a panic / boot into an unsafe state.

| ID | Title | Location |
|---|---|---|
| ECU-CTL-01 | ThrottleSlew safety pullback silently disarmed when sensor unhealthy | `engine/controllers/ThrottleSlew.h:50-58` |
| ECU-CTL-02 | DynamicIdle ramps throttle to 100% if `idleUseN2=true` but N2 sensor not fitted (default `n2Healthy=true`) | `engine/controllers/DynamicIdle.h:45-48` + `EngineData.h:61` |
| ECU-CTL-03 | OilPressureLoop failsafe forces 60% duty regardless of fault direction (over-pressure overrides) | `engine/controllers/OilPressureLoop.h:38-48` |
| ECU-ACT-01 | Fuel solenoid pin floats HIGH from power-on until `initActuators` runs | `Hardware.h` initActuators + `hardware_profile.h` |
| ECU-ACT-02 | `allOff()` does not zero `fuelSolOpen` / `igniterOn` / `starterDemand` in EngineData; next tick can re-open | `main.cpp:1047` (allOff) + `Hardware.h` |
| ECU-ACT-03 | Coil-igniter latches energised on first invocation due to uninitialised static `s_coilPhaseStart` | `Hardware.h` igniter dwell logic |
| ECU-LIM-01 | NULL dereference in `enterFaultShutdown` when `g_safety.lastFault()` returns null (panic at the moment of fault) | `main.cpp:961-975` + `engine/SafetyMonitor.h:278` |
| ECU-LIM-02 | Cooldown-skip gesture (hold START+STOP 1 s during SHUTDOWN) has no TOT guard; hot turbine goes to STANDBY with no airflow | `main.cpp:850-871` |
| ECU-RTOS-01 | `FlightRecorder::clear()` called on Core 1 with `portMAX_DELAY`; web-held mutex stalls ECU past 5 s TWDT, panic | `FlightRecorder.cpp` clear() |
| ECU-LOG-01 | Eviction mutex leak on `LittleFS.open` failure; subsequently every event is dropped silently | `FlightRecorder.cpp:265-283` |
| ECU-LOG-02 | `/api/log/raw` streams the live log file without holding the mutex; eviction mid-download corrupts the file | `WebServer.cpp:588-597` |
| ECU-WEB-01 | Unauthenticated remote engine start: `POST /api/start` on open AP starts the engine | `WebServer.cpp:642-645` |
| ECU-WEB-02 | `TOGGLE_DEV_MODE` then `TOGGLE_SAFETY_CHECKS` from `/api/command` fully disables overspeed/overtemp/oil checks on a running engine | `WebServer.cpp:683` + `main.cpp:1177-1186` |
| ECU-SEN-01 | `AnalogThSensor::isHealthy` always returns true; flame sensor sensor-loss never propagates | `hal/sensors/AnalogSensor.h:143` |
| ECU-COM-01 | `mavlinkIntervalMs=0` removes the rate gate; MAVLink floods UART2 and the blocking write stalls the main loop | `MAVLinkOutput.h` tick() |
| ECU-BOOT-01 | All actuator GPIO pins float from power-on reset until `initActuators` runs; relay loads see undefined level | `main.cpp` setup() + `Hardware.h` initActuators |

---

## High (37)

Items that produce incorrect control output under realistic conditions, allow
calibration corruption, or crash / DoS on common input.

### Controllers
- **ECU-CTL-04** - OilPressureLoop failsafe latch never exits cleanly when sensor recovers; snap from 60% back to stale `_outputPct`. *(`OilPressureLoop.h:38-51`)*
- **ECU-CTL-05** - Race on `Config::idleIGain` / `idleIMax` / `idleUseN2` between Core 0 web POST and Core 1 DynamicIdle tick; not mirrored by `applyConfig`. *(`DynamicIdle.h:45,85-86` + `WebServer.cpp:484-489`)*
- **ECU-CTL-06** - PowerTurbineGovernor pitch-saturation handoff fights ThrottleSlew rate limit; corrections truncated to slew rate. *(`PowerTurbineGovernor.h:101-123`)*
- **ECU-CTL-07** - `dt = 0` when two consecutive ticks land in the same `millis()`; DynamicIdle and ThrottleSlew lose corrections. *(`DynamicIdle.h:53-55` + `ThrottleSlew.h:42`)*
- **ECU-CTL-08** - ThrottleSlew loses its slew limit on first tick after a long pause; large dt makes maxStep > 1.0 in one tick. *(`ThrottleSlew.h:41-71`)*
- **ECU-CTL-09** - `enterRunning()` zeroes `throttleDemand` BEFORE `ThrottleSlew::begin()` captures it; controller slews up from 0 at the most fragile moment. *(`main.cpp:924-940`)*

### Actuators
- **ECU-ACT-04** - Oil pump `oilPumpPct` interpreted as 0..100 for LEDC/servo but 0..1 for relay; same field, two scales. *(`Hardware.h` updateActuators)*
- **ECU-ACT-05** - Multiple unsynchronised writers to `oilPumpPct` (controller / failsafe / standbyOilFeed / extraCooldown); no priority order. *(`Hardware.h` + `main.cpp:511-539`)*
- **ECU-ACT-07** - Igniter LEDC duty computed at object construction with compile-time defaults; runtime override may or may not be re-applied to LEDC peripheral. *(`Hardware.h` igniter init)*
- **ECU-ACT-08** - Starter ESC receives non-zero PWM when `starterEnabled=true` and `starterDemand=0`; mid-stick may jog motor. *(`Hardware.h` updateActuators)*
- **ECU-ACT-09** - NULL pointer dereference if `hasThrottle` / `hasStarter` is true but the `IActuator*` was not assigned (type mismatch). *(`Hardware.h` updateActuators)*

### Calibration / storage
- **ECU-CAL-01** - One-byte overflow in `g_webTxBuf` (`serializeJson` writes null terminator past end). *(`WebServer.cpp` serialize path)*
- **ECU-CAL-02** - `HardwareConfig::save` writes in-place, not via tmp+rename; power loss wipes both hw and settings sections. *(`HardwareConfig.cpp` save())*
- **ECU-CAL-03** - Atomic-rename window in `Config::save`: remove succeeds, rename fails, no config remains. `_savePending` is also cleared before save returns so failures are never retried. *(`Config.cpp` save())*
- **ECU-CAL-05** - No range validation on safety floats: `rpmLimit=0` and `rpmLimit=-1` both accepted via web POST. *(`Config.cpp` fromJson)*
- **ECU-CAL-07** - Oil pressure polynomial defaults all zero; first-boot or factory-reset reads 0 bar and immediately trips `LOW_OIL_PRESSURE` on RUNNING entry. *(`Config.cpp` defaults)*

### Limits / protection
- **ECU-LIM-03** - RulesEngine runs in SHUTDOWN and FAULT modes; rule can re-open bleed valve / fuel pump 2 immediately after `allOff()`. *(`RulesEngine.h:27-41` + `main.cpp:1657-1659`)*
- **ECU-LIM-04** - ThrottleSlew safety pullback is NOT suppressed by `skipSafetyChecks`; asymmetric bypass during dev-mode testing near limit. *(`ThrottleSlew.h:47-58`)*
- **ECU-LIM-05** - Limp-mode throttle cap (default 0.50) cannot override DynamicIdle minimum floor (~0.55); engine oscillates between floor and cap. *(`main.cpp:1638-1645` + `DynamicIdle.h:77-99`)*

### Web / OTA
- **ECU-WEB-03** - Shared 8 KB `g_webRxBuf` has no mutex; concurrent POSTs on AsyncTCP interleave and corrupt each other's bodies. *(`WebServer.cpp:57-59,472-479`)*
- **ECU-WEB-04** - OTA STANDBY mode guard runs only on first chunk (`index==0`); engine start mid-upload still commits firmware. *(`WebServer.cpp:831-858`)*
- **ECU-WEB-05** - `Update.begin(UPDATE_SIZE_UNKNOWN)` has no firmware size cap; no in-progress guard; deliberate repeated `/update` posts wear flash. *(`WebServer.cpp:840`)*
- **ECU-WEB-06** - `PATCH /api/config` accepted in FAULT mode (`isLocked` returns false); operator can edit safety limits during fault recovery. *(`WebServer.cpp:495-551`)*

### RTOS / cross-core
- **ECU-RTOS-02** - `webTask` spawned at `main.cpp:1598` before `Watchdog::begin` at `main.cpp:1613`; WiFi init runs with no WDT coverage. *(`main.cpp:1598`)*
- **ECU-RTOS-03** - `Hardware::applyConfig()` called from web PATCH path on Core 0 AND from `main.cpp:1149` on Core 1; concurrent writes to block instances. *(`main.cpp:1149,1327,1538`)*
- **ECU-RTOS-05** - RCInput ISRs perform float work without `xPortFPUIsCurrentTaskFPU()` save; FPU state can be clobbered. *(`hal/RCInput.h:87,96`)*
- **ECU-RTOS-06** - `xTaskCreatePinnedToCore` for webTask return value unchecked; silent heap exhaustion leaves no web server. *(`main.cpp:1598`)*
- **ECU-RTOS-07** - `CommandQueue::begin` xQueueCreate return value unchecked; NULL queue causes silent command loss. *(`CommandQueue.cpp:64`)*
- **ECU-RTOS-08** - FlightRecorder mutex 8 ms timeout vs web read holding it across LittleFS open; priority inversion on Core 0 starves Core 1 append. *(`FlightRecorder.cpp:292`)*

### Sequencer
- **ECU-SEQ-01** - Double `onExit()` invocation when abort triggers shutdown sequence re-entry (`_running` still true when `startSequence` runs). *(`engine/sequencer/SequenceEngine.h`)*
- **ECU-SEQ-02** - `SafetyHold` block lets engine reach RUNNING when N1 / oil sensor is unhealthy; final checks silently skipped. *(`engine/sequencer/blocks/SafetyHold.h`)*
- **ECU-SEQ-03** - `FuelPulse::onExit()` is empty; aborting mid-phase leaves fuel solenoid open until next tick's `ImmediateCut`. *(`engine/sequencer/blocks/AdvancedBlocks.h`)*
- **ECU-SEQ-04** - `_abTriggerPrev` stale across restart; trigger held through shutdown re-fires AB immediately on RUNNING re-entry. *(`main.cpp:746-843`)*

### Sensors
- **ECU-SEN-02** - PCNTRpmSensor SATURATED false-positive during normal deceleration. *(`hal/sensors/PCNTRpmSensor.h:133-135`)*
- **ECU-SEN-03** - RCInput ISR pulse-width acceptance window hardcoded (500-2500 us); does not match runtime-configurable `rcMinUs/rcMaxUs`. *(`hal/RCInput.h:92,101`)*
- **ECU-SEN-04** - PCNTRpmSensor does not guard against `_ppr == 0`; divide-by-zero produces inf RPM. *(`hal/sensors/PCNTRpmSensor.h:85`)*

### Comms
- **ECU-COM-02** - Web STOP and AB_STOP silently dropped when CommandQueue is full; HTTP 200 returned. *(`CommandQueue.h:68-71` + `WebServer.cpp:648-651`)*
- **ECU-COM-03** - MAVLink 303-byte burst exceeds 256-byte HardwareSerial TX buffer; write blocks the main loop. *(`MAVLinkOutput.h`)*

### Boot
- **ECU-BOOT-02** - `OT_FUEL_SOL_PIN = GPIO 12` is the ESP32 MTDI strapping pin; active-low solenoid configuration creates spurious-open risk during boot strap evaluation. *(`hardware_profile.h`)*

---

## Medium (38)

### Controllers
- **ECU-CTL-10** - Limp-mode cap applied after ThrottleSlew leaves `_current` out of sync. *(`main.cpp:1638-1645`)*
- **ECU-CTL-11** - Unclamped `governorKp` / `pitchKp` / `pitchRampSec`; `pitchRampSec=0` removes pitch slew. *(`PowerTurbineGovernor.h:54-56`)*
- **ECU-CTL-12** - DynamicIdle integrator not reset on early-return paths (`targetRpm <= 0`). *(`DynamicIdle.h:41-43`)*
- **ECU-CTL-13** - Both ThrottleSlew pullback branches subtract independently; combined hot+fast snaps target to 0. *(`ThrottleSlew.h:47-58`)*

### Actuators
- **ECU-ACT-06** - `LEDCActuator::begin()` does not `ledcDetach` before re-attaching; mid-run reconfig glitches. *(`hal/actuators/LEDCActuator.h`)*
- **ECU-ACT-10** - `applyConfig()` called on every START re-applies block params while sequence may already be running. *(`main.cpp:1149`)*
- **ECU-ACT-11** - `throttleDemand=0` in `enterRunning()` regardless of slew state; one-frame zero throttle. *(`main.cpp:927`)*
- **ECU-ACT-12** - `ServoActuator::set()` no overflow check on extreme `minUs/maxUs`. *(`hal/actuators/ServoActuator.h`)*
- **ECU-ACT-13** - LEDCActuator duty truncates float to uint32_t without rounding; off-by-one at 100%. *(`hal/actuators/LEDCActuator.h`)*

### Calibration
- **ECU-CAL-04** - `_savePending` volatile but not atomic; Core 1 write / Core 0 read race. *(`Config.cpp`)*
- **ECU-CAL-06** - `HardwareConfig::fromJson` has no `isLocked` / engine-state guard. *(`HardwareConfig.cpp`)*
- **ECU-CAL-09** - `LittleFS.begin(true)` silently formats flash on mount failure; all calibration and logs lost. *(`platform/esp32/PlatformInit.h:23`)*
- **ECU-CAL-10** - `wifiPassword` stored plaintext in `ecu_config.json` and echoed by `GET /api/hardware`. *(`HardwareConfig.cpp`)*
- **ECU-CAL-11** - `POST /api/config` does not re-check `isLocked` after body accumulation; TOCTOU. *(`WebServer.cpp`)*
- **ECU-CAL-12** - Legacy migration calls `save()` before `profileId` set; wrong profile_id written. *(`Config.cpp`)*

### Limits / protection
- **ECU-LIM-06** - `flameoutShutdownMs` stored as float; cast to `unsigned long` is UB if negative. *(`SafetyMonitor.h:34,145`)*
- **ECU-LIM-07** - `safetyCheckIntervalMs` int->unsigned long mismatch produces ~49-day interval if negative. *(`Config.h:82` + `SafetyMonitor.h:35`)*
- **ECU-LIM-08** - `setExternalFault` stores raw `const char*` with undocumented lifetime contract. *(`SafetyMonitor.h:49`)*
- **ECU-LIM-09** - Surge buffer not cleared on STARTUP entry; stale spindown samples cause false SURGE on next start. *(`SafetyMonitor.h:67-70`)*
- **ECU-LIM-10** - `skipSafetyChecks` bool with no memory barrier. *(`EngineData.h:117` + `SafetyMonitor.h:55`)*
- **ECU-LIM-11** - Relight callback fires while `_relightActive` already true; redundant igniterOn writes. *(`main.cpp:1578-1591`)*
- **ECU-LIM-12** - `minRpm >= rpmLimit` (misconfig) fires UNDERSPEED immediately on RUNNING entry. *(`SafetyMonitor.h:247-249`)*

### RTOS
- **ECU-RTOS-04** - SessionLogger `vTaskDelay(30 ms)` on Core 1 injects jitter into control loop. *(`SessionLogger.cpp`)*
- **ECU-RTOS-09** - EngineData char arrays (`lastEvent`, `faultDescription`) written Core 1, read Core 0 unsynchronised; torn string reads. *(`EngineData.h`)*
- **ECU-RTOS-10** - SessionLogger `_file` handle accessed from Core 1 endSession and Core 0 drainQueue concurrently. *(`SessionLogger.cpp`)*
- **ECU-RTOS-11** - `CONFIG_ASYNC_TCP_USE_WDT=0` disables WDT on async_tcp task; AsyncTCP stall undetectable. *(`platformio.ini`)*
- **ECU-RTOS-12** - CommandQueue depth 16 too small under captive-portal command burst; STOP can be dropped. *(`CommandQueue.cpp:61`)*
- **ECU-RTOS-13** - webTask priority (8) > ECU loopTask (1) on Core 0; FlightRecorder mutex inversion. *(`main.cpp:1598`)*

### Sequencer
- **ECU-SEQ-05** - Normal shutdown does not synchronously clear `abFuelOffset`; one-tick residual AB fuel. *(`main.cpp:enterShutdown`)*
- **ECU-SEQ-08** - `ABFlameConfirm` mode 2: `assumeIgnitedMs >= flameTimeoutMs` always produces Fault. *(`blocks/ABFlameConfirm.h`)*
- **ECU-SEQ-09** - `buildSequences()` silently drops unrecognized block names; shorter sequence with no error. *(`main.cpp:96-138`)*
- **ECU-SEQ-11** - `ABStabilize::onExit()` sets `abMode=Running` even on Fault; Core 0 may see transient. *(`blocks/ABStabilize.h`)*

### Sensors
- **ECU-SEN-05** - NTCSensor returns garbage on invalid calibration without flagging unhealthy. *(`hal/sensors/NTCSensor.h:55-56`)*
- **ECU-SEN-06** - AnalogLinearSensor extrapolates silently above calibrated range. *(`hal/sensors/AnalogSensor.h:119-128`)*
- **ECU-SEN-07** - RC idle channel stale value persists after failsafe. *(`hal/RCInput.h:56-61`)*
- **ECU-SEN-08** - MAX6675 ring buffer not cleared on fault; stale samples after reconnect. *(`hal/sensors/MAX6675TempSensor.h:53-64`)*
- **ECU-SEN-09** - PCNTRpmSensor::begin() leaks PCNT unit handle on repeated init. *(`hal/sensors/PCNTRpmSensor.h:29-67`)*

### Web
- **ECU-WEB-07** - WebSocket 7 KB full frame can silently drop on connect. *(`WebServer.cpp:1093-1096`)*
- **ECU-WEB-08** - Shared static `JsonDocument` in WS callback not re-entrant. *(`WebServer.cpp:1094`)*
- **ECU-WEB-09** - Open WiFi AP by default; password optional with no warning. *(`WebServer.cpp:71-72`)*
- **ECU-WEB-10** - `g_webRxBuf` overflow check off-by-one; 8191-byte write without null terminator. *(`WebServer.cpp:473`)*
- **ECU-WEB-11** - CommandQueue silent drop loses STOP/AB_STOP; HTTP 200 returned. *(`WebServer.cpp:648-651`)*
- **ECU-WEB-12** - No `setBodyLimit` on AsyncWebServer; oversized POST buffers in async_tcp. *(`WebServer.cpp:62`)*

### Persistence
- **ECU-LOG-03** - `partitions.csv` declares `subtype=spiffs` but firmware uses LittleFS; a stricter IDF will fail to mount. *(`partitions.csv:7`)*
- **ECU-LOG-04** - `s_lineCount` written by Core 0 without atomic protection. *(`FlightRecorder.cpp:20`)*
- **ECU-LOG-05** - 8 ms append mutex timeout drops fault events during log download. *(`FlightRecorder.cpp:292`)*
- **ECU-LOG-06/07** - `endSession()` / `startSession()` race `drainQueue()` via 30 ms sleep, not a real primitive. *(`SessionLogger.cpp:113-179`)*

### Boot / comms
- **ECU-BOOT-03** - webTask spawned before `FlightRecorder::begin()`; race window. *(`main.cpp:1598`)*
- **ECU-BOOT-04** - LittleFS double-failure (mount AND format) silent. *(`platform/esp32/PlatformInit.h:23`)*
- **ECU-BOOT-06** - `stopPin` configured INPUT_PULLUP with compile-time pin before runtime reassignment. *(`PlatformInit.h:61-64` + `main.cpp:1511-1512`)*
- **ECU-COM-04** - ClusterSerial schema not retransmitted after cluster reboot. *(`ClusterSerial.cpp`)*
- **ECU-COM-05** - `snprintf` truncation in ClusterSerial D-frame silently ignored; underflow in next size arg. *(`ClusterSerial.cpp`)*

---

## Low (21)

### Controllers
- **ECU-CTL-14** - OilPressureLoop deadband + P-only loop SS offset *(`OilPressureLoop.h:53-63`)*
- **ECU-CTL-15** - Header defaults vs Config defaults divergence risk *(`Hardware.h:396-530`)*
- **ECU-CTL-16** - DynamicIdle asymmetric ramp comment vs code/defaults conflict *(`DynamicIdle.h:70-76`)*

### Actuators
- **ECU-ACT-14** - Igniter 2 LEDC constructed with pin -1 freq 111 *(`Hardware.h`)*

### Limits / protection
- **ECU-LIM-13** - `_fallbackDesc` stack buffer pattern in `SafetyMonitor::_trigger` *(`SafetyMonitor.h:346-358`)*
- **ECU-LIM-14** - `checkStandbyOilFeed` ignores `n1Healthy`; stale-zero RPM blocks bearing protection *(`main.cpp:511-539`)*

### Sequencer
- **ECU-SEQ-06** - `FlameConfirm` bench mode does not clear `seqWaitReason` *(`blocks/FlameConfirm.h`)*
- **ECU-SEQ-07** - `ABIgnite::torchDurationMs` signed/unsigned mismatch with `elapsed` *(`blocks/ABIgnite.h`)*
- **ECU-SEQ-10** - `OilPrime::onExit()` arms `oilMinBar` on abort path *(`blocks/OilPrime.h`)*
- **ECU-SEQ-12** - `WaitForInput` with timeout=0 and invalid channelIdx hangs *(`blocks/WaitForInput.h`)*

### Sensors / persistence
- **ECU-SEN-10** - RCInput ISR pulseUs+fresh write without compiler barrier *(`hal/RCInput.h:91-93`)*
- **ECU-SEN-11** - MAX31856 SPI mode comment vs implementation mismatch *(`hal/sensors/MAX31856TempSensor.h:11`)*
- **ECU-LOG-09** - SESSION_MIN_FREE_BYTES capacity planning *(`SessionLogger.cpp:80`)*
- **ECU-LOG-10** - SessionLogger filename derived from unsynchronised runCount *(`SessionLogger.cpp:123`)*
- **ECU-LOG-12** - Dir entry handle not closed before reassignment in eviction loop *(`SessionLogger.cpp:_evictOldSessions`)*

### Calibration / boot / comms / RTOS
- **ECU-CAL-08** - `totalRunSeconds` uint32_t overflow at ~136 years *(`Config.cpp`)*
- **ECU-BOOT-07** - NVS Preferences failure fully silent *(`PlatformInit.h`)*
- **ECU-BOOT-08** - ClusterSerial blocking `delay(100)` x3 with webTask running *(`ClusterSerial.cpp` begin)*
- **ECU-BOOT-09** - `applyConfig` before initSensors/initActuators (safe but fragile) *(`main.cpp:1538`)*
- **ECU-BOOT-10** - DI channel pinMode order relies on implicit ordering *(`main.cpp:1554-1569`)*
- **ECU-WEB-13** - `serveStatic("/", LittleFS, "/")` exposes `ecu_config.json` (WiFi password) via direct URL *(`WebServer.cpp:436`)*
- **ECU-WEB-14** - No security headers (CSP / X-Frame-Options / CSRF) on any response *(`WebServer.cpp`)*
- **ECU-RTOS-14** - `RCInput::begin()` uses compile-time pins; no runtime validity check *(`hal/RCInput.h`)*
- **ECU-RTOS-15** - `extraCooldownUntilMs` unsigned long portability *(`EngineData.h`)*
- **ECU-COM-06** - CommandQueue comment misdescribes producer set *(`CommandQueue.h`)*
- **ECU-COM-07** - `_crc16()` helper dead code *(`MAVLinkOutput.h`)*

---

## Info (1)

- **ECU-BOOT-05** - Dead code: `setup()` FAULT-mode check after Config::load can never be true *(`main.cpp:1516-1519`)*

---

## Cross-cutting themes

Looking across all 113 findings, the following root causes appear repeatedly:

1. **"Sensor unhealthy" silently skips a check.** SafetyMonitor, ThrottleSlew pullback, SafetyHold final gate, standby oil feed, and others all share the pattern `if (healthy && condition)`. The recovery on a real sensor fault is to do nothing, which is unsafe for limit checks and unsafe for soft pullback. This pattern appears in at least 7 findings (ECU-CTL-01, ECU-LIM-14, ECU-SEQ-02, ECU-SEN-01, ECU-CTL-02, ECU-WEB-02 indirectly via DEV_MODE, ECU-CTL-03).

2. **Cross-core EngineData with no synchronisation.** Core 1 writes, Core 0 reads, no mutex, no atomics. Single 32-bit aligned fields are torn-read-safe on Xtensa, but composite state (mode + igniterOn + faultDescription) is not. ECU-RTOS-09, ECU-CAL-04, ECU-LIM-10, ECU-LOG-04 all stem from this.

3. **No bounds checking on safety-critical config values.** `rpmLimit`, `totLimit`, `minRpm`, `governorKp`, `pitchRampSec`, `safetyCheckIntervalMs`, `flameoutShutdownMs`, polynomial coefficients, pin numbers, igniter dwell — all accepted unbounded from web POST. ECU-CAL-05, ECU-CTL-11, ECU-LIM-06, ECU-LIM-07, ECU-LIM-12, ECU-SEN-04, ECU-SEQ-07 all share this root.

4. **No authentication / no rate-limit / no integrity on the web surface.** Open AP by default, no body limit, OTA accepts unsigned firmware, command endpoint dispatches dev-mode toggle. ECU-WEB-01/02/05/09/13/14 are all variations.

5. **Persistence is fragile.** Atomic-rename inconsistencies, silent format-on-mount-fail, mutex leaks in eviction, schema-vs-content mismatch in partitions.csv. ECU-CAL-02/03/09, ECU-LOG-01/03 all share this.

6. **`millis()` based dt has no upper or lower bound clamp** in most controllers. Fast-loop produces dt=0 (lost correction); a stall produces dt=N seconds (slew limit ignored). ECU-CTL-07, ECU-CTL-08 are direct hits.

7. **Boot-time GPIO is unspecified** until the actuator's `begin()` runs. Strapping pin GPIO 12 has documented power-on behaviour, but other relay/MOSFET pins land in their default reset state for ~1.5 s. ECU-ACT-01, ECU-BOOT-01, ECU-BOOT-02 are the surface symptoms.

8. **Sequencer onExit is informally a "cleanup"**, but Abort and Fault paths can skip it, leaving actuators in mid-block state. ECU-SEQ-01, ECU-SEQ-03, ECU-SEQ-10, ECU-SEQ-11 are all this pattern.

---

## Where to read each finding in full

| Subsystem | Subagent report |
|---|---|
| controllers | `.audit/controllers/bugs.md` |
| actuator-output | `.audit/actuator-output/bugs.md` |
| calibration-storage | `.audit/calibration-storage/bugs.md` |
| limits-protection | `.audit/limits-protection/bugs.md` |
| rtos-architecture | `.audit/rtos-architecture/bugs.md` |
| sequencer-state | `.audit/sequencer-state/bugs.md` |
| persistence-logging | `.audit/persistence-logging/bugs.md` |
| wireless-web | `.audit/wireless-web/bugs.md` |
| sensor-input | `.audit/sensor-input/bugs.md` |
| boot-init | `.audit/boot-init/bugs.md` |
| comms-protocols | `.audit/comms-protocols/bugs.md` |
