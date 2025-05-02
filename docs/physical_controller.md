The controller is a physical device that contains the ESP32.
The inputs connected to the microcontroller are:
- current sensor to measure the current passed to the computer.
- constant voltage divider to measure the voltage passed to the computer.
- buttons:
    - force charge button: to charge the computer to 100%.
    - new pair button: change the device bt name.
    - check button: ask the computer for a fast report rate, until the information is recieved, then that bt indicator returns to false.
    - reset button, directly connect to the reset pin of the ESP32.

The outputs connected to the microcontroller are:
- a relay that connect the power to the computer.
- chained few neopixel LEDs to show the status of the controller:
    - first LED: physical connection status, red connected to power (it must so be to show anything,) blue if it is also physically connected to the computer. blinking blue for a mismatch between the controller sensed state and the computer reported state.
    - second LED: BLE connection status, green if it is connected to the computer, red if it is not, blinking red if it is ready for a first time pairing situation.
    - third LED: charging status, green if it is actually charging now, black if not, magenta if it is in full charge mode.

The device is communicating via BLE with the service on the computer. The communication is described in `docs/comm_between_service_and_esp.md`.