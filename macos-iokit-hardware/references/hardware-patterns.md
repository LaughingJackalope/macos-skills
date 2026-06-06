# macOS Hardware Patterns

Complete hardware access patterns for macOS development: IOHIDManager gamepad input, IOPMAssertion for preventing sleep, AVAudioEngine with audio device selection, USB device enumeration, and battery monitoring with power assertions. Targets macOS 26+ with Swift 6.3.

## Contents

- [IOHIDManager for Gamepad Input](#iohidmanager-for-gamepad-input)
- [IOPMAssertion for Preventing Sleep](#iopmassertion-for-preventing-sleep)
- [AVAudioEngine with Audio Device Selection](#avaudioengine-with-audio-device-selection)
- [USB Device Enumeration Pattern](#usb-device-enumeration-pattern)
- [Battery Monitoring and Power Assertions](#battery-monitoring-and-power-assertions)

## IOHIDManager for Gamepad Input

Full gamepad/Joystick input with IOHIDManager. This pattern handles device connection/disconnection events, element discovery, and value parsing with usage pages.

```swift
import IOKit
import IOKit.hid

final class GamepadInputManager {

    // MARK: - Types

    enum Input {
        case axis(AxisID, Float)       // Normalized -1.0...1.0
        case button(Int, Bool)         // Button index, pressed
        case HatSwitch(Int, Float)     // Hat index, angle (0-360°)
    }

    enum AxisID: Int {
        case leftX = 0, leftY, rightX, rightY, leftTrigger, rightTrigger
    }

    typealias InputHandler = (Input) -> Void

    // MARK: - Properties

    private var manager: IOHIDManager!
    private var device: IOHIDDevice?
    private let onInput: InputHandler

    // Cache element usage->index mapping for fast lookup
    private var elementMap: [UInt32: IOHIDElement] = [:]

    // MARK: - Init

    init(onInput: @escaping InputHandler) {
        self.onInput = onInput
        setupManager()
    }

    deinit {
        stop()
    }

    // MARK: - Setup

    private func setupManager() {
        manager = IOHIDManagerCreate(kCFAllocatorDefault, IOOptionBits(kIOHIDOptionsTypeNone))

        // Match joysticks (usage page 1, usage 4) and game pads (usage page 1, usage 5)
        let joystickMatch: [String: Any] = [
            kIOHIDDeviceUsagePageKey: kHIDPage_GenericDesktop,
            kIOHIDDeviceUsageKey: kHIDUsage_GD_Joystick
        ]
        let gamepadMatch: [String: Any] = [
            kIOHIDDeviceUsagePageKey: kHIDPage_GenericDesktop,
            kIOHIDDeviceUsageKey: kHIDUsage_GD_GamePad
        ]

        let matchArray = [joystickMatch, gamepadMatch]
        IOHIDManagerSetDeviceMatchingMultiple(manager, matchArray as CFArray)

        let context = Unmanaged.passUnretained(self).toOpaque()
        IOHIDManagerRegisterDeviceMatchingCallback(manager, deviceConnected, context)
        IOHIDManagerRegisterDeviceRemovalCallback(manager, deviceDisconnected, context)
    }

    func start() {
        IOHIDManagerScheduleWithRunLoop(manager, CFRunLoopGetMain(), CFRunLoopMode.defaultMode.rawValue)
        let result = IOHIDManagerOpen(manager, IOOptionBits(kIOHIDOptionsTypeNone))
        if result != kIOReturnSuccess {
            print("Failed to open HID manager: \(result)")
        }
    }

    func stop() {
        IOHIDManagerClose(manager, IOOptionBits(kIOHIDOptionsTypeNone))
    }

    // MARK: - Element Discovery

    private func discoverElements(for device: IOHIDDevice) {
        elementMap.removeAll()

        guard let elements = IOHIDDeviceCopyMatchingElements(device, nil, IOOptionBits(0))?
            .takeRetainedValue() as? [IOHIDElement] else { return
        }

        for element in elements {
            let usagePage = IOHIDElementGetUsagePage(element)
            let usage = IOHIDElementGetUsage(element)
            let key = (usagePage << 16) | usage
            elementMap[key] = element

            let min = IOHIDElementGetLogicalMin(element)
            let max = IOHIDElementGetLogicalMax(element)
            print("Element: page=\(usagePage) usage=\(usage) range=[\(min),\(max)]")
        }
    }

    // MARK: - Input Handling

    private func handleInputValue(_ value: IOHIDValue) {
        let element = IOHIDValueGetElement(value)
        let intValue = IOHIDValueGetIntegerValue(element)
        let usagePage = IOHIDElementGetUsagePage(element)
        let usage = IOHIDElementGetUsage(element)

        let min = IOHIDElementGetLogicalMin(element)
        let max = IOHIDElementGetLogicalMax(element)
        let range = Float(max - min)
        let normalized = range > 0 ? (Float(intValue - min) / range) * 2.0 - 1.0 : 0

        switch usagePage {
        case kHIDPage_Button:
            onInput(.button(intValue, intValue != 0))

        case kHIDPage_GenericDesktop:
            switch usage {
            case kHIDUsage_GD_X:      onInput(.axis(.leftX, normalized))
            case kHIDUsage_GD_Y:      onInput(.axis(.leftY, normalized))
            case kHIDUsage_GD_Z:      onInput(.axis(.rightX, normalized))
            case kHIDUsage_GD_Rz:     onInput(.axis(.rightY, normalized))
            case kHIDUsage_GD_Rx:     onInput(.axis(.leftTrigger, normalized))
            case kHIDUsage_GD_Ry:     onInput(.axis(.rightTrigger, normalized))
            case kHIDUsage_GD_HatSwitch:
                onInput(.HatSwitch(intValue, Float(intValue) * 45.0))
            default: break
            }

        default: break
        }
    }
}

// MARK: - C Callbacks

private func deviceConnected(
    context: UnsafeMutableRawPointer?,
    result: IOReturn,
    sender: UnsafeMutableRawPointer?,
    device: IOHIDDevice
) {
    guard let ctx = context else { return }
    let manager = Unmanaged<GamepadInputManager>.fromOpaque(ctx).takeUnretainedValue()

    // Read device info
    let product = IOHIDDeviceGetProperty(device, kIOHIDProductKey as CFString) as? String ?? "Unknown"
    let vendor = IOHIDDeviceGetProperty(device, kIOHIDVendorIDKey as CFString) as? Int ?? 0
    print("Connected: \(product) (vendor: 0x\(String(vendor, radix: 16)))")

    manager.device = device
    manager.discoverElements(for: device)

    // Register input callback
    let cbContext = Unmanaged.passUnretained(manager).toOpaque()
    IOHIDDeviceRegisterInputValueCallback(device, inputValue, cbContext)
    IOHIDDeviceScheduleWithRunLoop(device, CFRunLoopGetMain(), CFRunLoopMode.defaultMode.rawValue)
    IOHIDDeviceOpen(device, IOOptionBits(kIOHIDOptionsTypeNone))
}

private func deviceDisconnected(
    context: UnsafeMutableRawPointer?,
    result: IOReturn,
    sender: UnsafeMutableRawPointer?,
    device: IOHIDDevice
) {
    guard let ctx = context else { return }
    let manager = Unmanaged<GamepadInputManager>.fromOpaque(ctx).takeUnretainedValue()
    print("Disconnected")
    manager.device = nil
}

private func inputValue(
    context: UnsafeMutableRawPointer?,
    result: IOReturn,
    sender: UnsafeMutableRawPointer?,
    value: IOHIDValue
) {
    guard let ctx = context else { return }
    let manager = Unmanaged<GamepadInputManager>.fromOpaque(ctx).takeUnretainedValue()
    manager.handleInputValue(value)
}
```

### Usage Notes

- The usage-to-axis mapping (`kHIDUsage_GD_X` → `leftX`) is a convention, not a standard. Different controllers map differently.
- For production use, query the element's `Cookie` for stable identification rather than usage codes, since usage codes may vary.
- For Apple platforms, prefer `GCController` from GameController.framework for standard Xbox/PlayStation controllers. Use IOHIDManager only for devices not supported by GameController.

## IOPMAssertion for Preventing Sleep

Prevent the system from sleeping during critical background work — long downloads, file processing, database migrations. The key principle: create the assertion before the work, and release it *immediately* after, even on failure.

```swift
import IOPMLib

/// Manages a scoped power assertion. Creates on init, releases on deinit.
final class ScopedPowerAssertion {

    private var assertionID: UInt32 = 0
    private let reason: String
    private let assertionType: CFString

    /// - Parameters:
    ///   - reason: Human-visible reason. Shown in `pmset -g assertions`.
    ///   - preventDisplaySleep: If true, also prevents the display from sleeping.
    init(reason: String, preventDisplaySleep: Bool = false) {
        self.reason = reason
        self.assertionType = preventDisplaySleep
            ? kIOPMAssertionTypePreventUserIdleDisplaySleep as CFString
            : kIOPMAssertionTypePreventUserIdleSystemSleep as CFString
        createAssertion()
    }

    deinit {
        releaseAssertion()
    }

    private func createAssertion() {
        let status = IOPMAssertionCreateWithName(
            assertionType,
            IOPMAssertionLevel(kIOPMAssertionLevelOn),
            reason as CFString,
            &assertionID
        )
        if status != kIOReturnSuccess {
            print("Power assertion failed: \(status)")
        }
    }

    private func releaseAssertion() {
        if assertionID != 0 {
            IOPMAssertionRelease(assertionID)
            assertionID = 0
        }
    }
}

// MARK: - Async Scoped Work

func withPowerAssertion<T>(
    reason: String,
    preventDisplaySleep: Bool = false,
    body: () async throws -> T
) async throws -> T {
    let assertion = ScopedPowerAssertion(reason: reason, preventDisplaySleep: preventDisplaySleep)
    let result: T
    do {
        result = try await body()
    } catch {
        // assertion is released by deinit when this scope exits
        throw error
    }
    return result
}

// MARK: - Synchronous scoped helper

func withPowerAssertion<T>(
    reason: String,
    preventDisplaySleep: Bool = false,
    body: () throws -> T
) throws -> T {
    let assertion = ScopedPowerAssertion(reason: reason, preventDisplaySleep: preventDisplaySleep)
    defer { _ = assertion } // deinit releases
    return try body()
}
```

### Practical Patterns: Batch Processing

```swift
// Prevent idle sleep during a batch export
func performBatchExport(items: [ExportItem]) async throws {
    try await withPowerAssertion(reason: "Exporting \(items.count) items") {
        for (index, item) in items.enumerated() {
            try await export(item)
            print("Exported \(index + 1)/\(items.count)")
        }
    }
    // Assertion is released automatically
}
```

### Practical Patterns: Media Playback

```swift
let playbackAssertion = ScopedPowerAssertion(
    reason: "Video playback in progress",
    preventDisplaySleep: true  // Keep display awake for video
)
// ... start playback ...

// When playback ends:
// deinit releases it, or create a property and nil it out.
```

### Assertion Debugging

View live assertions in Terminal:

```bash
pmset -g assertions
# Look for your process name and reason string
```

## AVAudioEngine with Audio Device Selection

Route audio to a specific output device (USB audio interface, Bluetooth speaker, etc.) through AVAudioEngine's audio unit interface.

```swift
import AVFoundation
import CoreAudio

final class DeviceAudioEngine {

    let engine = AVAudioEngine()
    private var selectedDeviceID: AudioDeviceID?

    /// Enumerate all available audio output devices.
    static func listOutputDevices() -> [(name: String, id: AudioDeviceID)] {
        var address = AudioObjectPropertyAddress(
            mSelector: kAudioHardwarePropertyDevices,
            mScope: kAudioObjectPropertyScopeGlobal,
            mElement: kAudioObjectPropertyElementMain
        )

        var size: UInt32 = 0
        AudioObjectGetPropertyDataSize(kAudioObjectSystemObject, &address, 0, nil, &size)

        let count = Int(size) / MemoryLayout<AudioDeviceID>.size
        var devices = [AudioDeviceID](repeating: 0, count: count)
        AudioObjectGetPropertyData(kAudioObjectSystemObject, &address, 0, nil, &size, &devices)

        var results: [(String, AudioDeviceID)] = []

        for devID in devices {
            // Check if this device has output channels
            var outputAddress = AudioObjectPropertyAddress(
                mSelector: kAudioDevicePropertyStreamConfiguration,
                mScope: kAudioDevicePropertyScopeOutput,
                mElement: kAudioObjectPropertyElementMain
            )

            var listSize: UInt32 = 0
            let listSizeResult = AudioObjectGetPropertyDataSize(devID, &outputAddress, 0, nil, &listSize)
            if listSizeResult != noErr || listSize == 0 { continue }

            let bufferList = UnsafeMutablePointer<AudioBufferList>.allocate(capacity: Int(listSize))
            defer { bufferList.deallocate() }
            AudioObjectGetPropertyData(devID, &outputAddress, 0, nil, &listSize, bufferList)

            let numBuffers = Int(bufferList.pointee.mNumberBuffers)
            if numBuffers > 0 {
                // Get device name
                var nameAddress = AudioObjectPropertyAddress(
                    mSelector: kAudioObjectPropertyName,
                    mScope: kAudioObjectPropertyScopeGlobal,
                    mElement: kAudioObjectPropertyElementMain
                )
                var nameSize = UInt32(MemoryLayout<CFString?>.size)
                var cfName: CFString?
                AudioObjectGetPropertyData(devID, &nameAddress, 0, nil, &nameSize, &cfName)

                let name = (cfName as String?) ?? "Unknown"
                results.append((name, devID))
            }
        }

        return results
    }

    /// Set a specific output device for the engine.
    func selectOutputDevice(_ deviceID: AudioDeviceID) throws {
        guard engine.isRunning else {
            selectedDeviceID = deviceID
            return  // Will be applied in start()
        }

        engine.stop()
        try setDeviceOnOutputNode(deviceID)
        selectedDeviceID = deviceID
        try engine.start()
    }

    private func setDeviceOnOutputNode(_ deviceID: AudioDeviceID) throws {
        guard let outputUnit = engine.outputNode.audioUnit else {
            throw AudioEngineError.noAudioUnit
        }

        var deviceID = deviceID
        let status = AudioUnitSetProperty(
            outputUnit,
            kAudioOutputUnitProperty_CurrentDevice,
            kAudioUnitScope_Global,
            0,
            &deviceID,
            UInt32(MemoryLayout<AudioDeviceID>.size)
        )

        guard status == noErr else {
            throw AudioEngineError.coreAudioError(status)
        }
    }

    /// Start the engine, optionally routing to a selected device.
    func start(deviceID: AudioDeviceID? = nil) throws {
        if let devID = deviceID ?? selectedDeviceID {
            try setDeviceOnOutputNode(devID)
        }
        try engine.start()
    }

    func stop() {
        engine.stop()
    }
}

enum AudioEngineError: Error {
    case noAudioUnit
    case coreAudioError(OSStatus)
}
```

### Detailed Note on Device Selection Flow

1. Call `DeviceAudioEngine.listOutputDevices()` to present options to the user.
2. User picks a device. Store the `AudioDeviceID`.
3. Call `engine.stop()` (safely stop the engine if running).  → Call `setDeviceOnOutputNode(deviceID)` to set `kAudioOutputUnitProperty_CurrentDevice`. → Call `engine.start()` to restart.
4. If the engine was already stopped, just set the device and call `start()` — no stop needed.

### Practical Patterns: Monitoring Default Device Changes

```swift
func observeDefaultOutputDeviceChanges() {
    var address = AudioObjectPropertyAddress(
        mSelector: kAudioHardwarePropertyDefaultOutputDevice,
        mScope: kAudioObjectPropertyScopeGlobal,
        mElement: kAudioObjectPropertyElementMain
    )

    AudioObjectAddPropertyListenerBlock(kAudioObjectSystemObject, &address, .main) { _, _ in
        print("Default audio output device changed")
        // Re-route if you follow system default
    }
}
```

## USB Device Enumeration Pattern

Enumerate all USB devices, filtering by vendor/product ID, and read extended properties from the IORegistry.

```swift
import IOKit
import IOKit.usb

struct USBDeviceDescription {
    let vendorID: Int
    let productID: Int
    let productName: String
    let vendorName: String
    let serialNumber: String
    let deviceAddress: Int
    let portNumber: Int
}

final class USBDeviceEnumerator {

    /// Enumerate all USB devices matching an optional vendor/product filter.
    static func enumerate(
        vendorID: Int? = nil,
        productID: Int? = nil
    ) -> [USBDeviceDescription] {
        var results: [USBDeviceDescription] = []

        var matchingDict = IOServiceMatching(kIOUSBDeviceClassName) as NSMutableDictionary

        if let vid = vendorID {
            matchingDict[kUSBVendorID] = vid
        }
        if let pid = productID {
            matchingDict[kUSBProductID] = pid
        }

        var iterator: io_iterator_t = 0
        let kr = IOServiceGetMatchingServices(kIOMainPortDefault, matchingDict, &iterator)
        guard kr == KERN_SUCCESS else { return results }

        defer { IOObjectRelease(iterator) }

        var service = IOIteratorNext(iterator)
        while service != 0 {
            defer { IOObjectRelease(service) }

            let readInt: (String) -> Int? = { key in
                IORegistryEntryCreateCFProperty(
                    service, key as CFString, kCFAllocatorDefault, 0
                )?.takeRetainedValue() as? Int
            }

            let readString: (String) -> String = { key in
                (IORegistryEntryCreateCFProperty(
                    service, key as CFString, kCFAllocatorDefault, 0
)?.takeRetainedValue() as? String) ?? ""
            }

            if let desc = USBDeviceDescription(
                vendorID: readInt("idVendor") ?? 0,
                productID: readInt("idProduct") ?? 0,
                productName: readString("USB Product Name"),
                vendorName: readString("USB Vendor Name"),
                serialNumber: readString(kUSBSerialNumberString),
                deviceAddress: readInt("USB Address") ?? 0,
                portNumber: readInt("PortNum") ?? 0
            ) as USBDeviceDescription? {
                results.append(desc)
            }

            service = IOIteratorNext(iterator)
        }

        return results
    }

    /// Enumerate hubs and their ports to find a specific device's bus path.
    static func findBusPath(for device: USBDeviceDescription) -> String? {
        let hubDict = IOServiceMatching("IOUSBHubDevice") as NSMutableDictionary
        var iterator: io_iterator_t = 0
        guard IOServiceGetMatchingServices(kIOMainPortDefault, hubDict, &iterator) == KERN_SUCCESS else {
            return nil
        }
        defer { IOObjectRelease(iterator) }

        var hub = IOIteratorNext(iterator)
        while hub != 0 {
            defer { IOObjectRelease(hub) }
            if let portNum = IORegistryEntryCreateCFProperty(
                hub, "PortNum" as CFString, kCFAllocatorDefault, 0
            )?.takeRetainedValue() as? Int, portNum == device.portNumber {
                if let locationID = IORegistryEntryCreateCFProperty(
                    hub, kUSBDevicePropertyLocationID as CFString, kCFAllocatorDefault, 0
                )?.takeRetainedValue() as? UInt32 {
                    return String(format: "0x%08x", locationID)
                }
            }
            hub = IOIteratorNext(iterator)
        }
        return nil
    }
}
```

### Important USB Gotchas

- **Iterator consumption**: Each `IOIteratorNext` advances the iterator. If you must pass through twice, use `IOIteratorReset`.
- **Hotplug**: USB devices can disconnect between enumeration and use. Always check return values on IOCalloutDevice operations previously-held references become stale after `IOObjectRelease`.
- **Composite devices**: A single USB device may have multiple interfaces (e.g., a webcam + microphone). These appear as separate children in the IORegistry. Iterate children to find interface-level entries.

## Battery Monitoring and Power Assertions

Combine battery monitoring with conditional power assertion management — e.g., pause background processing on low battery, or prevent sleep only when on AC power.

```swift
import IOPMLib
import Combine

final class PowerStateMonitor: ObservableObject {

    // MARK: - Published State

    @Published var isOnBattery: Bool = false
    @Published var batteryPercent: Int?
    @Published var isCharging: Bool = false
    @Published var estimatedMinutesRemaining: Int?
    @Published var warningLevel: IOPSLBatteryWarningLevel = kIOPSLBatteryWarningNone

    // MARK: - Power Assertion

    private var assertionID: UInt32 = 0

    // MARK: - Polling

    private var timer: Timer?

    func startPolling(interval: TimeInterval = 30) {
        refresh()
        timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { [weak self] _ in
            self?.refresh()
        }
    }

    func stopPolling() {
        timer?.invalidate()
        timer = nil
    }

    // MARK: - Refresh

    private func refresh() {
        guard let blob = IOPSCopyPowerSourcesInfo(),
              let sources = IOPSCopyPowerSourcesList(blob)?.takeRetainedValue() as? [CFTypeRef],
              let first = sources.first,
              let desc = IOPSGetPowerSourceDescription(blob, first)?.takeUnretainedValue() as? [String: Any]
        else {
            CFRelease(blob)
            return
        }
        CFRelease(blob)

        // Power source state
        let state = desc[kIOPSPowerSourceStateKey as String] as? String
        isOnBattery = (state == kIOPSProvidedValues.BatteryPower.rawValue)
        isCharging = desc[kIOPSIsChargingKey as String] as? Bool ?? false

        // Battery percentage
        let current = desc[kIOPSCurrentCapacityKey as String] as? Int
        let max = desc[kIOPSMaxCapacityKey as String] as? Int
        if let cur = current, let mx = max, mx > 0 {
            batteryPercent = Int((Double(cur) / Double(mx)) * 100)
        } else {
            batteryPercent = nil
        }

        // Time remaining
        let rawTime = desc[kIOPSTimeRemainingKey as String] as? Int ?? 0
        if rawTime == Int(kIOPSTimeRemainingUnknown) {
            estimatedMinutesRemaining = nil
        } else if rawTime == Int(kIOPSTimeRemainingUnlimited) {
            estimatedMinutesRemaining = Int.max  // On AC, charging
        } else {
            estimatedMinutesRemaining = rawTime
        }

        // Warning level
        warningLevel = IOPSGetBatteryWarningLevel()
    }

    // MARK: - Conditional Power Assertion

    /// Prevent system sleep, but only when it makes sense given battery state.
    func updatePowerAssertion(
        workActive: Bool,
        minimumBatteryPercent: Int = 20,
        reason: String = "Background processing"
    ) {
        let hasBattery = batteryPercent != nil
        let batteryTooLow = (batteryPercent ?? 100) < minimumBatteryPercent
        let shouldBeActive = workActive && (!hasBattery || !isOnBattery || !batteryTooLow)

        if shouldBeActive && assertionID == 0 {
            let status = IOPMAssertionCreateWithName(
                kIOPMAssertionTypePreventUserIdleSystemSleep as CFString,
                IOPMAssertionLevel(kIOPMAssertionLevelOn),
                reason as CFString,
                &assertionID
            )
            if status == kIOReturnSuccess {
                print("Power assertion active: \(reason)")
            }
        } else if !shouldBeActive && assertionID != 0 {
            IOPMAssertionRelease(assertionID)
            assertionID = 0
            print("Power assertion released")
        }
    }

    deinit {
        if assertionID != 0 {
            IOPMAssertionRelease(assertionID)
        }
    }
}
```

### Practical Patterns: Low-Battery-Aware Task Queue

```swift
// In your app's task manager:
let powerMonitor = PowerStateMonitor()
powerMonitor.startPolling()

// When dispatching work:
powerMonitor.updatePowerAssertion(
    workActive: !taskQueue.isEmpty,
    minimumBatteryPercent: 15,
    reason: "\(taskQueue.count) pending uploads"
)

// React to warning levels:
@Published var warningLevel: IOPSLBatteryWarningLevel = kIOPSLBatteryWarningNone

// In the polling callback:
switch warningLevel {
case kIOPSLBatteryWarningEarly:
    notifyUser("Battery low — background tasks will pause at 10%")
case kIOPSLBatteryWarningFinal:
    taskQueue.pause()
    notifyUser("Battery critically low — tasks paused")
case kIOPSLBatteryWarningNone:
    taskQueue.resume()
@unknown default: break
}
```
