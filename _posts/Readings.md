---
layout: post
title: Reading from the device
---


After the device is initialized and one of the „active measurements” mode is set, the sensor provides samples of data that can be read from the register called (in the datasheet) „Algorithm Results Data”.  This is an 8-byte read only register that contains the calculated eCO2 (ppm) and TVOC (ppb) values followed by the STATUS register, ERROR_ID register and the RAW_DATA register(which holds current + voltage). As you can see, the device provides multiple types of information: eCO2, TVOC, current through the sensor, voltage across the sensor. For each of these types of data we are going to define an IIO channel. 

An IIO channel is represented using `struct iio_chan_spec`.  I need 3 types of channels for my driver: IIO_CURRENT, IIO_VOLTAGE, IIO_CONCENTRATION. eCO2 and TVOC represent 2 separate data channels (2 different types of information) that translate into the same type of IIO channel: IIO_CONCENTRATION. To distinguish between them, we use modifiers to indicate the unique purpose of each channel. 

We also want to specify some **attributes** for our channels. For example, the value of eCO2 retrieved from the sensor's register ranges from 400 parts per million(ppm) to 8192 ppm. But if I tell you that the air in your room is marked by a value of 4600 ppm eCO2 you may not understand that you need to open the window and we'll suffer from lungs, liver, kidney & central nervous system damage + eye, nose, throat, skin irritation, headaches, nausea, dizziness, significant impairment of performance & decision-making. Sounds like I'm writing down the side effects of some medicine? I'm actually quoting from the sensor's documents. 

![Poor Air Quality]({{ site.baseurl }}/images/effects_of_poor_air.jpg "Poor air quality")

But don't worry (too much), those are the effects of _long_ term exposure to poor-quality air. Anyway, to help you better understand the meaning of eCO2 values, I will translate them  into percentage (%), by mapping the [400, 8192] interval to [0, 100] interval. This is done by applying an offset and a scale, so the end result will be the eCO2 concentration in the air. The same goes for TVOC channel, and we also make sure to add scales for voltage and current too, in order to have the end results expressed in the right measurements units (millivolts for voltage, milliamps for current, etc.) Available standard attributes for IIO devices [here](Documentation/ABI/testing/sysfs-bus-iio) and more about IIO channels [here](https://dbaluta.github.io/iiosubsys.html#iiochannel).

```c
static const struct iio_chan_spec ccs811_channels[] = {
	{
		.type = IIO_CURRENT,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW) |
				      BIT(IIO_CHAN_INFO_SCALE),
	}, {
		.type = IIO_VOLTAGE,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW) |
				      BIT(IIO_CHAN_INFO_SCALE),
	}, {
		.type = IIO_CONCENTRATION,
		.channel2 = IIO_MOD_CO2,
		.modified = 1,
		.info_mask_separate = BIT(IIO_CHAN_INFO_RAW) |
				      BIT(IIO_CHAN_INFO_OFFSET) |
				      BIT(IIO_CHAN_INFO_SCALE),
	}, {
		.type = IIO_CONCENTRATION,
		.channel2 = IIO_MOD_VOC,
		.modified = 1,
		.info_mask_separate =  BIT(IIO_CHAN_INFO_RAW) |
				       BIT(IIO_CHAN_INFO_SCALE),
	},
};
```

You can see that I used a bitmask called `info_mask_separate` to set up some properties for each channel (for example the eCO2 channel needed an offset to be applied so I set the offset bit to 1 into info_mask_separate mask). 

Every channel definition generates a sysfs file that can be read to retrieve the data corresponding to that channel.
For information related to eCO2 channel, 3 files are created:

* in_concentration_co2_raw
* in_concentration_co2_scale
* in_concentration_co2_offset

Obviously, the first one will output the raw value of eCO2, read from the sensor, the scond one will contain the value by which we scale the raw value, and the last one - the offset. 

We defined the channels, next step is to attach them to the device, in probe function:

```c
indio_dev->channels = ccs811_channels;
indio_dev->num_channels = ARRAY_SIZE(ccs811_channels);
```

We're ready to define the read_raw function. It's not the most elegant of functions, one of the reasons being that it is pretty long, so I will not output it here, but this is its prototype: 

```c
static int ccs811_read_raw(struct iio_dev *indio_dev, struct iio_chan_spec const *chan, int *val, int *val2, long mask);
```

Let's say we want to read in_concentration_co2_raw file. For this file, the _mask_ variable will have the value set to "IIO_CHAN_INFO_RAW" and the chan variable's type will be IIO_CONCENTRATION. Inside the read_raw function we first check the value of the mask against all the possible values it can take (given the attributes specified in masks, while defining the channels), then we determine the channel type, the channel modifiers, until we can exactly say which (unique) file is being read. We use *val and *val2 to output the results of the measurement for the current file.

The function used to retrieve the data from the sensor is this one: 

```c
static int ccs811_get_measurement(struct ccs811_data *data)
{
	int ret, tries = 11;

	/* Maximum waiting time: 1s, as measurements are made every second */
	while (tries-- > 0) {
		ret = i2c_smbus_read_byte_data(data->client, CCS811_STATUS);
		if (ret < 0)
			return ret;

		if ((ret & CCS811_STATUS_DATA_READY) || tries == 0)
			break;
		msleep(100);
	}
	if (!(ret & CCS811_STATUS_DATA_READY))
		return -EIO;

	return i2c_smbus_read_i2c_block_data(data->client,
					    CCS811_ALG_RESULT_DATA, 8,
					    (char *)&data->buffer);
}
```
I don't have support for hardware interrupts yet, so I need to poll the STATUS register to find out when new data samples are ready in ALG_RESULT_DATA register. The sensor is set to perform measurements every second. Therefore, I wait 1 second for the data ready bit to be set and I error out in case this doesn't happen in time. From the moment after the first time when I poll the register, to the moment before the last time I poll it, exactly 1 second had been spent "sleeping". That is, the driver sleeps 100ms after every try to read the data(10 times). I initilized the `tries` variable with 11, but if its value reaches 0, I break from the loop to avoid sleeping for another 100ms in vain. You can see that when `tries` becomes 0, we enter the loop for the last time, so either data is ready or not, but the decision is made inside this last loop, so there's no need to wait again 100ms at the end of this loop, because a decision has already been made.

One more thing to note:
I store the results inside the global data variable:

`struct ccs811_data *data = iio_priv(indio_dev);`

Readings and writings to this variable can happen simultaneously, so protecting it with a lock is necessary, in order to avoid inconsistencies and compromised data.

