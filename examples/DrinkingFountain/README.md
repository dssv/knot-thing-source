[<img src="http://knot.cesar.org.br/images/KNoT_logo_topo1.png" height="80">] (http://knot.cesar.org.br/) KNoT Network of Things
=============================================================================

#### _Documentation of Scale Application_

This tutorial explains how to hack a weight scale, model EB9322 of Camry Scales
® using a converter (ADC) for weight scales HX711.

Specifications of the Weight Scale
----------------

* Product Name: HEAVY DUTY GLASS SCALES
* Producer: Camry Scales
* Model: EB9322

Link for more details: [EB9322](http://www.camryscale.com/showroom/eb9322.html)

Similar model used in the application: [EB9321]
(http://www.camryscale.com/showroom/eb9321.html)

24-Bit Analog-to-Digital Converter (ADC) for Weigh Scales - HX711
----------------

HX711 is a precision 24-bit analog-to-digital converter (ADC) designed for
weigh scales and industrial control applications to interface directly with a
bridge sensor.

### Features of HX711 ###

*  Two selectable differential input channels
*   On-chip active low noise PGA with selectable gain of 32, 64 and 128
*   On-chip power supply regulator for load-cell and ADC analog power supply
*   On-chip oscillator requiring no external component with optional external
crystal
*   On-chip power-on-reset
*   Simple digital control and serial interface:
	pin-driven controls, no programming needed
*   Selectable 10SPS or 80SPS output data rate
*   Simultaneous 50 and 60Hz supply rejection
*   Current consumption including on-chip analog power supply regulator:
	normal operation < 1.5mA, power down < 1uA
*   Operation supply voltage range: 2.6 ~ 5.5V
*   Operation temperature range: -40 ~ +85°C
*   16 pin SOP-16 package

Link to the datasheet with more details about the board used:
[Datasheet](http://www.aviaic.com/uploadfile/hx711_brief_en.pdf)

### Library of HX711 ###

An Arduino library to interface the Avia Semiconductor HX711 24-Bit
Analog-to-Digital Converter (ADC) for Weight Scales.

Link to download: [Git](https://github.com/bogde/HX711)

#### Advantages of this library ####

Although other libraries exist, I needed a slightly different approach, so
here's how my library is different than others:

1. It provides a tare() function, which "resets" the scale to 0. Many other
implementations calculate the tare weight when the ADC is initialized only. I
needed a way to be able to set the tare weight at any time. Use case: place an
empty container on the scale, call tare() to reset the readings to 0, fill the
container and get the weight of the content.

2. It provides a power_down() function, to put the ADC into a low power mode.
According to the datasheet, "When PD_SCK pin changes from low to high and stays
at high for longer than 60μs, HX711 enters power down mode". Use case: battery
powered scales. Accordingly, there is a power_up() function to get the chip out
of the low power mode.

3. It has a set_gain(byte gain) function that allows you to set the gain factor
and select the channel. According to the datasheet, "Channel A can be programmed
with a gain of 128 or 64, corresponding to a full-scale differential input
voltage of ±20mV or ±40mV respectively, when a 5V supply is connected to AVDD
analog power supply pin. Channel B has a fixed gain of 32.". The same function
is used to select the channel A or channel B, by passing 128 or 64 for channel
A, or 32 for channel B as the parameter. The default value is 128, which means
"channel A with a gain factor of 128", so one can simply call set_gain(). Also,
the function is called from the constructor.

4. The constructor has an extra parameter "gain" that allows you to set the gain
factor and channel. The constructor calls the "set_gain" function mentioned
above.

5. The "get_value" and "get_units" functions can receive an extra parameter
"times", and they will return the average of multiple readings instead of a
single reading.

How to Hack the Weight Scale
----------------

Opening the back of the balance, you can find the controller board.
In order to capture the data read by the scale using the pins of the HX711 chip,
you need to solder the pins of the HX711 with those of the balance controller
board as follows:

| Pins in HX711 |             Reference             |Pins in PCB Weight Scale|
|---------------|-----------------------------------|------------------------|
|        E-     |                GND                |    Middle left down    |
|        E+     |AVDD Analog Power Supply Pin (5V)  |    Middle right up     |
|        A-     |      Input Analogic Negative      |    Middle right down   |
|        A+     |      Input Analogic Positive      |    Middle left up      |

Doing this now you are able to read the values and use them in your application.

Application DrinkingFountain.ino
----------------

Setting the analog inputs of the HX711 chip.
```c++
HX711 scale(A3, A2);
```

Setting the pin 3 or microcontroler as input of the button of tare
```c++
pinMode(BUTTON_PIN, INPUT);
```

Wakes up the HX711 chip.
```c++
scale.power_up();
```

This part of the function `get_weight(times)` converts the average of values
read from the HX711 chip to a readable value.
```c++
mes = scale.get_value(times);
a = ref_w/(k2 - k1);
b = (-1)*ref_w*k1/(k2-k1);
raw_kg = a*mes + b;
```

Logic to verify if the button to tare is pressed, if yes tare the scale with
the average of the 20 last values read.
```c++
if ((button_value_current == HIGH) && (button_value_previous == LOW)) {
    offset = get_weight(20);
}
```

Save offset on EEPROM
```c++
EEPROM.put(OFFSET_ADDR, offset);
```

The code below calculate the average of the last values reads from HX711 chip
and subtract this with the value read of tare.
```c++
kg = get_weight(TIMES_READING) - offset;
```

The function `remove_noise(value)` uses logic to filter the read data, avoiding
abrupt data variation.
```c++
 if(value > (previous_value + BOUNCE_RANGE) || value < (previous_value - BOUNCE_RANGE)) {
        previous_value = value;
}
```

The bounce range is defined with a value of 200
```
#define BOUNCE_RANGE 200
```

Logic to do the microcontroler read only on interval
```c++
currentMillis = millis();
if(currentMillis - previousMillis >= READ_INTERVAL){
	previousMillis = currentMillis;
	kg = get_weight(TIMES_READING) - offset;
}
```

The read interval is difined as 1 second
```
#define READ_INTERVAL      	1000
```

Converting from kg to g and calling the function `remove_noise(value)`
```c++
 *val_int = remove_noise(kg*1000);
```
