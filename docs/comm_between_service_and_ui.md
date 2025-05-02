** This is the description of the communication between the WinService and the WinUI. **
* Method: Native RPC over Named Pipes (as defined by the IDL below)
* The service creates an RPC service allowing the UI to manage devices and control charging.
* The UI is stateless; all device information and settings are stored and managed by the service.

**RPC Methods (Defined in IDL):**

*   `SetDesiredBatteryLevel(min, max)`: Sets the desired battery charge range (%). Returns success/error.
*   `GetCurrentState()`: Gets the current overall state: connected device info (if any), system power status, current settings. Returns `ServiceState` struct.
*   `ChargeFully(onlyThisTime)`: Charges the computer fully once or until disabled. Returns success/error.
*   `StartFirstTimePairing()`: Instructs the service to scan for nearby devices in pairing mode (button pressed) and attempt to bond with the first one found. **This is an asynchronous operation.** Returns success/error immediately based on whether the process could be *started*.
*   `GetCurrentState()`: Gets the current overall state. **Includes detailed status of any ongoing pairing process** (`currentPairingStatus`, `pairingStatusMessage`, `pairingDeviceMac`). The UI should poll this method periodically after calling `StartFirstTimePairing` to monitor progress and see the final result (Success/Failure).

### UI Pairing Status Flow

1.  Call `StartFirstTimePairing` to begin pairing.
2.  Enter a "Pairing in progress..." state in the UI.
3.  Poll `GetCurrentState` every second or two.
4.  Update the UI display based on `currentPairingStatus` and `pairingStatusMessage`. Show progress, errors, or success as appropriate.
5.  Exit the pairing state when `currentPairingStatus` shows Success or Failure (and optionally display the paired device info).

## Single IDL for Managed Client + Unmanaged Server

- **Create IDL** (`/idl/BatteryControl.idl`):
  ```idl
  import "rpc.idl";
  
  // Forward declarations
  struct KnownDevice;
  struct ServiceState;

  [
    uuid(12345678-1234-1234-1234-1234567890ab), // Keep consistent or generate new
    version(1.1), // Increment version due to changes
    pointer_default(unique)
  ]
  interface IBatteryControlService
  {
    // Configuration
    HRESULT SetDesiredBatteryLevel(
      [in] int min,
      [in] int max,
      [out, string] wchar_t** error
    );

    // State & Info
    HRESULT GetCurrentState(
      [out] ServiceState* state,
      [out, string] wchar_t** error
    );

    // Actions
    HRESULT ChargeFully(
      [in] BOOL onlyThisTime,
      [out, string] wchar_t** error
    );

    // Pairing & Device Management
    HRESULT StartFirstTimePairing(
      [out, string] wchar_t** error
    );

    HRESULT GetKnownDevices(
      [out] KnownDeviceList* devices,
      [out, string] wchar_t** error
    );

    HRESULT SetDeviceAlias(
      [in] unsigned __int64 macAddress,
      [in, string] const wchar_t* alias,
      [out, string] wchar_t** error
    );

    HRESULT ForgetDevice(
      [in] unsigned __int64 macAddress,
      [out, string] wchar_t** error
    );
  };

  // --- Data Structures --- 

  typedef enum _PairingStatusCode {
      PAIRING_IDLE = 0,
      PAIRING_SCANNING = 1,
      PAIRING_CONNECTING = 2, // Found device, trying to connect/bond
      PAIRING_SUCCESS = 3,    // Bonding successful
      PAIRING_FAILED_TIMEOUT = 4, // Scan timed out, no device found
      PAIRING_FAILED_BONDING = 5, // Connection/Bonding failed
      PAIRING_FAILED_ERROR = 6   // Other internal error
  } PairingStatusCode;

  [uuid( /* Generate new UUID */ ), version(1.0)]
  struct KnownDevice {
      unsigned __int64 macAddress;
      [string] wchar_t* advertisedName; // name generated after first pairing
      [string] wchar_t* alias; // user defined alias
      BOOL isConnected; // Currently connected?
  };

  [uuid( /* Generate new UUID */ ), version(1.0)]
  struct KnownDeviceList {
      [range(0, 50)] unsigned long count; // Max 50 devices in scan result
      [size_is(count)] KnownDevice devices[];
  };

  [uuid(87654321-4321-4321-4321-ba0987654321), version(1.2)] // Incremented version for pairing status
  struct ServiceState {
      // Current Settings
      int desiredMinLevel;
      int desiredMaxLevel;
      BOOL isChargeFullyActive;
      
      // Current Device Status (if one is connected)
      BOOL isDeviceConnected; // Is ANY paired device currently connected?
      unsigned __int64 connectedDeviceMac; // 0 if none connected
      [string] wchar_t* connectedDeviceAlias; // Alias of connected device
      int currentBatteryPercent;
      BOOL isComputerCharging; // Is the computer AC adapter providing power?
      BOOL isComputerPluggedIn; // Is the AC adapter physically plugged in?

      // Pairing Status
      PairingStatusCode currentPairingStatus; // Enum status code
      [string] wchar_t* pairingStatusMessage; // Human-readable status
      unsigned __int64 pairingDeviceMac; // Device being paired or just paired (0 if not applicable)
  };

  ```

- **MIDL Compilation**: (Command remains similar, ensure output paths are correct)
  ```powershell
  midl /env win64 /Oicf /out idl-gen \
    /header idl-gen/BatteryControl.h \
    /iid idl-gen/BatteryControl_i.c \
    /proxy idl-gen/BatteryControl_p.c \
    idl/BatteryControl.idl
  ```
  - Generates server skeleton + client stubs under `idl-gen/`.

- **Unmanaged Server**:
  - Compile `BatteryControl_s.c` into your service, implement the generated `IBatteryControlService_*` functions.
  - Use `RpcServerUseProtseqEp` with `ncacn_np` (named pipes) and `RpcServerRegisterIf` to host.

- **Managed Client (C#)**:
  - Compile the client stub into a native DLL (`BatteryControlClient.dll`).
  - Use P/Invoke to call RPC methods:
    ```csharp
    [DllImport("BatteryControlClient.dll", CharSet=CharSet.Unicode)]
    static extern int SetDesiredBatteryLevel(
      int min, int max, out IntPtr error);
    // ...

    // Create binding handle
    IntPtr binding;
    RpcBindingFromStringBindingW(
      L"ncacn_np:localhost[\\Pipe\\BatteryControlPipe]", out binding);
    ```
  - Free returned strings via `RpcStringFree`.

    // Example call
    IntPtr errorPtr;
    int hr = GetPairedDevices(binding, out pairedDeviceListPtr, out errorPtr);
    if (hr == 0) {
        // Process pairedDeviceListPtr (requires marshalling structs)
    } else {
        string errorMsg = Marshal.PtrToStringUni(errorPtr);
        RpcStringFree(ref errorPtr);
        // Handle error
    }
    // Free returned list struct memory appropriately