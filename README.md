# ESP8266 Temperature Sensor and Display
This is a project to read the temperature and air pressure outside, and display the values inside on a display, like this:
![Temperature and Pressure Display](/images/Display%20Module%20Complete.png "Temperature and Pressure Display")

Both the inside and the outside modules use a D1 mini ESP8266.  The outside sensor also includes an additional sensor to read the temperature and the air pressure.  

I've used the BMP180 Temperature and Barometric Pressure Sensor Module.

For the inside display, I used two 0.96" I2C 128X64 OLED Display - the SSD1306.  While rather small, it is the perfect size to show the temperature and the pressure. I use one diusplay for each value.  I liked the fixed yellow section of the top, which I used to display the words 'TEMPERATURE' and 'PRESSURE' accordingly, and then the blue section of the display for the values.  I think it is a 25%/75% split.

I designed custom cases for each which I printed on my Bambu Labs X1 Carbon printer. I included a graphic on the outside case for no other reason than to justify my dual AMS setup for the printer.  Definitly not required.

I used ESPHome and Home Assistant - More on this later.

My general requirements for the project were:
Read the termperature and air pressure outside
Wirelessly send that sesnor data to the inside
Display the sensor data inside on a display
Battery power supply is not required, and a simple USB connection to power is all I needed.


Alright, let's break the project down into consumable chunks.

Let's see, we need:
- Sensors
- Displays
- ESP8266
- Platform
- Cases

### Let's start with the sensors.
I knew I wanted to measure the outside temperature so that was easy.  I also wanted to measure the barometric pressure, because well my old knee joints like to look at the display and exclaim "I knew the pressure was dropping, I could feel it in my bones!" and thus justify the sensor.

The ESP8266 has an nice I2C interface so that seemed like a good option, as I had seen this as a popular choice for sensors these days.

I went with the BMP180 sensor, and into my Amazon cart it went.

![BMP180 sensor](/images/BMP180%20Sensor.png "BMP180 sensor")

### Displays
Ok, so for displays, there are a lot of options, and it turns out support for these is great for the ESP8266.  My requirements were:
I2C
Available, easy to use drivers
Support for multiple displays

I found the 0.96" OLED and it perfectly fit my needs.  It was about the right size, allowed for dual addressing, was I2C, and super cheap - perfect.  Driver support turned out to be super easy, and again, the silent click of 'Add to Cart' filled the air.

![OLE display](/images/OLED%20Dsiplay.png "OLED display")

### ESP8266
I used the ESP8266 because, well I had a bunch, and they are super fun, small, and very capable. Oh also, it has built in wireless comms, which is a requirement for my project. Anything more would be overkill for this kind of project.

![ESP8266](/images/ESP8266.png "ESP8266")

### Platform
Ok, so at first my thought was to use ESPNOW for a peer to peer conenction between the two devices. A colleague of mine suggested I use ESPHome.  I already have a Home Assistant setup and in fact I already have other ESP8266's with temperature and air quality sensors to monitor my 3D printing, which were done in ESPHome, feeding HA, so I thought that was a great idea, even though it wasn't a requirement for this project.

Anyways, so I did exactly that.  First, I setup the outside sensor module in ESPHome, and fed that into HA, and I could monitor the data in HA. Cool.

Then I configured the second, inside display module in ESPHome, and to my surprise, using the display and reading sensor values from HA was too easy. I was amazed how much things have advanced since I first started getting into microsontroller development.  Very cool.

So, in the end, I dropped the p2p thought and went ESPHome end to end.  This worked out quite well, and am very happy with the end result.  The code for both is included.

This is not intended to be a Home Assistant or ESPHome tutorial.  I will assume you know how to install and configure ESPHome and configure your device accordingly.

#### Outside Sensor Module Code
Once you have the boiler plate ESPHome code in place, the first thing you need to do is configure I2C.

    i2c:
      ## I²C Port - For Temp/Pressure sensor
      sda: GPIO5
      scl: GPIO4
      scan: true

Next you'll need to configure your sensor, in our case the BMP180 I2C device.

    sensor:
      ## Temp/Pressure Sensor
      - platform: bmp085
        temperature:
            name: $name Temperature
            id: bme_temp
        pressure:
            name:  $name Pressure
            id: bme_press
        update_interval: 1s

I found the sensor needed to be corrected.  I didn't have a precise thermometer I could use so this is a rough graph of what I did in the filter for correction.

      filters:
      - calibrate_linear:
         method: exact
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 6.5 -> 2.5
          - 11.8 -> 5.8
          - 24.5 -> 20.5

Of course you may or may not need this, or your values would be different.

#### Inside Display Module Code
To read values from Home Assistant you configure sensors, and by setting the platform to Home Assistant, ESPHome knows to retrieve the sensor data from your HA API accordingly.

First, as before, configure your I2C.  You'll note I have a line here to increase the frequency of the I2C bus for the display. This wasn't a functional requirement, but the display takes a bit of time to update on the I2C bus, and the logs were being filled up with warnings about this, so increassing the frequesncy just removes those for me.

    i2c:
      sda: GPIO04
      scl: GPIO05
      scan: false
      frequency: 400kHz

To setup your sensors all you need is your entity id's.

    sensor:
      - platform: homeassistant
      id: outside_temperature
      entity_id: "sensor.outside_environment_sensor_outside_environment_sensors_temperature"

      - platform: homeassistant
        id: outside_pressure
        entity_id: "sensor.outside_environment_sensor_outside_environment_sensors_pressure"

I added a sliding window averaging routine so the temperature was smoothed out, aberaging over 10 readings - I didn't need to see it flicker between 13.4 and 13.5 as it transitioned.  Luckily ESPHome has such a mechanism built in using filters.

    filters:
    - sliding_window_moving_average:
      window_size: 10
      send_every: 1

I imported a few fonts for displaying the values. Be sure to upload these to your ESPHome folder accordingly.
`config/ESPHome/fonts`

    font:
      - file: 'fonts/Courier-New.ttf'
      id: font1
      size: 17

    - file: 'fonts/BebasNeue-Regular.ttf'
      id: font2
      size: 50

Finally, the display is done via a Lambda.  Yay, some real code!

    display:
    # Temperature DIsplay
    - platform: ssd1306_i2c
        model: SSD1306 128x64
        address: 0x3C
        update_interval: 5s
        lambda: |-
        // Print the header
        it.printf(64, 0, id(font1), TextAlign::TOP_CENTER, "TEMPERATURE");

        // Print outside temperature (from homeassistant sensor)
        if (id(outside_temperature).has_state()) {
            it.printf(64, 16, id(font2), TextAlign::TOP_CENTER , "%.1f°", id(outside_temperature).state);
        }

    # Air Pressure Display
    - platform: ssd1306_i2c
        model: SSD1306 128x64
        address: 0x3D
        update_interval: 5s
        lambda: |-
        // Print the header
        it.printf(64, 0, id(font1), TextAlign::TOP_CENTER, "PRESSURE");

        // Print outside pressure (from homeassistant sensor)
        if (id(outside_pressure).has_state()) {
            int intValue {id(outside_pressure).state};
            it.printf(85, 16, id(font2), TextAlign::TOP_RIGHT , "%d", intValue);
            it.printf(90, 46, id(font1), TextAlign::TOP_LEFT, "hPa");
        }

The ESPHome configuration YAML files are included for you.

### Cases
Finally, let's talk about the cases.  Like every electronics project we have all done, the end result is a bunch of wires and a breadboard.  Obviously for this project I needed something more, as I needed to convince my wife to let me display these devices in plain sight. The outside module plugged into an outlet and hung nearby and the inside module was plugged in by the tv, on the cabinet for everyone to see, so yeah, you can see where I am going with this.

Into Fusion360 I went.  I spent way mroe time designing the custom cases for this than I am willing to admint, and the number of prototypes I printed for test fitting the boards and the case lid, etc filled my small office garbage can. Ryan:1, Environment:0

In the end, I had two awesome cases that help the boards inside quite well, allowing for an easy external USB connection.  The case lid is help down with four screws, which totally could have been accomplished with a simple click together system, but I didn't realize this until I started screwing in four M3 bolts, thinking this is kinda not needed. Lessons learned (?).

I've included the 3d models if you would like to print them.

#### Outside Module
For the outside module case, the sensor board screws down and is held in place by a set of shoulder offsets to keep it off of the bottom to avoid heat creep. The ESP8266 i slikewise held down with a screw and a simple printed clamp, again sitting on shoulders to get it at the correct height for the USB connection through the case.  
I added slots onto three sides to ensure a sufficient airflow for the sensors.

Here is the case test fitting the two boards.  Seems like things foit pretty well together.

![Test fitting the boards](/images/Sensor%20Module%20Inside%20Test%20Fit.png "Test fitting the boards")

Here it is wired up and boards mounted - yes I know, focus is hard.
![Inside wired up](/images/Sensor%20Module%20Inside%20Complete.png "Inside wired up")

And finally, here it is all together and mounted outside.
![Mounted outside](/images/Sensor%20Module%20Mounted.png "Mounted outside")

#### Inside Module
The inside module case was quite similar, though I ended up splitting the case into three parts, instead of the classis two parts (case bottom and case lid) as I did for the outside case. The third part was the front, where the displays are held.  I wanted to prototype these so knew I will have to print several out, adjusting along the way, and I wanted a clean finish on the front without having to use supports to print the case out. This worked out quite well, and once the displays were mounted using a practice common in the 70's and 80's (plastic posts through the circuit board mounting holes, and flower out with the touch of a soldering iron) I glued the front to the rest of the case and we were off to the races.  
Like the outside module, I included slots for a biut of airflow for the baords.

Because the visual side of this case was the front - the side with the displays, I opted not to include any graphics on the case.

![Prototyping case front 1](/images/Display%20Module%20Front%20Case%20Prototype%20Test%20Fit%20Back.png "Prototyping case front 1")

Here is one of the prototypes test fitting the front of the display.
![Prototyping case front 2](/images/Display%20Module%20Front%20Case%20Prototype%20Test%20Fit%20Front.png "Prototyping case front 2")

And with the inside all wired up.  You'll note the wiring is super simple for this, because of the different addressing for the I2C bus for the rtwo displays, I simply needed to daisy chain the displays up.
![Display board wired up](/images/Display%20Module%20Inside%20Complete.png "Display board wired up")

And here is is complete and displaying the outside temperature and the barometric pressure.
![Display module complete](images/Display%20Module%20Complete.png "Display module complete")

Well, I think that is everything. The system works great, I am overall pretty happy with the results.  Hope you enjoy!


### References:

ACEIRMC 6pcs GY-68 BMP180 Temperature Barometric Pressure Sensor Module  
https://www.amazon.ca/gp/product/B091GWXM8D

KeeYees Mini Development Board for ESP8266 4M Bytes  
https://www.amazon.ca/gp/product/B092ZMSVX5

IZOKEE 0.96'' I2C IIC 12864 128X64 Pixel OLED LCD Display Shield Board Module SSD1306  
https://www.amazon.ca/gp/product/B076PNP2VD