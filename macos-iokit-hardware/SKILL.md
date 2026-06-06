---
name: macos-iokit-hardware
description: "Hardware access on macOS — IOKit (IOServiceGetMatchingService, IORegistryEntry), USB/HID devices (IOHIDManager), serial ports, power management (IOPMAssertion), battery/power info, CoreAudio (AVAudioEngine, AudioUnit, audio device selection), NSSound. Use when accessing hardware devices, managing audio, or controlling power assertions on macOS."
sources: https://developer.apple.com/documentation/iokit, https://developer.apple.com/documentation/avfaudio/avaudioengine, https://developer.apple.com/documentation/iokit/iohidmanager_h, https://developer.apple.com/documentation/iokit/kiopmassertiontypepreventsystemsleep, https://developer.apple.com/library/archive/technotes/tn2187/_index.html, https://dev.to/paulcontr_/usb-hid-on-macos-talking-to-devices-with-iokit-3g2n
researched: 2026-06-06
platform: macOS 26+
---

# macOS IOKit & Hardware Access

Hardware access patterns for macOS 26+ with Swift 6.3. Covers IOKit service matching and registry traversal, HID device management (gamepads, keyboards, custom devices), USB device enumeration, serial port access, power management assertions, battery and battery-level monitoring, CoreAudio engine setup with device selection, and NSSound playback. For AppKit window management, see `appkit-foundations`. For entitlements and TCC, see `macos-security-entitlements`.

See [references/hardware-patterns.md](references/hardware-patterns.md) for complete IOHIDManager gamepad patterns, IOPMAssertion background-work recipes, AVAudioEngine device selection, USB enumeration, and battery monitoring code.

## Contents

- [IOKit Fundamentals](#iokit-fundamentals)
- [HID Devices](#hid-devices)
- [USB Device Enumeration](#usb-device-enumeration)
- [Serial Ports](#serial-ports)
- [Power Management](#power-management)
- [Battery & Power](#battery--power)
- [CoreAudio](#coreaudio)
- [NSSound](#nssound)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## IOKit Fundamentals

IOKit is the macOS framework for communicating with hardware devices through a tree of interconnected services. Every device and driver in the system appears as a node in the IORegistry, a hierarchical database that can be traversed and queried.

### IOService Matching

`IOServiceGetMatchingService` returns the first service matching a dictionary of criteria. Use `IOServiceMatching` to create a match dictionary for a class name, or `IORegistryEntrySearchCFProperty` to search by property.

```swift
import IOKit

// Match the first USB device
var matchDict = IOServiceMatching(kIOUSBDeviceClassName) as NSMutableDictionary
// Add vendor/product ID filters
matchDict[kUSBVendorID] = 0x05AC  // Apple
matchDict[kUSBProductID] = 0x12A8

var iterator: io_iterator_t = 0
let result = IOServiceGetMatchingServices(kIOMainPortDefault, matchDict, &iterator)
guard result == KERN_SUCCESS, iterator != 0 else { return }

var service: io_object_t
repeat {
    service = IOIteratorNext(iterator)
    if service != 0 {
        // Use the service
        IOObjectRelease(service)
    }
} while service != 0

IOObjectRelease(iterator)
```

### IORegistryEntry Traversal

Walk the registry tree to discover child entries, parent entries, and properties. Use `IORegistryEntryCreateCFProperty` to read a specific property as a CoreFoundation type.

```swift
func readRegistryProperty(for service: io_object_t, key: String) -> CFTypeRef? {
    let cfKey = key as CFString
    let value = IORegistryEntryCreateCFProperty(service, cfKey, kCFAllocatorDefault, 0)
    return value?.takeRetainedValue()
}

// Example: read the "USB Product Name" string
if let name = readRegistryProperty(for: service, key: "USB Product Name") as? String {
    print("Device: \(name)")
}

// Example: read the "idVendor" as a number
if let vendor = readRegistryProperty(for: service, key: "idVendor") as? Int {
    print("Vendor ID: 0x\(String(vendor, radix: 16))")
}
```

### Registry Plane

The IORegistry is divided into planes (IOService, IOPower, IOUSB, IOAudio, etc.). When traversing, specify the plane to scope your search:

| Plane | Scope |
|-------|-------|
| `kIOServicePlane` | Device and driver services (default for most work) |
| `kIOPowerPlane` | Power management entries |
| `kIOAudioPlane` | Audio device entries |
| `kIOFireWirePlane` | FireWire (legacy) |
| `kIOUSBPlane` | USB-specific entries |

```swift
// Traverse children in the IOService plane
var childIterator: io_iterator_t = 0
let plane = kIOServicePlane
let result = IORegistryEntryGetChildIterator(service, plane, &childIterator)
```

### Key IOKit Memory Rules

- Every `io_object_t` returned by `IOIteratorNext`, `IOServiceGetMatchingService`, or similar must be released with `IOObjectRelease`.
- `IORegistryEntryCreateCFProperty` returns a +1 retained CFTypeRef. Use `.takeRetainedValue()` to transfer ownership to ARC, or call `CFRelease` yourself.
- `IOServiceGetMatchingServices` returns an iterator that must be released with `IOObjectRelease`.

## HID Devices

The Human Interface Device (HID) framework provides access to gamepads, joysticks, keyboards, mice, and custom USB HID devices. `IOHIDManager` is the central manager.

### IOHIDManager Setup

```swift
import IOKit.hid

class HIDDeviceManager {
    private var manager: IOHIDManager?

    init() {
        manager = IOHIDManagerCreate(kCFAllocatorDefault, IOOptionBits(kIOHIDOptionsTypeNone))
    }

    func start() {
        guard let manager = manager else { return }

        // Match vendor/product for a specific device
        let deviceMatch: [String: Any] = [
            kIOHIDVendorIDKey: 0x05AC,
            kIOHIDProductIDKey: 0x12A8
        ]
        IOHIDManagerSetDeviceMatching(manager, deviceMatch as CFDictionary)

        // Set callbacks
        let context = Unmanaged.passUnretained(self).toOpaque()
        IOHIDManagerRegisterDeviceMatchingCallback(manager, deviceMatched, context)
        IOHIDManagerRegisterDeviceRemovalCallback(manager, deviceRemoved, context)
        IOHIDManagerRegisterInputValueCallback(manager, inputValueReceived, context)

        // Schedule with run loop
        IOHIDManagerScheduleWithRunLoop(manager, CFRunLoopGetCurrent(), CFRunLoopMode.defaultMode.rawValue)
        IOHIDManagerOpen(manager, IOOptionBits(kIOHIDOptionsTypeNone))
    }

    func stop() {
        guard let manager = manager else { return }
        IOHIDManagerClose(manager, IOOptionBits(kIOHIDOptionsTypeNone))
    }
}

// Callbacks
func deviceMatched(context: UnsafeMutableRawPointer?, result: IOReturn, sender: UnsafeMutableRawPointer?, device: IOHIDDevice) {
    print("HID device connected")
}
func deviceRemoved(context: UnsafeMutableRawPointer?, result: IOReturn, sender: UnsafeMutableRawPointer?, device: IOHIDDevice) {
    print("HID device disconnected")
}
func inputValueReceived(context: UnsafeMutableRawPointer?, result: IOReturn, sender: UnsafeMutableRawPointer?, value: IOHIDValue) {
    let element = IOHIDValueGetElement(value)
    let intValue = IOHIDValueGetIntegerValue(value)
    let usagePage = IOHIDElementGetUsagePage(element)
    let usage = IOHIDElementGetUsage(element)
    print("Input: usagePage=\(usagePage) usage=\(usage) value=\(intValue)")
}
```

### Gamepad / Joystick Access

For game controller support on macOS 13+, prefer Apple's GameController framework. Use IOHIDManager directly when you need raw HID access to vendor-specific devices, custom peripherals, or devices that GameController doesn't enumerate:

```swift
// Match all gamepads/joysticks
let match: [String: Any] = [
    kIOHIDDeviceUsagePageKey: kHIDPage_GenericDesktop,
    kIOHIDDeviceUsageKey: kHIDUsage_GD_Joystick
]
IOHIDManagerSetDeviceMatching(manager, match as CFDictionary)
```

### Reading Device Properties

```swift
func describe(device: IOHIDDevice) -> [String: Any] {
    var props: [String: Any] = [:]
    props["manufacturer"] = IOHIDDeviceGetProperty(device, kIOHIDManufacturerKey as CFString)
    props["product"] = IOHIDDeviceGetProperty(device, kIOHIDProductKey as CFString)
    props["vendorID"] = IOHIDDeviceGetProperty(device, kIOHIDVendorIDKey as CFString)
    props["productID"] = IOHIDDeviceGetProperty(device, kIOHIDProductIDKey as CFString)
    props["serial"] = IOHIDDeviceGetProperty(device, kIOHIDSerialNumberKey as CFString)
    return props
}
```

### USB HID Device Communication

For custom USB HID devices (e.g., gamepads, keyboards, peripherals), use feature reports via `IOHIDDeviceSetReport`/`IOHIDDeviceGetReport`:

```swift
// Open the device and send a feature report
let result = IOHIDDeviceOpen(device, IOOptionBits(kIOHIDOptionsTypeNone))
guard result == kIOReturnSuccess else { return }

// Send a feature report — buffer length must match maxFeatureReportSize exactly
var buffer = [UInt8](repeating: 0, count: 90)  // Use the exact maxFeatureReportSize
buffer[0] = 0x01  // Report ID (passed separately)

let sendResult = IOHIDDeviceSetReport(
    device,
    kIOHIDReportTypeFeature,
    0,  // Report ID
    buffer,
    buffer.count
)

// Read back after a short delay (200–300 ms for many devices)
DispatchQueue.global().asyncAfter(deadline: .now() + 0.25) {
    var readBuffer = [UInt8](repeating: 0, count: 90)
    IOHIDDeviceGetReport(
        device,
        kIOHIDReportTypeFeature,
        0,
        &readBuffer,
        &readBuffer.count
    )
    // Interpret response bytes per device protocol
}
```

**Critical details when working with USB HID devices:**
- Filter interfaces by `maxFeatureReportSize` to find the correct interface when a device has multiple
- Buffer length must exactly match the expected report size; an off-by-one causes error `0xE0005000`
- `IOHIDManagerCopyDevices` returns a `Set` with non-deterministic order — always filter properties
- Sandboxed apps will silently fail IOKit calls; entitlements are required

## USB Device Enumeration

### Matching by Vendor/Product ID

```swift
import IOKit.usb

func enumerateUSBDevices(vendorID: Int, productID: Int) {
    let matchingDict = IOServiceMatching(kIOUSBDeviceClassName) as NSMutableDictionary
    matchingDict[kUSBVendorID] = vendorID
    matchingDict[kUSBProductID] = productID

    var iterator: io_iterator_t = 0
    let kr = IOServiceGetMatchingServices(kIOMainPortDefault, matchingDict, &iterator)
    guard kr == KERN_SUCCESS else { return }

    var devices: [IOUSBDeviceInterface] = []
    var device = IOIteratorNext(iterator)
    while device != 0 {
        // Query the device for its interface
        var plugInInterface: UnsafeMutablePointer<UnsafeMutablePointer<IUnknownVTbl>?>?
        var score: Int32 = 0
        let result = IOCreatePlugInInterfaceForService(
            device,
            kIOUSBDeviceUserClientTypeID,
            kIOCFPlugInInterfaceID,
            &plugInInterface,
            &score
        )

        if result == KERN_SUCCESS {
            // Use the device interface
        }

        IOObjectRelease(device)
        device = IOIteratorNext(iterator)
    }
    IOObjectRelease(iterator)
}
```

### Common USB Registry Properties

| Property Key | Type | Description |
|---|---|---|
| `USB Vendor Name` | String | Human-readable vendor |
| `USB Product Name` | String | Human-readable product |
| `idVendor` | Int | USB Vendor ID |
| `idProduct` | Int | USB Product ID |
| `bcdDevice` | Int | Device release number |
| `kUSBSerialNumberString` | String | Device serial number |
| `kUSBContainerID` | Data | USB hub container ID |
| `PortNum` | Int | Physical port number |

### USB Interface Enumeration

To access interfaces (e.g., a USB device with multiple functions), iterate the device's children in the matching plane or use `IOUSBFindInterfaceRequest`:

```swift
let interfaceRequest = IOUSBFindInterfaceRequest(
    bInterfaceClass: UInt16(kIOUSBInterfaceClassHID),
    bInterfaceSubClass: 0,
    bInterfaceProtocol: 0,
    bAlternateSetting: 0
)
```

## Serial Ports

### Finding Serial Devices via IOKit

```swift
import IOKit.serial

func findSerialDevices() {
    let matchingDict = IOServiceMatching(kIOSerialBSDServiceValue) as NSMutableDictionary
    matchingDict[kIOSerialBSDTypeKey] = kIOSerialBSDAllTypes

    var iterator: io_iterator_t = 0
    let kr = IOServiceGetMatchingServices(
        kIOMainPortDefault,
        matchingDict,
        &iterator
    )
    guard kr == KERN_SUCCESS else { return }

    var device = IOIteratorNext(iterator)
    while device != 0 {
        // Get the /dev path from the registry entry
        if let path = IORegistryEntryCreateCFProperty(
            device,
            kIOCalloutDeviceKey as CFString,
            kCFAllocatorDefault,
            0
        )?.takeRetainedValue() as? String {
            print("Serial port: \(path)")
            // e.g., /dev/cu.usbserial-XXXXXX
        }
        IOObjectRelease(device)
        device = IOIteratorNext(iterator)
    }
    IOObjectRelease(iterator)
}
```

### Serial Device Paths

| Path Pattern | Type |
|---|---|
| `/dev/cu.*` | Callout (dial-out) — use for initiating connections |
| `/dev/tty.*` | Terminal (dial-in) — use for listening for incoming connections |

Use `/dev/cu.*` paths for typical serial device communication. The `/dev/cu.*` devices support non-blocking open even when no carrier detect is present.

### Bluetooth Serial

For Bluetooth serial port profiles (SPP), filter by device type:

```swift
matchingDict[kIOSerialBSDTypeKey] = kIOSerialBSDModemType  // or kIOSerialBSDRS232Type
```

## Power Management

### Preventing System Sleep with IOPMAssertion

`IOPMAssertionCreateWithName` creates a power assertion that prevents the system from sleeping. This is the recommended way to prevent idle sleep during long operations (downloads, backups, video playback).

```swift
import IOPMLib

class PowerAssertion {
    private var assertionID: UInt32 = 0

    func preventSleep(reason: String = "Operation in progress") -> Bool {
        let result = IOPMAssertionCreateWithName(
            kIOPMAssertionTypePreventUserIdleSystemSleep as CFString,
            IOPMAssertionLevel(kIOPMAssertionLevelOn),
            reason as CFString,
            &assertionID
        )
        return result == kIOReturnSuccess
    }

    func allowSleep() {
        if assertionID != 0 {
            IOPMAssertionRelease(assertionID)
            assertionID = 0
        }
    }

    deinit {
        allowSleep()
    }
}
```

### Assertion Types

| Type | Behavior |
|---|---|
| `kIOPMAssertionTypePreventUserIdleSystemSleep` | Prevent system idle sleep (display may still sleep) |
| `kIOPMAssertionTypePreventUserIdleDisplaySleep` | Prevent display sleep (system may still idle-sleep) |
| `kIOPMAssertionTypePreventSystemSleep` | Prevent all system sleep (Dark Wake preferred; may still sleep under thermal emergencies) |

**Recommended types (2025):** Use `kIOPMAssertionTypePreventUserIdleSystemSleep` for most workloads — it prevents the system from going to sleep while allowing display sleep. This mirrors `caffeinate -i` behavior. For full system+display sleep prevention, use `PreventUserIdleDisplaySleep` in combination.

**`kIOPMAssertionTypePreventSystemSleep`** keeps the system in Dark Wake. The OS may still sleep under low power or thermal emergencies — assertions are suggestions, not guarantees. This assertion type publishes `kIOPMAssertionTypePreventSystemSleep` in its properties dictionary.

### Sleep/Wake Notifications

To receive sleep/wake notifications (`caffeinate -i` replacement pattern), register for system power changes:

```swift
// Register for system power notifications
var notifyPort: IONotificationPortRef?
let rootPort = IORegisterForSystemPower(&connection, &notifyPort, { (refCon, service, messageType, messageArgument) in
    switch messageType {
    case kIOMessageSystemWillSleep:
        // Acknowledge or deny sleep
        IOCancelPowerChange(refCon as! io_connect_t, messageArgument)
    case kIOMessageCanSystemSleep:
        // Acknowledge or deny can-sleep
        IOCancelPowerChange(refCon as! io_connect_t, messageArgument)
    case kIOMessageSystemHasPoweredOn:
        // System woke up
        break
    default:
        break
    }
}, &connection)
```

**Important:** Forced sleep cannot be prevented — only delayed. Always handle `kIOMessageCanSystemSleep` by calling `IOCancelPowerChange` to delay, but the system may eventually force sleep anyway.

### Scope and Lifetime

- Assertions are reference-counted. Each `Create` must be balanced by a `Release`.
- Assertions are automatically released when the process exits.
- Use the `deinit` pattern to ensure cleanup. For Swift classes that manage a single assertion, `deinit` is the right place to release.
- Assertions can be inspected by users and system tools (`pmset -g assertions`).

## Battery & Power

### Power Source Information

`IOPSCopyPowerSourcesInfo` returns a `CFTypeRef` (a `CFArray` of `CFDictionary` entries) describing all power sources.

```swift
import IOPMLib

struct PowerInfo {
    let isCharging: Bool
    let isOnBattery: Bool
    let batteryPercent: Int?
    let timeRemaining: Int?  // minutes, or kIOPSTimeRemainingUnknown / kIOPSTimeRemainingUnlimited
    let powerSourceState: String
}

func getPowerInfo() -> PowerInfo? {
    guard let powerSourcesInfo = IOPSCopyPowerSourcesInfo() else { return nil }
    guard let sources = IOPSCopyPowerSourcesList(powerSourcesInfo)?.takeRetainedValue() as? [CFTypeRef],
          let firstSource = sources.first else {
        CFRelease(powerSourcesInfo)
        return nil
    }

    guard let description = IOPSGetPowerSourceDescription(powerSourcesInfo, firstSource)?.takeUnretainedValue() as? [String: Any] else {
        CFRelease(powerSourcesInfo)
        return nil
    }

    CFRelease(powerSourcesInfo)

    let powerSourceState = description[kIOPSPowerSourceStateKey as String] as? String
    let isOnBattery = powerSourceState == kIOPSProvidedValues.BatteryPower.rawValue
    let isCharging = description[kIOPSIsChargingKey as String] as? Bool ?? false
    let capacity = description[kIOPSCurrentCapacityKey as String] as? Int
    let maxCapacity = description[kIOPSMaxCapacityKey as String] as? Int
    let timeRemaining = description[kIOPSTimeRemainingKey as String] as? Int

    var batteryPercent: Int? = nil
    if let cap = capacity, let max = maxCapacity, max > 0 {
        batteryPercent = Int((Double(cap) / Double(max)) * 100)
    }

    return PowerInfo(
        isCharging: isCharging,
        isOnBattery: isOnBattery,
        batteryPercent: batteryPercent,
        timeRemaining: timeRemaining,
        powerSourceState: powerSourceState ?? "unknown"
    )
}
```

### Battery Warning Level

```swift
func batteryWarningLevel() -> IOPSLBatteryWarningLevel {
    return IOPSGetBatteryWarningLevel()
}
// Returns: kIOPSLBatteryWarningNone, kIOPSLBatteryWarningEarly, kIOPSLBatteryWarningFinal
```

### Key Power Source Dictionary Keys

| Key | Type | Description |
|---|---|---|
| `kIOPSPowerSourceStateKey` | String | `kIOPSAcPowerValue` or `kIOPSBatteryPowerValue` |
| `kIOPSCurrentCapacityKey` | Int | Current charge (mAh or %) |
| `kIOPSMaxCapacityKey` | Int | Maximum capacity |
| `kIOPSTimeRemainingKey` | Int | Estimated time remaining (minutes) |
| `kIOPSIsChargingKey` | Bool | Whether the battery is charging |
| `kIOPSIsPresentKey` | Bool | Whether a battery is present |
| `kIOPSTypeKey` | String | `kIOPSInternalBatteryType` or `kIOPSBatteryType` |
| `kIOPSNameKey` | String | Power source name |

## CoreAudio

### AVAudioEngine Setup

`AVAudioEngine` is the high-level CoreAudio API for routing audio between input, output, and effect nodes. It manages the AudioUnit graph for you.

```swift
import AVFoundation

class AudioEngineController {
    let engine = AVAudioEngine()
    let playerNode = AVAudioPlayerNode()
    let reverb = AVAudioUnitReverb()
    let delay = AVAudioUnitDelay()

    func setup() throws {
        // Attach nodes
        engine.attach(playerNode)
        engine.attach(reverb)
        engine.attach(delay)

        // Connect: player -> delay -> reverb -> output
        engine.connect(playerNode, to: delay, format: nil)
        engine.connect(delay, to: reverb, format: nil)
        engine.connect(reverb, to: engine.mainMixerNode, format: nil)

        // Configure effects
        reverb.loadFactoryPreset(.mediumHall)
        reverb.wetDryMix = 30
        delay.delayTime = 0.5
        delay.feedback = 20
        delay.wetDryMix = 15

        // Start the engine
        try engine.start()
    }

    func playFile(url: URL) throws {
        try playerNode.scheduleFile(url, at: nil, completionCallbackType: .dataPlayedBack) {
            print("Playback complete")
        }
        playerNode.play()
    }
}
```

### Input/Output Device Selection

On macOS, `AVAudioEngine` doesn't provide a direct high-level API to select input/output devices. You must configure them at the Core Audio / AudioUnit level.

**Setting the output device (macOS Ventura+):**

```swift
// Get the AUAudioUnit from the output node
let audioUnit = engine.outputNode.auAudioUnit
try audioUnit.setDeviceID(targetAudioDeviceID)
```

**Setting the input device (via AudioUnit property):**

```swift
let audioUnit = engine.inputNode.auAudioUnit.audioUnit  // Underlying AudioUnit
var deviceID = targetInputDeviceID
let status = AudioUnitSetProperty(
    audioUnit,
    kAudioOutputUnitProperty_CurrentDevice,
    kAudioUnitScope_Global,
    0,
    &deviceID,
    UInt32(MemoryLayout<AudioDeviceID>.size)
)
```

**Important:** `AVAudioEngine` can only be assigned to a single aggregate device (a device with both input and output channels). You cannot independently set separate input and output devices — create an `AudioAggregateDevice` if you need both.

**Finding an AudioDeviceID by name:**

```swift
import CoreAudio
import AVFoundation

func findAudioDevice(named deviceName: String) -> AudioDeviceID? {
    var propertyAddress = AudioObjectPropertyAddress(
        mSelector: kAudioHardwarePropertyDevices,
        mScope: kAudioObjectPropertyScopeGlobal,
        mElement: kAudioObjectPropertyElementMain
    )

    var dataSize: UInt32 = 0
    var result = AudioObjectGetPropertyDataSize(kAudioObjectSystemObject, &propertyAddress, 0, nil, &dataSize)
    guard result == noErr else { return nil }

    let deviceCount = Int(dataSize) / MemoryLayout<AudioDeviceID>.size
    var devices = [AudioDeviceID](repeating: 0, count: deviceCount)
    result = AudioObjectGetPropertyData(kAudioObjectSystemObject, &propertyAddress, 0, nil, &dataSize, &devices)
    guard result == noErr else { return nil }

    for device in devices {
        var nameAddress = AudioObjectPropertyAddress(
            mSelector: kAudioDevicePropertyDeviceNameCFString,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )
        var nameSize = UInt32(MemoryLayout<CFString?>.size)
        var name: CFString? = nil

        result = AudioObjectGetPropertyData(device, &nameAddress, 0, nil, &nameSize, &name)
        if result == noErr, let cfName = name as String?, cfName.contains(deviceName) {
            return device
        }
    }
    return nil
}
```

### Available AVAudioUnit Effects

| Class | Purpose |
|---|---|
| `AVAudioUnitReverb` | Reverb with factory presets or custom parameters |
| `AVAudioUnitDelay` | Delay/echo effect |
| `AVAudioUnitDistortion` | Distortion with presets (drums, speech, etc.) |
| `AVAudioUnitEQ` | Parametric or graphic equalizer |
| `AVAudioUnitTimePitch` | Pitch shifting and time stretching |
| `AVAudioUnitVarispeed` | Speed variation with optional pitch adjustment |
| `AVAudioUnitDynamicsProcessor` | Compression/limiting |

## NSSound

### Basic Playback

`NSSound` is the simplest way to play audio on macOS. It supports AIFF, WAV, and other common formats.

```swift
import AppKit

class SoundManager: NSObject, NSSoundDelegate {
    func playSound(name: String) {
        guard let sound = NSSound(named: name) else {
            print("Sound not found: \(name)")
            return
        }
        sound.delegate = self
        sound.play()
    }

    func playSound(from url: URL) throws {
        guard let sound = NSSound(contentsOf: url, byReference: false) else {
            throw SoundError.fileNotSupported
        }
        sound.delegate = self
        sound.play()
    }

    // MARK: - NSSoundDelegate

    func sound(_ sound: NSSound, didFinishPlaying flag: Bool) {
        if flag {
            print("Finished playing successfully")
        } else {
            print("Playback stopped (not finished)")
        }
    }
}
```

### Volume Control

```swift
sound.volume = 0.5  // 0.0 (silent) to 1.0 (full)
```

### Looping

```swift
sound.loops = true  // Continuously loops; delegate fires on each completion
```

### System Sound Names

Common system sounds available via `NSSound(named:)`:

| Name | Description |
|---|---|
| `NSSound.Name.basso` | Bass error sound |
| `NSSound.Name.blow` | Blow sound |
| `NSSound.Name.bottle` | Bottle pop |
| `NSSound.Name.frog` | Frog croak |
| `NSSound.Name.funk` | Funk sound |
| `NSSound.Name.glass` | Glass clink |
| `NSSound.Name.hero` | Hero sound |
| `NSSound.Name.morse` | Morse code beep |
| `NSSound.Name.ping` | Ping sound |
| `NSSound.Name.pop` | Pop sound |
| `NSSound.Name.purr` | Purr sound |
| `NSSound.Name.sosumi` | Sosumi alert |
| `NSSound.Name.submarine` | Submarine sonar |
| `NSSound.Name.tink` | Tink sound |

## Constraints & Gotchas

- **Entitlements**: Accessing certain hardware via IOKit may require entitlements. USB device access in a sandboxed app needs the `com.apple.security.device.usb` entitlement. HID device access is generally available without special entitlements.
- **TCC**: Starting macOS 10.15, camera, microphone, and input monitoring access requires TCC consent. Recording audio requires `NSMicrophoneUsageDescription` in Info.plist. Accessing HID devices that might be keyboards requires "Input Monitoring" permission in System Settings > Privacy & Security.
- **Main thread**: IOHIDManager callbacks fire on the run loop it's scheduled on. If you schedule on the main run loop, callbacks arrive on the main thread. Use a dedicated thread or queue for heavy HID processing.
- **IOKit iterator exhaustion**: `IOIteratorNext` consumes the iterator. If you need to re-enumerate, call `IOIteratorReset` or create a new matching query.
- **Power assertions are process-scoped**: If your process crashes without releasing its assertion, the system cleans it up on exit. But in long-running daemons, leaked assertions prevent sleep indefinitely.
- **CoreAudio format matching**: When connecting AVAudioEngine nodes with `format: nil`, the engine uses the node's default format. For multi-channel or non-standard hardware, pass an explicit `AVAudioFormat` to avoid silent failures.
- **USB hot-plug**: USB devices can appear and disappear at any time. Always handle removal callbacks and nil-check interfaces before use. Do not cache `io_object_t` values across event boundaries.
- **IOHIDManager on App Store apps**: The App Store may reject apps using certain IOKit functions. Use `NSOpenPanel` or `IOHIDManager` with known device matching rather than broad enumeration for consumer apps.
- **Audio device enumeration** requires `NSMicrophoneUsageDescription` if enumerating input devices on macOS 14+. The user will be prompted for permission.
- **HAL plugin access**: Direct HAL plugin manipulation (`kAudioHardwarePropertyPlugForCoreAudio`) requires the app to be signed and is not available to sandboxed apps.
- **AVAudioEngine device limitation**: `AVAudioEngine` on macOS can only be assigned to a single aggregate device with both input and output channels. For independent input/output device selection, create an `AudioAggregateDevice` or operate at the AudioUnit level directly.

## Common Mistakes

1. **Forgetting IOObjectRelease**: Every `io_object_t` must be released. Leaking kernel object references can cause resource exhaustion.

2. **Wrong matching dictionary keys**: `kUSBVendorID`/`kUSBProductID` are IOKit keys; `kIOHIDVendorIDKey`/`kIOHIDProductIDKey` are HID manager keys. They are different constants.

3. **Calling IOPMAssertionCreateWithName without IOPMAssertionRelease**: Leaves a permanent sleep assertion until the process exits, draining battery.

4. **Starting AVAudioEngine before connecting nodes**: Always connect and attach all nodes first. If the engine is running and you reconfigure the graph, stop it first (`engine.stop()`), call `engine.reset()`, make changes, then `engine.start()`.

5. **Using NSSound for streaming/real-time audio**: NSSound is fine for short sound effects but not for real-time audio. Use AVAudioEngine or AudioUnits for that.

6. **Ignoring NSSound delegate's `didFinishedPlaying` flag**: When `flag` is `false`, playback was stopped, not completed. Handle both cases.

7. **Accessing IOKit from a sandboxed app without the USB entitlement**: The app will get `KERN_NO_ACCESS` silently. Add `com.apple.security.device.usb` to entitlements and check the return value.

8. **Not scheduling IOHIDManager on a run loop**: Without `IOHIDManagerScheduleWithRunLoop`, no callbacks ever fire.

9. **Using `IORegistryEntryGetPath` on a wrong plane**: Returns empty or misleading results if the specified plane doesn't contain the entry.

10. **Ignoring `kIOPSTimeRemainingUnknown` (-1) and `kIOPSTimeRemainingUnlimited` (-2)**: These are valid return values from time-remaining queries. Don't treat them as errors or negative time.

11. **Wrong HID report buffer size**: Buffer length must exactly match the device's `maxFeatureReportSize`. Even one byte off causes error `0xE0005000`. Filter interfaces by `maxFeatureReportSize` to find the correct one.

12. **Holding strong references to HID drawables without autorelease**: Autorelease pools should wrap HID communication to prevent resource leaks in tight loops.

## Review Checklist

- [ ] All `io_object_t` values are released with `IOObjectRelease`
- [ ] IOHIDManager callbacks are scheduled on a run loop
- [ ] IOPMAssertion pairs (`Create`/`Release`) are balanced
- [ ] USB/vendor matching uses the correct key constants for the API (IOKit vs HID)
- [ ] AVAudioEngine graph is reconfigured only when stopped
- [ ] Power source `CFRelease` is called after `IOPSCopyPowerSourcesInfo`
- [ ] Audio device permissions (`NSMicrophoneUsageDescription`) are declared
- [ ] TCC/IOKit entitlements are included in the app target
- [ ] NSSound delegate handles `didFinishedPlaying: false` (stopped early)
- [ ] Serial port paths use `/dev/cu.*` for outgoing connections
- [ ] HID report buffer sizes match `maxFeatureReportSize` exactly
- [ ] Sleep/wake notification handlers acknowledge or deny with `IOCancelPowerChange`
- [ ] Aggregate devices created when AVAudioEngine needs separate input+output devices

## References

- [IOKit Framework Reference — Apple Developer](https://developer.apple.com/documentation/iokit)
- [AVAudioEngine — Apple Developer](https://developer.apple.com/documentation/avfaudio/avaudioengine)
- [IOPMLib (Power Management) — Apple Developer](https://developer.apple.com/documentation/iokit/iopmlib)
- [Core Audio HAL — Apple Developer](https://developer.apple.com/documentation/coreaudio/core_audio_hardware)
- [NSSound — Apple Developer](https://developer.apple.com/documentation/appkit/nssound)
- [IOHIDManager — Apple Developer](https://developer.apple.com/documentation/iokit/iohidmanager_h)
- [IOHIDDevice — Apple Developer](https://developer.apple.com/documentation/iokit/iohiddevice_h_user-space)
- [kIOPMAssertionTypePreventSystemSleep — Apple Developer](https://developer.apple.com/documentation/iokit/kiopmassertiontypepreventsystemsleep)
- [Technical Note TN2187: New HID Manager APIs](https://developer.apple.com/library/archive/technotes/tn2187/_index.html)
- [Technical Q&A QA1340: Sleep and Wake Notifications](https://developer.apple.com/library/archive/qa/qa1340/_index.html)
- [USB HID on macOS: Talking to Devices with IOKit](https://dev.to/paulcontr_/usb-hid-on-macos-talking-to-devices-with-iokit-3g2n) — Practical HID communication guide
- [USB Device Interface Guide — Apple Developer](https://developer.apple.com/library/archive/documentation/DeviceDrivers/Conceptual/USBBook/)
