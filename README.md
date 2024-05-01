# ESPHome Altitude Azimuth Sun Tracker
 Inspired by the wonderful Horizon card for HACS, and the multi-cog designs for Alt-Azimuth telescope mounts and Tellurions (Earth-Sun-Moon animated toys), this project builds a real-life 3D Horizon Card to track the Sun using ESPHome and an Altitude-Azimuth mount.

Sun Altitude-Azimuth Tracking with SG90 servos and ESPHome
===============================================================
by Phil Pownall

### Ingredients
- An ESP32 module
- A 608ZZ bearing
- Two SG90 or MG90 180-degree servo motors
- A 3D-printed mount consisting of a base, a cog, a turntable, and an arm
- An SSD1306 128x64 OLED display

### Optional ingredients
- An ESP32 Expansion board also from AliExpress to supply power and additional pins
- A Sun disk (e.g. a ping-pong ball, or a token) mounted on the servo arm
- An LED for inside the Sun ball for visual impact

### Derivative ingredients
- An ESP32-based camera module e.g. the ESP-CAM module from AliExpress.
- A smartphone telephoto lens and a mini garbage can to create your own observatory
- A compass rose for the base to show the azimuth
- A protractor to add to the platform to show the altitude


The Mount
---------

The mount was designed in TinkerCAD, as it has models for SG90 servos and for 608ZZ bearings (one is used in this design).   STL files for the design are available in Tinkercad thing 
[AltAzimuthMount](//www.tinkercad.com/things/iezULSBmK4e-altitude-azimuth-mount-for-espcam)

[AltAz2_2](AltAz2_2.png)

The mount design is minimalist: the 1X cog is part of the platform with the Altitude servo which sits inside the 608ZZ bearing in the base, and the 2X cog sits on the Azimuth SG90 servo.  This provides a 1:2 gear ratio for the Azimuth so that we can cover a full 360 degrees.  A Sun or Moon disk is attached to the servo arm, or an ESP-CAM sits inside a case attached to the servo arm.

Since we do not have a curved display to show the Sunrise/Sunset values, we use an SSD1306 OLED display showing 4 pages: one for the Sunrise time, two for the current Sun Altitude and Azimuth, and one for the Sunset time.  Attractive mdi icons for the Sunrise/Sun/Sunset are also shown on the display.


The ESPHome code
================

The ESPHome yaml code consists of the following sections:
- Number template and servo definitions for two servos: the Azimuth servo and the Altitude servo.
- The Sun platform, providing azimuth and altitude (Elevation) sensors, and text sensors for Sunrise and Sunset.
- The Display code to show information on the attached display.
- An *Interval:* automation to update the servo number templates from the Sun platform sensors.

Number Templates
----------------

### Example Definition of number templates to drive the Azimuth and Altitude servos

```yaml
number:
  # Define a number template to control servo 1
  # servo 1 is azimuth: 0 to 360: map to -1 to +1
  - platform: template
    name: Azimuth Number
    id: number_azimuth
    optimistic: True
    mode: slider
    min_value: 0
    initial_value: 180
    max_value: 360
    step: 0.1
    set_action:
      then:
        - servo.write:
            id: servo_azimuth
            level: !lambda 'return ((x / 360.0)* 2.0) - 1.0;'

  # Define a number template to control servo 2
  # servo 2 is altitude: 0 to 90: map to 0 to -1
  - platform: template
    name: Altitude Number
    id: number_altitude
    optimistic: true
    mode: slider
    min_value: 0
    initial_value: 0
    max_value: 90
    step: 0.1
    set_action:
      then:
        - servo.write:
            id: servo_altitude
            level: !lambda 'return -x / 90.0;'
```

Note the modification of the usual servo control number definitions to provide a 1:1 mapping between the Sun Azimuth and Elevation (Altitude) sensors and the servo control numbers.  The Azimuth servo number template thus varies from 0 to 360 (because we have a 1:2 gear cog attached to it), and the Altitude servo number template varies from 0 to 90.  We track the Sun if it is above the horizon, so the position of the platform will swing back around to the Sunrise position every morning.

Servo Definitions
-----------------

The servos are defined as usual with two outputs, and two servo definitions:

### Example Definition of outputs and servos for Azimuth and Altitude

```yaml
# Alt-Azimuth
substitutions:
  # pin assignments
  servo_azimuth: GPIO19
  servo_altitude: GPIO21

output:
  # Servo output 1   
  - platform: ledc
    pin: ${servo_azimuth}
    id: ledc_1
    channel: 0
    frequency: 50 Hz

  # Servo output 2
  - platform: ledc
    pin: ${servo_altitude}
    id: ledc_2
    channel: 1
    frequency: 50 Hz

servo:
    # servo 1
  - id: servo_azimuth
    output: ledc_1
    auto_detach_time: 1s
    transition_length: 4s

    # servo 2
  - id: servo_altitude
    output: ledc_2
    auto_detach_time: 1s
    transition_length: 4s
```


The Sun Integration, Sensors and Text sensors
---------------------------------------------

The Sun platform in ESPHome provides us with the Altitude (Elevation) and Azimuth sensors for the Sun at our location.  The sensors are defined as follows:

### Example Definition of sensors for Sun Azimuth and Elevation

```yaml
# Track the sun for this location
sun:
  latitude: 44.2333°
  longitude: -76.4810°

# At least one time source is required
time:
  - platform: homeassistant

sensor:
  - platform: sun
    name: Sun Altitude
    id: sun_altitude
    type: elevation

  - platform: sun
    name: Sun Azimuth
    id: sun_azimuth
    type: azimuth

```

We also require the sunrise and sunset times to show on the display.  These are provided by the Sun platform text sensors.

### Example Definition of text sensors for Sunrise and Sunset times

```yaml
text_sensor:
  - platform: sun
    name: Sun Next Sunrise
    id: sunrise_text
    type: sunrise
    format: "%I:%M %p"

  - platform: sun
    name: Sun Next Sunset
    id: sunset_text
    type: sunset
    format: "%I:%M %p"
```

The format definition for the text sensors is used to display the Sunrise and Sunset time strings as they are shown on the Home Assistant Horizon card (12-hour format with AM and PM).

Display Definition
------------------

The SSD1306 OLED display uses two fonts: Arial for the text, and the mdi font for the Sunrise/Sun/Sunset glyphs:

### Example fonts Definition

```yaml
font:
  # some fonts for the display
  - file: 'fonts/arial.ttf'
    id: font1
    size: 20

  - file: 'fonts/materialdesignicons-webfont.ttf'
    id: icons
    size: 44
    glyphs: [
        "󰖙", # sun
        "󱦥", # sun-compass
        "󱬨", # sun-angle-outline
        "󰖜", # sunrise
        "󰖚" # weather-sunset
     ]
```

The display definition is 4 pages, quite similar to the text sections of the horizon card, with the addition of the glyphs:

### Example Display Definition for SSD1306

```yaml
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    reset_pin: GPIO25 # an unused pin
    address: 0x3C
    id: my_display
    flip_x: True
    flip_y: True
    pages:
      - id: page1
        lambda: |-
          it.print(90, 5, id(icons), "󰖜");
          it.print(15, 24, id(font1), "Sunrise");
          it.printf(5, 44, id(font1), "%s", id(sunrise_text).state.c_str());
      - id: page2
        lambda: |-
          it.print(90, 0, id(icons), "󱬨");
          it.print(10, 24, id(font1), "Altitude");
          it.printf(10, 44, id(font1), "%.0f°", id(sun_altitude).state);
      - id: page3
        lambda: |-
          it.print(90, 0, id(icons), "󱦥");
          it.print(10, 24, id(font1), "Azimuth");
          it.printf(10, 44, id(font1), "%.0f°", id(sun_azimuth).state);
      - id: page4
        lambda: |-
          it.print(90, 5, id(icons), "󰖚");
          it.print(15, 24, id(font1), "Sunset");
          it.printf(5, 44, id(font1), "%s", id(sunset_text).state.c_str());
```


The Interval Automation
-----------------------

The ESPHome *Interval:* section is used to initiate the update of the servo altitude and azimuth values from the Sun platform azimuth and elevation values using an *if:* condition to check that the sun is above the horizon.

### Example Interval Automation Definition for Azimuth and Altitude update

```yaml
interval:
  # update the display page every 20 seconds
  - interval: 20s
    then:
      - display.page.show_next: my_display
      - component.update: my_display
  
  # update the servo motors every 5 minutes
  - interval: 300s
    then:
      - if:
          condition:
            - sun.is_above_horizon:
          then:
            # - logger.log: Sun is above horizon!
            - number.set:
                id: number_azimuth
                value: !lambda 'return id(sun_azimuth).state ;'
            - number.set:
                id: number_altitude
                value: !lambda 'return id(sun_altitude).state ;'
```

And that's it!  Now you can spend hours watching the sun and moon models move.  Next up: integrating two Sun and Moon trackers into a single object to provide a full Earth-Moon-Sun Tellurion.

[3DHorizoncard](3DHorizonCard.jpg)
[AltAz2](AltAz2.jpg)
