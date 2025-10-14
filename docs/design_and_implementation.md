[Click here](../README.md) to view the README.

## Design and implementation

The over-the-air (OTA) update feature enables devices (mobile phones, vehicles, and other IoT devices) to download and update their software/firmware using a wireless network connection. OTA updates let you deploy firmware updates to one or more devices in your fleet. 

Jobs are used to accomplish OTA updates in a repeatable, reliable, controlled, and auditable way. A job document (Json document) is a description of the remote operations to be performed by the devices.

The OTA library utilizes MQTT and TLS to securely connect to an MQTT broker to download and update the user application. 

Wi-Fi update flows:

1. **Direct flow:** The device downloads a known file from the server/broker
2. **Job flow:** the device downloads a job document that provides information about the update

> **Note:** Use *Makefile*'s `define OTA_MQTT_SUPPORT` to use the MQTT protocol support from the OTA library

The application image can be stored in two slots on the device:

1. **Primary slot (BOOT):** stores the currently executing application
2. **Secondary slot (UPDATE):** Stores the new application being downloaded

Upon reset, the Edge Protect Bootloader (running on device) performs OTA update by overwriting the updating the image from the secondary slot (Slot 1) to primary slot (Slot 0).

**Figure 1. OTA MQTT update flow**
![](images/ota_mqtt_update_flow.png) 



**Table 1. Application projects**

Project | Description
--------|------------------------
*proj_cm33_s* | Project for CM33 secure processing environment (SPE)
*proj_cm33_ns* | Project for CM33 non-secure processing environment (NSPE)
*proj_cm55* | CM55 project

<br>

In this code example, at device reset, the secure boot process starts from the ROM boot with the secure enclave (SE) as the root of trust (RoT). From the secure enclave, the boot flow is passed on to the system CPU subsystem, where the Edge Protect Bootloader is the first application to be started. The Edge Protect Bootloader then verifies and launces the secure CM33 application. After all necessary secure configurations, the flow is passed on to the non-secure CM33 application. Resource initialization for this example is performed by this CM33 non-secure project. It initializes the OTA and MQTT configurations and starts the OTA firmware update task. It then enables the CM55 core using the `Cy_SysEnableCM55()` function. 


<br>