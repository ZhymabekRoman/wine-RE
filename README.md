# Wine-RE: Wine Staging Patches for Modern Windows Apps & UWP/WinUI 3

Experimental Wine Staging patch tree focused on reverse-engineering and implementing modern Windows NT kernel and userland subsystems required by **WinUI 3**, **Windows App SDK (WASDK)**, and native UWP runtimes.

Developed during native bring-up for **WhatsApp for Windows** (`v2.2620.102.0`, the final native WinUI 3 / XAML line using `WindowsAppRuntime.1.6` and `Microsoft.UI.Xaml 3.0.0.2411`).

**Base**: Wine Staging **v11.19** (`11dd86bd`) / Upstream Wine `a15ee2c`.

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
* **Patches (0001–0008)**:
  * `0001`: Implements `NtCreateWaitCompletionPacket`, `NtAssociateWaitCompletionPacket`, and `NtCancelWaitCompletionPacket` + wineserver completion packet object.
  * `0002`: Emulates Windows kernel ALPC connection handshake behavior expected by `CoreMessagingXP.dll`, status `0x87b2080c`, and flag byte checks at offset `+0x3a`.
  * `0003`: Transports ALPC connection-message payload and `ClientId`.
  * `0004`: Implements `NtAlpcImpersonateClientOfPort` (in-process impersonation).
  * `0005`: ALPC post-accept message routing + per-client `PortContext` and `ClientId` delivery.
  * `0006`: Stamps ALPC message `Type` and assigns `MessageId`; fixes send use-after-free.
  * `0007`: Corrects per-message `PortContext` for reply direction.
  * `0008`: Implements ALPC Section Views (`NtAlpcCreatePortSection`, `NtAlpcCreateSectionView`, `NtAlpcDeletePortSection`, `NtAlpcDeleteSectionView`) and `ALPC_MESSAGE_VIEW_ATTRIBUTE` (0x40000000) wire propagation for shared memory transport.

### 3. `kernelbase-PackageIdentity`
* **Path**: `patches/kernelbase-PackageIdentity/`
* **Target Subsystem**: KernelBase / AppModel
* **Patches (0001–0003)**:
  * `0001`: Reports package identity parsed from environment variables (`PACKAGE_FAMILY_NAME`, `PACKAGE_FULL_NAME`) for unpackaged / standalone runtime launch.
  * `0002`: Implements `GetCurrentPackageId`, `GetCurrentPackageInfo`, `GetCurrentPackagePath`, and `GetPackagePathByFullName`.
  * `0003`: `AppPolicyGetWindowingModel` returns `AppPolicyWindowingModel_Universal` for packaged apps to bypass AppContainer sandboxing checks in CoreMessaging.

### 4. `twinapi-appcore-UWP`
* **Path**: `patches/twinapi-appcore-UWP/`
* **Target Subsystem**: `twinapi.appcore.dll`, `windows.ui.dll`, Dispatcher
* **Depends on**: `kernelbase-PackageIdentity`
* **Patches (0001–0009)**:
  * `0001`: Implements `ICoreApplication` interface on CoreApplication factory.
  * `0002`: Implements `CoreApplication.Run` and minimal `CoreApplicationView`.
  * `0003`: Wires `CoreWindow` into `FrameworkView.SetWindow`.
  * `0004`: Implements `IStaticLifetime` and `IMap` on CoreApplication factory.
  * `0005`: Stubs `ApiInformation` present-checks to return `TRUE`; shares `CoreApplicationView`.
  * `0006`: Attribution fix on `static_lifetime_map.c`.
  * `0007`: Adds `ICoreApplicationUnhandledError` to CoreApplication factory.
  * `0008`: Extends `IStaticLifetime` vtable and adds `ICoreApplication2`.
  * `0009`: Creates `DispatcherQueueController` and `DispatcherQueue` pump before `FrameworkView.Initialize` (allows execution past `Microsoft.UI.Xaml.Application.Start`).

### 5. `dcomp-DCompositionCreateDevice2`
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
    kernelbase-PackageIdentity \
    twinapi-appcore-UWP \
    dcomp-DCompositionCreateDevice2
```

---

## Repository & Branches

* **Current Base**: Synced to Wine-Staging `v11.19` (`11dd86bd`)
* **Upstream Wine**: `https://gitlab.winehq.org/wine/wine.git`
* **Wine-Staging**: `https://gitlab.winehq.org/wine/wine-staging.git`
* **GitHub**: `https://github.com/ZhymabekRoman/wine-RE`
