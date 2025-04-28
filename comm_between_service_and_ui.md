** This is the description of the communication between the WinService and the WinUI. **
* Method: WCF with NetNamedPipeBinding
* The service is create an WCF RPC service that allows the UI to set the desired battery level, and get the current setting.
* There are the following methods:
    * `SetDesiredBatteryLevel` - sets the desired battery level, take 2 parameter of type `int` `min`, `max` as percentage, and returns `[bool success, string error]`.
    * `GetCurrentState` - gets the current state, no parameters, returns `min`, `max` (`int`)as the range of the desired battery level, `current_battery_level` (`int`) as percentage, also three booleans `charge_controller_connected`, `computer_charging` and `charger_pluged`.
    * `DiscoverChargeController` - discovers the charge controller, no parameters, returns `[bool success, string error, string mac_address]`.
    * `ChargeFully` - charges the computer fully, parameter: `onlyThisTime` (`bool`), returns `[bool success, string error]`.