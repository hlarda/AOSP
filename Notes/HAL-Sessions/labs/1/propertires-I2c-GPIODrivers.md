# Lab Steps

## Add Custom VHAL Properties Changing in HAL

`hardware/interfaces/automotive/vehicle/aidl/impl/utils/test_vendor_properties/android/hardware/automotive/vehicle/TestVendorProperty.aidl`

```aidl
 /**
     * ITI_INT_Properties
     */
    ITI_DOOR_KEY    = 0x0F33 + 0x20000000 + 0x01000000 + 0x00400000,
    ITI_SPEED_METER = 0x0F32 + 0x20000000 + 0x01000000 + 0x00400000,
```

`packages/services/Car/service/src/com/android/car/hal/fakevhal/FakeVhalConfigParser.java`

```java
    private static final Map<String, Integer> CONSTANTS_BY_NAME = Map.ofEntries(
            Map.entry("ITI_SPEED_METER", TestVendorProperty.ITI_DOOR_KEY),
            Map.entry("ITI_DOOR_KEY", TestVendorProperty.ITI_SPEED_METER),
            //EXISTING ENTRIES
    );
```

`hardware/interfaces/automotive/vehicle/aidl/impl/default_config/config/TestProperties.json`

```json
{
    "properties": [
        {
            "property": "TestVendorProperty::ITI_DOOR_KEY",
            "defaultValue": {
                "int32Values": [ 0 ]
            },
            "access": "VehiclePropertyAccess::READ_WRITE",
            "changeMode": "VehiclePropertyChangeMode::ON_CHANGE",
            "minSampleRate": 1
        },
        {
            "property": "TestVendorProperty::ITI_SPEED_METER",
            "defaultValue": {
                "int32Values": [ 777 ]
            },
            "access": "VehiclePropertyAccess::READ",
            "changeMode": "VehiclePropertyChangeMode::ON_CHANGE",
            "minSampleRate": 1
        },
        // REST OF PROPERTIES
]
}
```

`hardware/interfaces/automotive/vehicle/aidl/impl/fake_impl/hardware/src/FakeVehicleHardware.cpp`

1. Include drivers

    ```cpp
    #include "/home/ubuntu/aosp/device/brcm/iti/VHalFullStack/gpio/1.0/default/GpioHal.h"
    #include "/home/ubuntu/aosp/device/brcm/iti/VHalFullStack/itoc/1.0/default/ItoCHal.h"
    ```

2. `FakeVehicleHardware::ValueResultType FakeVehicleHardware::maybeGetSpecialValue`

   ```cpp
   switch (propId) {
        case toInt(TestVendorProperty::ITI_DOOR_KEY):
            {
                *isSpecialValue = true;
                GpioHal gpioHal;
                int gpioPin = 18;

                 // Ensure GPIO is set as input to read its state
                gpioHal.exportGpio(gpioPin);
                gpioHal.setGpioDirection(gpioPin, "in");

                // Read GPIO state
                bool gpioState = false;
                gpioHal.getGpioValue(gpioPin, gpioState);

                // Return GPIO state as an integer (1 or 0)
                result = mValuePool->obtainInt32(gpioState ? 1 : 0);
                result.value()->prop = propId;
                result.value()->areaId = 0;
                result.value()->timestamp = elapsedRealtimeNano();
                return result;
            }
        case toInt(TestVendorProperty::ITI_SPEED_METER):
            {
                *isSpecialValue = true;
                int speedVal = getItoCAnalogReading();  // Call your function to retrieve the temperature value
                result = mValuePool->obtainInt32(speedVal);
                ALOGD("temp %d",speedVal);
                result.value()->prop = propId;
                result.value()->areaId = 0;
                result.value()->timestamp = getItoCAnalogReading();
                return result;
                
            }
        // REST OF CASES
    }
    ```

3. `VhalResult<void> FakeVehicleHardware::maybeSetSpecialValue`

    ```cpp
    switch (propId) {
        case toInt(TestVendorProperty::ITI_DOOR_KEY):
        {
            *isSpecialValue = true;
            GpioHal gpioHal;
            int gpioPin = 18;

            gpioHal.exportGpio(gpioPin);
            gpioHal.setGpioDirection(gpioPin, "out");

            if (value.value.int32Values[0] == 1){
                gpioHal.setGpioValue(gpioPin, true);
                ALOGI("ITI: High PIN");
            }else{
                gpioHal.setGpioValue(gpioPin, false);
                ALOGI("ITI: Low PIN");
            }
        }
        return{};
        // REST OF CASES
    }
    ```

Add libs to this module `Android.bp`
    `hardware/interfaces/automotive/vehicle/aidl/impl/fake_impl/hardware/Android.bp`

```bp
cc_defaults {
name: "FakeVehicleHardwareDefaults",
header_libs: [
    "IVehicleHardware",
    "libbinder_headers",
],
export_header_lib_headers: ["IVehicleHardware"],
static_libs: [
    "VehicleHalJsonConfigLoaderEnableTestProperties",
    "VehicleHalUtils",
    "FakeVehicleHalValueGenerators",
    "FakeObd2Frame",
    "FakeUserHal",
],
required: [
    "Prebuilt_VehicleHalDefaultProperties_JSON",
    "Prebuilt_VehicleHalTestProperties_JSON",
    "Prebuilt_VehicleHalVendorClusterTestProperties_JSON",
],
shared_libs: [
    "libgrpc++",
    "libjsoncpp",
    "libprotobuf-cpp-full",
    "libgpiohal", "libitochal", // GPIO and ItoC drivers
],
export_static_lib_headers: ["VehicleHalUtils"],
}   
```

make the rc service uses this module root

`hardware/interfaces/automotive/vehicle/aidl/impl/vhal/vhal-default-service.rc`

```rc
service vendor.vehicle-hal-default /vendor/bin/hw/android.hardware.automotive.vehicle@V3-default-service
class early_hal
user root
group system inet root
```

Enable I2c ans ADC
`device/brcm/rpi4/boot/config.txt`

```txt
## Enable I2c ans ADC ##

# I2C
dtparam=i2c_arm=on
dtparam=i2c_arm_baudrate=400000
dtoverlay=ads1115

#ADC 
dtparam=cha_enable
dtparam=chb_enable
dtparam=chc_enable
dtparam=chd_enable
```

## GPIO HAL Driver 

`Android.bp`

```bp
cc_library_shared {
        name: "libgpiohal",    
        srcs: ["GpioHal.cpp"],    
        shared_libs: ["liblog", "libutils"],
        vendor: true,  
        stl: "libc++"
}
```

`GpioHal.h`

```cpp
#pragma once
#include <string>
class GpioHal {public:    bool exportGpio(int pin);
bool setGpioDirection(int pin, const std::string& direction);
bool setGpioValue(int pin, bool value);    
bool getGpioValue(int pin, bool &value);};
```

`GpioHal.cpp`

```cpp
#include "GpioHal.h"

#include <fstream>

#include <string>

bool GpioHal::exportGpio(int pin) {
    std::ofstream exportFile("/sys/class/gpio/export");
    if (!exportFile) return false;
    exportFile << pin;
    return exportFile.good();
}

bool GpioHal::setGpioDirection(int pin, const std::string& direction) {

std::string directionPath = "/sys/class/gpio/gpio" + std::to_string(pin) + "/direction";
    std::ofstream directionFile(directionPath);
    if (!directionFile) return false;
    directionFile << direction;
    return directionFile.good();
}

bool GpioHal::setGpioValue(int pin, bool value) {
std::string valuePath = "/sys/class/gpio/gpio" + std::to_string(pin) + "/value";
    std::ofstream valueFile(valuePath);
    if (!valueFile) return false;
    valueFile << (value ? "1" : "0");
    return valueFile.good();
}

bool GpioHal::getGpioValue(int pin, bool &value) {
std::string valuePath = "/sys/class/gpio/gpio" + std::to_string(pin) + "/value";
    std::ifstream valueFile(valuePath);
    if (!valueFile) return false;
    int gpioValue;
    valueFile >> gpioValue;
    value = (gpioValue == 1);
    return valueFile.good();
}
```

## ItoC HAL Driver

`Android.bp`

```bp
cc_library_shared {
        name: "libitochal",    
        srcs: ["ItoCHal.cpp"],    
        shared_libs: ["liblog", "libutils"],
        vendor: true,  
        stl: "libc++"
}
```

`ItoCHal.h`

```cpp
#include <iostream>
#include <unistd.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>
#include <stdint.h>
#include <stdexcept>

#define ADS1115_ADDRESS 0x48  // Default I2C address for ADS1115
#define ADS1115_CONVERSION_REGISTER 0x00
#define ADS1115_CONFIG_REGISTER 0x01


int open_i2c(const char* device, int address) ;
void configure_ads1115(int file, int channel);
int16_t read_ads1115(int file);
int getItoCAnalogReading();
```

`ItoCHal.cpp`

```cpp
#include "ItoCHal.h"

#define ADS1115_ADDRESS 0x48  // Default I2C address for ADS1115
#define ADS1115_CONVERSION_REGISTER 0x00
#define ADS1115_CONFIG_REGISTER 0x01


// Function to open I2C device and set slave address
int open_i2c(const char* device, int address) {
    int file = open(device, O_RDWR);
    if (file < 0) {
        std::cerr << "Failed to open the bus" << std::endl;
        return -1;
    }

    if (ioctl(file, I2C_SLAVE, address) < 0) {
        std::cerr << "Failed to acquire bus access and/or talk to slave" << std::endl;
        close(file);
        return -1;
    }
    return file;
}

// Function to configure the ADS1115 (set input channel)
void configure_ads1115(int file, int channel) {
    uint16_t config = 0xC383;  // Default settings for 16-bit, 128 SPS, ±4.096V range

    // Set MUX to the selected channel
    switch (channel) {
        case 0: config |= 0x4000; break;  // AIN0
        case 1: config |= 0x5000; break;  // AIN1
        case 2: config |= 0x6000; break;  // AIN2
        case 3: config |= 0x7000; break;  // AIN3
        default:break;
            //throw std::invalid_argument("Invalid channel. Choose from 0, 1, 2, 3.");
    }

    uint8_t data[3];
    data[0] = ADS1115_CONFIG_REGISTER;
    data[1] = (config >> 8) & 0xFF;  // High byte
    data[2] = config & 0xFF;         // Low byte
    if (write(file, data, 3) != 3) {
        std::cerr << "Failed to write to the configuration register" << std::endl;
        close(file);
        exit(1);
    }
}

// Function to read the conversion result from the ADS1115
int16_t read_ads1115(int file) {
    uint8_t buf[2];
    buf[0] = ADS1115_CONVERSION_REGISTER;
    if (write(file, buf, 1) != 1) {
        std::cerr << "Failed to set conversion register" << std::endl;
        close(file);
        exit(1);
    }

    if (read(file, buf, 2) != 2) {
        std::cerr << "Failed to read conversion result" << std::endl;
        close(file);
        exit(1);
    }

    // Swap byte order
    int16_t result = (buf[0] << 8) | buf[1];
    return result;
}



int getItoCAnalogReading()
{
    const char* device = "/dev/i2c-1";  // Use the correct I2C bus (usually /dev/i2c-1 on Raspberry Pi)
    int file = open_i2c(device, ADS1115_ADDRESS);
    if (file < 0) return  1;

    int channel = 1;  // Set the channel to read (you can change this)
    
        // Start a new conversion
        configure_ads1115(file, channel);
        
        // Wait for the conversion to complete (typical delay 8 ms)
        usleep(10000);

        // Read the ADC value
        int16_t value = read_ads1115(file);
  
        sleep(1);
  
    close(file);
    return (int)value;
}
```
