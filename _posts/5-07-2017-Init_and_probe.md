---
layout: post
title: Init and Probe
---

A driver is described using the **i2c_driver** struct. In order to register our chip driver, we first need to populate this struct.

 * Providing a name for the driver is done using the driver.name field.
 * Then, we need to pass some function pointers that interests us. I will implement the _probe and _remove functions, so I'm populating the corresponding fields. 
 * Another member that I will initialize is the “id_table”, which lists the i2c_devices supported by the driver. On that account, I define an array that will contain entries of type **struct i2c_device_id**. This is a struct that provides 2 members: name and driver_data.
I'll add one single entry to my array,  meaning that we support one chip named “ccs811”. This means that when the kernel discovers a “ccs811” device, it will look for a driver that can handle this “ccs811”. 

Now that we defined our i2c_driver struct, we pass it to the helper macro **module_i2c_driver** that registers our driver. Basically, `module_init()` and `module_exit()` are replaced by this macro.

```c
static const struct i2c_device_id ccs811_id[] = {
	{"ccs811", 0},
	{	}
};
MODULE_DEVICE_TABLE(i2c, ccs811_id);

static struct i2c_driver ccs811_driver = {
	.driver = {
		.name = "ccs811",
	},
	.probe = ccs811_probe,
	.remove = ccs811_remove,
	.id_table = ccs811_id,
};
module_i2c_driver(ccs811_driver);
```

The **MODULE_DEVICE_TABLE** macro is used for hot plugging. It exports the table of supported devices, so when the kernel discovers a new device, the right module will be loaded for it. 

As mentioned above, our driver knows that it supports "ccs811" device. But when we plug in an I2C device, how does the kernel know what it is named? It doesn't! We must explicitly instantiate the device. Different methods of instantiating devices are documented here: [instantiating-devices](https://www.kernel.org/doc/Documentation/i2c/instantiating-devices). 
For example we can instantiate from userspace using the special attribute file new_device. 

```sh
$ echo ccs811 0x5B > /sys/bus/i2c/devices/i2c-0/new_device
```
Here we informed the kernel about a device named "ccs811" which lives at the address 0x5B. As we listed "ccs811" as a supported device, our driver now shows up and says: "I can handle this!"

But can it really? It can't be sure, until it performs a probe.

### The probe

The probe function will first check functionality. Different adapters implement different I2C specifications, so we have to test to see what functionalities are implemented.
Then it goes on and confirms the detected device is a CCS811 by reading the hardware id and version.
Any device is identified by an id. In this sensor's case, the hardware id is stored in a single byte read only register and has to have the value 0x81. The probe reads the value from this register and checks to see if it is indeed 0x81:

`ret = i2c_smbus_read_byte_data(client, CCS811_HW_ID);
	if (ret < 0)
		return ret;

	if (ret != CCS881_HW_ID_VALUE) {
		dev_err(&client->dev, "hardware id doesn't match CCS81x\n");
		return -ENODEV;
	}`

Memory for the IIO device is allocated using devm_iio_device_alloc.

```c
indio_dev = devm_iio_device_alloc(&client->dev, sizeof(*data));
```
Note the "sizeof(*data)" param. "data" is a pointer to a local struct that we define in order to hold some private information. Calling the above function also allocates memory for this struct. A pointer to the reserved memory for this struct is later acquired using the iio_priv function.
After this, we can initialize some iio_dev fields:

```c
	indio_dev->dev.parent = &client->dev;
	indio_dev->name = id->name;
	indio_dev->info = &ccs811_info;
```
Lastly, calling `iio_device_register()` will register the device with the I2C core.


