╔══════════════════════════════════════════════════════════════════════════════╗
║                  FANUC R-30iB Plus  –  TwinCAT EtherCAT Library              ║
╚══════════════════════════════════════════════════════════════════════════════╝

  PURPOSE
  ───────
  Ready-made function blocks for controlling a FANUC R-30iB Plus robot over
  EtherCAT using the FANUC EtherCAT Slave option (R-30iB Plus, PDO3 mapping).
  Handles the full enable sequence, interlock validation, safety signal
  management, fault reset, homing, cycle stop, and program start in both
  RSR and PNS modes.

  ──────────────────────────────────────────────────────────────────────────────
  LIBRARY CONTENTS
  ──────────────────────────────────────────────────────────────────────────────

  Data types (DUTs\)
    ST_FanucStatus          Live robot status decoded from EtherCAT inputs
    ST_FanucControl         Robot control commands written to EtherCAT outputs
    E_FanucEnableState      State enum for FB_FanucEnable
    E_FanucFaultResetState  State enum for FB_FanucFaultReset
    E_FanucHomingState      State enum for FB_FanucHoming
    E_FanucCstopiMode       Cycle-stop mode selector (Standard / Abort)
    E_FanucStartMode        RSR or PNS program start mode selector

  Function blocks (POUs\)
    FB_FanucStatus          Reads aDI byte array → ST_FanucStatus
    FB_FanucControl         Writes ST_FanucControl → aDO byte array
    FB_FanucEnable          Enable sequence + interlock validation + safety signals
    FB_FanucFaultReset      Fault reset pulse (FAULT_RESET UI[5]) with confirmation
    FB_FanucHoming          Homing sequence (HOME UI[7]) with completion detection
    FB_FanucCycleStop       Cycle stop (CSTOPI UI[4]) in Standard or Abort mode
    FB_FanucStartProgram    Program start in RSR or PNS mode with ACK detection

  ──────────────────────────────────────────────────────────────────────────────
  SIGNAL MAPPING  (FANUC PDO3)
  ──────────────────────────────────────────────────────────────────────────────

  EtherCAT INPUTS  →  aDI (GVL_Fanuc.DI0)   Robot UO → PLC
  ┌────────┬──────────┬────────────────────────────────────────────────────┐
  │ Byte   │ Bit      │ Signal                                             │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDI[0] │ 0        │ UO[1]  CMDENBL   – remote cmd enable confirmed     │
  │        │ 1        │ UO[2]  SYSRDY    – servo on, no alarm              │
  │        │ 2        │ UO[3]  PROGRUN   – program executing               │
  │        │ 3        │ UO[4]  PAUSED    – program paused                  │
  │        │ 4        │ UO[5]  HELD      – motion held (HOLD active)       │
  │        │ 5        │ UO[6]  FAULT     – alarm present                   │
  │        │ 6        │ UO[7]  ATPERCH   – robot at home position          │
  │        │ 7        │ UO[8]  TPENBL    – teach pendant switch ON         │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDI[1] │ 0        │ UO[9]  BATALM    – battery alarm                   │
  │        │ 1        │ UO[10] BUSY      – controller busy                 │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDI[2] │ 0..7     │ UO[11..18] ACK1..ACK8 (RSR) / SNO1..SNO8 (PNS)   │
  └────────┴──────────┴────────────────────────────────────────────────────┘

  EtherCAT OUTPUTS  →  aDO (GVL_Fanuc.DO0)   PLC → Robot UI
  ┌────────┬──────────┬────────────────────────────────────────────────────┐
  │ Byte   │ Bit      │ Signal                                             │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDO[0] │ 0        │ UI[1]  *IMSTP      NC – FALSE = emergency stop     │
  │        │ 1        │ UI[2]  *HOLD       NC – FALSE = pause robot        │
  │        │ 2        │ UI[3]  *SFSPD      NC – FALSE = safe-speed limit   │
  │        │ 3        │ UI[4]  CSTOPI         – pulse to abort cycle       │
  │        │ 4        │ UI[5]  FAULT_RESET    – pulse to clear alarms      │
  │        │ 5        │ UI[6]  START          – start / resume program     │
  │        │ 6        │ UI[7]  HOME           – trigger homing macro       │
  │        │ 7        │ UI[8]  ENBL        NC – FALSE = disable motion     │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDO[1] │ 0..7     │ UI[9..16] RSR1..RSR8 (RSR) / PNS1..PNS8 (PNS)    │
  ├────────┼──────────┼────────────────────────────────────────────────────┤
  │ aDO[2] │ 1        │ UI[18] PROD_START  – triggers PNS execution        │
  └────────┴──────────┴────────────────────────────────────────────────────┘

  NC = Normally Closed. These signals MUST be held TRUE for the robot to run.
  Dropping them to FALSE stops or limits the robot – a broken cable fails safe.

  ──────────────────────────────────────────────────────────────────────────────
  ENABLE SEQUENCE  (FB_FanucEnable)
  ──────────────────────────────────────────────────────────────────────────────

  Set bEnable to TRUE. All interlock preconditions must be met before the FB
  transitions out of Idle. The sequence then steps through automatically:

    Idle        bEnable = TRUE AND all interlock preconditions met → ClearFault.
                bInterlockX outputs indicate which signal is blocking (HMI use).
    ClearFault  Waits for bFault = FALSE. Does not send a reset pulse – use
                FB_FanucFaultReset for that. Aborts to Idle on interlock loss.
    CheckTP     Waits for TP switch OFF (bTP_Enabled = FALSE).
                bTPWarning = TRUE while waiting – wire to HMI indicator.
    WaitReady   Waits for CMDENBL AND SYSRDY (5 s timeout → Error).
    Ready       bReady = TRUE. Monitors continuously.

  Additional states (monitored from Ready, Paused, SafeSpeed):
    EmergencyStop  bExtIMSTP = FALSE. ENBL stays asserted; returns via ClearFault.
    SafeSpeed      bExtSFSPD = FALSE. Robot runs at safe speed; bReady = FALSE.
    Paused         bMotion_Held confirmed. HOLD dropped; ENBL stays asserted.
    Error          Timeout in WaitReady. Toggle bEnable FALSE → TRUE to reset.

  Interlock preconditions (all checked in Idle before starting the sequence,
  and monitored mid-sequence in ClearFault / CheckTP / WaitReady):
    bExtIMSTP must be TRUE  – no E-stop requested
    bExtHold  must be TRUE  – no hold/pause requested
    bExtSFSPD must be TRUE  – fence closed, no safe-speed requested
    bExtStart must be FALSE – no pending start command

  NC signal behaviour when disabled (Idle / Error):
    IMSTP, HOLD and SFSPD are held TRUE (non-asserting, Option A).
    ENBL = FALSE is the primary gate; asserting stops on top causes alarms.
    When in any active state the signals follow bExt* inputs directly.

  Start signal:
    bExtStart is forwarded to stControl.Start only when in the Ready state.
    It is a precondition that bExtStart = FALSE before enabling.

  ──────────────────────────────────────────────────────────────────────────────
  FAULT RESET  (FB_FanucFaultReset)
  ──────────────────────────────────────────────────────────────────────────────

  Sends a 20 ms FAULT_RESET pulse (UI[5]) and waits for bFault to clear.

  Trigger with a rising edge on bReset. Typically called from application logic
  when fbEnable.eState = ClearFault and the operator requests a reset.
  FB_FanucEnable.ClearFault waits passively for bFault = FALSE; this FB is
  what actually clears it.

  States: Idle → SendPulse (20 ms) → WaitClear (5 s timeout) → Done → Idle.
  bDone goes TRUE for one scan when bFault is confirmed cleared.
  bError goes TRUE if the fault persists after the pulse; set bReset FALSE to reset.

  ──────────────────────────────────────────────────────────────────────────────
  HOMING  (FB_FanucHoming)
  ──────────────────────────────────────────────────────────────────────────────

  Triggers the robot homing macro via a 20 ms HOME pulse (UI[7]) and waits
  for bAt_Home (UO[7]) to confirm completion.

  Gated by bReady – homing only starts when the robot is enabled and ready.
  Rising edge on bStart begins the sequence.
  bDone goes TRUE for one scan when bAt_Home is confirmed (30 s timeout → Error).

  ──────────────────────────────────────────────────────────────────────────────
  CYCLE STOP  (FB_FanucCycleStop)
  ──────────────────────────────────────────────────────────────────────────────

  Sends a CSTOPI pulse (UI[4]) to stop the running program.

  Set eCstopiMode to match the FANUC controller "CSTOPI for ABORT" setting:
    Standard  – 20 ms pulse; bDone goes TRUE immediately after the pulse.
    Abort     – 20 ms pulse; bDone waits until PROGRUN = FALSE (program stopped).

  Rising edge on bTrigger sends the pulse.

  ──────────────────────────────────────────────────────────────────────────────
  PROGRAM START  (FB_FanucStartProgram)
  ──────────────────────────────────────────────────────────────────────────────

  RSR mode  (eMode = E_FanucStartMode.RSR)
    Programs 1..8. Each maps to one RSR bit (UI[9..16]), pulsed for 200 ms.
    Robot acknowledges on ACK bits (UO[11..18]) → fbStart.bAck TRUE one scan.
    FANUC controller must be configured for RSR mode.

  PNS mode  (eMode = E_FanucStartMode.PNS)
    Programs 1..255. Binary-encoded across PNS bits (UI[9..16]).
    PROD_START (UI[18]) is pulsed simultaneously for 200 ms.
    Robot acknowledges via PROGRUN rising edge → fbStart.bAck TRUE one scan.
    FANUC controller must be configured for PNS mode.

  RSR and PNS are mutually exclusive – configured on the robot controller,
  not switchable at runtime. Set eStartMode to match the controller config.

  ──────────────────────────────────────────────────────────────────────────────
  MAIN TEMPLATE  (execution order)
  ──────────────────────────────────────────────────────────────────────────────

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

  // 3. Program start (VAR_IN_OUT stamps start bits into stControl)
  fbStart(
      bStart    := bStartProgram,
      nProgram  := nProgram,
      eMode     := eStartMode,
      bReady    := fbEnable.bReady,
      stStatus  := stStatus,
      stControl := stControl);

  // 4. Cycle stop
  fbCycleStop(
      bTrigger  := bCycleStop,
      eMode     := eCstopiMode,
      stStatus  := stStatus,
      stControl := stControl);

  // 5. Homing
  fbHoming(
      bStart    := bStartHome,
      bReady    := fbEnable.bReady,
      stStatus  := stStatus,
      stControl := stControl);

  // 6. Fault reset (call when fbEnable.eState = ClearFault and operator requests)
  fbFaultReset(
      bReset    := bFaultReset,
      stStatus  := stStatus,
      stControl := stControl);

  // 7. Write outputs – always last
  fbControl(aDO := GVL_Fanuc.DO0, stControl := stControl);

  ──────────────────────────────────────────────────────────────────────────────
  LIBRARY DESIGN NOTES
  ──────────────────────────────────────────────────────────────────────────────

  I/O injection
    No AT %I* or AT %Q* hardware-linked variables anywhere in the library.
    The caller passes GVL_Fanuc.DI0 / DO0 at the call site (VAR_IN_OUT).
    This is required for TwinCAT library compatibility and allows the same FBs
    to be used with any EtherCAT slave address or PDO mapping.

  stControl ownership pattern
    FB_FanucEnable owns stControl as VAR_OUTPUT. It sets IMSTP/HOLD/SFSPD/ENBL/START
    and clears all other bits on every scan.
    Every other FB (FB_FanucFaultReset, FB_FanucHoming, FB_FanucCycleStop,
    FB_FanucStartProgram) receives stControl as VAR_IN_OUT – they each write
    only their own bit(s) and leave the rest untouched.
    The calling program passes fbEnable.stControl to each of these FBs in turn.

  Outputs in the state machine
    Each FB's outputs (bReady, bError, bDone, bBusy, etc.) are derived after
    the state machine as simple expressions of the current state, e.g.:
      bReady := (_eState = E_FanucEnableState.Ready);
    No output is scattered across multiple state branches.

  Interlock signals
    bExtIMSTP, bExtHold, bExtSFSPD and bExtStart may be driven by HMI,
    safety PLC, or TwinCAT logic. Signal arbitration (AND/OR combining) is
    done at the call site; the FB sees one value per signal.
    When disabled (Idle / Error) the NC signals are held TRUE (Option A –
    non-asserting) because ENBL = FALSE is the correct gate. Asserting stops
    on top would cause robot alarms on every disable/enable cycle.
