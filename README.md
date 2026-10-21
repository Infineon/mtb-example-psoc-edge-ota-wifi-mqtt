# PSOC&trade; Edge MCU: OTA Wi-Fi MQTT

This code example demonstrates the firmware update of user application over-the-air (OTA) via the Wi-Fi interface using MQTT protocol, using Arm&reg; Cortex&reg; M33 CPU on PSOC&trade; Edge MCU coupled with AIROC&trade; CYW55513/CYW55512 in ModusToolbox&trade; environment.

The device establishes a connection with the designated MQTT broker – this example uses AWS. It periodically checks the job document to see if a new update is available. When a new update is available, it is downloaded and written to the secondary slot (flash). On the next reboot, the Edge Protect Bootloader handles image verification and upgrades. This example uses overwrite-based upgrade. In an overwrite-based upgrade, the new image from the secondary slot is simply copied to the primary slot after a successful validation.

This code example has a three project structure: CM33 secure, CM33 non-secure, and CM55 projects. All three projects are programmed to the external QSPI flash and executed in Execute in Place (XIP) mode. It also requires you to add the Edge Protect Bootloader project (*proj_bootloader*). The Edge Protect bootloader is responsible for validation and updation of the images downloaded to the device.
The extended boot launches the Edge Protect Bootloader from the RRAM and the Edge Protect Bootloader launches the CM33 Secure project from a known location in the external flash, which then launches the CM33 non-secure application. CM33 non-secure application is responsible for starting CM55 CPU and handling the OTA task.


This example has been tested with the following (out-of-box) configurations:

* Device lifecycle - Development
* Secure Boot - Disabled

[View this README on GitHub.](https://github.com/Infineon/mtb-example-psoc-edge-ota-wifi-mqtt)

[Provide feedback on this code example.](https://yourvoice.infineon.com/jfe/form/SV_1NTns53sK2yiljn?Q_EED=eyJVbmlxdWUgRG9jIElkIjoiQ0UyMzk5NDQiLCJTcGVjIE51bWJlciI6IjAwMi0zOTk0NCIsIkRvYyBUaXRsZSI6IlBTT0MmdHJhZGU7IEVkZ2UgTUNVOiBPVEEgV2ktRmkgTVFUVCIsInJpZCI6InZhc3VtYS5uYXJheWFuYUBpbmZpbmVvbi5jb20iLCJEb2MgdmVyc2lvbiI6IjIuMi4wIiwiRG9jIExhbmd1YWdlIjoiRW5nbGlzaCIsIkRvYyBEaXZpc2lvbiI6Ik1DRCIsIkRvYyBCVSI6IklDVyIsIkRvYyBGYW1pbHkiOiJQU09DIn0=)

See the [Design and implementation](docs/design_and_implementation.md) for the functional description of this code example.


## Requirements

- [ModusToolbox&trade;](https://www.infineon.com/modustoolbox) v3.8 or later (tested with v3.9)
- Board support package (BSP) minimum required version: 1.0.0
- Programming language: C
- Associated parts: All PSOC&trade; Edge E84 MCU parts
- Python (tested with v3.13.7)


## Supported toolchains (make variable 'TOOLCHAIN')

- GNU Arm&reg; Embedded Compiler v14.2.1 (`GCC_ARM`) – Default value of `TOOLCHAIN`
- Arm&reg; Compiler v6.22 (`ARM`)
- IAR C/C++ Compiler v9.70.4 (`IAR`)
- Arm&reg; Toolchain for Embedded v22.1 (`LLVM_ARM`)


## Supported kits (make variable 'TARGET')

- [PSOC&trade; Edge E84 Evaluation Kit](https://www.infineon.com/KIT_PSE84_EVAL) (`KIT_PSE84_EVAL_EPC2`) – Default value of `TARGET`
- [PSOC&trade; Edge E84 Evaluation Kit](https://www.infineon.com/KIT_PSE84_EVAL) (`KIT_PSE84_EVAL_EPC4`)
- [PSOC&trade; Edge E846 Evaluation Kit](https://www.infineon.com/KIT_PSE846_EVAL) (`KIT_PSE846_EVAL_EPC2`)


## Hardware setup

This example uses the board's default configuration. See the kit user guide to ensure that the board is configured correctly.

Ensure the following jumper and pin configuration on the board.
- The BOOT SW pin (P17.6) is in the LOW/OFF position
- J20 and J21 must be in the tristate/not connected (NC) position (for the PSOC&trade; Edge E84 Evaluation Kit)
- All SW8 DIP switches in OFF position (for the PSOC&trade; Edge E846 Evaluation Kit)


## Software setup

See the [ModusToolbox&trade; tools package installation guide](https://www.infineon.com/ModusToolboxInstallguide) for information about installing and configuring the tools package.

Install a terminal emulator if you do not have one. Instructions in this document use [Tera Term](https://teratermproject.github.io/index-en.html).

This example requires no additional software or tools.


## Operation

### Firmware update using AWS MQTT broker

<details><summary><b>Add the Edge Protect Bootloader to application</b></summary>

1. Add *proj_bootloader* to this code example as a first step. Follow step 2 to step 9 in the **Operation** section of the [Edge Protect Bootloader](https://github.com/Infineon/mtb-example-edge-protect-bootloader) code example's *README.md* file

  > **Note:**
  > 1. [Edge Protect Bootloader](https://github.com/Infineon/mtb-example-edge-protect-bootloader) *README.md* describes how to add the bootloader to **Basic Secure App**. Add it to **this** code example instead
  > 2. (For the `LLVM_ARM` compiler) **Edge Protect Bootloader** does not support `LLVM_ARM`. In the *proj_bootloader* directory, edit the *Makefile* to force other supported compilers for EPB (for example: `TOOLCHAIN=GCC_ARM`)

</details>

<details><summary><b>Configure the memory map</b></summary>

2. Configure the memory map: this code example bundles a custom *design.modus* file compatible with the Edge Protect Bootloader. Hence, you need not make any further customizations here to use EPB and this project together

</details>

<details><summary><b>Set up the AWS MQTT broker</b></summary>

3. Set up the MQTT device (also known as a **Thing**) in the AWS IoT core as described in the [Getting started with AWS IoT tutorial](https://docs.aws.amazon.com/iot/latest/developerguide/iot-gs.html)

   > **Note:** While setting up your device, ensure that the policy associated with this device permits all MQTT operations (**iot:Connect**, **iot:Publish**, **iot:Receive**, and **iot:Subscribe**) for the resource used by this device. For testing purposes, it is recommended to have the following policy document, which allows all **MQTT Policy Actions** on all **Amazon resource names (ARNs)**

    ```
   {
        "Version": "2024-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "iot:*",
                "Resource": "*"
            }
        ]
   }
   ```

 4. Download the following certificates and keys that are created and activated in the previous step:
       - A **certificate for the AWS IoT Thing:** *xxxxxxxxxx.aws-client-certificate.crt*
       - A **private key:** *xxxxxxxxxx.aws-private_key.pem*
       - Root CA "RSA 2048 bit key: Amazon Root CA 1" for AWS IoT from [CA certificates for server authentication](https://docs.aws.amazon.com/iot/latest/developerguide/server-authentication.html#server-authentication-certs) - *xxxx.AmazonRootCA1.crt*

 5. Copy the certificates and key, paste it in the *\<My-OTA-MQTT-example>/scripts* folder

 6. Rename the following file names in the *\<My-OTA-MQTT-example>/scripts* folder
      - *xxxx.AmazonRootCA1.crt* to *amazon_ca.crt*
      - *xxxxxxxxxx.aws-client-certificate.crt* to *amazon_client.crt*
      - *xxxxxxxxxx.aws-private_key.pem* to *amazon_private_key.pem*

 </details>

<details><summary><b>Build and program the bootloader and application </b></summary>

7. Open the *common.mk* file in the root of the application and set the `COMBINE_SIGN_JSON` variables to the configurator-generated *boot_with_bldr.json* file, as shown below:

    ```
    COMBINE_SIGN_JSON?=./bsps/TARGET_$(TARGET)/config/GeneratedSource/boot_with_bldr.json
    ```

8. Ensure IMG_TYPE is set to BOOT, as shown below:

    ```
     IMG_TYPE=BOOT
    ```


 9. Provide your python path in the *common.mk* file located at the top level directory of your PSOC&trade; Edge MCU : OTA WIFI MQTT application

    ```
    CY_PYTHON_PATH = <path-to-python>/python.exe
    ```

    > **Note:** Install Python if not already installed. If you see the **ModuleNotFoundError: No module named 'click'** error during application build, install the respective python module

    ```
    pip install click
    ```

10. Update all the required connection parameters in the configuration file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h* <br>


    - **Wi-Fi connection parameters**

        ```
        /* Name of the Wi-Fi network */
        #define WIFI_SSID                            ""

        /* Password for the Wi-Fi network */
        #define WIFI_PASSWORD                        ""

        /* Security type of the Wi-Fi access point. See 'cy_wcm_security_t' structure
        * in "cy_wcm.h" for more details.
        */
        #define WIFI_SECURITY
        ```

    - **MQTT broker information**

        ```
        #define MQTT_BROKER_AWS_URL                  ""

        #define MQTT_BROKER_AWS_USERNAME             ""

        #define MQTT_BROKER_AWS_PASSWORD             ""
        ```

**Certificates and key**

11. Navigate to *\<My-OTA-MQTT-example>/scripts* directory. Execute format_cert_key.py python script to generate the string format of the certificate and key files that can be added as macro. Pass the name of the certificate or key with the extension as an argument to the Python script

```
python format_cert_key.py <one-or-more-file-name-of-certificate-or-key-with-extension>
```

Example:

```
python format_cert_key.py amazon_ca.crt amazon_client.crt amazon_private_key.pem
```

12. Copy the string format into the *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h.* file

```
#define ROOT_CA_CERTIFICATE \
    "-----BEGIN CERTIFICATE-----\n" \
	"..........base64 data......\n" \
    "-----END CERTIFICATE-----\n";

#define CLIENT_CERTIFICATE \
    "-----BEGIN CERTIFICATE-----\n" \
	"..........base64 data......\n" \
    "-----END CERTIFICATE-----\n";


#define CLIENT_KEY \
    "-----BEGIN RSA PRIVATE KEY-----\n" \
	"..........base64 data......\n" \
    "-----END RSA PRIVATE KEY-----\n";

 ```

13. Clean, build, and program the application. See [Using the code example](docs/using_the_code_example.md) for instructions on creating a project, opening it in various supported IDEs, and performing tasks, such as building, programming, and debugging the application within the respective IDEs

14. After programming, the application starts automatically. Confirm "PSOC Edge MCU : OTA WIFI MQTT Application" is displayed on the UART terminal


    **Figure 1. Terminal output on program startup**

    ![](images/terminal_ota_mqtt_boot.png)

</details>

<details><summary><b>Build the application image for overwrite update</b></summary>

15. Ensure `IMG_TYPE` is set to `UPDATE` in the *common.mk* file of the this code example, found in the top level directory of the application

    ```
    IMG_TYPE?=UPDATE
    ```

  > **Note:**
  > 1. Now the application version will be changed to 1.1.0 in the same *common.mk* file based on `IMG_TYPE`
  > 2. When you select the `IMG_TYPE` to `UPDATE`, the `COMBINE_SIGN_JSON` variable is set automatically in the *common.mk* file, depending on the value of `UPDATE_TYPE`

16. Clean, build the application **DO NOT PROGRAM** the application. See [Using the code example](docs/using_the_code_example.md) for instructions on creating a project, opening it in various supported IDEs, and performing tasks, such as clean and build the application

</details>


<details><summary><b> Perform the OTA MQTT firmware update using AWS MQTT Broker</b></summary>

17.  Open the job document  *ota_update.json* in the  *\<My-OTA-MQTT-example>/scripts* directory and ensure that the version is set to to **1.1.0** or later. This version must be different than what has been already programmed on the target

        ```
        {
        "Message":"Update Available",
        "Manufacturer":"Infineon",
        "ManufacturerId":"IFX",
        "Product":"PSOC Edge",
        "SerialNumber":"ABC213450001",
        "Board":"APP_KIT_PSE84_EVAL_EPC2",
        "Version": "1.1.0",
        "Connection":"MQTT",
        "Broker": "",
        "Port": "8883",
        "UniqueTopicName":"ota-mqtt"
        }

        ```

18. Open a CLI terminal

    On Linux and macOS, you can use any terminal application. On Windows, open 'modus-shell' app from the **Start** menu

19. Navigate to *\<My-OTA-MQTT-example>/scripts/* directory

20. Run the publisher script

    ```
    python publisher.py tls -b amazon -k APP_KIT_PSE84_EVAL_EPC2 -f ./../build/ota-update.tar
    ```

    **Figure 2. Run the MQTT publisher script**

    ![](images/start_mqtt_publisher.png)

    > **Note:** If you see an error saying **"paho-mqtt" is missing in your python installation**, install it using `pip install paho-mqtt==1.6.1`

21. Since the current application version is 1.0.0 while the job document version is 1.1.0, the OTA update starts automatically

    **Figure 3. Terminal output for OTA update start**

    ![](images/ota_mqtt_transfer_started.png)

    **Figure 4. Publisher transfer start**

    ![](images/mqtt_publisher_transfer_started.png)

22. After the image transfer is complete, the Edge Protect Bootloader validates the image and copies the image to the primary slot and boots it

    **Figure 5. Terminal output for OTA download completed**

    ![](images/mqtt_transfer_complete.png)


    **Figure 6. Publisher transfer done**

    ![](images/mqtt_publisher_transfer_done.png)


    **Figure 7. Terminal output for image validation by the bootloader**

    ![](images/mqtt_bootloader_image_validation.png)

23. After successful image upgrade the following display message will be seen on the terminal window. Verify the application version is for the upgrade image

     **Figure 8. Upgrade image**

    ![](images/terminal_ota_mqtt_update.png)

</details>


### Firmware update using the Mosquitto or Eclipse broker

<details><summary><b>Using the Mosquitto broker</b></summary>

The steps to perform an OTA upgrade remains same as described in step 1 to step 2 in the [Operation](#operation) section above. The following instructions indicate only the steps that differ for the Mosquitto broker.

#### TLS mode

1. Follow the instructions in this [link](https://test.mosquitto.org/ssl/) to generate your own keys and certificates

2. Place the CA certificate, client key, and client certificate in the *\<My-OTA-MQTT-example>/scripts/ directory*

    > **Note:** Ensure the filenames are the same as used in the publisher script:
    >- CA Certificate: *mosquitto.org.crt*
    >- Client certificate: *mosquitto_client.crt*
    >- Client key: *mosquitto_client.key*

3. Open a CLI terminal

   On Linux and macOS, use any terminal application. On Windows, open 'modus-shell' app from the **Start** menu

4. Navigate to the *\<My-OTA-MQTT-example>/scripts/* directory. Run the publisher script:

    ```
    python publisher.py tls -b mosquitto -k APP_KIT_PSE84_EVAL_EPC2 -f ./../build/ota-update.tar
    ```

   > **Note:** Ensure the upgrade image (ota-update.tar) is already available the specified path. See [Operation](#Operation)

5. Update the following connection information in the configuration file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h*

    **Wi-Fi connection parameters**

    ```
    /* Name of the Wi-Fi network */
    #define WIFI_SSID                            ""

    /* Password for the Wi-Fi network */
    #define WIFI_PASSWORD                        ""

    /* Security type of the Wi-Fi access point. See 'cy_wcm_security_t' structure
    * in "cy_wcm.h" for more details.
    */
    #define WIFI_SECURITY
    ```

    **MQTT broker information**

    ```
    #define OTA_MQTT_BROKER                      MQTT_BROKER_MOSQUITTO_URL

    #define OTA_MQTT_BROKER_PORT                 MQTT_BROKER_MOSQUITTO_TLS_PORT

    #define ENABLE_TLS                           1

    #define OTA_UPDATE_FLOW                      CY_OTA_JOB_FLOW
    ```


**Certificates and key**

6. Navigate to the *\<My-OTA-MQTT-example>/scripts* directory. Execute the *format_cert_key.py* python script to generate the string format of the certificate and key files that can be added as macro. Pass the name of the certificate or key with the extension as an argument to the Python script

    ```
    python format_cert_key.py <one-or-more-file-name-of-certificate-or-key-with-extension>
    ```

    Example:

    ```
    python format_cert_key.py mosquitto.org.crt mosquitto_client.crt mosquitto_client.key
    ```

7. Copy the string format into the *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h.* file

    ```
    #define ROOT_CA_CERTIFICATE \
        "-----BEGIN CERTIFICATE-----\n" \
        "..........base64 data......\n" \
        "-----END CERTIFICATE-----\n";

    #define CLIENT_CERTIFICATE \
        "-----BEGIN CERTIFICATE-----\n" \
        "..........base64 data......\n" \
        "-----END CERTIFICATE-----\n";


    #define CLIENT_KEY \
        "-----BEGIN RSA PRIVATE KEY-----\n" \
        "..........base64 data......\n" \
        "-----END RSA PRIVATE KEY-----\n";

    ```

8. Update the MbedTLS configuration file to use TLS v1.2 . Open the file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/mbedtls_user_config.h.*

   Comment out the following defines

   ```
   //#define MBEDTLS_SSL_PROTO_TLS1_3
   ```

   ```
   //#define FORCE_TLS_VERSION MBEDTLS_SSL_VERSION_TLS1_3
   ```

#### Non TLS mode

9. Open a CLI terminal

   On Linux and macOS, use any terminal application. On Windows, open 'modus-shell' app from the **Start** menu

10. Navigate to the *\<My-OTA-MQTT-example>/scripts/* directory. Run the publisher script

    ```
    python publisher.py -b mosquitto -k APP_KIT_PSE84_EVAL_EPC2 -f ./../build/ota-update.tar
    ```

    > **Note:** Ensure the upgrade image (*ota-update.tar*) is already available the specified path. See [Operation](#Operation) for more information

11. Update the following connection information in the configuration file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h*


    **Wi-Fi connection parameters**

    ```
    /* Name of the Wi-Fi network */
    #define WIFI_SSID                            ""

    /* Password for the Wi-Fi network */
    #define WIFI_PASSWORD                        ""

    /* Security type of the Wi-Fi access point. See 'cy_wcm_security_t' structure
    * in "cy_wcm.h" for more details.
    */
    #define WIFI_SECURITY
    ```
    <br>

    **MQTT broker information**

    ```
    #define OTA_MQTT_BROKER                      MQTT_BROKER_MOSQUITTO_URL

    #define OTA_MQTT_BROKER_PORT                 MQTT_NON_TLS_PORT

    #define ENABLE_TLS                           0

    #define OTA_UPDATE_FLOW                      CY_OTA_JOB_FLOW
    ```

</details>

<details><summary><b>Using the Eclipse broker</b></summary>

The steps to perform an OTA upgrade remains same as described in the steps 1 to step 5 in the [Operation](#operation) section above. Below instructions indicate only those steps that are different for the Eclipse mode.


#### Non TLS mode

1. Open a CLI terminal

   On Linux and macOS, you can use any terminal application. On Windows, open 'modus-shell' app from the **Start** menu

2. Navigate to *\<My-OTA-MQTT-example>/scripts/* directory

3. Run the publisher script

    ```
    python publisher.py -b eclipse -k APP_KIT_PSE84_EVAL_EPC2 -f ./../build/ota-update.tar
    ```
   > **Note:** Ensure the upgrade image (ota-update.tar) is already available the specified path. See [Operation](#Operation)

4. Update the following connection information in the configuration file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/ota_app_config.h* <br>


    **Wi-Fi connection parameters**

    ```
    /* Name of the Wi-Fi network */
    #define WIFI_SSID                            ""

    /* Password for the Wi-Fi network */
    #define WIFI_PASSWORD                        ""

    /* Security type of the Wi-Fi access point. See 'cy_wcm_security_t' structure
    * in "cy_wcm.h" for more details.
    */
    #define WIFI_SECURITY
    ```

    **MQTT broker information**

    ```
    #define OTA_MQTT_BROKER                      MQTT_BROKER_ECLIPSE_URL

    #define OTA_MQTT_BROKER_PORT                 MQTT_NON_TLS_PORT

    #define ENABLE_TLS                           0

    #define OTA_UPDATE_FLOW                      CY_OTA_DIRECT_FLOW
    ```

    </details>

### Octal SPI flash memory support

The default configuration for this code example enables the Edge Protect Bootloader to launch the user applications from the external QSPI flash. Follow the instructions given below only if you need to configure the Edge Protect Bootloader to launch the user application from the external Octal SPI flash memory:

1. Replace the default configuration with the Octal SPI flash configuration by copying the `design.modus` and `design.cyqspi` files provided in the **OSPI_config** folder into the <My-OTA-MQTT-example>/bsps/<TARGET>/config folder. For more details about enabling Octal SPI support in Edge Protect Bootloader, see "Octal SPI flash memory support" Section in [AN237857 - Edge Protect Bootloader for PSOC™ Edge MCU](https://www.infineon.com/AN237857).

2. Clean, build, and program the application as per the steps provided in the **Operation** Section. The Edge Protect Bootloader boot the application from the external Octal SPI flash.


## Related resources

Resources  | Links
-----------|----------------------------------
Application notes  | [AN235935](https://www.infineon.com/AN235935) – Getting started with PSOC&trade; Edge E84 MCU on ModusToolbox&trade; software
Code examples  | [Using ModusToolbox&trade;](https://github.com/Infineon/Code-Examples-for-ModusToolbox-Software) on GitHub
Device documentation | [PSOC&trade; Edge E84 MCU datasheet](https://www.infineon.com/products/microcontroller/32-bit-psoc-arm-cortex/32-bit-psoc-edge-arm/psoc-edge-e84#Documents) <br> [PSOC&trade; Edge E84 MCU reference manuals](https://www.infineon.com/products/microcontroller/32-bit-psoc-arm-cortex/32-bit-psoc-edge-arm/psoc-edge-e84#Documents)
Development kits | Select your kits from the [Evaluation board finder](https://www.infineon.com/cms/en/design-support/finder-selection-tools/product-finder/evaluation-board)
Libraries  | [mtb-dsl-pse8xxgp](https://github.com/Infineon/mtb-dsl-pse8xxgp) – Device support library for PSE8XXGP <br> [mtb-dsl-pse8x6](https://github.com/Infineon/mtb-dsl-pse8x6) – Device support library for PSE8X6 <br> [retarget-io](https://github.com/Infineon/retarget-io) – Utility library to retarget STDIO messages to a UART port <br> [serial-memory](https://github.com/Infineon/serial-memory) - Serial Flash Middleware for External memory interface <br> [DFU](https://github.com/Infineon/dfu) – Device Firmware Update Middleware (DFU MW) <br> [emusb-device](https://github.com/Infineon/emusb-device) – USB Device stack for embedded applications
Tools  | [ModusToolbox&trade;](https://www.infineon.com/modustoolbox) – ModusToolbox&trade; software is a collection of easy-to-use libraries and tools enabling rapid development with Infineon MCUs for applications ranging from wireless and cloud-connected systems, edge AI/ML, embedded sense and control, to wired USB connectivity using PSOC&trade; Industrial/IoT MCUs, AIROC&trade; Wi-Fi and Bluetooth&reg; connectivity devices, XMC&trade; Industrial MCUs, and EZ-USB&trade;/EZ-PD&trade; wired connectivity controllers. ModusToolbox&trade; incorporates a comprehensive set of BSPs, HAL, libraries, configuration tools, and provides support for industry-standard IDEs to fast-track your embedded application development

<br>


## Other resources

Infineon provides a wealth of data at [www.infineon.com](https://www.infineon.com) to help you select the right device, and quickly and effectively integrate it into your design.


## Document history

Document title: *CE239944* - *PSOC&trade; Edge MCU: OTA Wi-Fi MQTT*

 Version | Description of change
 ------- | ---------------------
 1.x.0   | New code example <br> Early access release
 2.0.0   | GitHub release
 2.1.0   | Added instructions to use external Octal SPI (OSPI) flash <br> Added WIFI SDIO DeepSleep Callback <br> Updated design files to fix ModusToolbox™ v3.7 build warnings
 2.2.0   | Updated to support wifi-core-freertos-lwip-mbedtls v4.X <br> Updated Makefile to increase the MQTT event thread stack size <br> Added support for KIT_PSE846_EVAL_EPC2
<br>


All referenced product or service names and trademarks are the property of their respective owners.

The Bluetooth&reg; word mark and logos are registered trademarks owned by Bluetooth SIG, Inc., and any use of such marks by Infineon is under license.

PSOC&trade;, formerly known as PSoC&trade;, is a trademark of Infineon Technologies. Any references to PSoC&trade; in this document or others shall be deemed to refer to PSOC&trade;.

---------------------------------------------------------

(c) 2026, Infineon Technologies AG, or an affiliate of Infineon Technologies AG. All rights reserved.
This software, associated documentation and materials ("Software") is owned by Infineon Technologies AG or one of its affiliates ("Infineon") and is protected by and subject to worldwide patent protection, worldwide copyright laws, and international treaty provisions. Therefore, you may use this Software only as provided in the license agreement accompanying the software package from which you obtained this Software. If no license agreement applies, then any use, reproduction, modification, translation, or compilation of this Software is prohibited without the express written permission of Infineon.
<br>
Disclaimer: UNLESS OTHERWISE EXPRESSLY AGREED WITH INFINEON, THIS SOFTWARE IS PROVIDED AS-IS, WITH NO WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, ALL WARRANTIES OF NON-INFRINGEMENT OF THIRD-PARTY RIGHTS AND IMPLIED WARRANTIES SUCH AS WARRANTIES OF FITNESS FOR A SPECIFIC USE/PURPOSE OR MERCHANTABILITY. Infineon reserves the right to make changes to the Software without notice. You are responsible for properly designing, programming, and testing the functionality and safety of your intended application of the Software, as well as complying with any legal requirements related to its use. Infineon does not guarantee that the Software will be free from intrusion, data theft or loss, or other breaches (“Security Breaches”), and Infineon shall have no liability arising out of any Security Breaches. Unless otherwise explicitly approved by Infineon, the Software may not be used in any application where a failure of the Product or any consequences of the use thereof can reasonably be expected to result in personal injury.
