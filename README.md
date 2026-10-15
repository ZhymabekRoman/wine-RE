# Wine-RE: Wine Staging Patches for Modern Windows Apps & UWP/WinUI 3

Experimental Wine Staging patch tree focused on reverse-engineering and implementing modern Windows NT kernel and userland subsystems required by **WinUI 3**, **Windows App SDK (WASDK)**, and native UWP runtimes.

Developed during native bring-up for **WhatsApp for Windows** (`v2.2620.102.0`, the final native WinUI 3 / XAML line using `WindowsAppRuntime.1.6` and `Microsoft.UI.Xaml 3.0.0.2411`).

---

## Reverse-Engineered Patch Sets

### 1. `ntdll-NtAlpcCreatePort`
* **Path**: `patches/ntdll-NtAlpcCreatePort/`
* **Target Subsystem**: NTDLL & Wineserver ALPC Engine
* **Components**:
  * Implements missing ALPC (Advanced Local Procedure Call) kernel interfaces:
    * `NtAlpcCreatePort`
    * `NtAlpcConnectPort`
    * `NtAlpcAcceptConnectPort`
    * `NtAlpcDisconnectPort`
    * `NtAlpcSendWaitReceivePort`
  * New `server/alpc.c` server object and 4 wineserver protocol requests (`create_alpc_port`, `connect_alpc_port`, `accept_alpc_connection`, `send_wait_receive_alpc_port`).
  * Roundtrip-verified message queueing, connection state machine, attribute negotiation, and context passing.
* **Why Required**: CoreMessaging (`CoreMessagingXP.dll` / `CoreMessaging.dll`) in Windows App SDK uses ALPC endpoints for in-process and inter-process message dispatch and registrar services. Without ALPC, WASDK crashes immediately on initialization.

### 2. `ntdll-WaitCompletionPacket`
* **Path**: `patches/ntdll-WaitCompletionPacket/`
* **Target Subsystem**: NTDLL & Wineserver Async Wait Completion Engine
* **Components**:
  * `0001-ntdll-server-Implement-wait-completion-packets-ALPC-.patch`:
    * Implements `NtCreateWaitCompletionPacket`, `NtAssociateWaitCompletionPacket`, and `NtCancelWaitCompletionPacket`.
    * Wineserver completion packet object and association with I/O completion ports / ALPC events.
  * `0002-ntdll-ALPC-connection-handshake-for-CoreMessaging.patch`:
    * Emulates Windows kernel ALPC connection handshake behavior expected by `CoreMessagingXP.dll`.
    * Solves PortContext population and status `0x87b2080c` handling.
    * Resolves flag byte check crashes at offset `+0x3a` in CoreMessaging connection state machine.
* **Depends on**: `ntdll-NtAlpcCreatePort`

### 3. `dcomp-DCompositionCreateDevice2` (Prior Art / Staging)
* **Path**: `patches/dcomp-DCompositionCreateDevice2/`
* **Target Subsystem**: DirectComposition (`dcomp.dll`), `dxgi`
* **Components**:
  * 66-patch set (Zhiyi Zhang @ CodeWeavers).
  * Implements `IDCompositionDevice`, `IDCompositionDevice2`, `IDCompositionDevice3`, `IDCompositionDesktopDevicePartner`, `IDCompositionTarget`, `IDCompositionVisual2`, `IDCompositionSurface`.
  * Adds `CreateSwapChainForComposition` stubs in DXGI.
* **Why Required**: WinUI 3 / XAML rendering compositor requires DirectComposition devices and swapchains.

---

## Companion Wine Work (in `wine-src`)

In addition to staging patches, running modern WinUI 3 apps requires companion implementations inside Wine core:

1. **Package Identity & Windowing Model**:
   * `AppPolicyGetWindowingModel` stubbed/forced to `AppPolicyWindowingModel_Universal` (`eb718a3`, `e4b6320`) to satisfy CoreMessaging ALPC port initialization without sandboxed AppContainer crash.
2. **UWP Application Lifecycle (`twinapi.appcore.dll`)**:
   * `ICoreApplication`, `CoreApplication.Run`, and `CoreWindow` creation (`2ca23f0` through `a84f092`).
   * Passes `CoreWindow` into `FrameworkView.SetWindow`.
3. **DispatcherQueue & CoreDispatcher**:
   * `IStaticLifetime`, `IMap`, `DispatcherQueueController` and thread dispatcher pump support (`99e0c52` through `fec4ec7`).
   * Allows execution past `Microsoft.UI.Xaml.Application.Start`.

---

## How to Apply Patches

Using the staging `patchinstall.py` tool on your Wine source tree:

```bash
# Apply ALPC infrastructure
./staging/patchinstall.py DESTDIR="/path/to/wine-src" ntdll-NtAlpcCreatePort

# Apply Wait Completion Packets + CoreMessaging handshake
./staging/patchinstall.py DESTDIR="/path/to/wine-src" ntdll-WaitCompletionPacket

# Or apply all custom patches
./staging/patchinstall.py DESTDIR="/path/to/wine-src" ntdll-NtAlpcCreatePort ntdll-WaitCompletionPacket dcomp-DCompositionCreateDevice2
```

---

## Upstream & Base

* Base: Wine Staging 11.10 (`v11.10`)
* Upstream Wine: https://gitlab.winehq.org/wine/wine.git
* Wine-Staging: https://gitlab.winehq.org/wine/wine-staging.git
