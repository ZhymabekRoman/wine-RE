# wine-ultimate: Wine Staging Patches for Modern Windows Apps & UWP/WinUI 3

Experimental Wine Staging patch tree focused on reverse-engineering and implementing modern Windows NT kernel and userland subsystems required by **WinUI 3**, **Windows App SDK (WASDK)**, and native UWP runtimes.

Developed during native bring-up for **WhatsApp for Windows** (`v2.2620.102.0`, the final native WinUI 3 / XAML line using `WindowsAppRuntime.1.6` and `Microsoft.UI.Xaml 3.0.0.2411`).

**Base**: Wine Staging **v11.19** (`11dd86bd`) / Upstream Wine `a15ee2c`.

---

## Current Status: WhatsApp Desktop Running

* **Process State**: Running stably (`WhatsApp.Root.exe`, PID active, 440MB+ RSS).
* **Display / Window**: Top-level 1016×734 window is successfully mapped to X11 display (`Map State: IsViewable`).
* **Graphics Stack**: DirectWrite, Direct2D, DXGI, and Direct3D 11 device contexts initialized.
* **CoreMessaging / ALPC Bus**: Full bidirectional handshake and data message exchange functional across UI, Registrar, and DWM manipulation threads.

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
* **Why Required**: CoreMessaging (`CoreMessagingXP.dll` / `CoreMessaging.dll`) in Windows App SDK uses ALPC endpoints for in-process and inter-process message dispatch and registrar services.

### 2. `ntdll-WaitCompletionPacket`
* **Path**: `patches/ntdll-WaitCompletionPacket/`
* **Target Subsystem**: NTDLL & Wineserver Async Wait Completion Engine & ALPC Message Bus
* **Depends on**: `ntdll-NtAlpcCreatePort`
* **Patches (0001–0010)**:
  * `0001`: Implements `NtCreateWaitCompletionPacket`, `NtAssociateWaitCompletionPacket`, and `NtCancelWaitCompletionPacket` + wineserver completion packet object.
  * `0002`: Emulates Windows kernel ALPC connection handshake behavior expected by `CoreMessagingXP.dll`, status `0x87b2080c`, and flag byte checks at offset `+0x3a`.
  * `0003`: Transports ALPC connection-message payload and `ClientId`.
  * `0004`: Implements `NtAlpcImpersonateClientOfPort` (in-process impersonation).
  * `0005`: ALPC post-accept message routing + per-client `PortContext` and `ClientId` delivery.
  * `0006`: Stamps ALPC message `Type` and assigns `MessageId`; fixes send use-after-free.
  * `0007`: Corrects per-message `PortContext` for reply direction.
  * `0008`: Implements `OBJ_OPENIF` named port reuse and correlated synchronous request/reply matching by `MessageId` in wineserver async queue (resolves CoreMessaging in-process registrar collision `0xc0000035` and `CoreUICallReceive` empty parameter decode crash).
  * `0009`: Server-side multi-connection reply routing by `MessageId` across multiple client connection endpoints on the listening port.
  * `0010`: Routes client data messages directly to connected communication `peer` rather than fallback listening port, resolving `0x87B2000C` validate buffer error in `CoreMessagingXP.dll` during "Lifted" endpoint registration.

### 3. `kernelbase-PackageIdentity`
* **Path**: `patches/kernelbase-PackageIdentity/`
* **Target Subsystem**: KernelBase / AppModel & Wineserver Object Handles
* **Patches (0001–0004)**:
  * `0001`: Reports package identity parsed from environment variables (`PACKAGE_FAMILY_NAME`, `PACKAGE_FULL_NAME`) for unpackaged / standalone runtime launch.
  * `0002`: Implements `GetCurrentPackageId`, `GetCurrentPackageInfo`, `GetCurrentPackagePath`, and `GetPackagePathByFullName`.
  * `0003`: `AppPolicyGetWindowingModel` returns `AppPolicyWindowingModel_Universal` for packaged apps to bypass AppContainer sandboxing checks in CoreMessaging.
  * `0004`: Allows in-process handle access right escalation (`entry->access |= access`) in wineserver to eliminate false-positive `STATUS_ACCESS_DENIED` (`0xc0000022` / `0xd0000022`), and makes `CloseHandle` non-fatal on invalid/unclosable handles to avoid WIL failfast assertions in `FrameworkUdk.dll`.

### 4. `twinapi-appcore-UWP`
* **Path**: `patches/twinapi-appcore-UWP/`
* **Target Subsystem**: `twinapi.appcore.dll`, `windows.ui.dll`, Dispatcher
* **Depends on**: `kernelbase-PackageIdentity`
* **Patches (0001–0011)**:
  * `0001`: Implements `ICoreApplication` interface on CoreApplication factory.
  * `0002`: Implements `CoreApplication.Run` and minimal `CoreApplicationView`.
  * `0003`: Wires `CoreWindow` into `FrameworkView.SetWindow`.
  * `0004`: Implements `IStaticLifetime` and `IMap` on CoreApplication factory.
  * `0005`: Stubs `ApiInformation` present-checks to return `TRUE`; shares `CoreApplicationView`.
  * `0006`: Attribution fix on `static_lifetime_map.c`.
  * `0007`: Adds `ICoreApplicationUnhandledError` to CoreApplication factory.
  * `0008`: Extends `IStaticLifetime` vtable and adds `ICoreApplication2`.
  * `0009`: Creates `DispatcherQueueController` and `DispatcherQueue` pump before `FrameworkView.Initialize`.
  * `0010`: Implements real Win32 `HWND` backing for `CoreWindow`, `ICoreWindowInterop` (`45d64a29-a63e-4cb6-b498-5781d297cb98`), `WS_VISIBLE`, `ShowWindow(SW_SHOW)`, and UWP `IActivatedEventArgs` launch event dispatch.
  * `0011`: Implements `Windows.ApplicationModel.AppInstance` factory and `IAppInstanceStatics::GetActivatedEventArgs` to satisfy `Microsoft.Windows.AppLifecycle.AppInstance.GetActivatedEventArgs()` in WASDK.

### 5. `combase-AgileObject`
* **Path**: `patches/combase-AgileObject/`
* **Target Subsystem**: COMBase Free-Threaded Marshaler
* **Patches (0001)**:
  * `0001`: Implements `IID_IAgileObject` (`94ea2b94-e9cc-49e0-c0ff-ee64ca8f5b90`) in `ftmarshaler_inner_QueryInterface`.

### 6. `coremessaging-DispatcherQueue`
* **Path**: `patches/coremessaging-DispatcherQueue/`
* **Target Subsystem**: `coremessaging.dll`
* **Patches (0001)**:
  * `0001`: Implements `IDispatcherQueue` and `IDispatcherQueue2` (`CreateTimer`, `TryEnqueue`, `get_HasThreadAccess`), wires `dispatcher_queue_controller_get_DispatcherQueue`, and implements `CoreUICreate`.

### 7. `windows.graphics-DisplayInformation`
* **Path**: `patches/windows.graphics-DisplayInformation/`
* **Target Subsystem**: `windows.graphics.dll`
* **Patches (0001)**:
  * `0001`: Implements `IDisplayInformation` and `IDisplayInformation2` (DPI 96, Scale 100%, Landscape, 1.0 raw pixels) and wires `DisplayInformation.GetForCurrentView`.

### 8. `profapi-UserToken`
* **Path**: `patches/profapi-UserToken/`
* **Target Subsystem**: `profapi.dll`
* **Patches (0001)**:
  * `0001`: Implements ordinal 114 user profile token stub returning `ERROR_SUCCESS`.

### 9. `dcomp-DCompositionCreateDevice2`
* **Path**: `patches/dcomp-DCompositionCreateDevice2/`
* **Target Subsystem**: DirectComposition (`dcomp.dll`), `dxgi`
* **Patches (0001–0067)**:
  * `0001–0066`: CodeWeavers DComposition patchset (Zhiyi Zhang). Implements `IDCompositionDevice/2/3`, `IDCompositionDesktopDevicePartner`, `IDCompositionTarget`, `IDCompositionVisual2`, `IDCompositionSurface`, and `CreateSwapChainForComposition` stubs.
  * `0067`: Exports 11 `DCompPrivates_*` stubs needed for `Microsoft.Internal.FrameworkUdk.dll` delay-import resolution.

---

## How to Apply Patches

Using Wine Staging `patchinstall.py`:

```bash
# Apply full WASDK / WinUI 3 stack to Wine tree:
./staging/patchinstall.py DESTDIR="/path/to/wine" \
    ntdll-NtAlpcCreatePort \
    ntdll-WaitCompletionPacket \
    combase-AgileObject \
    coremessaging-DispatcherQueue \
    kernelbase-PackageIdentity \
    profapi-UserToken \
    windows.graphics-DisplayInformation \
    twinapi-appcore-UWP \
    dcomp-DCompositionCreateDevice2
```

---

## Repository & Branches

* **Current Base**: Synced to Wine-Staging `v11.19` (`11dd86bd`)
* **Upstream Wine**: `https://gitlab.winehq.org/wine/wine.git`
* **Wine-Staging**: `https://gitlab.winehq.org/wine/wine-staging.git`
* **GitHub**: `https://github.com/ZhymabekRoman/wine-ultimate`
