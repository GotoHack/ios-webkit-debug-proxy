# iOS WebKit Debug Proxy

The ios_webkit_debug_proxy (aka _iwdp_) allows developers to inspect MobileSafari and UIWebViews on real and simulated iOS devices via the [Chrome DevTools UI](https://developers.google.com/chrome-developer-tools/) and [Chrome Remote Debugging Protocol](https://developer.chrome.com/devtools/docs/debugger-protocol).  DevTools requests are translated into Apple's [Remote Web Inspector service](https://developer.apple.com/technologies/safari/developer-tools.html) calls.

## Installation

Linux and OS X are currently supported.  The fantastic [ios-webkit-debug-proxy-win32](https://github.com/artygus/ios-webkit-debug-proxy-win32) project implements Windows support.

On a Mac, it's easiest to install with [homebrew](http://brew.sh/):

```console
brew install ios-webkit-debug-proxy
```

On Linux or Mac:

```console
sudo apt-get install autoconf automake libusb-dev libusb-1.0-0-dev libplist-dev libplist++-dev usbmuxd libtool libimobiledevice-dev

git clone git@github.com:google/ios-webkit-debug-proxy.git
cd ios-webkit-debug-proxy

./autogen.sh
make
sudo make install
```

## Usage

On Linux, you must run the `usbmuxd` daemon.  The above install adds a /lib/udev rule to start the daemon whenever a device is attached.  To verify that usbmuxd can list your attached device(s), run `idevice_id -l`

### Start the simulator or device

The iOS Simulator is supported, but it must be started **before** the proxy.  The simulator can be started in XCode,  standalone, or via the command line:

```sh
# Xcode changes these paths frequently, so doublecheck them
SDK_DIR="/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs"
SIM_APP="/Applications/Xcode.app/Contents/Developer/Applications/Simulator.app/Contents/MacOS/Simulator"
$SIM_APP -SimulateApplication $SDK_DIR/iPhoneSimulator8.4.sdk/Applications/MobileSafari.app/MobileSafari
```

#### Enable the inspector

Your attached iOS devices must have ≥1 open browser tabs and the inspector enabled via:
  `Settings > Safari > Advanced > Web Inspector = ON`


### Start the proxy

```console
ios_webkit_debug_proxy
```

* `--debug` for verbose output.
* `--frontend` to specify a frontend
* `--help` for more options.
* `Ctrl-C` to quit. Also, the proxy can be left running as a background process.

### View and inspect debuggable tabs

Navigate to [localhost:9221](http://localhost:9221). You'll see a listing of all connected devices.

Click through to view tabs available on each, and click through again to open the DevTools for a tab.

## Configuration

### Setting the DevTools UI URL

By default, the DevTools UI frontend that iwdp uses is from:

    http://chrome-devtools-frontend.appspot.com/static/18.0.1025.74/devtools.html

You can use the `-f` argument to specify different frontend source, like Chrome's local DevTools, a local
[Chromium checkout](https://chromium.googlesource.com/chromium/src/+/master/third_party/WebKit/Source/devtools/) or another URL:

```console
# examples:
ios_webkit_debug_proxy -f chrome-devtools://devtools/bundled/inspector.html
ios_webkit_debug_proxy -f ~/chromium/src/third_party/WebKit/Source/devtools/front_end/inspector.html
ios_webkit_debug_proxy -f http://foo.com:1234/bar/inspector.html
```

If you use `-f chrome-devtools://devtools/bundled/inspector.html`, you won't be able to click the links shown in `localhost:9222` as Chrome blocks clicking these URLs. However, you can copy/paste them into the address bar.

Just the same, you can apply the appropriate port (9222) and page (2) values below.

    chrome-devtools://devtools/bundled/inspector.html?ws=localhost:9222/devtools/page/1

The `-f` value must end in ".html". As of Chrome 45, the primary URL [changed](https://codereview.chromium.org/1144393004/) from `devtools.html` to `inspector.html`.

To disable the frontend proxy, use the `--no-frontend` argument.

#### Port assigment

The default configuration works well for most developers. The device_id-to-port assignment defaults to:

    :9221 for the device list
    :9222 for the first iOS device that is attached
    :9223 for the second iOS device that is attached
    ...
    :9322 for the max device

If a port is in use then the next available port will be used, up to the range limit.

The port assignment is first-come-first-serve but is preserved if a device is detached and reattached, assuming that the proxy is not restarted, e.g.:

  1. start the proxy
  1. the device list gets :9221
  1. attach A gets :9222
  1. attach B gets :9223
  1. detach A, doesn't affect B's port
  1. attach C gets :9224 (not :9222)
  1. reattach A gets :9222 again (not :9225)

The port assignment rules can be set via the command line with `-c`.  The default is equivalent to:

    ios_webkit_debug_proxy -c null:9221,:9222-9322

where "null" represents the device list.  The following example restricts the proxy to a single device and port:

    ios_webkit_debug_proxy -c 4ea8dd11e8c4fbc1a2deadbeefa0fd3bbbb268c7:9227


### Troubleshooting

##### undefined reference to symbol 'log10@@GLIBC_2.2.5'
```console
/usr/bin/ld: ios_webkit_debug_proxy-char_buffer.o: undefined reference to symbol 'log10@@GLIBC_2.2.5'
//lib/x86_64-linux-gnu/libm.so.6: error adding symbols: DSO missing from command line
```

Run this before `make`: `./configure LIBS="-lm"`

##### error while loading shared libraries: libimobiledevice.so.6
```console
ios_webkit_debug_proxy: error while loading shared libraries: libimobiledevice.so.6: cannot open shared object file: No such file or directory
```

Run `sudo ldconfig`

##### idevice_id not found

The `idevice_id` executable may be found as part of the libimobiledevice-utils package.

##### could not start com.apple.webinspector! success

[Remove and rebuild libimobiledevice](https://github.com/google/ios-webkit-debug-proxy/issues/82#issuecomment-74205898).

##### Could not connect to lockdownd
> Could not connect to lockdownd. Exiting.: No such file or directory. Unable to attach <long id> inspector ios_webkit_debug_proxy

Check the device for [a prompt to trust the connected computer](http://i.stack.imgur.com/hPaqX.png). Choose "Trust" and try again.

##### If no luck so far...
Lastly, always try replugging in the USB cable.


## Design

![Alt overview](overview.png "Overview")

The proxy detects when iOS devices are attached/removed and provides the current device list on <http://localhost:9221>.  A developer can click on a device's link (e.g. <http://localhost:9222>) to list that device's open tabs, then click on a tab link (e.g. <http://localhost:9222/devtools/page/1>) to inspect that tab in their browser's DevTools UI.

Equivalent JSON-formatted APIs are provided for programmatic clients: <http://localhost:9221/json> to list all devices,    <http://localhost/9222/json> to list device ":9222"'s tabs,    and [ws://localhost:9222/devtools/page/1]() to inspect a tab.  See the [examples/README](examples/README.md) for example clients.

See [design.md](design.md) for an overview of the source layout and architecture.

## License and Copyright

Google BSD license <http://code.google.com/google_bsd_license.html>
Copyright 2012 Google Inc.  <wrightt@google.com>

The proxy uses the following open-source packages:
   - [libplist 1.10](http://cgit.sukimashita.com/libplist.git)
   - [libusbmuxd 1.0.8](http://cgit.sukimashita.com/usbmuxd.git/)
   - [libimobiledevice 1.1.5](http://cgit.sukimashita.com/libimobiledevice.git)
