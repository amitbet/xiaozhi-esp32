# Fork Notes

This fork (`amitbet/xiaozhi-esp32`) adds a server-initiated **intercom / PA** mode and two MQTT/UDP
fixes so the firmware works with a self-hosted hub and a standard Mosquitto broker. The server side is
`xiaozhi-hub` in the [homeauto](https://github.com/amitbet) repository (`compose/xiaozhi-hub`).

- Upstream: `78/xiaozhi-esp32`, forked at `64b57d0`.
- Fork commits: `747f58d` (intercom + MQTT fixes), `eb276d4` (this file), then the room-name tool (item 5), audio controls (item 6) and silent calls plus `notify` without the popup (item 7) and the silent wake word (item 8).
- Protocol reference: [`docs/intercom.md`](docs/intercom.md).

Line numbers below are as of `747f58d`; search for the quoted symbols after an upstream merge, since lines drift.

## Merging upstream

```sh
git remote add upstream https://github.com/78/xiaozhi-esp32.git   # once
git fetch upstream
git merge upstream/main
```

Conflicts are most likely in `main/application.cc` (upstream changes it often), the long `depends on`
list of `USE_DEVICE_AEC` in `main/Kconfig.projbuild`, and the LED `switch` statements. Resolve them by
keeping upstream's code **and** re-adding the fork's lines described below. Then run the checks at the end.

## 1. Intercom device state (feature, `CONFIG_ENABLE_INTERCOM`)

The server sends `{"type":"intercom","state":"start","mode":"duplex"|"receive","caller":"..."}` or
`{"type":"intercom","state":"stop"}`. The device enters a dedicated state that plays incoming audio and,
in duplex mode, streams the microphone continuously. A repeated `start` changes mode mid-call
(push-to-talk). The button hangs up; a busy device answers `busy`.

| What | Where |
|---|---|
| New state `kDeviceStateIntercom` (after `kDeviceStateNotifying`) | `main/device_state.h:13` |
| State name `"intercom"` (must stay at the same index as the enum) | `main/device_state_machine.cc`, `STATE_STRINGS` |
| Transitions `Idle→Intercom`, `Connecting→Intercom`, `Intercom→Idle` | `main/device_state_machine.cc:79`, `:88`, `:103` (`IsValidTransition`) |
| Message dispatch `"intercom"` (guarded by `#if CONFIG_ENABLE_INTERCOM`) | `main/application.cc:623`, in the `OnIncomingJson` lambda of `InitializeProtocol()` |
| Play incoming audio while in Intercom (not only Speaking) | `main/application.cc:559`, `OnIncomingAudio` lambda |
| State entry: status text, caller, performance power, wake word off, `EnableVoiceProcessing(mic)`, popup sound only when the start message has `"chime": true` (`intercom_chime_`) | `main/application.cc:1074`, `case kDeviceStateIntercom` in `HandleStateChangedEvent()` |
| Handlers `HandleIntercomMessage`, `StartIntercom`, `ContinueStartIntercom`, `SetIntercomMic`, `StopIntercom` | `main/application.cc:1193`–`1300`; declarations `main/application.h:177` |
| Members `intercom_mic_enabled_`, `intercom_caller_`, `intercom_chime_` | `main/application.h:154` |
| Button (`ToggleChatState`) hangs up during a call | `main/application.cc:796`, top of `HandleToggleChatEvent()` |
| `WakeWordInvoke` (board buttons) hangs up during a call | `main/application.cc:1431` |
| Network loss ends the call | `main/application.cc:322`, `HandleNetworkDisconnectedEvent()` |
| Device→server `intercom` `stop`/`busy` | `Protocol::SendIntercomState`, `main/protocols/protocol.cc:108`, `main/protocols/protocol.h:69` |
| Kconfig option `ENABLE_INTERCOM` (default `n`) | `main/Kconfig.projbuild:1020`, after `RECEIVE_CUSTOM_MESSAGE` |
| Status string `INTERCOM` (en-US; he-IL translation; other locales fall back to en-US) | `main/assets/locales/en-US/language.json:30`, `he-IL/language.json:27` |
| LEDs treat Intercom like Speaking | `main/led/gpio_led.cc`, `single_led.cc`, `circular_strip.cc` (`case kDeviceStateIntercom`) |
| Wi-Fi config mode resets the protocol from Intercom too | `main/boards/common/wifi_board.cc:203`, `EnterWifiConfigMode()` |

Merge notes:

- If upstream adds device states, keep `kDeviceStateIntercom` and its `STATE_STRINGS` entry at matching
  positions. `GetStateName()` indexes the array by enum value.
- If upstream adds a new `switch (state)` over `DeviceState` (LEDs, boards), add `kDeviceStateIntercom`
  next to `kDeviceStateSpeaking`/`kDeviceStateNotifying`.
- If upstream changes how `Notifying` is handled (the closest analogue), mirror the change for Intercom.
- `SetIntercomMic(true)` relies on `AudioService::EnableVoiceProcessing(true)` resetting the decoder. If
  upstream changes that, re-check the popup sound in the state-entry code and the push-to-talk switch.
- Not cherry-pickable in pieces: the state, the state machine entries, the handlers, and the Kconfig
  option depend on each other. With `CONFIG_ENABLE_INTERCOM=n` the device never enters the state.

## 2. Board variant with AEC for the Waveshare ESP32-S3-AUDIO-Board

| What | Where |
|---|---|
| Build variant `esp32-s3-audio-board-intercom` (existing variant unchanged) adding `CONFIG_ENABLE_INTERCOM=y`, `CONFIG_USE_DEVICE_AEC=y` | `main/boards/waveshare/esp32-s3-audio-board/config.json:21` |
| Wake word **Jarvis** instead of 你好小智 (`CONFIG_SR_WN_WN9_NIHAOXIAOZHI_TTS=n`, `CONFIG_SR_WN_WN9_JARVIS_TTS=y`). The model is packed into the assets image (`generated_assets.bin` at `0x800000`), so flash it along with the app | same `config.json` variant |
| Allow `USE_DEVICE_AEC` on this board (it has a hardware reference input, `AUDIO_INPUT_REFERENCE true`) | `main/Kconfig.projbuild:972`, end of the `USE_DEVICE_AEC` `depends on` list |

Merge notes: upstream keeps extending that `depends on` list. On conflict, take upstream's list and
append `|| BOARD_TYPE_WAVESHARE_ESP32_S3_AUDIO_BOARD`. The variant name is the OTA-reported board name;
the hub offers firmware by that name (`<data>/firmware/esp32-s3-audio-board-intercom.bin`).

With device AEC on, `GetDefaultListeningMode()` returns realtime, so voice-assistant conversations also
become full duplex. That is intended.

## 3. MQTT: subscribe to `subscribe_topic` (fix, needed for Mosquitto)

Upstream never subscribes to a topic; it relies on xiaozhi's own broker pushing messages to the
client. Mosquitto (and standard brokers) only deliver to subscribers, so server messages never arrived.

| What | Where |
|---|---|
| Read optional `subscribe_topic` from the `mqtt` settings (written from the OTA response) | `main/protocols/mqtt_protocol.cc:88`, `StartMqttClient()` |
| Subscribe on every connect | `main/protocols/mqtt_protocol.cc:112`, `mqtt_->OnConnected` lambda |
| Member `subscribe_topic_` | `main/protocols/mqtt_protocol.h:44` |

This is standalone and cherry-pickable. It is a no-op when the key is absent, so it is safe with the official
server. It could be proposed upstream.

## 4. MQTT/UDP: header-only announce packet when the channel opens (fix)

The server learns a device's UDP address from its first datagram. In a receive-only intercom call (PA),
the mic is off, so without this packet the server could never send audio.

| What | Where |
|---|---|
| After the UDP connect, send the 16-byte header with `payload_len` 0 and the next sequence number | `main/protocols/mqtt_protocol.cc:384`–`392`, end of `OpenAudioChannel()` |

Merge notes: `CreateUdp()` returns a `unique_ptr`; the code converts it to `shared_ptr` before storing it in
`udp_`. If upstream changes channel ownership, keep the send after `udp_` is set, outside
`channel_mutex_`. Servers must ignore zero-length packets as audio; `xiaozhi-hub` does. This is standalone
and cherry-pickable.

## 5. Room name shown on the idle screen (feature, `CONFIG_ENABLE_INTERCOM`)

The hub names each speaker after its room and pushes the name with an MCP tool call over MQTT
(`tools/call` of `self.intercom.set_name` with `{"name": "Kitchen"}`). The device stores it in NVS and
shows it instead of "Standby" when idle. An empty name restores "Standby".

| What | Where |
|---|---|
| User-only MCP tool `self.intercom.set_name` (max 40 chars) | `main/mcp_server.cc`, end of `AddUserOnlyTools()`, guarded by `#if CONFIG_ENABLE_INTERCOM` |
| `Application::SetDeviceName` (schedules, persists NVS `intercom`/`name`, refreshes idle status) and `GetDeviceName` | `main/application.cc`, before `Application::Schedule`; declarations `main/application.h` (public section, after `PlaySound`) |
| Name loaded at boot | top of `Application::Initialize()` |
| `StandbyStatus()` replaces `Lang::Strings::STANDBY` in `DismissAlert()` and the `kDeviceStateIdle` case of `HandleStateChangedEvent()` | `main/application.cc` |

Merge notes: if upstream adds new places that set the idle status to `Lang::Strings::STANDBY`, use
`StandbyStatus()` there too. Some emoji displays (`electron_emoji_display.cc`, `otto_emoji_display.cc`,
`esp-hi/emoji_display.cc`, `emote_display.cc`) match the status text against `STANDBY`. The name only
replaces it on intercom builds, where the name is set, so those boards are unaffected unless they enable
intercom. NVS key `intercom`/`name` is persistent API.

## 6. Speaker volume and mic sensitivity from the hub (feature, `CONFIG_ENABLE_INTERCOM`)

The hub reads and sets both per speaker over MQTT MCP calls. Volume already had a tool upstream
(`self.audio_speaker.set_volume`, persisted by `AudioCodec` in NVS `audio`/`output_volume`). The fork adds:

| What | Where |
|---|---|
| User-only tools `self.intercom.set_mic_gain` (`gain` 0-37 dB, persisted in NVS `intercom`/`mic_gain` as tenths of a dB) and `self.intercom.get_audio` (returns `{"volume", "mic_gain", "mic_gain_max"}`) | `main/mcp_server.cc`, `AddUserOnlyTools()`, in the same `#if CONFIG_ENABLE_INTERCOM` block as `set_name` |
| Saved mic gain restored at boot | `Application::Initialize()`, right after `audio_service_.Initialize(codec)` |
| `BoxAudioCodec::SetInputGain` override so a change applies while the mic is open (upstream applies the gain only in `EnableInput`) | `main/audio/codecs/box_audio_codec.cc`/`.h` |

Merge notes: if upstream starts calling `SetInputGain` from board code or persists input gain itself,
reconcile with the NVS `intercom`/`mic_gain` value (the fork's value is applied after codec init). The
`BoxAudioCodec` override takes `data_if_mutex_` like `EnableInput`. Keep that if upstream changes the locking.

## 7. `notify` without the popup (`"chime": false`)

Not behind `CONFIG_ENABLE_INTERCOM`: it extends the upstream `notify` message. The hub rings speakers
with `notify` and a ring sound, and the popup before a ring would be a second alert.

| What | Where |
|---|---|
| Parse optional `chime` (default true) and pass it on | `main/application.cc`, `"notify"` branch of the `OnIncomingJson` lambda |
| `StartNotification(url, subtitles, chime = true)` skips `PlaySound(OGG_POPUP)` when false | `main/application.cc`, `Application::StartNotification`; declaration `main/application.h` |

Merge notes: if upstream changes `StartNotification`'s signature or moves the popup, keep the flag.
Documented in `docs/notify.md`.

## 8. Wake word: lights only, no sound (`CONFIG_ENABLE_INTERCOM`)

With no assistant behind the hub, a false "Jarvis" should not make noise. The hub no longer sends an
alert (the firmware plays `OGG_VIBRATION` with every alert) and closes the channel after 3 s.

| What | Where |
|---|---|
| `kWakeWordSound` (false in intercom builds) replaces the popup on wake: `PlaySound(OGG_POPUP)` in the Listening branch of `HandleWakeWordDetectedEvent()`, and `play_popup_on_listening_ = true` there and in `ContinueWakeWordInvoke()` | `main/application.cc`, top of file and those functions |
| LED ring shows a spinning blue light (`Scroll`) in Listening instead of solid red | `main/led/circular_strip.cc`, `OnStateChanged()`, `#if CONFIG_ENABLE_INTERCOM` case |

Merge notes: if upstream adds more popup calls on the wake path, gate them with `kWakeWordSound` too.

## 9. Documentation

- `docs/intercom.md`: the intercom protocol, transport requirements, and items 3–4.
- `FORK_NOTES.md`: this file.

## Checks after a merge

1. Build the variant (the ESP-IDF `release-v6.1` image currently fails on the `78__uart-uhci` component;
   `v6.0.2` works):

   ```sh
   docker run --rm -v "$PWD":/project -w /project espressif/idf:v6.0.2 bash -c \
     'git config --global --add safe.directory /project && . $IDF_PATH/export.sh >/dev/null && \
      python3 scripts/build.py waveshare/esp32-s3-audio-board --name esp32-s3-audio-board-intercom'
   ```

2. Run the host tests: `python3 -m unittest discover -s scripts/tests`.
3. Run the hub's end-to-end test with a simulated device (homeauto `compose/xiaozhi-hub/tests/e2e.py`).
   It exercises the MQTT hello, UDP announce, push-to-talk mode switch, and hangup.
4. On hardware: flash it (or update it over the air: put the `.bin` in the hub's `data/firmware/`
   and call the `self.upgrade_firmware` MCP tool with its `http://<hub>:8003/firmware/<file>.bin` URL), confirm it reaches `idle` over MQTT, then call it from the hub UI or the Home
   Intercom DynApp. Check hold-to-talk (speaker plays and its mic is off), release (the device mic is heard),
   the button hangup, announce to all, and an open call without echo.
