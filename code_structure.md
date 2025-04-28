There are three subprojects:

1. **WinService**: A Windows service that runs in the background and communicate with the esp32 over bluetooth and report to it the charging status, desired battery level,and the current battery level. Using the status of the bluetooth connection with the esp, it changes the performance profile of the windows machine to that of either the performance when not plugged, or the performance when plugged. There is also a way to comminicate with it from another program and set the desired battery level. The settings are stored in the registry. The program is written in C++.

The following is the current structure of this program:
* it is currently empty.
2. **WinUI**: A Windows application that allows the user to set the desired battery level range, it communicates with the WinService to set the desired battery level. The program is written in C#.
The following is the current structure of this program:
* it is currently empty.
3. **ESP32**: A microcontroller that is used to switch the charging on and off according to the information it gets from the WinService. It also measures the current and voltage of the charging to calculate how much it charged the computer battery, so when the computer goes to sleep, it can estimate when to stop charging the computer battery according to the last battery level reported and the desired battery level. The program is written in C++ with Arduino platform and PlatformIO.

The following is the current structure of this program:
* it is currently empty.
