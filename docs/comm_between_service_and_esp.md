## General protocol functions
   # first time pairing:
    - The ESP32 device does *not* advertise by default.
    - When the physical **pairing button** is pressed, the device enters a **pairing mode** for a limited time (e.g., 60 seconds).
    - In pairing mode, it advertises with:
        - A specific, recognizable name (e.g., `"Battery Limiter Pair"`).
        - Its primary service UUID.
        - Flags indicating discoverability/connectability.
    - The Windows Service, when instructed by the UI, scans *only* for this specific pairing name and service UUID.
    - Upon finding the device, the Service connects and immediately initiates **BLE Bonding** (e.g., "Just Works" if no device input/output is available).
    - Both the Service (OS BLE stack) and the ESP32 store the bonding keys and the peer's MAC address.
    - After successful bonding, the ESP32 **exits pairing mode**.

   # Post-Bonding Advertising & Normal Operation:
    - After bonding, the ESP32 should advertise persistently (or periodically) with:
        - Its **Service UUID** (essential for reconnection scans).
        - Recommended: A stable, unique advertised name derived from its MAC (e.g., `"BatteryLimiter-XXXXXX"` where XXXXXX is the last 3 bytes of the MAC). This aids user identification in generic scans; this also allow a fallback to the MAC address for connection.
        - Necessary flags (connectable, etc.).
        - Consider using Resolvable Private Addresses (RPAs) for enhanced privacy if needed.
    - The Windows Service maintains a list of **bonded device MAC addresses**.
    - To connect to a known device, the Service first attempts a **direct connection** using the stored MAC address (fastest method).
    - If direct connection fails, the Service scans for the device's specific Service UUID or unique advertised name (`BatteryLimiter-XXXXXX`) as a fallback, then connects via MAC once found.
    - Once connected, the Service proceeds with normal operation: sending `SYSTEM_POWER_STATUS`, desired range, etc.
   # normal operation:
    - the windows service will send the current SYSTEM_POWER_STATUS, and the desired battery level range to the esp32 device periodically, or charge fully command.
    - the windows service will monitor for disconnection and attempt to reconnect. connection/disconnection will also affect the performance profile of the windows machine.
    - there is an indication `high_frequency_mode` that the esp sets to indicate to the windows service that it should send the information more often. This will allow the esp to be able to check whether it is physiscally connected to the computer, by checking the plugged status, which the esp can turn on temporarily just for the test.

## WinRT (.NET/C++ UWP or Desktop Bridge):
    # Pairing & Bonding
    1. Scan for Pairing Devices (when user initiates):
        ```cpp
        // Construct AQS filter for the pairing name and service UUID
        string aqsFilter = "System.Devices.Aep.DeviceAddress:\"" + pairingDeviceName + "\" AND System.Devices.Aep.IsPaired:=System.StructuredQueryType.Boolean#False AND System.Devices.AepService.ServiceId:=\"{" + serviceUuidString + "}\"";
        auto devices = co_await Windows::Devices::Enumeration::DeviceInformation::FindAllAsync(aqsFilter, { L"System.Devices.Aep.DeviceAddress" });
        // Display found devices (by BluetoothAddress) to user
        ```
    2. Initiate Bonding (User selects device from scan results):
        ```cpp
        // Get DeviceInformation for the selected BluetoothAddress
        auto deviceInfo = co_await Windows::Devices::Enumeration::DeviceInformation::CreateFromIdAsync(selectedDeviceAepId);
        // Set protection level for bonding
        auto pairingKind = Windows::Devices::Enumeration::DevicePairingKinds::ConfirmOnly; // Or JustWorks, etc.
        auto customPairing = deviceInfo.Pairing().Custom();
        customPairing.PairingRequested({this, &MyClass::PairingRequestedHandler}); // Handle passkey display/entry if needed
        auto result = co_await customPairing.PairAsync(pairingKind, Windows::Devices::Enumeration::DevicePairingProtectionLevel::EncryptionAndAuthentication);

        if (result.Status() == Windows::Devices::Enumeration::DevicePairingResultStatus::Paired ||
            result.Status() == Windows::Devices::Enumeration::DevicePairingResultStatus::AlreadyPaired)
        {
            // Success: Store the device's BluetoothAddress (ulonglong) persistently
            uint64_t deviceAddress = //... Get from deviceInfo or result
            // ... store deviceAddress ...
        }
        ```
    3. Connect to Bonded Device:
        ```cpp
        uint64_t bondedDeviceAddress = // ... load stored address ...
        auto device = co_await Windows::Devices::Bluetooth::BluetoothLEDevice::FromBluetoothAddressAsync(bondedDeviceAddress);
        if (device != nullptr) {
            // Connection successful (or already connected)
            // Proceed with GATT operations
        }
        ```
    - Use `DeviceWatcher` for continuous scanning if needed, applying the appropriate AQS filter.
    - Handle `DeviceInformation.Pairing.IsPaired` status.

    # Disconnection detection
    - Subscribe to `BluetoothLEDevice.ConnectionStatusChanged` event in WinRT.
    - In native code, handle `ERROR_DEVICE_NOT_CONNECTED` on GATT calls or GATT event callback indicating status change.

    3. Communication (GATT):
        1. Discover services:
            - WinRT: `auto device = co_await BluetoothLEDevice::FromBluetoothAddressAsync(addr);`
              `auto services = co_await device.GetGattServicesAsync();`
            - Win32: `BluetoothGATTGetServices(hDevice, 0, nullptr, &count, 0);` then allocate array and call again.
        2. Discover characteristics:
            - WinRT: `auto chars = co_await service.GetCharacteristicsAsync();`
            - Win32: `BluetoothGATTGetCharacteristics(hDevice, &service, 0, nullptr, &count, 0);` then read.
        3. Read / Write:
            - WinRT: `co_await characteristic.ReadValueAsync();`
              `co_await characteristic.WriteValueAsync(buffer);
            - Win32: `BluetoothGATTReadCharacteristic(hDevice, &service, &characteristic, ...);`
              `BluetoothGATTWriteCharacteristic(hDevice, &service, &characteristic, ...);`
        4. Subscribe to notifications:
            - WinRT: register `ValueChanged` event and call `WriteClientCharacteristicConfigurationDescriptorAsync(GattClientCharacteristicConfigurationDescriptorValue::Notify)`;
            - Win32: Use `BluetoothGATTSetCharacteristicValue` with `BLUETOOTH_GATT_FLAG_NONE` on the CCCD handle (0x2902) to enable notifications, then register a `BluetoothGATTEventCallback`.

    ## Removing Bonds (Forgetting Device)
    - If the user wants to "forget" a device:
        ```cpp
        // Get DeviceInformation for the device to unpair
        auto deviceInfo = co_await Windows::Devices::Enumeration::DeviceInformation::CreateFromIdAsync(deviceAepId_to_forget);
        auto unpairResult = co_await deviceInfo.Pairing().UnpairAsync();
        if (unpairResult.Status() == Windows::Devices::Enumeration::DeviceUnpairingResultStatus::Unpaired ||
            unpairResult.Status() == Windows::Devices::Enumeration::DeviceUnpairingResultStatus::AlreadyUnpaired) {
            // Success: Remove the device's BluetoothAddress from persistent storage
        }
        ```

    ## Notifications vs. Indications

    - **Notifications**:
      - Unacknowledged; peripheral sends without waiting for confirmation.
      - Lower protocol overhead, higher throughput.
      - Possible data loss if central misses a packet.
    - **Indications**:
      - Acknowledged; peripheral waits for a confirmation write from central.
      - Ensures reliable delivery at the cost of throughput and added latency.

    Use Notifications for frequent, non-critical updates, and Indications for important or ordered data.

    # ESP32 Initiated Communication (Notifications/Indications)
    - While the central (Windows service) usually initiates reads/writes, the peripheral (ESP32) can proactively send data to the central if the central has subscribed to a characteristic.
    - **Method**:
        1. **ESP32**: Define a characteristic with `notify` or `indicate` properties.
        2. **Windows Service**: Discover the characteristic and its Client Characteristic Configuration Descriptor (CCCD).
        3. **Windows Service**: Write to the CCCD to enable notifications (`0x0001`) or indications (`0x0002`).
        4. **Windows Service**: Register a callback/event handler (`ValueChanged` in WinRT, `BluetoothGATTEventCallback` in Win32) to receive the data when the ESP32 sends it.
        5. **ESP32**: When data needs to be sent, update the characteristic value and trigger the notification/indication.
    - This allows the ESP32 to push status updates, alerts, or other data to the Windows service without waiting for a poll.