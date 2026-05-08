# FANUC R-30iB Plus – TwinCAT EtherCAT Library

Ready-made function blocks for controlling a FANUC R-30iB Plus robot over EtherCAT using the FANUC EtherCAT Slave option (R-30iB Plus, PDO3 mapping). Handles the full enable sequence, safety signal management, fault reset, homing, cycle stop, program start (RSR), and user-defined output overwrite.

---

## Installation

A pre-compiled TwinCAT library file (`.library`) is available for download. Installing the library is the recommended approach - it does not require copying individual FB source files into your project.

**To install Library:** open TwinCAT XAE → PLC → References → Library repository → install.

**To Add Library:** open TwinCAT XAE → PLC → References → Add Library → browse under Miscellaneous. The FBs, DUTs, and GVLs will be available immediately.

The source files in this repository are the reference implementation. Use them if you need to modify the library or understand the internals.

---

## Library Contents

### Data Types (DUTs)

| Name | Description |
|---|---|
| `ST_FanucStatus` | Live robot status decoded from EtherCAT inputs |
| `ST_FanucControl` | Robot control commands written to EtherCAT outputs |
| `E_FanucEnableState` | State enum for `FB_FanucEnable` |
| `E_FanucFaultResetState` | State enum for `FB_FanucFaultReset` |
| `E_FanucHomingState` | State enum for `FB_FanucHoming` |
| `E_FanucCstopiState` | State enum for `FB_FanucCycleStop` |
| `E_FanucStartProgramState` | State enum for `FB_FanucStartProgram` |

### Function Blocks (POUs)

| Name | Description |
|---|---|
| `FB_FanucStatus` | Reads aDI byte array → `ST_FanucStatus` |
| `FB_FanucControl` | Writes `ST_FanucControl` → aDO byte array (bytes 0–3) |
| `FB_FanucEnable` | Enable sequence + safety signal management |
| `FB_FanucFaultReset` | Fault reset pulse (FAULT_RESET UI[5]) with confirmation |
| `FB_FanucHoming` | Homing sequence (HOME UI[7]) with completion detection |
| `FB_FanucCycleStop` | Cycle stop – 20 ms CSTOPI pulse (UI[4]) |
| `FB_FanucStartProgram` | Program start in RSR mode (programs 1..8) with ACK detection |
| `FB_FanucOverwrite` | Writes a decimal value (0–255) to `stControl.nDO3` (GI[25..33]) |

---

## Signal Mapping (FANUC PDO3)

### EtherCAT Inputs → `aDI` (`GVL_Fanuc.DI0`) - Robot UO → PLC

| Byte | Bit | Signal |
|---|---|---|
| `aDI[0]` | 0 | `UO[1]  CMDENBL`  – remote cmd enable confirmed |
| | 1 | `UO[2]  SYSRDY`   – servo on, no alarm |
| | 2 | `UO[3]  PROGRUN`  – program executing |
| | 3 | `UO[4]  PAUSED`   – program paused |
| | 4 | `UO[5]  HELD`     – motion held (HOLD active) |
| | 5 | `UO[6]  FAULT`    – alarm present |
| | 6 | `UO[7]  ATPERCH`  – robot at home position |
| | 7 | `UO[8]  TPENBL`   – teach pendant switch ON |
| `aDI[1]` | 0 | `UO[9]  BATALM`   – battery alarm |
| | 1 | `UO[10] BUSY`     – controller busy |
| `aDI[2]` | 0–7 | `UO[11..18]` ACK1..ACK8 (RSR) |

### EtherCAT Outputs → `aDO` (`GVL_Fanuc.DO0`) - PLC → Robot UI

| Byte | Bit | Signal |
|---|---|---|
| `aDO[0]` | 0 | `UI[1]  *IMSTP`      NC – FALSE = emergency stop |
| | 1 | `UI[2]  *HOLD`       NC – FALSE = pause robot |
| | 2 | `UI[3]  *SFSPD`      NC – FALSE = safe-speed limit |
| | 3 | `UI[4]  CSTOPI`         – pulse to stop cycle |
| | 4 | `UI[5]  FAULT_RESET`    – pulse to clear alarms |
| | 5 | `UI[6]  START`          – 20 ms pulse to start/resume |
| | 6 | `UI[7]  HOME`           – pulse to trigger homing |
| | 7 | `UI[8]  ENBL`        NC – FALSE = disable motion |
| `aDO[1]` | 0–7 | `UI[9..16]` RSR1..RSR8 – one-hot program select |
| `aDO[3]` | 0–7 | GI[25..33] – user-defined output (`stControl.nDO3`) |

> **NC = Normally Closed.** These signals must be held TRUE for the robot to run. Dropping them to FALSE stops or limits the robot - a broken cable fails safe.

---

## Enable Sequence (`FB_FanucEnable`)

Set `bEnable` to TRUE. The sequence steps through automatically:

| State | Behaviour |
|---|---|
| `Idle` | `bEnable = TRUE` → `ClearFault`. |
| `ClearFault` | Waits for `bFault = FALSE`. Does not send a reset pulse - use `FB_FanucFaultReset` for that. |
| `CheckTP` | Waits for TP switch OFF (`bTP_Enabled = FALSE`). `bTPWarning = TRUE` while waiting - wire to HMI indicator. |
| `WaitReady` | Waits for CMDENBL AND SYSRDY (5 s timeout → Error). |
| `Ready` | `bReady = TRUE`. Monitors continuously. |
| `EmergencyStop` | `bExtIMSTP = FALSE`. ENBL stays asserted; returns via ClearFault. |
| `SafeSpeed` | `bExtSFSPD = FALSE`. Robot runs at safe speed; `bReady = FALSE`. |
| `Paused` | `bMotion_Held` confirmed. HOLD dropped; ENBL stays asserted. |
| `Error` | Timeout in WaitReady. Toggle `bEnable` FALSE → TRUE to reset. |

### NC Signal Behaviour When Disabled (Idle / Error)

IMSTP, HOLD and SFSPD are held TRUE (non-asserting, Option A). ENBL = FALSE is the primary gate; asserting stops on top causes alarms. When in any active state the signals follow `bExt*` inputs directly.

### Start Signal

Rising edge on `bExtStart` triggers a 20 ms pulse on `stControl.Start` (UI[6]), only when in Ready state. `bExtStart` must be FALSE before the enable sequence can begin.

---

## Fault Reset (`FB_FanucFaultReset`)

Sends a 20 ms FAULT_RESET pulse (UI[5]) and waits for `bFault` to clear.

Trigger with a rising edge on `bReset`. Typically called from application logic when `fbEnable.eState = ClearFault` and the operator requests a reset. `FB_FanucEnable.ClearFault` waits passively for `bFault = FALSE`; this FB is what actually clears it.

**States:** Idle → SendPulse (20 ms) → WaitClear (5 s timeout) → Done → Idle.  
`bDone` goes TRUE for one scan when `bFault` is confirmed cleared.  
`bError` goes TRUE if the fault persists after the pulse; set `bReset` FALSE to reset.

---

## Homing (`FB_FanucHoming`)

Triggers the robot homing macro via a 20 ms HOME pulse (UI[7]) and waits for `bAt_Home` (UO[7]) to confirm completion.

Gated by `bReady` - homing only starts when the robot is enabled and ready. Rising edge on `bStart` begins the sequence. `bDone` goes TRUE for one scan when `bAt_Home` is confirmed (30 s timeout → Error).

---

## Cycle Stop (`FB_FanucCycleStop`)

Sends a 20 ms CSTOPI pulse (UI[4]) to stop the running program. Rising edge on `bTrigger` triggers the sequence.

**States:** Idle → Pulsing (20 ms) → Done → Idle.  
`bBusy` is TRUE while the pulse is active. `bDone` goes TRUE for one scan when complete.

This FB sends the pulse unconditionally.

---

## Program Start (`FB_FanucStartProgram`)

Starts programs 1–8 in RSR mode. `nProgram` maps to a one-hot 200 ms pulse on RSR1..RSR8 (UI[9..16]). The robot acknowledges on the matching ACK bit (UO[11..18]); `bAck` goes TRUE for one scan on confirmation.

Gated by `bReady`. Rising edge on `bStart` triggers the sequence.  
`bError` latches if `nProgram` is out of range (1..8); set `bStart` FALSE to reset.  
FANUC controller must be configured for RSR mode.

---

## Speed Overwrite (`FB_FanucOverwrite`)

Writes a decimal value (0–`nMax`) to `stControl.nDO3`, which `FB_FanucControl` maps to `aDO[3]` (GI[25..33]). 

Set `bEnable` TRUE to write `nValue`; FALSE clears the byte to 0.  
`bError` goes TRUE if `nValue` exceeds `nMax`; default `nMax` = 100.  
Must be called **before** `fbControl` in MAIN so the value is included in the final write.

---

## Main Template (Execution Order)

```pascal
// 1. Read status – always first
fbStatus(aDI := GVL_Fanuc.DI0, stStatus => stStatus);

// 2. Enable sequence (outputs stControl; other FBs stamp into it below)
fbEnable(
    bEnable   := bEnableRobot,
    stStatus  := stStatus,
    bExtIMSTP := bExtIMSTP,   // Wire from: NOT bSoftEStop
    bExtHold  := bExtHold,    // Wire from: NOT bPauseButton
    bExtSFSPD := bExtSFSPD,   // Wire from: bFenceOK / area scanner
    bExtStart := bExtStart,   // Wire from: HMI start button
    stControl => stControl);

// 3. Homing
fbHoming(
    bStart    := bStartHome,
    bReady    := fbEnable.bReady,
    stStatus  := stStatus,
    stControl := stControl);

// 4. Program start
fbStartProgram(
    bStart    := bStartProgram,
    nProgram  := nProgram,
    bReady    := fbEnable.bReady,
    stStatus  := stStatus,
    stControl := stControl);

// 5. Cycle stop
fbCycleStop(
    bTrigger  := bCycleStop,
    stControl := stControl);

// 6. Fault reset (call when fbEnable.eState = ClearFault and operator requests)
fbFaultReset(
    bReset    := bFaultReset,
    stStatus  := stStatus,
    stControl := stControl);

// 7. Output overwrite (e.g. speed override to GI[25..33])
fbOverwrite(
    nValue    := byOverwrite,
    bEnable   := TRUE,
    nMax      := 100,
    stControl := stControl);

// 8. Write outputs – always last
fbControl(aDO := GVL_Fanuc.DO0, stControl := stControl);
```

---

## Library Design Notes

### I/O Injection

No `AT %I*` or `AT %Q*` hardware-linked variables anywhere in the library. The caller passes `GVL_Fanuc.DI0` / `DO0` at the call site (`VAR_IN_OUT`). This is required for TwinCAT library compatibility and allows the same FBs to be used with any EtherCAT slave address or PDO mapping.

### `stControl` Ownership Pattern

`FB_FanucEnable` owns `stControl` as `VAR_OUTPUT`. It sets IMSTP/HOLD/SFSPD/ENBL and generates the START pulse. Every other FB (`FB_FanucFaultReset`, `FB_FanucHoming`, `FB_FanucCycleStop`, `FB_FanucStartProgram`, `FB_FanucOverwrite`) receives `stControl` as `VAR_IN_OUT` - each writes only its own bit(s) or field and leaves the rest untouched. `FB_FanucControl` is called last and maps the complete struct to `aDO`.

### Outputs in the State Machine

Each FB's outputs (`bReady`, `bError`, `bDone`, `bBusy`, `eState`, etc.) are derived after the state machine as simple expressions of the current state, e.g.:

```pascal
bReady := (_eState = E_FanucEnableState.Ready);
```

No output is scattered across multiple state branches.

### External Signals

`bExtIMSTP`, `bExtHold`, `bExtSFSPD` and `bExtStart` may be driven by HMI, safety PLC, or TwinCAT logic. Signal arbitration (AND/OR combining) is done at the call site; the FB sees one value per signal. When disabled (Idle / Error) the NC signals are held TRUE (Option A - non-asserting) because `ENBL = FALSE` is the correct gate. Asserting stops on top would cause robot alarms on every disable/enable cycle.
