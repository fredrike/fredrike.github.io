---
title: Building a environment sensor with a display using D1-mini and ESPHome
last_modified_at: 2025-08-14 10:29:30 +0200
---

A fun exercise to getting to know some hardware is to start with ESP8266 and in particular Wemos D1-mini. In this post I show how to build an environment sensor (with temperature and pressure) that shows the result on a display.

Bill of hardware:

* [D1 Mini NodeMcu with ESP8266-12F](https://www.amazon.se/-/en/dp/B0D8W8N2DP) 
* [BMP280 Temperature+Pressure Sensor](https://www.amazon.se/-/en/dp/B07D8TPVVY)
* [OLED 128 x 64 pixels I2C](https://www.amazon.se/-/en/dp/B078J78R45)

As both BMP280 and the OLED display is using I2C for communication, the wiring is quite straight forward as seen here:

<div style="position: relative; width: 100%; padding-top: calc(max(56.25%, 400px));">
  <iframe src="https://app.cirkitdesigner.com/project/48e9be7c-39be-44cf-861c-c53e05d0b851?view=interactive_preview" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; border: none;"></iframe>
</div>

Just make sure to connect SDA to GPIO4 and SDC to GPIO5 and provide GND and Vcc.

To keep programming to a minimum we are using [ESPHome](https://esphome.io/), a framework that instead of coding uses `yaml` config. Both the [BMP280 sensor](https://esphome.io/components/sensor/bmp280.html) and the [OLED display](https://esphome.io/components/display/ssd1306.html) have native support in ESPHome.

SPI setup:

```yaml
i2c:
  sda: GPIO4
  scl: GPIO5
  scan: true
```

For the sensor the example config looks like this:

```yaml
sensor:
  - platform: bmp280_i2c
    temperature:
      id: bmp280_temperature
      name: "Temperature"
      oversampling: 16x
    pressure:
      id: bmp280_pressure
      name: "Pressure"
    address: 0x77
    update_interval: 60s
```

For the display we first must define a font:

```yaml
font:
  - file:
      type: gfonts
      family: Roboto
      weight: 900
    id: roboto_16
    size: 16
```

And use it like this:

```yaml
display:
  - platform: ssd1306_i2c
    model: "SH1106 128x64"
    address: 0x3C
    lambda: |-
      it.print(0, 0, id(roboto_16), "Hello World!");
```

What we like to do is update the display whenever our sensor updates..

<details markdown="1">
  <summary>yaml for updating display dynamically</summary>

```yaml
display:
  - platform: ssd1306_i2c
    model: "SH1106 128x64"
    address: 0x3C
    lambda: |-
      it.printf(0, 12, id(roboto_16), TextAlign::TOP_LEFT, "Temperature");

      // Print temperature
      if (id(bmp280_temperature).has_state()) {
        it.printf(
          127,
          23,
          id(roboto_16),
          TextAlign::TOP_RIGHT,
          "%.1f",
          id(bmp280_temperature).state
        );
      }

      // Print pressure
      if (id(bmp280_pressure).has_state()) {
        it.printf(
            127,
            60,
            id(roboto_16),
            TextAlign::BASELINE_RIGHT,
            "%.1f",
            id(bmp280_pressure).state
        );
      }
```

</details>

Final code: [ESPHome.yaml](https://gist.github.com/fredrike/6f230d2e828717db8960e6e7e9e4dbf8)
