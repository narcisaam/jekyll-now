---
layout: post
title: Device initialization
---

### Device initialization

Some drivers use a setup/device-initialization function to set up default values, to put the device in a known state, etc.
I defined a function like this for ccs811 sensor:

```c
static int ccs811_setup(struct i2c_client *client)
{
	int ret;

	ret = ccs811_start_sensor_application(client);
	if (ret < 0)
		return ret;

	return i2c_smbus_write_byte_data(client, CCS811_MEAS_MODE,
					         CCS811_MODE_IAQ_1SEC);
}
```

You can see that this function calls ccs811_start_sensor_application() and performs a write to the MEAS_MODE register. Let's take a closer look.

`ccs811_start_sensor_application()` - what does this do ?

CCS811 powers-up in boot mode. This allows new firmware to be loaded, in case an update is needed. Firmware is the (usually) small piece of code that orchestrates the functions of a hardware device. It is stored in non-volatile memory. When ccs811 is powered up, code starts executing on the microcontroller on board. The bootloader performs the basic initialization of different hardware elements and after the boot process finishes, the state in which the device is left is called **boot mode**. In order to take measurements with our sensor, we now want to transition from boot mode to **application mode**. Similarly to when you power up your computer, after the boot process is finished, the operating system is given full control of the hardware. Here, we need to explicitly give control to the _application_ loaded to the chip. The _application_ is just another set of instructions stored in memory, responsible with managing hardware functions.
To sum up:
  * the **boot mode** allows us to reprogram the device. We can update the application loaded to the chip, when a new version is available. You can see the version of the current loaded app by reading the FW_App_Version register (0x24). The bootloader version is also available in the 0x23 register
  * the **application mode** lets us use the device for the purpose it was created: to take measurements of the indoor air quality

Maybe I overemphasized on these device-specific details, but when I first tried to take measurements with my sensor, I wrongly assumed that the application mode starts automatically. I first read the hardware id and version and everything was fine. Then I  intended to choose an operating mode for the device by writing the MEAS_MODE register and surprise! The error register was set with “invalid register address ID”. I double checked the address of MEAS_MODE register from the datasheet - it _was_ valid. Only... it was valid while in application mode! And I was still in boot mode. Basically, if you read the datasheet you can see there are 2 register maps: 
 * application register map
 * bootloader register map
  
You can check the current mode by reading the FW_MODE flag (7th bit of STATUS register). Note that STATUS register is available in both ways of operating. From the datasheet, if the FW_MODE flag's value is:

	0: Firmware is in boot mode, this allows new firmware to be loaded 
	1: Firmware is in application mode. CCS811 is ready to take ADC measurements

Going back to `ccs811_start_sensor_application()`: to transition to application mode, we first check the STATUS register's flag APP_VALID (4th bit). This tells us if a valid application is loaded in memory. If the answer is 'yes', we can try to start it by making a setup write to the APP_START register. A setup write means sending just the address of the APP_START register to the device, with no other data to follow. After this, we go and check the STATUS register again, to make sure we successfully transitioned to application mode (FW_MODE flag is 1). If the operation is successful, we're ready to take ADC measurements (we can read voltage across the sensor and current).

```c
static int ccs811_start_sensor_application(struct i2c_client *client)
{
	int ret;

	ret = i2c_smbus_read_byte_data(client, CCS811_STATUS);
	if (ret < 0)
		return ret;

	if ((ret & CCS811_STATUS_APP_VALID_MASK) !=
	    CCS811_STATUS_APP_VALID_LOADED)
		return -EIO;

	ret = i2c_smbus_write_byte(client, CCS811_APP_START);
	if (ret < 0)
		return ret;

	ret = i2c_smbus_read_byte_data(client, CCS811_STATUS);
	if (ret < 0)
		return ret;

	if ((ret & CCS811_STATUS_FW_MODE_MASK) !=
	    CCS811_STATUS_FW_MODE_APPLICATION) {
		dev_err(&client->dev, "Application failed to start. Sensor is still in boot mode.\n");
		return -EIO;
	}

	return 0;
}
```

### Selecting operating mode

To be able to measure TVOC and eCO2 we need to place the device in a state in which measurements are enabled. This is done by writing the desired operating mode to MEAS_MODE register. By default, the device is idle, which means all IAQ measurements are off. We can choose between 4 drive modes, that enable measurements:

	- Mode 1 – Constant power mode, IAQ measurement every second
	- Mode 2 – Pulse heating mode IAQ measurement every 10 seconds
	- Mode 3 – Low power pulse heating mode IAQ measurement every 60 seconds
	- Mode 4 – Constant power mode, sensor measurement every 250ms

```c
return i2c_smbus_write_byte_data(client, CCS811_MEAS_MODE, CCS811_MODE_IAQ_1SEC);
```
Here, I set the measurement mode to Mode 1

For visualization on the entire setup process, check out this nice diagram, taken from [CCS811_Programming_Guide](https://cdn.sparkfun.com/datasheets/BreakoutBoards/CCS811_Programming_Guide.pdf).

![Initialization Flow]({{ site.baseurl }}/images/initflowdiagram.png "Initialization Flow")





