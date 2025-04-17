# QuarkLink Getting Started

Similarly to [quarklink-getting-started-esp32-platformio](https://github.com/cryptoquantique/quarklink-getting-started-esp32-platformio), this project is a simple demonstration on how to get started with QuarkLink to secure an esp32 device.

In particular, this project uses the [esp-idf framework](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/) and is designed specifically for the [M5Stack EduKit](https://shop.m5stack.com/products/m5stack-core2-esp32-iot-development-kit-for-aws-iot-edukit) unit, that comes with an on-board Secure Element (Microchip ATECC-608).

The result is an M5Stack (ESP32-Eco3) that is secured by using:
- Secure Boot v2
- Flash Encryption
- Hardware Root-of-Trust

and which can only be updated Over-The-Air with firmware signed by a key from the QuarkLink Hardware Security Module (HSM).

See the [QuarkLink Getting Started Guide](https://docs.quarklink.io/docs/getting-started-with-quarklink-ignite) for more detailed information.

## Requirements

There are a few requirements needed in order to get started with this project:

- **M5Stack EduKit**
    This project is only meant for this specific device.
- **esp-idf v5.3**
    This project is meant for the esp-idf framework and it needs at least v5.3 due to some updates that are necessary to properly use the secure element and the TLS1.3 stack.  
    Instructions on how to setup the esp-idf environment can be found [here](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/index.html).
- **quarklink-client libraries**
    The quarklink-client library comes in the form of compiled binaries and can be found in the [quarklink-binaries repository](https://github.com/cryptoquantique/quarklink-binaries/tree/main/quarklink-client).  
    The files needed for this project are labelled with *m5edukit-ecc608* and need to copied inside the `lib` folder in this project.

## Setup

### Provision the ATECC-608 Secure Element
When configuring a new device, there is need to provision the Secure Element in order to use it.  
The esp-cryptoauthlib component included in this project, is a port of Microchip's [cryptoauthlib](https://github.com/MicrochipTech/cryptoauthlib) for ESP-IDF and includes all the utilities needed to configure, provision and use the Secure Element module. The component can also be found [here](https://github.com/espressif/esp-cryptoauthlib).  
Instructions on how to provision the ATECC-608 are included in the `esp_cryptoauth_utility` [README.md](./components/esp-cryptoauthlib/esp_cryptoauth_utility/README.md).  

### Project configuration
>**Note**  
It is assumed that the user has already familiarised with the benefits of using the *Digital Signature peripheral* and the difference between *virtual-efuse* and *release* during the QuarkLink provisioning process.  
More information can be found as part of the espressif programming guide:  
\- [Digital Signature](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-reference/peripherals/ds.html)  
\- [Virtual eFuses](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/efuse.html#virtual-efuses)

Use ESP-IDF's `idf.py` command to configure the project:
- Run `idf.py set-target esp32` to select the target device (note: M5Stack edukit mounts an esp32 rev3)  
    This will create the file *sdkconfig*.
- The created configuration files will already contain the changes imposed by *sdkconfig.defaults*
- The default configuration uses virtual-efuses, if this is not the desired configuration update it following the instructions below:  
    There are two ways a user can switch between *virtual-efuses* and *release* configurations:
    1. Copy the content of the relevant sdkconfig file (i.e. [*sdkconfig-release*](./sdkconfig-release)) and paste it into the newly generated *sdkconfig* to replace its content.
    2. Update [*sdkconfig.defaults*](./sdkconfig.defaults) by commenting out the vefuse section and un-commenting the release section, delete the generated *sdkconfig* to allow the generation of a new, updated one.

>**⚠️ Important**  
Updating a device from **vefuse** to **release** or vice-versa, will break the device. When performing an OTA update, make sure to select a firmware that is of the same type as the one provisioned.

### Select quarklink-client library
To change the quarklink-client compiled library to use, simply modify the [CMakeLists.txt](./main/CMakeLists.txt) in the main folder and update with the path and name of the file needed.
E.g.
```c
add_prebuilt_library(quarklink_client "../lib/libquarklink-client-esp32-m5edukit-ecc608-v1.4.6.a"
                    PRIV_REQUIRES nvs_flash esp_http_client esp_https_ota app_update mbedtls esp-cryptoauthlib)
```
Becomes
```c
add_prebuilt_library(quarklink_client "../lib/libquarklink-client-esp32-m5edukit-ecc608-v1.4.6-debug.a"
                    PRIV_REQUIRES nvs_flash esp_http_client esp_https_ota app_update mbedtls esp-cryptoauthlib)
```


## Build
Run `idf.py build` to build the project. 
This command will generate the firmware that needs to be uploaded to QuarkLink for the OTA update.
```
idf.py build

. . .

Creating esp32 image...
Merged 25 ELF sections
Successfully created esp32 image.
Generated <path/to/>quarklink-getting-started-m5edukit-ecc608/build/quarklink-getting-started-m5edukit-ecc608.bin

. . .

```

You might notice that at the end the utility prompts you to sign the firmware and suggests what command to run to flash the device. This is not possible, since the device has already been provisioned via QuarkLink and can only be updated with firmware that is signed with the same key, securely stored in the QuarkLink HSM.

In order to update your device, you need to upload the generated binary `build/quarklink-getting-started-m5edukit-ecc608.bin` to your QuarkLink instance, where it will be signed and provided to the running device as an over-the-air update.

## ML-KEM-768 support
This project now supports hybrid Post-Quantum Cryptography using ML-KEM-768. Because hybrid key exchange is only supported in TLS1.3 and above, TLS1.3 has been enabled as part of the sdkconfig.defaults. Furthermore, ESP-IDF 5.3.1 or above must be used.
In the root folder there is a patch file that applies the necessary changes to the mbedtls component: adds all the ML-KEM-768 related files and enables its use in the handshake process. 

The patch only needs to be applied once to the mbedtls component installed in your <IDF_PATH>. To apply the patch run the following command from terminal:
```sh
patch -p1 -i <PATH/TO/PROJECT>/mlkem_mbedtls.patch -d <IDF_PATH>/components/mbedtls/mbedtls
```
**On Windows**: the above command needs a recent version of `patch` (2.7.6) that comes as part of Git. If not in PATH, run the above from the Git installation folder, generally:
```sh
"C:\Program Files\Git\usr\bin\patch.exe" -p1 -i <PATH/TO/PROJECT>/mlkem_mbedtls.patch -d <IDF_PATH>/components/mbedtls/mbedtls
```

Once the patch has been applied build the application as explained above, the compiled binary can be used for hybrid PQC enabled communication.  

## Further Notes
**Custom Partition Table:** users might be interested in using their own partition table with QuarkLink. Currently, support for this feature is only for paid tiers, however users are welcome to request a custom partition table via the GitHub issues on this project.  
**Firmware size reduction:** Users may wanted to reduce the firmware footprint for this getting started program. This can be achieved by enabling `CONFIG_COMPILER_OPTIMIZATION_SIZE=y` in the sdkconfig file. Moreover further memory optimization techniques can be found in [the guide](https://docs.espressif.com/projects/esp-idf/en/v5.4/esp32/api-guides/performance/size.html)
