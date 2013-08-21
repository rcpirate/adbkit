# stf-adb

**stf-adb** is a pure [Node.js][nodejs] client for the [Android Debug Bridge][adb-site] server. It can be used either as a library in your own application, or simply as a convenient utility for playing with your device.

Most of the `adb` command line tool's functionality is supported (including pushing/pulling files, installing APKs and processing logs), with some added functionality such as being able to generate touch/key events and take screenshots.

## Requirements

Please note that although it may happen at some point, currently **this project is NOT an implementation of the ADB _server_**. The target host (where the devices are connected) must still have ADB installed and either already running (e.g. via `adb start-server`) or available in `$PATH`. An attempt will be made to start the server locally via the aforementioned command if the initial connection fails. This is the only case where we fall back to the `adb` binary.

When targeting a remote host, starting the server is entirely your responsibility.

Alternatively, you may want to consider using the Chrome [ADB][chrome-adb] extension, as it includes the ADB server and can be started/stopped quite easily.

## API

### adb.createClient([options])

Creates a client instance with the provided options. Note that this will not automatically establish a connection, it will only be done when necessary.

* **options** An object compatible with [Net.connect][net-connect]'s options:
    - **port** The port where the ADB server is listening. Defaults to `5037`.
    - **host** The host of the ADB server. Defaults to `'localhost'`.
    - **bin** As the sole exception, this option provides the path to the `adb` binary, used for starting the server locally if initial connection fails. Defaults to `'adb'`.
* Returns: The client instance.

### client.version(callback)

Queries the ADB server for its version. This is mainly useful for backwards-compatibility purposes.

* **callback(err, version)**
    - **err** `null` when successful, `Error` otherwise.
    - **version** The version of the ADB server.
* Returns: The client instance.

### client.listDevices(callback)

Get the list of currently connected devices and emulators.

* **callback(err, devices)**
    - **err** `null` when successful, `Error` otherwise.
    - **devices** An array of device objects. The device objects are plain JavaScript objects with two properties: `id` and `type`.
        * **id** The ID of the device. For real devices, this is usually the USB identifier.
        * **type** The device type. Values include `'emulator'` for emulators, `'device'` for devices, and `'offline'` for offline devices. `'offline'` can occur for example during boot, in low-battery conditions or when the ADB connection has not yet been approved on the device.
* Returns: The client instance.

### client.listDevicesWithPaths(callback)

Like `client.listDevices(callback)`, but includes the "path" of every device.

* **callback(err, devices)**
    - **err** `null` when successful, `Error` otherwise.
    - **devices** An array of device objects. The device objects are plain JavaScript objects with the following properties:
        * **id** See `client.listDevices()`.
        * **type** See `client.listDevices()`.
        * **path** The device path. This can be something like `usb:FD120000` for real devices.
* Returns: The client instance.

### client.trackDevices(callback)

Gets a device tracker. Events will be emitted when devices are added, removed, or their type changes (i.e. to/from `offline`). Note that the same events will be emitted for the initially connected devices also, so that you don't need to use both `client.listDevices()` and `client.trackDevices()`.

Note that as the tracker will keep a connection open, you must call `tracker.end()` if you wish to stop tracking devices.

* **callback(err, tracker)**
    - **err** `null` when successful, `Error` otherwise.
    - **tracker** The device tracker, which is an [`EventEmitter`][node-events]. The following events are available:
        * **add_(device)_** Emitted when a new device is connected, once per device. See `client.listDevices()` for details on the device object.
        * **remove_(device)_** Emitted when a device is unplugged, once per device. This does not include `offline` devices, those devices are connected but unavailable to ADB. See `client.listDevices()` for details on the device object.
        * **change_(device)_** Emitted when the `type` property of a device changes, once per device. The current value of `type` is the new value. This event usually occurs the type changes from `'device'` to `'offline'` or the other way around. See `client.listDevices()` for details on the device object and the `'offline'` type.
        * **changeSet_(changes)_** Emitted once for all changes reported by ADB in a single run. Multiple changes can occur when, for example, a USB hub is connected/unplugged and the device list changes quickly. If you wish to process all changes at once, use this event instead of the once-per-device ones. Keep in mind that the other events will still be emitted, though.
            - **changes** An object with the following properties always present:
                * **added** An array of added device objects, each one as in the `add` event. Empty if none.
                * **removed** An array of removed device objects, each one as in the `remove` event. Empty if none.
                * **changed** An array of changed device objects, each one as in the `change` event. Empty if none.
* Returns: The client instance.

### client.kill(callback)

This kills the ADB server. Note that the next connection will attempt to start the server again when it's unable to connect.

* **callback(err)**
    - **err** `null` when successful, `Error` otherwise.
* Returns: The client instance.

### client.getSerialNo(serial, callback)

Get the serial number of the device identified by the given serial number. With our API this doesn't really make much sense, but it has been implemented for completeness. _FYI: in the raw ADB protocol you can specify a device in other ways, too._

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, serial)**
    - **err** `null` when successful, `Error` otherwise.
    - **serial** The serial number of the device.
* Returns: The client instance.

### client.getDevicePath(serial, callback)

Get the device path of the device identified by the given serial number.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, path)**
    - **err** `null` when successful, `Error` otherwise.
    - **path** The device path. This corresponds to the device path in `client.listDevicesWithPaths()`.
* Returns: The client instance.

### client.getState(serial, callback)

Get the state of the device identified by the given serial number.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, state)**
    - **err** `null` when successful, `Error` otherwise.
    - **state** The device state. This corresponds to the device type in `client.listDevices()`.
* Returns: The client instance.

### client.getProperties(serial, callback)

Retrieves the properties of the device identified by the given serial number. This is analogous to `adb shell getprop`.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, properties)**
    - **err** `null` when successful, `Error` otherwise.
    - **properties** An object of device properties. Each key corresponds to a device property. Convenient for accessing things like `'ro.product.model'`.
* Returns: The client instance.

### client.getFeatures(serial, callback)

Retrieves the features of the device identified by the given serial number. This is analogous to `adb shell pm list features`. Useful for checking whether hardware features such as NFC are available (you'd check for `'android.hardware.nfc'`).

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, features)**
    - **err** `null` when successful, `Error` otherwise.
    - **features** An object of device features. Each key corresponds to a device feature, with the value being either `true` for a boolean feature, or the feature value as a string (e.g. `'0x20000'` for `reqGlEsVersion`).
* Returns: The client instance.

### client.forward(serial, local, remote, callback)

Forwards socket connections from the ADB server host (local) to the device (remote). This is analogous to `adb forward <local> <remote>`. It's important to note that if you are connected to a remote ADB server, the forward will be created on that host.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **local** A string representing the local endpoint on the ADB host. At time of writing, can be one of:
    - `tcp:<port>`
    - `localabstract:<unix domain socket name>`
    - `localreserved:<unix domain socket name>`
    - `localfilesystem:<unix domain socket name>`
    - `dev:<character device name>`
* **remote** A string representing the remote endpoint on the device. At time of writing, can be one of:
    - Any value accepted by the `local` argument
    - `jdwp:<process pid>`
* **callback(err)**
    - **err** `null` when successful, `Error` otherwise.
* Returns: The client instance.

### client.shell(serial, command, callback)

Runs a shell command on the device. Note that you'll be limited to the permissions of the `shell` user, which ADB uses.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **command** The shell command to execute.
* **callback(err, output)**
    - **err** `null` when successful, `Error` otherwise.
    - **output** An output [`Stream`][node-stream] in non-flowing mode. Unfortunately it is not possible to separate stdin and stdout, you'll get both of them in one stream. It is also not possible to access the exit code of the command. If access to any of these individual properties is needed, the command must be constructed in a way that allows you to parse the information from the output.
* Returns: The client instance.

### client.remount(serial, callback)

Attempts to remount the `/system` partition in read-write mode. This will usually only work on emulators and developer devices.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err)**
    - **err** `null` when successful, `Error` otherwise.
* Returns: The client instance.

### client.framebuffer(serial, callback)

Fetches the current framebuffer (i.e. what is visible on the screen) from the device and converts it into PNG using [gm][node-gm], which requires either [GraphicsMagick][graphicsmagick] or [ImageMagick][imagemagick] to be installed and available in `$PATH`. Note that we don't bother supporting really old framebuffer formats such as RGB_565. If for some mysterious reason you happen to run into a `>=2.3` device that uses RGB_565, let us know.

Note that high-resolution devices can have quite massive framebuffers. For example, a device with a resolution of 1920x1080 and 32 bit colors would have a roughly 8MB (`1920*1080*4` byte) RGBA framebuffer. Empirical tests point to about 5MB/s bandwidth limit for the ADB USB connection, which means that even excluding all other processing such as the PNG conversion, it can take ~1.6 seconds for the raw data to arrive, or even more if the USB connection is already congested.

* **serial** The serial number of the device. Corresponds to the device ID in `client.listDevices()`.
* **callback(err, info, png, raw)**
    - **err** `null` when successful, `Error` otherwise.
    - **info** Meta information about the framebuffer. Includes the following properties:
        * **version** The framebuffer version. Useful for patching possible backwards-compatibility issues.
        * **bpp** Bits per pixel (i.e. color depth).
        * **size** The raw byte size of the framebuffer.
        * **width** The horizontal resolution of the framebuffer. This SHOULD always be the same as screen width. We have not encountered any device with incorrect framebuffer metadata, but according to rumors there might be some.
        * **height** The vertical resolution of the framebuffer. This SHOULD always be the same as screen height.
        * **red_offset** The bit offset of the red color in a pixel.
        * **red_length** The bit length of the red color in a pixel.
        * **blue_offset** The bit offset of the blue color in a pixel.
        * **blue_length** The bit length of the blue color in a pixel.
        * **green_offset** The bit offset of the green color in a pixel.
        * **green_length** The bit length of the green color in a pixel.
        * **alpha_offset** The bit offset of alpha in a pixel.
        * **alpha_length** The bit length of alpha in a pixel. `0` when not available.
        * **format** The framebuffer format for convenience. This can be one of `'bgr'`,  `'bgra'`, `'rgb'`, `'rgba'`.
    - **png** The converted PNG stream.
    - **raw** The raw framebuffer stream.
* Returns: The client instance.

## Debugging

We use [debug][node-debug], and our debug namespace is `adb`. Some of the dependencies may provide debug output of their own. To see the debug output, set the `DEBUG` environment variable. For example, run your program with `DEBUG=adb:* node app.js`.

## Links

* [Android Debug Bridge][adb-site]
    - [SERVICES.TXT][adb-services] (ADB socket protocol)
* [Android ADB Protocols][adb-protocols] (a blog post explaining the protocol)
* [adb.js][adb-js] (another Node.js ADB implementation)
* [ADB Chrome extension][chrome-adb]

## License

Restricted until further notice.

[nodejs]: <http://nodejs.org/>
[adb-js]: <https://github.com/flier/adb.js>
[adb-site]: <http://developer.android.com/tools/help/adb.html>
[adb-services]: <https://github.com/android/platform_system_core/blob/master/adb/SERVICES.TXT>
[adb-protocols]: <http://blogs.kgsoft.co.uk/2013_03_15_prg.htm>
[file_sync_service.h]: <https://github.com/android/platform_system_core/blob/master/adb/file_sync_service.h>
[chrome-adb]: <https://chrome.google.com/webstore/detail/adb/dpngiggdglpdnjdoaefidgiigpemgage>
[node-debug]: <https://npmjs.org/package/debug>
[net-connect]: <http://nodejs.org/api/net.html#net_net_connect_options_connectionlistener>
[node-events]: <http://nodejs.org/api/events.html>
[node-stream]: <http://nodejs.org/api/stream.html>
[node-gm]: <https://github.com/aheckmann/gm>
[graphicsmagick]: <http://www.graphicsmagick.org/>
[imagemagick]: <http://www.imagemagick.org/>
