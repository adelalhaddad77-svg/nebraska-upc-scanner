# Claude Code prompt — fix cash drawer not opening

This repository only hosts the app's privacy-policy page, so the fix cannot be made here.
Open Claude Code **in the repository/project that contains the Nebraska UPC Scanner app source code**
(the Xcode project / app codebase with the checkout screen and printer code) and paste everything
below the horizontal line as your prompt.

---

Fix the cash drawer bug in this point-of-sale app (Nebraska UPC Scanner).

## Context

A thermal receipt printer is attached to the app (Bluetooth, LAN, or accessory connection), and the
cash drawer is connected to that printer's drawer-kick port (the RJ11/RJ12 "DK" jack on the back of
the printer). The drawer has no connection of its own — it only opens when the printer receives a
"drawer kick" pulse command from the software.

**Bug:** the cash drawer never opens:

1. Tapping the cash drawer icon/button in the app does nothing.
2. Completing a checkout (which prints the receipt on the attached printer) does not open the
   drawer automatically.

## Step 1 — Investigate and report the root cause before fixing

- Locate the printing layer: search the codebase for things like `print`, `printer`, `receipt`,
  `ESC`, `0x1B`, `escpos`, `epos`, `Epson`, `StarIO`, `StarPRNT`, `bluetooth`, `9100`,
  `ExternalAccessory`, `EAAccessory`, `CBPeripheral`, `OutputStream`.
- Identify the protocol/SDK used to print: Epson ePOS2, Star (StarXpand/StarPRNT or Star line
  mode), or raw ESC/POS bytes over a Bluetooth/TCP/accessory stream. Note: AirPrint /
  `UIPrintInteractionController` cannot kick a cash drawer — if receipts go through AirPrint, call
  that out; the drawer feature needs the raw-bytes/SDK path to the printer.
- Find the cash drawer icon's tap handler (search `drawer`, `openDrawer`, `cashDrawer`, `noSale`,
  or the icon's asset name) and check whether it sends anything to the printer at all.
- Find the checkout / complete-sale flow and check whether a kick command is included in (or sent
  alongside) the receipt print job.
- State clearly which defect it is: command never sent, wrong bytes/pin, wrong command set for the
  printer brand, or bytes written but the connection closed before they were flushed.

## Step 2 — Implement one shared `openCashDrawer()` in the printer service

Every drawer open must go through a single function in the printer layer, used by both the icon and
checkout. Use the correct command for the protocol found in Step 1:

- **Raw ESC/POS** (Epson-compatible generic thermal printers):
  - Standard pulse `ESC p m t1 t2` → bytes `1B 70 00 19 FA` (pin 2). For maximum hardware
    compatibility also send pin 5: `1B 70 01 19 FA`. (`t1=0x19` ≈ 50 ms on, `t2=0xFA` ≈ 500 ms
    off — a too-short pulse is a classic reason the solenoid never fires.)
  - Real-time fallback that works even while the printer is busy or in a recoverable error state
    (cover open / paper low): `DLE DC4 1 m t` → `10 14 01 00 05` (pin 2) / `10 14 01 01 05` (pin 5).
- **Epson ePOS2 SDK:** `addPulse(EPOS2_DRAWER_2PIN, EPOS2_PULSE_100)` on the printer/builder
  object, then send the job.
- **Star:** use the SDK drawer API (StarXpand `openCashDrawer` / StarPRNT
  `appendPeripheralChannel(.no1)`). In Star *line mode*, drawer 1 fires with the single byte `BEL`
  (`0x07`) — Star line mode does **not** use `ESC p`.
- If the printer brand/model is configurable in the app, branch on it; otherwise default to the
  ESC/POS sequence.

Hard requirements for this function:

- Must work with an **empty print job** (icon tap = kick only, no receipt paper fed).
- Flush/drain the output stream and keep the connection/session open until the write completes —
  a very common bug is writing the kick bytes and immediately closing the Bluetooth session or
  socket so the bytes never leave the buffer.
- If no printer is connected/reachable, fail gracefully with a clear user-visible message
  ("Printer not connected — cash drawer could not be opened"), never a silent no-op.
- Log send success/failure so future issues can be diagnosed.

## Step 3 — Wire the cash drawer icon

- The icon's tap handler must call `openCashDrawer()`.
- Debounce rapid taps (ignore further taps for ~1 second after a kick) so the solenoid is not
  hammered.

## Step 4 — Auto-open on checkout

- When a sale is finalized, open the drawer automatically through the same attached printer:
  - If a receipt is printed, embed the kick bytes **at the start of the same print job** so the
    drawer pops immediately and there is one job on one connection — no separate write racing the
    receipt.
  - If a checkout can complete without printing a receipt, still call `openCashDrawer()` on
    completion.
- Ensure the drawer opens exactly **once** per checkout (no double kick when both the print job and
  a completion handler could fire it).
- If the app has a settings screen, add a toggle **"Open cash drawer on checkout"**, default ON.
  The icon must keep working regardless of the toggle. If the app distinguishes cash vs. card
  payments, optionally auto-open only for cash payments.

## Step 5 — Verify

- The project must build; run any existing tests and linters.
- If the printer layer is unit-testable, add tests asserting the exact byte sequences above are
  queued for the drawer command.
- Include manual test steps in your summary:
  1. Tap the drawer icon with the printer connected → drawer opens, nothing prints.
  2. Complete a checkout that prints a receipt → receipt prints AND drawer opens.
  3. Complete a checkout with printing disabled (if possible) → drawer still opens.
  4. Tap the icon with the printer disconnected → clear error message, no crash, no hang.

## Hardware sanity notes (include these in your summary for the operator)

If the software verifiably sends the correct bytes and the drawer still stays shut, the fault is
physical — check:

- The drawer cable is plugged into the printer's **DK/drawer** RJ11/RJ12 port — not the LINE/phone
  jack and not the LAN port (they look alike).
- The cable is a real drawer-kick cable; an ordinary telephone cord is often wired differently.
- The printer's own self-test / utility "drawer kick" function opens the drawer (this proves the
  drawer, cable, and printer are fine and isolates the bug to the app).
- The drawer's key lock is in the unlocked position.

## Deliverables

- The root cause, stated in one or two sentences.
- The code changes implementing all of the above.
- The manual test checklist and (if run) results.
