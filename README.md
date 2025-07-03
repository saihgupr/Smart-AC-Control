# Smart AC Upgrade with ESP8266 & Home Assistant

One of my favorite DIY projects was turning a regular air conditioner into a smart one using a transistor and an ESP8266. The goal? Full remote control and climate automation without buying an expensive smart unit.

---

## DIY Smart AC with ESP8266 & Home Assistant: Step-by-Step Guide

### Step 1: Capture Your Remote's IR Codes

I won’t go deep into that here since there are plenty of guides online, but the basic idea is:

1. **Set up an IR receiver LED** (connected to your ESP8266 running ESPHome).
2. Put the ESP into "listening" mode and point your AC remote at it.
3. Press each button and log the output. I used the **NEC protocol**, but **RAW** might work depending on your remote.

### Step 2: Test the IR Transmission
Once you’ve got your codes, set up an **IR transmitting LED** to send them back.

- Use ESPHome’s `remote_transmitter` component
- Verify that your AC responds as if the remote was used

When the AC beeps and obeys your command — congrats, you've cloned the remote.

---

### Step 3: Wire into the AC’s Internal IR Receiver
Now for the fun part — **hardwiring your IR signal into the air conditioner** so it doesn’t rely on line-of-sight.

1. **Open your AC’s front panel**
   - Mine (a Frigidaire) had two screws on either side
2. **Swing the panel open** to reveal the control board
3. **Unscrew the control panel** to access the IR receiver solder points

You’ll need to identify:
- **VCC (likely 5V or 3.3V)** — powers the ESP
- **GND**
- **IR signal input pin** — where you’ll inject your ESP’s IR output

In my case, the IR receiver had **three pins**, and I had to test all of them to figure out which one carried the data.

> ⚠️ Tip: Get IR transmission working **through an LED first**, before wiring into the AC. It’s way easier to debug.

![Dashboard Preview](https://i.imgur.com/a983jT1.jpeg)
![Dashboard Preview](https://i.imgur.com/b2hh7qh.jpeg)
![Dashboard Preview](https://i.imgur.com/aGnhOht.jpeg)
![Dashboard Preview](https://i.imgur.com/8KQyNrL.jpeg)
![Dashboard Preview](https://i.imgur.com/s603CnE.jpeg)

---

### Step 4: Power the ESP from the AC (Optional)
If you want a clean setup, grab **5V and GND directly from the AC** board to power the Wemos D1 Mini.

> Note: If you're not 100% sure about the voltage, use a multimeter or add a voltage regulator.

I eventually soldered wires from the control board directly to my ESP and left the ESP sitting on top of the AC for easy access.

![Dashboard Preview](https://i.imgur.com/ZB0Q5ms.jpeg)
![Dashboard Preview](https://i.imgur.com/ZNSlAJa.jpeg)
---

### Step 5: Connect to ESPHome
At this point:
- You’ve captured IR codes
- Verified them by sending over an LED
- Hardwired your signal into the AC
- (Optionally) powered your ESP from the AC

Now configure your **ESPHome YAML** with `remote_transmitter` + `template` buttons. You’ll be able to send commands like:
- Power On
- Power Off
- Temp Up / Down
- Set Mode to Cool

![Dashboard Preview](https://i.imgur.com/LvmkjzH.png)

---

### Step 6: Create a Home Assistant Switch
Use a **template switch** in Home Assistant to wrap your ESPHome buttons:
- If your AC remote uses a **single power toggle**, you’ll need to track state manually.
  - Option 1: Use an **input_boolean** to fake the state
  - Option 2: Use a **power-sensing smart plug** to infer ON/OFF via a binary sensor

---

### Step 7: Add a Thermostat
If you have a nearby temperature sensor (like from the ESP or Zigbee), you can create a `generic_thermostat` entity in Home Assistant:
- Target temperature
- Auto on/off logic
- Feels like magic

![Dashboard Preview](https://i.imgur.com/ktx1YJJ.png)

---

## ESPHome Configuration

```yaml
remote_transmitter:
  pin: D2
  carrier_duty_percent: 50%

button:
  - platform: template
    name: Air Conditioner Cool Mode
    id: my_button_ac_cool_mode
    icon: "mdi:emoticon-outline"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xF609
          command_repeats: 1

  - platform: template
    name: Air Conditioner On
    id: my_button_ac_on
    icon: "mdi:emoticon-outline"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xEE11
          command_repeats: 1
      - delay: 3s
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xF609
          command_repeats: 1

  - platform: template
    name: Air Conditioner Off
    id: my_button_ac_off
    icon: "mdi:emoticon-outline"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xEE11
          command_repeats: 1

  - platform: template
    name: Air Conditioner Up
    id: my_button_ac_up
    icon: "mdi:emoticon-outline"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xF10E
          command_repeats: 1

  - platform: template
    name: Air Conditioner Down
    id: my_button_ac_down
    icon: "mdi:emoticon-outline"
    on_press:
      - remote_transmitter.transmit_nec:
          address: 0xF508
          command: 0xF20D
          command_repeats: 1
```

---

## Home Assistant Configuration

```yaml
climate:
  - platform: generic_thermostat
    name: Air Conditioner
    heater: switch.air_conditioner
    target_sensor: sensor.nodemcu_temperature
    precision: 0.5
    ac_mode: true
    cold_tolerance: 0.5
    hot_tolerance: 0.5
    min_cycle_duration:
      minutes: 1

switch:
  - platform: template
    switches:
      air_conditioner:
        value_template: "{{ states('binary_sensor.air_conditioner') }}"
        friendly_name: "Air Conditioner"
        icon_template: mdi:air-filter
        turn_on:
          - condition: not
            conditions:
              - condition: state
                entity_id: binary_sensor.air_conditioner
                state: 'on'
          - service: button.press
            data:
              entity_id: button.air_conditioner_on
        turn_off:
          - condition: state
            entity_id: binary_sensor.air_conditioner
            state: 'on'
          - service: button.press
            data:
              entity_id: button.air_conditioner_off
```

---

## Final Notes

If you made it this far — congrats, your dumb AC is now a **smart climate-controlled machine**. This is just the beginning — any IR device (TVs, fans, fireplaces) can be tamed in the same way.

> Don’t forget to share your own wiring diagram or code if you improve it. This project is weirdly satisfying, and way cheaper than buying a "smart" air conditioner.

---

## License

MIT License — feel free to copy, remix, or improve!
