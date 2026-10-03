# Server-Initiated Intercom

The `intercom` message lets the server open a live audio call with an idle device, for a room-to-room intercom or a PA system. Unlike [`notify`](notify.md), a call can stream the device microphone back to the server. The feature is disabled by default. Enable it with `CONFIG_ENABLE_INTERCOM`. For the Waveshare ESP32-S3-AUDIO-Board, build the `esp32-s3-audio-board-intercom` variant, which also enables device-side AEC.

## Transport

The server can only reach a device that has a control connection open. The MQTT/UDP transport keeps its MQTT connection open while idle, so it is the recommended transport. The WebSocket transport connects only when an audio channel is needed, so an idle WebSocket device cannot receive an `intercom` message.

When a call starts, the device opens the audio channel if it is not already open (MQTT `hello` followed by UDP, or WebSocket `hello`). Audio uses the normal channel format: Opus frames at the parameters negotiated in `hello`.

Two MQTT/UDP details make standard brokers and receive-only calls work:

- If the OTA response's `mqtt` section includes `subscribe_topic`, the device subscribes to it on every connect. Brokers such as Mosquitto only deliver messages to subscribed clients.
- Right after opening the UDP channel, the device sends one header-only packet (`payload_len` 0). This tells the server the device's UDP address before any microphone audio is sent, so `receive` calls can play audio immediately. Servers should ignore zero-length packets as audio.

## Server to device

Start a call:

```json
{
  "type": "intercom",
  "state": "start",
  "mode": "duplex",
  "caller": "Kitchen"
}
```

- `mode` is optional. `duplex` (the default) streams the microphone continuously for the whole call. `receive` plays incoming audio only and keeps the microphone off, which suits one-way PA announcements.
- `caller` is optional. The device shows it on the display.
- `chime` is optional. A call opens silently unless it is `true`, in which case the device plays its popup sound first. To ring a device before talking, send a [`notify`](notify.md) with a ring sound instead.

Sending `start` again during a call changes `mode` without ending the call. A server can use this for push-to-talk on devices without AEC: send `receive` while the remote side talks and `duplex` while it listens. Enabling the microphone resets the decoder, so queued playback is dropped at the switch.

End a call:

```json
{ "type": "intercom", "state": "stop" }
```

Queued audio finishes playing after `stop`. The audio channel stays open. Send `goodbye` (MQTT) or close the WebSocket to release it. A server-side `goodbye` also ends an active call.

Wait for the call key (room-to-room calls):

```json
{ "type": "intercom", "state": "ring", "text": "Call from Kitchen" }
{ "type": "intercom", "state": "dial", "text": "Calling Bedroom" }
```

- `ring` marks the device as being called: the call key answers. `dial` marks it as the caller: the call key cancels.
- While either is pending the device shows `text` as its status and blinks its LED ring green, also while a `notify` ring sound plays.
- The device plays no sound by itself. Send a `notify` with a ring sound (or a ringback tone) alongside.
- The pending state expires 15 seconds after the last `ring`/`dial`, so a server repeats them with each ring sound. `stop` or `start` clears it at once.

## Device to server

The device sends `intercom` messages with its `session_id`:

| `state` | Meaning |
|---|---|
| `stop` | The user hung up on the device (button press or wake-word invoke), or cancelled a pending `dial` with the call key. |
| `answer` | The call key was pressed while a `ring` was pending. The server starts the call (`start` to both devices). |
| `busy` | A `start` arrived while the device was not idle, for example during a voice assistant conversation or an OTA upgrade. The call was rejected. |

## Room name

The server can name the device after its room with an MCP tool call over the control channel:

```json
{ "type": "mcp", "payload": { "jsonrpc": "2.0", "id": 1, "method": "tools/call",
  "params": { "name": "self.intercom.set_name", "arguments": { "name": "Kitchen" } } } }
```

The device stores the name (at most 40 characters) and shows it instead of "Standby" while idle. An empty name restores "Standby". The tool is user-only, so it is not offered to the voice assistant's model.

## Device behavior

- The device accepts a call only while it is `Idle` or `Notifying`. An active notification is cancelled.
- During a call the device is in the `Intercom` state. Wake-word detection is off, a popup sound plays at the start, and incoming audio is played as it arrives.
- In `duplex` mode the device streams the microphone with the same AFE path as realtime conversations, including AEC when enabled. Without AEC the remote side hears its own voice echoed from the speaker, so use push-to-talk (`receive`/`duplex` switching) instead.
- Network loss, a protocol error, or a server `goodbye` returns the device to `Idle`.
- The device treats the channel as timed out after 120 seconds without incoming data. During a long, silent call, the server must keep sending audio (silence frames are fine).
- On the Waveshare ESP32-S3-AUDIO-Board, **K1** is the call key: it answers a pending `ring`, cancels a pending `dial`, hangs up a call, and otherwise works like BOOT (starts listening, so a server with speech recognition can take "call the kitchen").
