## General protocol functions
   # first time pairing:
    - esp32 device is first advertize itself as a BLE device with the generic name `ChargeController`
    - after initial pairing, it will be advertised as `ChargeController - <device mac address>` to allow other charge controllers to be paired with other computers. the windows service will record the mac address.
   # normal operation:
    - the windows service will connect to the esp32 device using the mac address.
    - the windows service will send the current SYSTEM_POWER_STATUS, and the desired battery level range to the esp32 device periodically, or charge fully command.
    - the windows service will monitor for disconnection and attempt to reconnect. connection/disconnection will also affect the performance profile of the windows machine.

## WinRT (.NET/C++ UWP or Desktop Bridge):
    # Pairing
    1. Find device selector:
        ```cpp
        auto selector = Windows::Devices::Bluetooth::BluetoothLEDevice::GetDeviceSelectorFromPairingState(false);
        auto devices = co_await Windows::Devices::Enumeration::DeviceInformation::FindAllAsync(selector);
        ```
    2. Pair:
        ```cpp
        auto deviceInfo = devices.GetAt(0);
        auto result = co_await deviceInfo.Pairing().PairAsync();
        if (result.Status() == Windows::Devices::Enumeration::DevicePairingResultStatus::Paired) {
            // success
        }
        ```
    - Offers non-blocking UI-less pairing and supports BLE-specific pairing scenarios.

    - WM_DEVICECHANGE: monitor for `DBT_DEVICEARRIVAL` to detect when pairing is completed at OS level.

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

