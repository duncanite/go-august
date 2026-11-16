# go-august: the design

Research into August-family smart locks (August, and Yale locks on the Yale Access or Yale
Home apps) with an August Connect Wi-Fi bridge, done on 2026-11-15 from public sources, to
decide what a Go module for them would be. Nothing is built and nothing here was run
against a lock. The decisions at the end are open.

In short: the lock can be driven entirely locally, over Bluetooth, with a key the lock
shares with the vendor's cloud. The protocol is known and maintained in Python; nothing
exists in Go. The hard part is getting that key, since the vendor's cloud stopped
accepting the API key hobby projects used. The bridge has no interface on the LAN.

## The device

- **The lock** speaks Bluetooth Low Energy, with August's own encryption on top
  ([Quarkslab's analysis](https://blog.quarkslab.com/examining-the-august-smart-lock.html)).
- **The bridge** (August Connect) plugs into a wall outlet near the lock: 2.4 GHz Wi-Fi to
  the vendor's cloud on one side, Bluetooth to the lock on the other, one bridge to a lock
  ([August's FAQ](https://support.august.com/august-connect-faq-BylGvUyA_G)). It relays
  what the cloud sends. Nothing found describes a LAN interface on it.
- **Two clouds** since the brand was split, per the
  [yalexs](https://github.com/Yale-Libs/yalexs) library's constants: August and Yale
  Access (United States) at `api-production.august.com`, Yale Home (elsewhere) at
  `api.aaecosystem.com`.

## Ways in

### Bluetooth, with the offline key (local)

From the source of [yalexs-ble](https://github.com/Yale-Libs/yalexs-ble) (`lock.py`,
`session.py`) and [Home Assistant's page for it](https://www.home-assistant.io/integrations/yalexs_ble/):

- The lock holds "offline keys", each in a numbered slot. A client that has one key and
  its slot opens a session: two exchanges of eight random bytes under the offline key
  give a session key, and every frame after that is 18 bytes, AES-CBC over the first 16.
- Read: lock state, the door sensor where DoorSense is calibrated, battery voltage, the
  auto-lock setting. Do: lock and unlock.
- Home Assistant lists the Yale Assure YRD216, 226, 256, 420 and 450 and the August
  ASL-02, 03 and 05 as supported.
- The key can change after a firmware update; Home Assistant advises a second owner
  account kept for the integration. Bluetooth range and timing are the other weakness: it
  recommends a good adapter or a proxy near the lock.

Where the key comes from is the obstacle. It is in the cloud's answer for a lock
(`OfflineKeys.loaded`, a key and a slot, in yalexs's `lock.py`), and in the phone app's own
storage. Home Assistant's cloud integrations pass it to the Bluetooth one.

### The vendor's cloud, as the app uses it

Closed to a new client. The yalexs README, under "API status": "The public API key
historically embedded in this library is no longer accepted by either backend", and the
August brand "now requires an official partner key issued by the vendor"; Yale Home goes
through an OAuth flow tied to Home Assistant's account linking
([the issue that tracks it](https://github.com/Yale-Libs/yalexs/issues/167)).

### The vendor's cloud, officially

A partner program, by arrangement with the vendor; and [Seam](https://www.seam.co/docs/device-and-system-integration-guides/august-locks),
a third party with partner access that offers lock, unlock and access codes through the
bridge.

### Matter, HomeKit

Yale sells a Matter-over-Thread module for the Assure Lock SL (YRD256). It takes the
lock's one module slot, in place of the Yale Access module, and that lock then leaves the
Yale Access app. Some locks carry HomeKit over Bluetooth.

## Three ways in, compared

| | Bluetooth with the offline key | Cloud through a partner | Matter module |
|---|---|---|---|
| Local at run time | yes | no | yes |
| Needs | the key and slot once; a Bluetooth radio in range | an account with the partner | a YRD256, a Thread border router, giving up the app |
| Gives | state, door, battery, lock and unlock | lock, unlock, codes | lock, unlock, state |
| Risk | the key changes with firmware; the way to fetch it is closed | a third party in the path | a hardware change |

## What this module would be

Not decided. The options, most local first:

1. **A Bluetooth client of the lock**, a port of yalexs-ble's protocol to Go, taking the
   key and its slot as configuration. The protocol is small. Getting the key is the open
   part: once, from Home Assistant's storage, from a phone backup, or through partner
   access.
2. **A client of a partner's cloud**, where local is given up.
3. **A Matter controller**, where the lock is a YRD256 and the module is acceptable.

## Open questions

- The lock's exact model, and which app it is on (Yale Access, August, or Yale Home).
- Whether a key can be had: Home Assistant already linked to the account, an iPhone
  backup, or neither.
- Whether a Bluetooth radio can sit within range of the lock, on the machine that runs
  the command or as a proxy.

## Not verified

- Nothing above was observed on a lock or a bridge.
- The bridge's chips and any port it listens on: no teardown or filing was found.
- The Matter module's terms come from a search excerpt of Yale's FAQ, whose page did not
  render; the partner program's from an excerpt of developer.august.com.
- Whether the Yale Access phone app still stores the offline key where earlier reports
  found it. One 2024 report says the Yale Home Android app no longer does.
- The first exchange of the handshake is described as AES-ECB by one third-party port,
  not read in yalexs-ble itself.

## Sources

- [yalexs-ble](https://github.com/Yale-Libs/yalexs-ble) and
  [yalexs](https://github.com/Yale-Libs/yalexs): the Bluetooth protocol, the cloud's
  hosts, where the offline key is, the state of the public API key.
- [Home Assistant, Yale Access Bluetooth](https://www.home-assistant.io/integrations/yalexs_ble/):
  supported locks, where keys come from, their rotation, range advice.
- [Home Assistant, Yale](https://www.home-assistant.io/integrations/yale/) and
  [August](https://www.home-assistant.io/integrations/august/): the two cloud
  integrations.
- [Quarkslab, Examining the August smart lock](https://blog.quarkslab.com/examining-the-august-smart-lock.html):
  the protocol's analysis.
- [August Connect FAQ](https://support.august.com/august-connect-faq-BylGvUyA_G): one
  bridge to a lock, 2.4 GHz.
- [Seam's August guide](https://www.seam.co/docs/device-and-system-integration-guides/august-locks):
  what a partner's cloud offers.
