# Architektur-Roadmap: seL4-Style Synchronous IPC

Diese Roadmap beschreibt die schrittweise Ablösung der POSIX-Pipes durch ein synchrones Endpoint-Messaging-System (Rendezvous) mit "Short IPC" (Control Plane) und "Shared Memory" (Data Plane) in D3OS.

## 1. Erweiterung des Thread Control Blocks (TCB)
Die Register für den schnellen Datenaustausch und Capability-Transfer müssen direkt im TCB hinterlegt werden.

- **[MODIFY]** `os/kernel/src/process/thread.rs`
  - **Virtual Message Registers:** Hinzufügen von `ipc_buffer: [u64; 4]` zum `Thread`-Struct.
  - **Capability Slot:** Hinzufügen von `ipc_cap: Option<Capability<NamingObject>>` für den Transport während des Rendezvous.
  - **Badge & Status:** Hinzufügen von `ipc_badge: Option<u64>` (Identifikation des Senders via Capability-Badge) und eines Status/Fehler-Feldes.
  - **Thread-States:** Erweitern des `ThreadState`-Enums um `BlockedOnSend` und `BlockedOnReceive`.

## 2. Implementierung der Endpoint-Abstraktion (Control Plane)
Endpoints ersetzen die Puffer-Logik der Pipes durch pure Synchronisation (Warteschlangen).

- **[NEW]** `os/kernel/src/sync/endpoint.rs`
  - Erstellung der `Endpoint`-Struktur mit zwei Warteschlangen: `send_queue: VecDeque<Arc<Thread>>` und `recv_queue: VecDeque<Arc<Thread>>`.
  - Integration von `Endpoint` in den `NamingObject`-Enum (oder als eigenes Kernel-Objekt), damit es über den `CSpace` referenziert werden kann.
  - Definition neuer Capability-Rechte: `READ` (darf empfangen), `WRITE` (darf senden) und `GRANT` (darf Capabilities über den Endpoint delegieren).

## 3. Scheduler Fastpath (Direct Process Switch)
Um den Overhead des synchronen IPCs zu minimieren, überspringen wir die reguläre Ready-Queue, wenn ein Rendezvous stattfindet.

- **[MODIFY]** `os/kernel/src/process/scheduler.rs`
  - Implementierung einer `direct_switch(target_thread: Arc<Thread>)`-Methode.
  - Logik: Wenn Thread A an Thread B sendet (und B bereits im `recv_queue` wartete), wird A in die `ready_queue` geschoben und B wird sofort zum `current_thread`. Es erfolgt ein direkter Kontextwechsel via `Thread::switch(A, B)`.

## 4. Capability Delegation (Der Rendezvous-Transfer)
Der Kernmechanismus, um Capabilities von einem `CSpace` in einen anderen zu injizieren.

- **[MODIFY]** `os/kernel/src/sync/endpoint.rs` (Rendezvous-Logik)
  - Wenn Senden und Empfangen matchen:
    1. Kopiere `sender.ipc_buffer` nach `receiver.ipc_buffer`.
    2. Prüfe, ob `sender.ipc_cap` gesetzt ist UND die Sender-Endpoint-Capability das `GRANT`-Flag besitzt.
    3. Falls ja: Sperre (lock) den `receiver.cspace`, rufe `receiver.cspace.receive_naming_capability(cap)` auf.
    4. Schreibe das resultierende Capability-Handle (den `usize` Index im CSpace des Empfängers) als Rückgabewert in `receiver.ipc_buffer[0]`, damit der Userspace-Empfänger weiß, wo das neue Handle liegt.

## 5. System Call Interface
Neue syscalls, die direkt auf die TCB-Felder mappen.

- **[MODIFY]** `os/kernel/src/syscall/sys_ipc.rs` (Neu oder bestehend)
  - `sys_seL4_Send(endpoint_cap, msg0, msg1, msg2, msg3, cap_to_transfer)`
  - `sys_seL4_Recv(endpoint_cap)`
  - `sys_seL4_Call(endpoint_cap, ...)` (Kombination aus Send + BlockOnReceive)

## 6. Data Plane (Shared Memory)
Gemäß Anforderung erfolgt für lange Nachrichten keine Kopie durch den Kernel.

- **User-Space Architektur** (Keine direkten Kernel-Änderungen für den Datentransport erforderlich):
  - Der Sender allokiert Shared Memory Pages.
  - Die Capability für dieses Shared Memory wird via `sys_seL4_Send` (Short IPC + Capability Transfer) über den Endpoint gesendet.
  - Sender und Empfänger greifen direkt auf den Speicher zu. Der Endpoint (`sys_seL4_Call`) dient ab sofort nur noch als "Ring / Wakeup", um zu signalisieren, dass neue Daten im Shared Memory bereitliegen.

---

## Offene Fragen an dich:
1. Sollen wir die IPC-Register (`msg0..msg3`) direkt aus den Hardware-Registern (`r8`-`r11`) in den TCB kopieren (als echter Fastpath in Assembler) oder reicht der Umweg über reguläre Systemcall-Argumente in C/Rust-Manier für den ersten Prototyp?
2. Möchtest du, dass wir mit Phase 1 & 2 (TCB und Endpoint Structs) beginnen, sobald du diese Roadmap freigibst?
