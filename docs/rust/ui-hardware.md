# UI & Hardware

Crates for talking to people (terminals, windows, tray icons) and to USB devices, all written with few or no third-party crates. **`noroi`** is a curses-style terminal UI with zero crates.io dependencies. **`stipple`** is a self-drawn, cross-platform GUI toolkit that renders through the pure-Rust `oxideav` media stack. **`ldtray`** puts an icon in the system tray and loads every OS GUI library at runtime, so a headless daemon still links and runs. On the hardware side, **`rawusb`** is a libusb replacement with no dependencies, and **`usbmagic`** builds on it to drive USB test instruments, starting with the Great Scott Gadgets Cynthion.

## Quick pick
| Need | Use |
|------|-----|
| Full-screen terminal UI (widgets, mouse, colors) with no dependencies, or a `no_std` TUI core | [`noroi`](#noroi) |
| A C-callable curses replacement | [`noroi`](#noroi) (`capi` feature) |
| Native desktop/web GUI without winit/wgpu/GTK/Qt (pre-alpha) | [`stipple`](#stipple) |
| Tray icon, context menu and desktop notifications from a daemon that must also run headless | [`ldtray`](#ldtray) |
| Enumerate USB devices, control/bulk/interrupt/isochronous transfers, no libusb | [`rawusb`](#rawusb) |
| USB HID, mass storage, USB serial (CDC-ACM/FTDI), UVC cameras or USB Ethernet from userspace | [`rawusb`](#rawusb) (class-helper features) |
| Capture USB 2.0 traffic to pcap with a Cynthion, or send USB Power Delivery messages | [`usbmagic`](#usbmagic) |

## noroi

**Repo:** https://github.com/KarpelesLab/noroi · **Crate:** `noroi` (git only, not on crates.io) · **License:** MIT · **Status:** usable, early (0.1.0). The backend supports Linux/Android only.

A terminal UI library in the style of curses/ncurses, with **zero crates.io dependencies**. It gets raw mode and the window size by declaring libc symbols with `extern "C"` (std already links libc), not through the `libc` crate. With `--no-default-features` the core is `#![no_std]` + `alloc`: buffers, styling, the input parser, layout, widgets and the line editor. Rendering is diffed and flicker-free: widgets paint into a cell buffer, and only the cells that changed are written. Its API is shaped like ratatui (`Terminal::draw`, `Frame::render_widget`, `Block`, `Paragraph`).

**Use it when:**
- You want a full-screen TUI with an empty dependency graph.
- You want the TUI core (parsing, layout, widgets) on an embedded or `no_std` target, with your own backend.
- You need a curses-like library callable from C (`include/noroi.h`).

**Don't use it when / limits:**
- You need Windows or macOS terminals. The backend is unix only, and only the Linux/Android `termios` layout is present. Porting to another unix means adding its `struct termios` in `src/sys/unix.rs`.
- You need crates.io publication or a stable API.

**Add it:**
```toml
[dependencies]
noroi = { git = "https://github.com/KarpelesLab/noroi" }
# core only (no_std + alloc):
# noroi = { git = "https://github.com/KarpelesLab/noroi", default-features = false }
```

**Key features / cargo features:**
- `std` (default): the unix TTY backend, `Terminal`/`Frame`, and a threaded event reader that delivers events over a channel. It detects resize without a `SIGWINCH` handler.
- `capi`: a C ABI (implies `std`). Run `make capi` to build `target/release/libnoroi.{so,a}`.
- Input: keys with Ctrl/Alt/Shift, SGR and legacy mouse, bracketed paste, focus events and UTF-8. The parser handles escape sequences split across reads.
- Colors: 16, 256 and 24-bit, downgraded automatically to the depth the terminal supports.
- Layout: a constraint solver plus `row`/`column`/`grid`/`spacer` helpers.
- Widgets: `Block`, `Paragraph`, `List`, `Gauge`, `Button`, `Clear`, `Spinner`, and a `LineEditor` with history and emacs keybindings.
- Theming: `Theme::ofuda()` (the default) and `Theme::mono()`. Animation (`anim::Tween`, `Pulse`) is clock-free: the app passes `advance(dt)` a time delta each frame.
- Demo binary: `cargo run --bin noroidemo`.

**Example:**
```rust
use noroi::terminal::Terminal;
use noroi::widget::{Block, Borders, Paragraph, Wrap};
use noroi::event::{Event, KeyCode};

fn main() -> std::io::Result<()> {
    let mut term = Terminal::open()?;        // raw mode + alternate screen
    loop {
        term.draw(|frame| {
            let block = Block::bordered().borders(Borders::ALL).title("noroi");
            let inner = block.inner(frame.area());
            frame.render_widget(&block, frame.area());
            frame.render_widget(
                &Paragraph::new("Hello (q quits)").wrap(Wrap { trim: true }),
                inner,
            );
        })?;
        if let Some(Event::Key(k)) = term.events().poll(None)? {
            if k.code == KeyCode::Char('q') { break; }
        }
    }
    Ok(()) // Terminal restores the screen on drop, even on panic
}
```

**Gotchas:**
- The environment variable `NOROI_REDUCED_MOTION=1` freezes animations. Respect it in your own animators too.
- `Terminal::open()` fails when stdin/stdout is not a TTY, for example under CI or a pipe. Fall back to plain output in that case.

## stipple

**Repo:** https://github.com/KarpelesLab/stipple · **Crate:** `stipple` umbrella plus a workspace of `stipple-*` crates (git only, version 0.0.1) · **License:** MIT · **Status:** pre-alpha. APIs are unstable.

A cross-platform, **self-drawn**, fully themeable GUI toolkit. Every platform renders pixel-identical output through a CPU rasterizer. It has native backends that it writes itself for X11, Wayland, Win32 and Cocoa, plus a web target (wasm + `<canvas>`, without wasm-bindgen). It uses **no** winit, wgpu, taffy, lyon, GTK or Qt. The only heavy dependency it accepts is the pure-Rust `oxideav` stack (scene graph, rasterizer, font shaping, image and SVG). MSRV is 1.88 (edition 2024).

**Use it when:**
- You want a native window GUI with a small, auditable dependency tree and one consistent theme on every OS.
- You want headless-testable UI. `App` has `render_once()`, `click_at(pos)`, `type_text(..)`, `press_key(..)`, `focus_next()` and `accessibility_tree()`, so an agent can drive and screenshot a UI without a display.

**Don't use it when / limits:**
- You need a stable API or a mature widget set. It has about 12 widgets.
- You need mobile targets (Android/iOS are roadmap items with demo stubs) or GPU rendering. GPU is experimental: `stipple-gpu` is an EGL + GLES2 blitter.

**Add it:**
```toml
[dependencies]
stipple = { git = "https://github.com/KarpelesLab/stipple" }
```

**Workspace crates:**

| Crate | Role |
|---|---|
| `stipple` | Umbrella: `App` builder, `prelude`, re-exports |
| `stipple-core` | `View` trait, element IR, layout and paint passes, state and events |
| `stipple-widgets` | Standard self-drawn widgets |
| `stipple-style` | Design tokens and `Theme` (`light()`, `dark()`, `with_accent`, `high_contrast`) |
| `stipple-layout` | Flex/box layout solver |
| `stipple-render` | Scene to oxideav raster to `Surface` |
| `stipple-platform` | Per-OS windowing, input, IME, clipboard, vsync (the only crate with per-OS code) |
| `stipple-anim` | Easing, tweens, springs |
| `stipple-geometry` | `Point`, `Size`, `Rect`, `Affine` in logical pixels |
| `stipple-gpu` | Experimental EGL/GLES2 present path |
| `stipple-web` | wasm target that exposes an RGBA framebuffer for a canvas |

**Example** (from `examples/clickdemo`; the view function takes `(&State, &mut Cx<State>)` and returns an `Element`):
```rust
use stipple::prelude::*;

struct Clicks { n: u32 }

fn view(state: &Clicks, cx: &mut Cx<Clicks>) -> Element {
    let theme = *cx.theme();
    Element::stack(
        Axis::Horizontal,
        vec![Element::text(format!("Clicks: {}", state.n), 64.0, theme.palette.on_primary)],
    )
    .fill(theme.palette.primary)
    .align(Align::Center, Align::Center)
    .on_tap(cx, |s: &mut Clicks| s.n += 1)
}

fn main() {
    let mut app = App::new(Clicks { n: 0 }, view)
        .title("Stipple Clicks")
        .theme(Theme::dark())
        .logical_size(Size::new(640.0, 480.0));
    if let Some(font) = Font::system_default() { app = app.font(font); }
    app.run(); // native window, or a one-shot headless render with no display
}
```

**Gotchas:**
- The README's `Column((..))`/`Button(..)` snippet with a one-argument `view(&State)` describes the intended API. The examples in `examples/*` compile against the current API, so copy from those.
- Examples run with `cargo run -p <name>`, for example `window`, `clickdemo`, `textinput`, `themegallery`, `calculator` or `tabsdemo`.

## ldtray

**Repo:** https://github.com/KarpelesLab/ldtray · **Crate:** `ldtray` (crates.io `0.1.2`) · **License:** MIT · **Status:** usable. The Linux backend is validated end to end on KDE. The Windows and macOS backends are smoke-tested at runtime in CI.

Cross-platform tray icons that are **never linked against a GUI library at compile time**. libdbus (Linux StatusNotifierItem + dbusmenu), `shell32`/`user32` (Windows `Shell_NotifyIcon`) and AppKit/objc (macOS `NSStatusItem`) are all `dlopen`ed through `libloading`, which is the only dependency. On a headless machine `Tray::new` returns a clean `Err`: it does not fail to link and does not crash.

**Use it when:**
- A daemon or CLI should show a tray icon, menu and notifications when a desktop is present, and keep running normally when it is not.
- You want one binary for desktops and servers.

**Don't use it when / limits:**
- You need a full GUI. It is only a tray icon, a menu and notifications.
- On Windows, clicking a notification balloon maps to the first action only. On Linux, notification actions are shown. Elsewhere the message is shown and the actions are ignored.

**Add it:**
```toml
[dependencies]
ldtray = "0.1"
```

**Key features:**
- `Icon::from_rgba(w, h, rgba)`. Menus are built from `MenuItem::button`, `checkbox`, `separator` and `submenu`.
- Events: `Event::Menu(id)`, `Event::NotificationAction(id)`, plus left, right, middle and double-click triggers.
- A `TrayHandle` (`Send + Sync`) offers `set_icon`, `set_tooltip`, `set_menu`, `clear_menu`, `notify` and `quit`.
- `tray.run(cb)` blocks and is correct on the main thread, which macOS requires. `tray.spawn(cb)` runs it in the background on Linux/Windows and returns a `TrayHandle`.

**Example:**
```rust
use ldtray::{Event, Icon, Menu, MenuItem, Notification, Tray, TrayConfig};

fn main() {
    let icon = Icon::from_rgba(16, 16, [220u8, 40, 40, 255].repeat(256)).expect("icon");
    let menu = Menu::new()
        .item(MenuItem::button(1, "Say hi"))
        .item(MenuItem::separator())
        .item(MenuItem::button(2, "Quit"));
    let tray = match Tray::new(TrayConfig::new(icon).tooltip("demo").menu(menu)) {
        Ok(t) => t,
        Err(e) => { eprintln!("no tray ({e}); running headless"); return; }
    };
    let handle = tray.handle();
    let _ = tray.run(move |event| match event {
        Event::Menu(id) if id.0 == 1 => { let _ = handle.notify(Notification::new("demo", "hi")); }
        Event::Menu(id) if id.0 == 2 => { let _ = handle.quit(); }
        _ => {}
    });
}
```

**Gotchas:**
- Always handle `Tray::new` returning `Err`, because that is the headless path.
- On macOS, use `run()` on the main thread, not `spawn()`.

## rawusb

**Repo:** https://github.com/KarpelesLab/rawusb · **Crate:** `rawusb` (crates.io `0.1.4`) · **License:** MIT · **Status:** usable, pre-1.0. Linux is tested against real hardware. Windows and macOS follow libusb's call sequences but have had less time on real devices.

Cross-platform USB access in the spirit of libusb, with **no dependencies, no C library and no build script**. It talks directly to usbfs on Linux, WinUSB on Windows and IOKit on macOS. The API has two levels. The convenience layer covers open/claim and synchronous control, bulk and interrupt transfers. The low-level layer is a `Transfer` object that you can submit, cancel, give a callback, or `.await` from any async runtime. One background event thread per `Context` drives completions. MSRV is 1.89.

**Use it when:**
- You would otherwise use `rusb`/libusb but want pure Rust with no system library.
- You need isochronous transfers, hotplug, or ready-made class drivers from userspace.

**Don't use it when / limits:**
- On Windows, only devices bound to WinUSB can be claimed. Bind one with Zadig, WCID descriptors or an INF file. There is no device reset on Windows, and only the current configuration can be set.
- On macOS, interfaces owned by a kernel driver (HID, mass storage, CDC) cannot be claimed.
- Portable isochronous transfers need every packet to be exactly the endpoint's max packet size, because WinUSB requires that.

**Add it:**
```toml
[dependencies]
rawusb = "0.1"
# rawusb = { version = "0.1", features = ["hotplug", "serial", "hid"] }
```

**Cargo features** (all off by default):
- `hotplug`: arrival and removal events, filtered by vendor, product or class, delivered as a callback or an iterator.
- `hid`: `HidDevice` (hidapi-style reports) and a `ReportDescriptor` parser.
- `msc`: `MassStorage` (SCSI over bulk-only transport) and `BlockDevice` (`Read + Write + Seek`).
- `serial`: `SerialPort` for CDC-ACM and FTDI (`std::io::Read`/`Write`, line settings, DTR/RTS).
- `uvc`: `Camera` and `Stream` (probe/commit negotiation, isochronous or bulk streaming, frame reassembly).
- `net`: `NetDevice` for CDC-ECM, CDC-NCM and RNDIS. `pktkit` implements `pktkit::L2Device` for it, and is the only feature that pulls in a dependency.

**Example:**
```rust
use rawusb::{Context, ControlType, Direction, Recipient, request_type};
use std::time::Duration;

fn main() -> rawusb::Result<()> {
    let ctx = Context::new()?;
    for dev in ctx.devices()? {
        let d = dev.device_descriptor();
        println!("{:03}:{:03} {:04x}:{:04x}", dev.bus_number(), dev.address(), d.vendor_id, d.product_id);
    }
    let handle = ctx.open_device_with_vid_pid(0x1234, 0x5678)?;
    handle.set_auto_detach_kernel_driver(true);
    handle.claim_interface(0)?;
    let mut buf = [0u8; 64];
    let n = handle.bulk_read(0x81, &mut buf, Duration::from_secs(1))?;
    let rt = request_type(Direction::In, ControlType::Vendor, Recipient::Device);
    let m = handle.control_read(rt, 0x01, 0, 0, &mut buf, Duration::from_secs(1))?;
    println!("bulk {n} B, control {m} B");
    Ok(())
}
```

**Gotchas:**
- On Linux, the user needs read/write access to `/dev/bus/usb/BBB/DDD`, usually through a udev rule or the `plugdev` group. Enumeration and descriptors work without opening the device.
- On Linux, class helpers detach the kernel driver (usbhid, ftdi_sio and so on) while they are alive. A process killed before dropping one leaves the driver detached until the device is replugged.
- On composite devices, call `handle.claim_all_interfaces()` first, then `SerialPort::open_all(&handle)`, `HidDevice::open_all(&handle)` and so on. Helpers never share an interface.
- Examples: `list_devices`, `hotplug`, `hid_dump`, `msc_info`, `serial_monitor`, `uvc_capture`. Each needs its feature, for example `cargo run --features uvc --example uvc_capture`.

## usbmagic

**Repo:** https://github.com/KarpelesLab/usbmagic · **Crate:** `usbmagic` library + CLI (git only, 0.1.0) · **License:** BSD-3-Clause (its protocol code derives from GSG Packetry) · **Status:** experimental. Analyzer capture and USB-PD tooling work. The USB host role is in progress.

A library and CLI for programmable USB test instruments ("magic USB ports"), built on `rawusb` without libusb or Python. It supports one device today: the Great Scott Gadgets **Cynthion**. With the stock USB Analyzer gateware (`1d50:615b`) it passively captures Low, Full and High speed USB 2.0 traffic between a host and a device plugged across its TARGET-A and TARGET-C ports, and writes Wireshark-readable pcap (`LINKTYPE_USB_2_0`, nanosecond timestamps). The CLI also flashes FPGA bitstreams itself over Apollo. With its PD-bridge gateware it can listen to, source and send raw USB Power Delivery messages and VDMs through the board's FUSB302B controllers.

**Use it when:**
- You need scripted USB 2.0 capture to pcap from a Cynthion, for example to debug a device driver.
- You want to observe or inject USB-C Power Delivery traffic, including non-compliant messages, for forensics.

**Don't use it when / limits:**
- You don't have a Cynthion. It is the only backend. Others can be added by implementing `Backend` and `MagicDevice`.
- You need USB 3 / SuperSpeed. The Cynthion hardware cannot route it.
- You want to drive a device as a USB host. The `UsbHost`, `PowerDelivery` and `PowerMonitor` traits exist, but only `mock::MockHost` implements them fully so far.

**Install / run:**
```sh
cargo install --git https://github.com/KarpelesLab/usbmagic
usbmagic list                                          # connected instruments
usbmagic info                                          # speeds, gateware version, state
usbmagic capture --speed auto --duration 5 --output capture.pcap
usbmagic capture --speed high --count 100              # print packet summaries
usbmagic capture -o - | wireshark -k -i -              # live into Wireshark
usbmagic flash [--bit x.bit] [--persistent]            # flash FPGA SRAM or SPI flash
usbmagic pd-listen | pd-source | pd-send | pd-dump     # USB-PD tools (pcapng LINKTYPE_USB_TYPE_C_PD)
```
Use `--device <serial-substring>` to pick one board when several are connected. On Linux, add a udev rule for `1d50:615b` (the README has it).

**Library example:**
```rust
use usbmagic::{discover, CaptureData, CaptureOptions, Speed};

fn main() -> usbmagic::Result<()> {
    let mut dev = discover()?.into_iter().next().expect("a device").open()?;
    println!("speeds: {:?}", dev.capabilities().supported_speeds);
    let opts = CaptureOptions { speed: Speed::Auto, ..Default::default() };
    for item in dev.start_capture(opts)?.take(10) {
        let item = item?;
        match item.data {
            CaptureData::Packet(bytes) => println!("{} ns: {} B", item.timestamp_ns, bytes.len()),
            CaptureData::Event(code) => println!("{} ns: event {code:#04x}", item.timestamp_ns),
        }
    }
    Ok(())
}
```

**Gotchas:**
- Capture is passive. With no host and device attached across TARGET-A and TARGET-C, it waits forever. Use `--count` or `--duration`.
- The bitstreams (`usbmagic-blinky.bit` and `usbmagic-pd-bridge.bit`) are stored in `firmware/` via Git LFS. Run `scripts/pull-gateware.sh` if they are missing.

**Related: [usbmagic-gateware](https://github.com/KarpelesLab/usbmagic-gateware)** (BSD-3-Clause) holds the Amaranth/LUNA FPGA designs for the Cynthion's ECP5. They are built reproducibly in Docker with `docker run ... python3 build.py`. The designs are a bring-up blinky, the PD bridge (FUSB302B I2C over JTAG registers) and an early full-speed USB host PHY bring-up. It is Python rather than Rust, and its bitstreams are flashed with `usbmagic flash`.
