# Walkie-Talkie Image Prompts

Nov 14, 2026 · @Chris Barlow

Five prompts for an image generator: one flat lay of every part with labels, three concepts for how the finished walkie could look, and a back view of the pocket pebble.

## How to use

Paste one prompt at a time. Each is self-contained, so the generator needs no other context.

- **Labels.** Image generators often garble small text. Prompt 1 asks for numbered tags and gives the key below it, so a wrong label does not matter. If your generator handles text well, use the "with names" line instead.
- **Consistency.** Every concept prompt repeats the same five features (big orange button, knob, small screen, speaker grille, PUSH / OPEN toggle switch), so the three results stay comparable.
- **Aspect ratio.** 16:9 for the parts layout, 4:5 for the concepts. Change the last line of each prompt if your tool sets this elsewhere.
- **The switch.** The PUSH / OPEN toggle was added after the first set of images was made, so the pictures in `images/` do not show it yet. Generators often garble the two small labels; the lever is what matters.
- **No antenna.** It works over Wi-Fi, so the prompts say so. Generators love adding one.

## Prompt 1: all parts laid out

```
Top-down flat lay product photograph of electronic hobby parts, neatly arranged in a grid (knolling style) on a matte light grey workbench mat. Soft even studio lighting, sharp focus, no hands, no tools, generous spacing between parts. Exactly thirteen items, each with a small white round numbered tag (1 to 13) placed beside it:

1. A small green ESP32-S3 microcontroller dev board, about 60 x 25 mm, with two USB-C ports, a silver metal radio shield and two rows of gold header pins.
2. A tiny round purple microphone breakout board, about 15 mm across, with one small silver MEMS microphone and six pins (INMP441).
3. A small blue amplifier breakout board, about 18 x 18 mm, with a green two-pin screw terminal (MAX98357A).
4. A round black 50 mm loudspeaker with a paper cone, metal frame, and two short red and black wires.
5. A large round orange arcade-style push button, 30 mm wide, with a black threaded body and nut.
6. A rotary encoder with a metal shaft on a small board, with a black ribbed knob beside it.
7. A small one-colour OLED display module, 1.3 inch, black glass with a blue board and four pins, showing plain white text.
8. A flat silver lithium polymer battery pouch, about 50 x 34 mm, with a red and black wire and small white plug.
9. A tiny USB-C battery charger board, about 25 x 17 mm, blue.
10. A single red-green-blue LED and a small black slide switch.
11. A small coil of thin multi-coloured hookup wire.
12. An empty matte black ABS plastic project box with its lid beside it, about 120 x 60 x 30 mm.
13. A small chrome toggle switch with a short metal bat-shaped lever, a threaded collar with a hex nut, and three solder lugs.

Clean technical catalogue look, realistic scale between parts, neutral colours. No extra components, no antenna, no brand logos. Aspect ratio 16:9.
```

**With names instead of numbers.** Swap the tag sentence for: "each with a small white printed label beside it reading its name in clean black sans-serif capitals", and use these names: ESP32-S3 BOARD, MICROPHONE, AMPLIFIER, SPEAKER, TALK BUTTON, CHANNEL KNOB, SCREEN, BATTERY, CHARGER, LED AND SWITCH, WIRE, CASE, MODE SWITCH.

**Key to the numbers.**

| No. | Part | Job |
| --- | --- | --- |
| 1 | ESP32-S3 dev board | The brain: Wi-Fi, voice coding, encryption |
| 2 | I2S microphone (INMP441) | Hears you |
| 3 | I2S amplifier (MAX98357A) | Drives the speaker |
| 4 | 4 ohm 3 W speaker | Loud enough for Grandad |
| 5 | Big push button | Hold to talk |
| 6 | Rotary encoder and knob | Picks the channel |
| 7 | One-colour OLED screen | Shows channel name and state |
| 8 | LiPo battery, 1000 mAh | Power off the dock |
| 9 | USB-C charger board | Charges the battery |
| 10 | Status LED and switch | Colour for talk and listen; on and off |
| 11 | Hookup wire | Joins it all up |
| 12 | Project box | The case |
| 13 | Toggle switch | Chooses push-to-talk or open channel |

## Prompts 2a to 2d: the built walkie

Three shapes for the same insides: a classic handheld, a bedside radio, and a pocket pebble.

### 2a. Classic handheld

The familiar walkie-talkie shape, so nobody needs it explained.

```
Studio product photograph of a simple homemade internet walkie-talkie, three-quarter front view, standing upright on a pale wooden table. A chunky rounded rectangular handheld in matte cream plastic, about 130 mm tall, 65 mm wide and 30 mm deep, sized for an elderly person's hand. Front face from top to bottom: a round perforated speaker grille filling the top third; a small one-colour OLED screen in the middle showing the word "CHRIS" in large white letters on black; one very large round orange push button, 30 mm wide, in the lower third. A small chrome toggle switch with a short metal lever on the right-hand side of the case, with the tiny engraved labels "PUSH" and "OPEN" either side of it. On the top edge, a single black ribbed rotary knob like a radio volume knob. A small status light glowing green beside the screen. A USB-C port on the bottom edge. No antenna, no keypad, no other buttons or switches, no brand logos. Friendly, sturdy, slightly retro design, like a well-made 3D printed gadget. Soft daylight, shallow depth of field, plain light background. Aspect ratio 4:5.
```

### 2b. Bedside radio

Sits on a table by the chair, always on charge. The button is on top where a hand lands on it.

```
Studio product photograph of a small homemade internet walkie-talkie designed as a tabletop radio, three-quarter front view, sitting on a wooden bedside table next to a reading lamp and a cup of tea. A squat wedge-shaped box in matte sage green plastic with rounded corners, about 120 mm wide, 80 mm deep and 70 mm tall, with a gently sloped front face. On the top surface: one very large round orange arcade-style push button, easy to press with a whole palm. On the sloped front face: a wide perforated speaker grille on the left, a small one-colour OLED screen on the right showing the word "CHRIS" in large white letters on black, and below the screen a large black ribbed rotary knob with a white pointer line. A small chrome toggle switch with a short metal lever on the front face below the speaker grille, with the tiny engraved labels "PUSH" and "OPEN" either side of it. A small status light glowing green next to the screen. A USB-C cable plugged into the back. No antenna, no keypad, no other buttons or switches, no brand logos. Warm, homely, sturdy design, like a modern take on a 1960s transistor radio. Soft warm indoor light, shallow depth of field. Aspect ratio 4:5.
```

### 2c. Pocket pebble

Small enough for a pocket or a lanyard. The knob is a ring around the button, so there are only two things to touch.

```
Studio product photograph of a small homemade internet walkie-talkie shaped like a smooth pebble, three-quarter front view, resting in an open adult hand for scale. A rounded oval puck in matte navy blue plastic, about 85 mm long, 65 mm wide and 28 mm thick, with a woven lanyard loop at the top. On the front face: one very large round orange push button in the centre, surrounded by a black ribbed rotating ring that turns to change channel; above the button a small one-colour OLED screen showing the word "CHRIS" in large white letters on black; below the button a row of small speaker holes. A small slide switch on the right edge with the tiny engraved labels "PUSH" and "OPEN". A small status light glowing green beside the screen. A USB-C port on the bottom edge. No antenna, no keypad, no other buttons or switches, no brand logos. Soft, friendly, minimal design, like a well-made 3D printed gadget. Soft daylight, plain light background, shallow depth of field. Aspect ratio 4:5.
```

### 2d. Pocket pebble, back view

The reverse of 2c, to show how it is carried: a sprung clip for a belt or shirt pocket, and the lanyard loop. Generate it straight after 2c so the colour and hand match.

```
Studio product photograph of the back of a small homemade internet walkie-talkie shaped like a smooth pebble, rear view, resting face down in an open adult hand for scale. A rounded oval puck in matte navy blue plastic, about 85 mm long, 65 mm wide and 28 mm thick, with a woven navy lanyard loop threaded through a small hole at the top. The back is plain and smooth except for one sturdy sprung belt clip in matching navy plastic running down the centre, hinged near the top with a visible small steel coil spring, with a ribbed grip at its lower tip. A small slide switch on the right edge. A USB-C port on the bottom edge. No screen, no buttons, no speaker holes and no labels on the back, no antenna, no brand logos. Soft, friendly, minimal design, like a well-made 3D printed gadget. Soft daylight, plain light background, shallow depth of field. Aspect ratio 4:5.
```

**Variations worth trying.** Add "exploded view showing the internal circuit boards, speaker and battery floating apart above the case" to any concept for a cutaway. Swap the screen word for "LISTENING" or "TALK" and the status light to red to show other states. Add "sitting in a matching charging dock" for the dock.
