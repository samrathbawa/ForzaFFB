# forza_ffb

Force feedback for Forza, built from the game's telemetry.

Forza's **Data Out** feature streams car physics over UDP. forza_ffb listens to that stream (default `127.0.0.1:2066`), turns the physics into a steering force, and sends the force to your wheel. You pick one of three outputs:

- **`ffbwheel`** drives a real wheel motor through SDL_Haptic. I built it for a MOZA R3, and any DirectInput FFB wheel should work.
- **`vjoy`** writes the effect channels to vJoy axes, so Joystick Gremlin, SimHub or a DIY device can read them.
- **`console`** prints the channels in your terminal. Use it to check that data arrives and to tune. It runs on any OS.

## Read this before you plug in a wheel

**Forza sends no force-feedback signal.** Data Out carries slip angles, lateral G, surface rumble, suspension travel and wheel speeds. forza_ffb computes a force from those numbers: cornering load plus tyre self-aligning torque, road texture, kerb hits, and a lighter wheel when the front tyres wash out. The force in your hands comes from this tool's math, so expect it to feel different from the game's built-in FFB.

**I have tested one setup: a MOZA R3 base with the MOZA ES wheel.** Other bases can expose different effects, gains and force directions. You run this at your own risk. On a new wheel, start with a low `--wheel-gain`, keep a hand near the wheel or the base's power switch, and drive slow laps until you trust the feel. The authors accept no liability for damage or injury (see the MIT license).

---

## Supported games

I have tested Forza Horizon 6 and nothing else. The other titles below use the same Data Out format, or one the parser detects by packet length, so they should work. Treat them as untested and tune with care.

| Game | Data Out | Packet format | Status |
|------|----------|---------------|--------|
| **Forza Horizon 6** | Yes | Horizon, 324 B | **Tested** |
| Forza Horizon 5 | Yes | Horizon, 324 B | Untested, same format as FH6 |
| Forza Horizon 4 | Yes | Horizon, 324 B | Untested, same format as FH6 |
| Forza Motorsport (2023) | Yes | Car Dash, 331 B | Untested, detected by length |
| Forza Motorsport 7 | Yes | Car Dash, 311 B | Untested, detected by length |
| Forza Horizon 3 and older | No | n/a | Can't work |

Turn 10 added Data Out in Forza Motorsport 7 (2017), and FH4 was the first Horizon game to ship it. FH3 and earlier send no telemetry on any port, so forza_ffb has nothing to read.

---

## How it works

```
Forza game        ──UDP packets──▶  TelemetryListener ──▶  FFBEngine ──▶  Output backend
(Data Out, :2066)                    (parse, autodetect)    (physics→force)  ffbwheel / vjoy / console
```

The parser picks the packet format from its length: Sled 232, FM7 311, Horizon 324 or FM2023 331 bytes. It reads each field from one table, computes the offsets at import and asserts the sizes. If an offset is wrong, you get an error at startup before any bad value reaches your wheel.

Forza streams while you drive. Menus, pause screens and replays send nothing. When the packets stop, forza_ffb outputs neutral force after `stale_timeout_s` and the wheel goes slack.

---

## Effect channels

| Channel | Range | Derived from | Feel |
|---------|-------|--------------|------|
| `steer_force` | −1…+1 | lateral G + front slip angle, gated by speed, reduced when the fronts lose grip | main wheel torque |
| `g_lat` | −1…+1 | `AccelerationX` | cornering G |
| `g_long` | −1…+1 | `AccelerationZ` | accel / brake G |
| `road_texture` | 0…1 | front `SurfaceRumble` | road roughness, fine vibration |
| `kerb` | 0…1 | sudden `SuspensionTravelMeters` changes + rumble strips | kerb and bump jolts |
| `understeer` | 0…1 | front `TireCombinedSlip` past the grip limit | front washing out |
| `oversteer` | 0…1 | rear `TireCombinedSlip` past the grip limit | rear sliding or wheelspin |

`ffbwheel` turns `steer_force` into constant motor torque and mixes `road_texture` and `kerb` into a sine vibration. `vjoy` maps each channel to an axis, and you can change the mapping in the config.

---

## Install

You need **Python 3.8+**. The core (parsing and force synthesis), the console backend and the test suite use the standard library alone.

```bash
pip install -e .                 # core + console backend
pip install -e .[ffbwheel]       # + real FFB to a physical wheel (pysdl2 + bundled SDL2)
pip install -e .[vjoy]           # + vJoy axis output (Windows + vJoy driver)
```

The install gives you a `forza_ffb` command, and `python -m forza_ffb` works from any folder. To run without installing, `set PYTHONPATH=src` and then `python -m forza_ffb ...`.

| Backend | Extra packages | Notes |
|---------|----------------|-------|
| `console` | none | any OS |
| `ffbwheel` | `pysdl2` + `pysdl2-dll` | Windows. `pysdl2-dll` ships SDL2.dll for you |
| `vjoy` | `pyvjoy` + vJoy driver | Windows. Enable a device in *Configure vJoy* |

---

## Quick start

**1. Turn on Data Out in the game.** Go to Settings → HUD & Gameplay → **Data Out** and set
`Data Out = ON`, `IP = 127.0.0.1` (or the IP of the PC running forza_ffb), and `Port = 2066` (or whatever you pass to `--port`).

**2. Check that data arrives, using the console backend:**
```bat
python -m forza_ffb --backend console --port 2066 -v
```
Start driving. You should see channel values updating and a steering meter like `steer[--##--]`. No game handy? Run `python tools/fake_forza_sender.py --scenario sweep --port 2066` in a second terminal to send synthetic packets.

**3. Send the force to your wheel (a MOZA R3 in this example):**
```bat
python -m forza_ffb --list-devices
REM ->  [0] MOZA R3 Racing Wheel  (FFB-capable)
python -m forza_ffb --backend ffbwheel --device-name "MOZA" --port 2066
```

Press **`Ctrl+C`** to stop. forza_ffb zeroes the force and releases the wheel on exit.

---

## Output backends

### `ffbwheel`: real force feedback

forza_ffb sends `steer_force` as an SDL_Haptic **constant-force** effect (on Windows, SDL drives DirectInput underneath) and can add a **sine** vibration from `road_texture` and `kerb`. It picks the first device whose name contains `device_name_match` (default `"moza"`), then falls back to the first FFB-capable device. Pass `--device-index` to choose one yourself.

> I have tested this backend on a **MOZA R3 + MOZA ES** and no other hardware. See the warning at the top before you try another wheel.

> **One app at a time can drive a wheel's FFB.** forza_ffb takes the wheel over, so set the game's wheel FFB to 0 and/or turn off FFB in MOZA Pit House's Horizon compatibility mode. If you leave both on, the game and forza_ffb fight over the motor.
>
> **Centering spring:** forza_ffb adds no spring and turns off the DirectInput autocenter. On a MOZA, Pit House owns the centering spring. For a raw feel, set `Spring`, `Damper`, `Friction` and `Inertia` to 0 in Pit House.

### `vjoy`: effect channels as joystick axes

forza_ffb writes each channel to a vJoy axis. The defaults are X=steer_force, Y=g_long, Z=road_texture, Rx=kerb, Ry=understeer and Rz=oversteer, and you can remap them with `output.vjoy.axis_map`. vJoy creates a virtual **input** device, so an axis can't move a wheel motor by itself. Point Joystick Gremlin, SimHub or a DIY device at it.

### `console`: print the channels

Prints one of every `output.console.every` updates with an ASCII meter. It runs on any OS. Use it to confirm data flow and rough in your tuning before you switch to the wheel.

---

## CLI reference

```
python -m forza_ffb [options]      (or: forza_ffb [options] after install)
```

| Flag | Type | Applies to | Description |
|------|------|------------|-------------|
| `--config PATH` | path | all | JSON config file, deep-merged over the built-in defaults |
| `--ip IP` | str | all | Listen IP (default `127.0.0.1`) |
| `--port PORT` | int | all | Listen UDP port (default `2066`, must match the game's Data Out port) |
| `--backend NAME` | choice | all | `console`, `vjoy`, `ffbwheel` (aliases `wheel`/`moza`/`sdl`) or `null` |
| `--device-id N` | int | vjoy | vJoy device id (default `1`) |
| `--device-index N` | int | ffbwheel | Wheel index from `--list-devices` (`-1` = auto) |
| `--device-name STR` | str | ffbwheel | Match the wheel by a name substring, e.g. `moza` (default `moza`) |
| `--gain F` | float | ffb | `master_gain`, the overall steering-force strength |
| `--wheel-gain F` | float | ffbwheel | `constant_gain`, the peak motor torque. Raise it if the wheel feels light |
| `--lat-g-ref F` | float | ffb | `lateral_g_ref_mps2`. Raise it to slow how fast force builds with cornering and speed |
| `--rumble-gain F` | float | ffbwheel | Master vibration multiplier. Lower it for less off-road buzz (`0` = none) |
| `--no-rumble` | flag | ffbwheel | Turn off the sine vibration and keep steering force alone |
| `--invert` | flag | ffb | Flip the steering-force sign (use it if the wheel pulls the wrong way) |
| `-v`, `--verbose` | count | all | `-v` = info logging, `-vv` = debug |
| `--show-format` | flag | n/a | Print the Forza packet formats and key offsets, then exit |
| `--list-devices` | flag | n/a | List the FFB-capable wheels and joysticks SDL can see, then exit |

CLI flags beat the config file, and the config file beats the built-in defaults.

### Every config key is also a flag

Each option in the [configuration reference](#configuration-reference) also has a generated `--section-key` flag, so you can change anything without editing a file. To build the flag, take the dotted config path and turn each `.` and `_` into `-`. Booleans need an explicit `true` or `false`.

| Config key | Flag |
|------------|------|
| `ffb.smoothing_alpha` | `--ffb-smoothing-alpha 0.3` |
| `ffb.understeer.drop` | `--ffb-understeer-drop 0.4` |
| `output.ffbwheel.disable_autocenter` | `--output-ffbwheel-disable-autocenter false` |
| `output.ffbwheel.rumble_road_gain` | `--output-ffbwheel-rumble-road-gain 0.2` |
| `output.vjoy.axis_map.steer_force` | `--output-vjoy-axis-map-steer-force RZ` |
| `stale_timeout_s` | `--stale-timeout-s 1.5` |

`python -m forza_ffb --help` lists them all. The short flags in the first table cover the keys you'll touch most. If you pass a short flag and its long form together, the short flag wins.

---

## Configuration reference

Copy `config.example.json`, edit it, and pass `--config my.json`. You can include any subset of keys, and forza_ffb deep-merges your file over the defaults below. You can also set any of these keys on the command line (see [Every config key is also a flag](#every-config-key-is-also-a-flag)).

### `listen`
| Key | Default | Meaning |
|-----|---------|---------|
| `listen.ip` | `"127.0.0.1"` | Interface the UDP listener binds to |
| `listen.port` | `2066` | UDP port that receives Data Out |

### `output`
| Key | Default | Meaning |
|-----|---------|---------|
| `output.backend` | `"console"` | `console` / `vjoy` / `ffbwheel` / `null` |
| `output.rate_hz` | `0` | Output rate cap in Hz. `0` = one output per received packet (~60 Hz) |
| `output.console.every` | `10` | Print one of every N updates (console backend) |
| `output.vjoy.device_id` | `1` | vJoy device id |
| `output.vjoy.axis_map` | see below | Channel → axis (`X Y Z RX RY RZ SL0 SL1`). Leave a channel out to skip it |
| `output.ffbwheel.device_index` | `-1` | Wheel index (`-1` = first FFB-capable device) |
| `output.ffbwheel.device_name_match` | `"moza"` | Name substring to match, case-insensitive |
| `output.ffbwheel.constant_gain` | `1.0` | Scales `steer_force` → motor torque (peak strength) |
| `output.ffbwheel.invert` | `false` | Flip force direction in the backend |
| `output.ffbwheel.disable_autocenter` | `true` | Turn off the device's DirectInput autocenter spring |
| `output.ffbwheel.rumble` | `true` | Add a sine vibration from `road_texture` + `kerb` |
| `output.ffbwheel.rumble_gain` | `1.0` | Master multiplier on all rumble (lower = less off-road buzz) |
| `output.ffbwheel.rumble_road_gain` | `0.6` | Road texture (surface roughness) → rumble strength |
| `output.ffbwheel.rumble_kerb_gain` | `1.0` | Kerb and bump jolts → rumble strength |
| `output.ffbwheel.rumble_period_ms` | `20` | Sine period (smaller = higher-pitched buzz) |

Default `axis_map`: `steer_force→X, g_long→Y, road_texture→Z, kerb→RX, understeer→RY, oversteer→RZ`.

### `ffb` (force synthesis, which shapes the feel)
| Key | Default | Meaning |
|-----|---------|---------|
| `ffb.master_gain` | `1.0` | Overall strength of `steer_force` |
| `ffb.invert_steer` | `false` | Flip the steering-force sign |
| `ffb.steer_deadzone` | `0.02` | Drops tiny forces around centre to stop hum |
| `ffb.weight_lateral` | `0.6` | Share of lateral G (cornering load) |
| `ffb.weight_aligning` | `0.4` | Share of front slip angle (self-aligning torque) |
| `ffb.lateral_g_ref_mps2` | `18.0` | Lateral accel that maps to full force. **Higher = gentler ramp** |
| `ffb.slip_angle_ref_rad` | `0.22` | Front slip angle that maps to the full aligning term (≈12.6°) |
| `ffb.speed_ref_mps` | `6.0` | Below this speed the wheel goes light (parking) |
| `ffb.understeer.threshold` | `1.0` | Front combined slip where lightening starts |
| `ffb.understeer.limit` | `1.8` | Front combined slip for full understeer |
| `ffb.understeer.drop` | `0.6` | Share of force removed at full understeer |
| `ffb.oversteer.threshold` | `1.0` | Rear combined slip where oversteer starts to register |
| `ffb.oversteer.limit` | `2.0` | Rear combined slip for full oversteer |
| `ffb.road_gain` | `1.0` | Surface rumble → `road_texture` |
| `ffb.kerb_gain` | `6.0` | Suspension compression spikes → `kerb` |
| `ffb.kerb_strip_boost` | `0.4` | Extra `kerb` while a wheel sits on a rumble strip |
| `ffb.smoothing_alpha` | `0.5` | Per-channel EMA. `1.0` = off, lower = smoother but laggier |

### Top level
| Key | Default | Meaning |
|-----|---------|---------|
| `stale_timeout_s` | `0.5` | Seconds without a packet before output goes neutral and the wheel relaxes |

---

## Tuning the feel

Start on `--backend console`, take a corner, then switch to `ffbwheel`. You can set each knob below as a CLI flag, so you never need to open a file to tune.

- **Wheel goes heavy too fast in normal corners:** raise `--lat-g-ref`. Try `18`, then `24`, then `30`. Higher values save full torque for high-G moments.
- **Too light at the limit:** raise `--wheel-gain` to around `1.3`.
- **Everything too strong or too weak:** change `--gain`, or the FFB percentage in your wheel's driver.
- **Wheel pulls the wrong way:** add `--invert`.
- **Too much buzz off-road:** lower `--rumble-gain` (try `0.4`) or pass `--no-rumble`. To control surface buzz and kerb hits apart from each other, set `rumble_road_gain` and `rumble_kerb_gain` in the config.
- **Notchy or jittery:** lower `ffb.smoothing_alpha` to about `0.3`. If it feels laggy, push it toward `1.0`.

The R3 is a low-torque base at about 3.8 Nm. Leave Pit House FFB strength near 100% and shape the feel here. To try new values, hit `Ctrl+C` and relaunch with different flags.

---

## Stopping safely

- **Normal stop:** `Ctrl+C` in the forza_ffb terminal. It zeroes the force, stops the effects and releases the wheel. If you close the window instead, Windows can kill the process before that cleanup runs.
- **Emergency stop:** power off the wheelbase.
- **Auto-relax:** Forza stops sending telemetry when you pause or leave a race, and the wheel goes neutral within `stale_timeout_s` (0.5 s by default).
- **No force at all:** run `--backend console`, or set `output.ffbwheel.constant_gain` to `0`.

---

## Testing and development

The tests use the standard library alone. You don't need the game, a wheel or SDL.

```bash
python -m unittest discover -s tests       # parser, FFB math, axis/level scaling, UDP loopback
python -m forza_ffb --show-format          # print packet layouts & key offsets
```

`tools/fake_forza_sender.py` sends real 324-byte Horizon packets for five scenarios (`sweep`, `corner`, `kerbs`, `straight`, `idle`), so you can run the whole pipeline offline.

---

## Project layout

```
src/forza_ffb/
  packet.py      Forza packet parsing (field table + length autodetect; self-asserting offsets)
  ffb.py         FFBEngine: physics -> normalized effect channels (smoothing, understeer, NaN-safe)
  telemetry.py   UDP listener (stdlib sockets, receive timeout)
  config.py      defaults + JSON deep-merge
  bridge.py      listen -> parse -> synth -> output loop, plus the CLI
  outputs/       base.py (scaling), console.py, vjoy.py, ffbwheel.py (SDL_Haptic); make_output() factory
tools/           fake_forza_sender.py  (synthetic telemetry generator)
tests/           test_packet.py  test_ffb.py  test_ffbwheel.py  test_integration.py
config.example.json   full config you can copy and edit
```

---

## Packet format reference (Horizon / FH4, FH5, FH6: 324 B, little-endian)

Bytes 0 to 231 hold the "sled" block that every Forza title shares. Horizon games insert 12 bytes at 232 to 243, which pushes the dash section to offset 244. I cross-checked these offsets against two community sources (see Credits):

```
IsRaceOn @0(s32)   AccelerationX @20(f32)   TireSlipAngle FL @164(f32)
TireCombinedSlip FL @180   SurfaceRumble FL @148   SuspensionTravelMeters FL @196
Speed @256(f32, m/s)   Accel @315(u8)   Brake @316(u8)   Gear @319(u8)   Steer @320(s8, -127..127)
```
Run `python -m forza_ffb --show-format` for the full list.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| No data, or "telemetry stale" | Check that Data Out is on and the IP and port match. Forza sends nothing from menus or replays. Confirm with `--backend console` first. |
| `WinError 10013` on start | Hyper-V/WSL reserved the UDP port, or another app holds it. Run `netsh int ipv4 show excludedportrange protocol=udp` and `netstat -ano \| findstr :2066`, then pick a free port and set the same one in-game. |
| Wheel missing from `--list-devices` | Power on the base, check that its current mode exposes FFB, and install `pysdl2 pysdl2-dll`. |
| Force pulls the wrong way | `--invert`. |
| Wheel feels dead or too strong | Adjust `--wheel-gain` or `--gain`. Check `speed_ref_mps` too: the wheel goes light at a standstill on purpose. |
| Gets heavy too fast | Raise `--lat-g-ref`. |
| Centering spring won't go away | Set `Spring` and `Damper` to 0 in MOZA Pit House. The spring lives in the driver, outside forza_ffb's reach. |
| vJoy "failed to set axis" | Enable that axis for the device in *Configure vJoy*. |
| Forza Horizon 3 or older | Unsupported. Those games have no Data Out telemetry. |

---

## Credits and references

I built forza_ffb on these projects and checked the packet format against them.

**Output targets**
- [vJoy](https://github.com/jshafer817/vJoy): virtual joystick driver (original on [SourceForge](https://sourceforge.net/projects/vjoystick/))
- [pyvjoy](https://github.com/tidzo/pyvjoy): Python bindings for vJoy (pip-installable fork at [maxofbritton/pyvjoy](https://github.com/maxofbritton/pyvjoy))
- [Joystick Gremlin](https://github.com/WhiteMagic/JoystickGremlin): joystick remapping and scripting that reads vJoy
- [PySDL2](https://github.com/py-sdl/py-sdl2): the SDL2 bindings behind the `ffbwheel` backend ([pysdl2-dll](https://github.com/a-hurst/pysdl2-dll) bundles the runtime)
- [SDL](https://github.com/libsdl-org/SDL): the [SDL_Haptic](https://wiki.libsdl.org/SDL2/CategoryHaptic) force-feedback API underneath

**Forza telemetry format**
- [richstokes/Forza-data-tools](https://github.com/richstokes/Forza-data-tools): the field tables I derived the parser offsets from
- [satyajiit/forza-horizon-6-moza-bridge](https://github.com/satyajiit/forza-horizon-6-moza-bridge): FH6 format and a second check on the driver-input offsets
- [Forza "Data Out" telemetry structure (community forum)](https://forums.forza.net/t/data-out-telemetry-variables-and-structure/535984) and the [official FH6 Data Out docs](https://support.forza.net/hc/en-us/articles/51744149102611-Forza-Horizon-6-Data-Out-Documentation)
- [MOZA SDK](https://mozaracing.com/pages/sdk): reference for MOZA-native FFB, if you'd rather layer effects on top of the game's FFB than replace it

forza_ffb is an independent project. Microsoft, Turn 10, Playground Games, MOZA Racing and the projects listed above have no affiliation with it and don't endorse it.

## License

MIT. See `LICENSE`.
