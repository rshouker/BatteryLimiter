# Battery Status Retrieval in Win32

## GetSystemPowerStatus

- Function: `BOOL GetSystemPowerStatus(LPSYSTEM_POWER_STATUS lpSystemPowerStatus);`
- Data struct: 
  ```cpp
  typedef struct {
      BYTE  ACLineStatus;      // 0=offline,1=online,255=unknown
      BYTE  BatteryFlag;       // Flags: high, low, critical, charging, etc.
      BYTE  BatteryLifePercent;// 0–100%, 255=unknown
      DWORD BatteryLifeTime;   // Seconds of runtime remaining, –1=unknown
      DWORD BatteryFullLifeTime; // Seconds at full charge, –1=unknown
  } SYSTEM_POWER_STATUS;
  ```
- This covers current charge % (`BatteryLifePercent`) and charging state (`BatteryFlag` bit 8).

## Battery Gauge Outline & Clip Mask (WPF/WinUI)

Use a `Path` for the outline and a `Rectangle` fill clipped by a `RectangleGeometry` whose height is bound to the battery percent:

```xml
<Grid Width="60" Height="120">
  <!-- Outline -->
  <Path Data="M2,2 H58 V100 H2 Z M58,20 H62 V80 H58 Z"
        Stroke="Black" StrokeThickness="2" Fill="Transparent"/>
  <!-- Fill -->
  <Rectangle Fill="{Binding BatteryFillBrush}"
             Width="56"
             Height="{Binding BatteryLevel, Converter={StaticResource PercentToHeightConverter}}"
             VerticalAlignment="Bottom"
             Margin="2">
    <Rectangle.Clip>
      <RectangleGeometry Rect="0,0,56,120"/>
    </Rectangle.Clip>
  </Rectangle>
</Grid>
```

**Converters**:
- `PercentToHeightConverter`: returns `BatteryLevel/100.0 * 120`.
- `BatteryFillBrush`: converter or `DataTrigger` to switch to green/yellow/red based on `BatteryLevel`.

## Bluetooth Pairing & Disconnection Detection

### 1. Classic Serial Port Profile (SPP)

- Pairing:
  - User can pair in Windows Settings > Bluetooth, or programmatically via `BluetoothAuthenticateDevice` / `BluetoothAuthenticateDeviceEx`.
  - Once paired, an RFCOMM COM port appears (e.g., `COM5`).
- Connecting & detecting disconnect:
  - Open serial port (`CreateFile("\\.\\COM5", ...)`).
  - Monitor for read/write errors or use `WaitCommEvent()` on EV_CTS/EV_RXCHAR and check for `ERROR_DEVICE_NOT_CONNECTED`.
  - Alternatively, use Winsock with `AF_BTH` sockets and register for `FD_CLOSE` (via `WSAEventSelect`) to detect disconnection.

### 2. Bluetooth Low Energy (BLE)

- Pairing & connecting:
  - Use WinRT/C++ API: `Windows::Devices::Bluetooth::BluetoothLEDevice::FromBluetoothAddressAsync(address)`.
  - For classic Win32: `BluetoothGATTRegisterEvent` on the device handle.

### 3. Device-Interface Notifications

- Register for device arrival/removal:
  ```cpp
  DEV_BROADCAST_DEVICEINTERFACE filter = {};
  filter.dbcc_size = sizeof(filter);
  filter.dbcc_devicetype = DBT_DEVTYP_DEVICEINTERFACE;
  filter.dbcc_classguid = GUID_DEVINTERFACE_BLUETOOTH_DEVICE;

  HDEVNOTIFY hNotify = RegisterDeviceNotification(
      hwnd, &filter, DEVICE_NOTIFY_WINDOW_HANDLE);
  ```
- Handle `WM_DEVICECHANGE` in your window proc:
  - `DBT_DEVICEREMOVECOMPLETE` for loss of a paired device.
  - Check `dbcc_name` to match your ESP’s address.

## Shared Bluetooth Protocol Header (C++)

To keep service and ESP32 in sync, define all UUIDs, packet structs, and constants in a single header (e.g. `include/BatteryProtocol.h`):

```cpp
#pragma once

#include <cstdint>
#include <array>

// BLE Service & Characteristic UUIDs
static constexpr GUID BATTERY_SERVICE_UUID = /* 128-bit UUID */;
static constexpr GUID STATUS_CHAR_UUID   = /* 128-bit UUID */;
static constexpr GUID COMMAND_CHAR_UUID  = /* 128-bit UUID */;

// Packet formats
struct BatteryStatusPacket {
    uint8_t  acLineStatus;
    uint8_t  batteryFlag;
    uint8_t  batteryLifePercent;
    uint32_t batteryLifeTime;
    uint32_t batteryFullLifeTime;
};

struct ControlCommand {
    uint8_t commandId;
    uint8_t param1;
    uint8_t param2;
};

**ControlCommand Usage**
- `commandId`: identifies the command type, e.g. `0x01=SetRange`, `0x02=ChargeFully`, `0x03=HighFreqMode`.
- `param1`, `param2`: command-specific parameters (e.g. min/max % for SetRange, boolean flag for ChargeFully, on/off for HighFreqMode).

**High Frequency Mode Indication**
- `commandId = 0x03` signifies HighFreqMode.
- ESP32 sends this via a GATT **Indication** on `COMMAND_CHAR_UUID` (CCCD value `0x0002`).
- Windows service must enable indications (not just notifications) to receive these updates.

static_assert(sizeof(BatteryStatusPacket) == 1 + 1 + 1 + 4 + 4,
              "BatteryStatusPacket size mismatch");
```

- Place this header in a shared `include/` folder at repo root.  
- In your Windows service project, add `#include "BatteryProtocol.h"` and link against definitions.  
- In your ESP32 code, copy or symlink `include/BatteryProtocol.h` and use the same structs and UUIDs.  

This ensures both sides use identical protocol definitions and avoids drift.  

## Symlinking Shared Headers on Windows

- Git tracks symlinks if `core.symlinks` is enabled and the filesystem supports it.
- **Enable Developer Mode** on Win10+ or run as Administrator to allow symlinks without elevation.

- **Command Prompt** (Directory link):
  ```bat
  cd path\to\ESP32\project
  mklink /D include ..\..\include
  ```

- **PowerShell**:
  ```powershell
  New-Item -ItemType SymbolicLink -Path include -Target ..\..\include
  ```

- Use `#include "BatteryProtocol.h"` in ESP32 code as the header now appears under `include/`.
- Ensure `.git/config` sets `core.symlinks=true`, then `git add include` and commit; clones will recreate the link.

## Requesting Administrator Privileges

### Application Manifest
- In your app’s manifest (app.manifest), include:
  ```xml
  <trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
    <security>
      <requestedPrivileges>
        <requestedExecutionLevel level="requireAdministrator" uiAccess="false" />
      </requestedPrivileges>
    </security>
  </trustInfo>
  ```

## P/Invoke Wrapper for RPC Error Strings

To simplify error handling when using native RPC stubs, wrap your P/Invoke call and string cleanup:

```csharp
using System;
using System.Runtime.InteropServices;

static class NativeRpc
{
    // Generated P/Invoke stub
    [DllImport("BatteryControlClient.dll", CharSet=CharSet.Unicode)]
    private static extern int SetDesiredBatteryLevel(
      int min, int max, out IntPtr errorPtr);

    // RPC helper to free unmanaged strings
    [DllImport("Rpcrt4.dll", CallingConvention = CallingConvention.Winapi)]
    private static extern int RpcStringFree(ref IntPtr stringPtr);

    public static bool SetDesiredBatteryLevelSafe(int min, int max, out string error)
    {
        error = null;
        IntPtr errPtr;
        int hr = SetDesiredBatteryLevel(min, max, out errPtr);

        if (errPtr != IntPtr.Zero)
        {
            error = Marshal.PtrToStringUni(errPtr);
            RpcStringFree(ref errPtr);
        }

        return hr >= 0;
    }
}
```

## Estimating SOC from Charger-Side Measurements

- If you only measure adapter voltage (V_in) and input current (I_in), you can approximate battery SOC with coulomb-counting and a simple converter model:
  1. Approximate battery-side current:
     ```
     I_batt ≈ I_in × (V_in / V_batt_est) × η
     ```
     - **V_batt_est**: estimated battery voltage (use a lookup table or average ~3.7–4.2 V depending on SOC)
     - **η**: DC–DC converter efficiency (~0.9–0.95)
  2. Integrate to get charge (Q):
     ```
     Q_accum += I_batt × Δt;
     SOC = Q_accum / Q_nominal × 100%;
     ```
  3. For energy mapping:
     ```
     E_accum += V_batt_est × I_batt × Δt;
     ```
  4. Update **V_batt_est** as SOC changes (using battery voltage vs. SOC curve).

- **Simplified fallback**: assume η≈1 and V_in≈V_batt_est constant, then use I_in directly for coulomb-count (introduces systematic error but linear mapping).