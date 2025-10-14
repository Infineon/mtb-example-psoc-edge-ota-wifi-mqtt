# PSOC&trade; Edge MCU: OTA Wi-Fi MQTT

This code example demonstrates the firmware update of user application over-the-air (OTA) via the Wi-Fi interface using MQTT protocol, using Arm&reg; Cortex&reg; M33 CPU on PSOC&trade; Edge MCU coupled with AIROC&trade; CYW55513 in ModusToolbox&trade; environment.  

The device establishes a connection with the designated MQTT broker – this example uses AWS. It periodically checks the job document to see if a new update is available. When a new update is available, it is downloaded and written to the secondary slot (flash). On the next reboot, the Edge Protect Bootloader handles image verification and upgrades. This example uses overwrite-based upgrade. In an overwrite-based upgrade, the new image from the secondary slot is simply copied to the primary slot after a successful validation.

This code example has a three project structure: CM33 secure, CM33 non-secure, and CM55 projects. All three projects are programmed to the external QSPI flash and executed in Execute in Place (XIP) mode. It also requires you to add the Edge Protect Bootloader project (*proj_bootloader*). The Edge Protect bootloader is responsible for validation and updation of the images downloaded to the device.
The extended boot launches the Edge Protect Bootloader from the RRAM and the Edge Protect Bootloader launches the CM33 Secure project from a known location in the external flash, which then launches the CM33 non-secure application. CM33 non-secure application is responsible for starting CM55 CPU and handling the OTA task. 


This example has been tested with the following (out-of-box) configurations:

1. Device lifecycle - Development
2. Secure Boot - Disabled 

[View this README on GitHub.](https://github.com/Infineon/mtb-example-psoc-edge-ota-wifi-mqtt)

[Provide feedback on this code example.](https://cypress.co1.qualtrics.com/jfe/form/SV_1NTns53sK2yiljn?Q_EED=eyJVbmlxdWUgRG9jIElkIjoiQ0UyMzk5NDQiLCJTcGVjIE51bWJlciI6IjAwMi0zOTk0NCIsIkRvYyBUaXRsZSI6IlBTT0MmdHJhZGU7IEVkZ2UgTUNVOiBPVEEgV2ktRmkgTVFUVCIsInJpZCI6InZhc3VtYSBuYXJheWFuYSIsIkRvYyB2ZXJzaW9uIjoiMi4wLjAiLCJEb2MgTGFuZ3VhZ2UiOiJFbmdsaXNoIiwiRG9jIERpdmlzaW9uIjoiTUNEIiwiRG9jIEJVIjoiSUNXIiwiRG9jIEZhbWlseSI6IlBTT0MifQ==)

See the [Design and implementation](docs/design_and_implementation.md) for the functional description of this code example.


## Requirements

- [ModusToolbox&trade;](https://www.infineon.com/modustoolbox) v3.6 or later (tested with v3.6)
- Board support package (BSP) minimum required version: 1.0.0
- Programming language: C
- Associated parts: All PSOC&trade; Edge E84 MCU parts
- Python (tested with v3.13.7)


## Supported toolchains (make variable 'TOOLCHAIN')

- GNU Arm&reg; Embedded Compiler v14.2.1 (`GCC_ARM`) – Default value of `TOOLCHAIN`
- Arm&reg; Compiler v6.22 (`ARM`)
- IAR C/C++ Compiler v9.50.2 (`IAR`)
- LLVM Embedded Toolchain for Arm&reg; v19.1.5 (`LLVM_ARM`)


## Supported kits (make variable 'TARGET')

- [PSOC&trade; Edge E84 Evaluation Kit](https://www.infineon.com/KIT_PSE84_EVAL) (`KIT_PSE84_EVAL_EPC2`) – Default value of `TARGET`
- [PSOC&trade; Edge E84 Evaluation Kit](https://www.infineon.com/KIT_PSE84_EVAL) (`KIT_PSE84_EVAL_EPC4`)


## Hardware setup

This example uses the board's default configuration. See the kit user guide to ensure that the board is configured correctly.

Ensure the following jumper and pin configuration on board:
- The BOOT SW pin (P17.6) is in the LOW/OFF position
- J20 and J21 are in the tristate/not connected (NC) position


## Software setup

See the [ModusToolbox&trade; tools package installation guide](https://www.infineon.com/ModusToolboxInstallguide) for information about installing and configuring the tools package.

Install a terminal emulator if you do not have one. Instructions in this document use [Tera Term](https://teratermproject.github.io/index-en.html).

This example requires no additional software or tools.


## Operation

### Firmware Update using AWS MQTT broker

<details><summary><b>Add the Edge Protect Bootloader to application</b></summary>

1. Add *proj_bootloader* to this code example as a first step. Follow steps 2 to 9 in the **Operation** section of the [Edge Protect Bootloader](https://github.com/Infineon/mtb-example-edge-protect-bootloader) code example's *README.md* file
  
  > **Note:** 
  > 1. [Edge Protect Bootloader](https://github.com/Infineon/mtb-example-edge-protect-bootloader) *README.md* describes how to add the bootloader to **Basic Secure App**. Add it to **this** code example instead
  > 2. (For the `LLVM_ARM` compiler) **Edge Protect Bootloader** does not support `LLVM_ARM`. In the *proj_bootloader* directory, edit the *Makefile* to force other supported compilers for EPB (for example: `TOOLCHAIN=GCC_ARM`)
 
</details>

<details><summary><b>Configure the memory map</b></summary>

2. Configure the memory map: this code example bundles a custom *design.modus* file compatible with the Edge Protect Bootloader. Hence, you need not make any further customizations here to use EPB and this project together

</details>

<details><summary><b>Set up the AWS MQTT broker</b></summary>

3. Set up the MQTT device (also known as a **Thing**) in the AWS IoT core as described in the [Getting started with AWS IoT tutorial](https://docs.aws.amazon.com/iot/latest/developerguide/iot-gs.html).

  > **Note:** While setting up your device, ensure that the policy associated with this device permits all MQTT operations (**iot:Connect**, **iot:Publish**, **iot:Receive**, and **iot:Subscribe**) for the resource used by this device. For testing purposes, it is recommended to have the following policy document, which allows all **MQTT Policy Actions** on all **Amazon resource names (ARNs)**.
      
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
       - A **private key:** *xxxxxxxxxx.aws-private.key* or *xxxxxxxxxx.aws-private.pem*
       - Root CA "RSA 2048 bit key: Amazon Root CA 1" for AWS IoT from [CA certificates for server authentication](https://docs.aws.amazon.com/iot/latest/developerguide/server-authentication.html#server-authentication-certs) - *xxxx.AmazonRootCA1.crt*

 5. Copy the certificates and key, paste it in the *\<My-OTA-MQTT-example>/scripts* folder

 6. Rename the following file names in the *\<My-OTA-MQTT-example>/scripts* folder
      - *xxxx.AmazonRootCA1.crt* to *amazon_ca.crt*
      - *xxxxxxxxxx.aws-client-certificate.crt* to *amazon_client.crt*
      - *xxxxxxxxxx.aws-private.key* to *amazon_private.key*

 </details>

<details><summary><b>Build and program the bootloader and application </b></summary>

7. Open the *common.mk* file in the root of the application and set the `COMBINE_SIGN_JSON` variables to the configurator-generated *boot_with_bldr.json* file as shown here:

    ```
    COMBINE_SIGN_JSON?=./bsps/TARGET_$(TARGET)/config/GeneratedSource/boot_with_bldr.json
    ```

8. Ensure IMG_TYPE is set to BOOT as shown below:

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

11. Navigate to *\<My-OTA-MQTT-example>/scripts* directory. Execute format_cert_key.py python script to generate the string format of the certificate and key files that can be added as macro. Pass the name of the certificate or key with the extension as an argument to the Python script.

```
python format_cert_key.py <one-or-more-file-name-of-certificate-or-key-with-extension>
```

Example:

```
python format_cert_key.py amazon_ca.crt amazon_client.crt amazon_private.pem
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

17.  Open the job document  *ota_update.json* in the  *\<My-OTA-MQTT-example>/scripts* directory and ensure the version set to to **1.1.0** or later. This version must be different than what has been already programmed on the target

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

    On Linux and macOS, you can use any terminal application. On Windows, open 'modus-shell' app from the Start menu

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

    **Figure 5. Terminal output for ota download completed**

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

The steps to perform an OTA upgrade remains same as described in steps 1-2 in the [Operation](#operation) section above. The following instructions indicate only the steps that differ for the Mosquitto broker.

#### TLS mode

1. Follow the instructions in this [link](https://test.mosquitto.org/ssl/) to generate your own keys and certificates

2. Place the CA certificate, client key, and client certificate in the *\<My-OTA-MQTT-example>/scripts/ directory*
    
  > **Note:** Ensure the filenames are the same as used in the publisher script:
  >- CA Certificate: *mosquitto.org.crt*
  >- Client certificate: *mosquitto_client.crt*
  >- Client key: *mosquitto_client.key*

3. Open a CLI terminal.

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

8. Update the MbedTLS config file to use TLS v1.2 . Open the file *\<My-OTA-MQTT-example>/proj_cm33_ns/app_ota/configs/mbedtls_user_config.h.*

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

The steps to perform an OTA upgrade remains same as described in the Steps 1-5 in the [Operation](#operation) section above. Below instructions indicate only those steps that different for the Eclipse mode.

#### Non TLS mode

1. Open a CLI terminal

   On Linux and macOS, you can use any terminal application. On Windows, open 'modus-shell' app from the Start menu

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


## Related resources

Resources  | Links
-----------|----------------------------------
Application notes  | [AN235935](https://www.infineon.com/AN235935) – Getting started with PSOC&trade; Edge E84 MCU on ModusToolbox&trade; software
Code examples  | [Using ModusToolbox&trade;](https://github.com/Infineon/Code-Examples-for-ModusToolbox-Software) on GitHub
Device documentation | [PSOC&trade; Edge E84 MCU datasheet](https://www.infineon.com/products/microcontroller/32-bit-psoc-arm-cortex/32-bit-psoc-edge-arm/psoc-edge-e84#Documents) <br> [PSOC&trade; Edge E84 MCU reference manuals](https://www.infineon.com/products/microcontroller/32-bit-psoc-arm-cortex/32-bit-psoc-edge-arm/psoc-edge-e84#Documents)
Development kits | Select your kits from the [Evaluation board finder](https://www.infineon.com/cms/en/design-support/finder-selection-tools/product-finder/evaluation-board)
Libraries  | [mtb-dsl-pse8xxgp](https://github.com/Infineon/mtb-dsl-pse8xxgp) – Device support library for PSE8XXGP <br> [retarget-io](https://github.com/Infineon/retarget-io) – Utility library to retarget STDIO messages to a UART port <br> [serial-memory](https://github.com/Infineon/serial-memory) - Serial Flash Middleware for External memory interface <br> [DFU](https://github.com/Infineon/dfu) – Device Firmware Update Middleware (DFU MW) <br> [emusb-device](https://github.com/Infineon/emusb-device) – USB Device stack for embedded applications
Tools  | [ModusToolbox&trade;](https://www.infineon.com/modustoolbox) – ModusToolbox&trade; software is a collection of easy-to-use libraries and tools enabling rapid development with Infineon MCUs for applications ranging from wireless and cloud-connected systems, edge AI/ML, embedded sense and control, to wired USB connectivity using PSOC&trade; Industrial/IoT MCUs, AIROC&trade; Wi-Fi and Bluetooth&reg; connectivity devices, XMC&trade; Industrial MCUs, and EZ-USB&trade;/EZ-PD&trade; wired connectivity controllers. ModusToolbox&trade; incorporates a comprehensive set of BSPs, HAL, libraries, configuration tools, and provides support for industry-standard IDEs to fast-track your embedded application development

<br>


## Other resources

Infineon provides a wealth of data at [www.infineon.com](https://www.infineon.com) to help you select the right device, and quickly and effectively integrate it into your design.


## Document history

Document title: *CE239944* - *PSOC&trade; Edge MCU: OTA WIFI MQTT*

 Version | Description of change
 ------- | ---------------------
 1.x.0   | New code example <br> Early access release
 2.0.0   | GitHub release
<br>


All referenced product or service names and trademarks are the property of their respective owners.

The Bluetooth&reg; word mark and logos are registered trademarks owned by Bluetooth SIG, Inc., and any use of such marks by Infineon is under license.

PSOC&trade;, formerly known as PSoC&trade;, is a trademark of Infineon Technologies. Any references to PSoC&trade; in this document or others shall be deemed to refer to PSOC&trade;.

---------------------------------------------------------

© Cypress Semiconductor Corporation, 2025. This document is the property of Cypress Semiconductor Corporation, an Infineon Technologies company, and its affiliates ("Cypress").  This document, including any software or firmware included or referenced in this document ("Software"), is owned by Cypress under the intellectual property laws and treaties of the United States and other countries worldwide.  Cypress reserves all rights under such laws and treaties and does not, except as specifically stated in this paragraph, grant any license under its patents, copyrights, trademarks, or other intellectual property rights.  If the Software is not accompanied by a license agreement and you do not otherwise have a written agreement with Cypress governing the use of the Software, then Cypress hereby grants you a personal, non-exclusive, nontransferable license (without the right to sublicense) (1) under its copyright rights in the Software (a) for Software provided in source code form, to modify and reproduce the Software solely for use with Cypress hardware products, only internally within your organization, and (b) to distribute the Software in binary code form externally to end users (either directly or indirectly through resellers and distributors), solely for use on Cypress hardware product units, and (2) under those claims of Cypress's patents that are infringed by the Software (as provided by Cypress, unmodified) to make, use, distribute, and import the Software solely for use with Cypress hardware products.  Any other use, reproduction, modification, translation, or compilation of the Software is prohibited.
<br>
TO THE EXTENT PERMITTED BY APPLICABLE LAW, CYPRESS MAKES NO WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, WITH REGARD TO THIS DOCUMENT OR ANY SOFTWARE OR ACCOMPANYING HARDWARE, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.  No computing device can be absolutely secure.  Therefore, despite security measures implemented in Cypress hardware or software products, Cypress shall have no liability arising out of any security breach, such as unauthorized access to or use of a Cypress product. CYPRESS DOES NOT REPRESENT, WARRANT, OR GUARANTEE THAT CYPRESS PRODUCTS, OR SYSTEMS CREATED USING CYPRESS PRODUCTS, WILL BE FREE FROM CORRUPTION, ATTACK, VIRUSES, INTERFERENCE, HACKING, DATA LOSS OR THEFT, OR OTHER SECURITY INTRUSION (collectively, "Security Breach").  Cypress disclaims any liability relating to any Security Breach, and you shall and hereby do release Cypress from any claim, damage, or other liability arising from any Security Breach.  In addition, the products described in these materials may contain design defects or errors known as errata which may cause the product to deviate from published specifications. To the extent permitted by applicable law, Cypress reserves the right to make changes to this document without further notice. Cypress does not assume any liability arising out of the application or use of any product or circuit described in this document. Any information provided in this document, including any sample design information or programming code, is provided only for reference purposes.  It is the responsibility of the user of this document to properly design, program, and test the functionality and safety of any application made of this information and any resulting product.  "High-Risk Device" means any device or system whose failure could cause personal injury, death, or property damage.  Examples of High-Risk Devices are weapons, nuclear installations, surgical implants, and other medical devices.  "Critical Component" means any component of a High-Risk Device whose failure to perform can be reasonably expected to cause, directly or indirectly, the failure of the High-Risk Device, or to affect its safety or effectiveness.  Cypress is not liable, in whole or in part, and you shall and hereby do release Cypress from any claim, damage, or other liability arising from any use of a Cypress product as a Critical Component in a High-Risk Device. You shall indemnify and hold Cypress, including its affiliates, and its directors, officers, employees, agents, distributors, and assigns harmless from and against all claims, costs, damages, and expenses, arising out of any claim, including claims for product liability, personal injury or death, or property damage arising from any use of a Cypress product as a Critical Component in a High-Risk Device. Cypress products are not intended or authorized for use as a Critical Component in any High-Risk Device except to the limited extent that (i) Cypress's published data sheet for the product explicitly states Cypress has qualified the product for use in a specific High-Risk Device, or (ii) Cypress has given you advance written authorization to use the product as a Critical Component in the specific High-Risk Device and you have signed a separate indemnification agreement.
<br>
Cypress, the Cypress logo, and combinations thereof, ModusToolbox, PSoC, CAPSENSE, EZ-USB, F-RAM, and TRAVEO are trademarks or registered trademarks of Cypress or a subsidiary of Cypress in the United States or in other countries. For a more complete list of Cypress trademarks, visit www.infineon.com. Other names and brands may be claimed as property of their respective owners.
