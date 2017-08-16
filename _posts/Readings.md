

Reading from the device

After the device is initialized and one of the „active measurements” mode is set, the sensor provides samples of data that can be read from the register called (in the datasheet) „Algorithm Results Data”.  This is an 8-byte read only register that contains the calculated eCO2
 (ppm) and TVOC (ppb) values followed by the STATUS register,
 ERROR_ID register and the RAW_DATA register(which holds current + voltage). As you can see, the device provides multiple types of information: eCO2, TVOC, current through the sensor, voltage across the sensor. For each of this types of data we are going to define an IIO channel. 

An IIO channel is represented using struct iio_chan_spec.  I need 3 types of channels for my driver: IIO_CURRENT, IIO_VOLTAGE, IIO_CONCENTRATION. eCO2 and TVOC represent  2 separate data channels (2 different types of information) that translate into the same type of IIO channel:  IIO_CONCENTRATION. To distinguish between them, we use modifiers to indicate the unique purpose of each channel. 

We also want to specify some attributes for our channels. For example, the value of eCO2 retrieved from the sensor's register ranges from 400 parts per million(ppm) to 8192 ppm. But if I tell you that the air in your room is marked by a value of 2600 ppm eCO2 you may not understand that you need to open the window and we'll suffer from lungs, liver,
 kidney & central nervous system damage
 +
eye, nose, throat, skin irritation, headaches,
 nausea, dizziness
, significant impairment of performance &
 decision-making. No, I'm not writing down the side effects of some medicine, I'm actually quoting from the sensor's documents. But don't worry (too much), those are the effects of long term exposure to poor quality air. Anyway, to help you better understand the meaning of eCO2 values, I will translate them  into percentage (%), by mapping the [400, 8192] interval to [0, 100] interval. This is done by applying an offset and then a scale, so the end result will be the eCO2 concentration in the air. The same goes for TVOC channel, and we also make sure to add scales for voltage and current too, in order to have the end results expressed in the right measurements units (millivolts for voltage, milliamps for current, etc.) Available standard attributes for IIO devices here [Documentation/ABI/testing/sysfs-bus-iio] and more about IIO channels here [https://dbaluta.github.io/iiosubsys.html#iiochannel]. 

Every channel definition generates a sysfs file that can be read to retrieve the data corresponding to that channel.
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

You can see that I used a bitmask called “info_mask_separate” to set up some properties for each channel (for example the eCO2 channel needed an offset to be applied so I set the offset bit to 1 into info_mask_separate mask). 

