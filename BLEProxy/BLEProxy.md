# Bluetooth Low Energy logging by placing a Mac-in-the-middle
## Why?
In the past two years or so I have been delving into Bluetooth Low Energy (BLE) for a project I did for one of our customers at Luminis Arnhem. In this project we have been tasked with implementing mobile applications for Android and iOS that used BLE to communicate with various products made by our customer. Because of this, a lot of my focus has gone to the BLE stacks that Apple and Google have created on their platforms for application developers to use. These stacks take care of the lower levels of the BLE protocol for developers and in very many ways this is a good thing. I don't know if it is even possible, without first having to hack your own device, to completely create your own BLE stack for these platforms on use that on iOS and Android devices instead but I certainly would not recommend you to try it.

On issue with using these stacks however, is that they make it hard as a developer to know what is actually being communicated at the protocol level. The API's that are offered are pretty high level so that they are easy to use, which is good. However the documentation provided with the API's, at least the documentation I have been able to find, is not very descriptive or transparant about how calling those API's translates to actual communication in the BLE protocol. So when your connection does not act as you expected it would and chances are that it will, what you really want to do is take a look at the packets that are actually being transferred and their contents.

## What's Mac got to do with it?
Depending on the platform that your developing for there may already be options available that you can use to get access to such logging without having to put in a lot of effort. For iOS devices with an OS version higher than 13 for example, [Apple already offers a solution](https://www.bluetooth.com/blog/a-new-way-to-debug-iosbluetooth-applications/) using PacketLogger that is fairly easy to set up. I have not actually tried this solution myself yet, but as the solution I am about to propose builds on some of the same concepts I can say with some confidence that this will work. In fact if iOS is the only target you are focussed on I would probably advise you to use this and skip my solution (Keep readin though! It never hurts to learn).

If you are also targetting Android, the challenge becomes a lot bigger. If you google `logging ble on android` you will find plenty of blogs that explain your how to enable [Bluetooth HCI snoop logging](https://source.android.com/devices/bluetooth/verifying_debugging#debugging-with-bug-reports) to achieve this, however I have a few problems with this. First of all the procedure requires a couple of manual steps on the device that you want to log which arent always clear. The official Android documentation and most blogs seem to agree on changing two settings and power cycling your Bluetooth connection (turn Bluetooth off and then on again), however I have also read blogs that insisted you needed to reboot your whole device to make it work. The location on your device where the logs are stored also tends to differ. It could be that these problems are caused by by different Android OS versions or Android devices. The second issue that I have with this solution is that it simply logs everything your device does over Bluetooth to a file and leaves you with the task of figuring out where that one bit of communication your were actually interested in went. There is no option to view the logs during the connection and any filtering that you may want to do on the traffic completely relies on you to find a tool or way to do that. All in all capturing relevant BLE traffic using this method is a exercise in patience I have yet to succeed at.

Another option you could consider is to get what is called a packet sniffer, one that supports BLE. Packet sniffers are devices that allow you to listen in on traffic. Usually you can plug them into your computer and they come with software that allow you to view the traffic and filter it. The problem with this option is that there are actually two types of such devices for BLE, singleband and multiband sniffers. You can buy a singleband packet sniffer for around 50 euros which is afforable enough, but these do not really work well for our purposes. A connection between two devices over Bluetooth Low Energy tends to jump between bands multiple times during the connection. As singleband sniffers by their design are only able to listen in on one band at a time you will find it difficult if not impossible to capture a complete connection. Multiband packet sniffers on the other are able to listen in on multiple bands at the same time and should therefore not have these difficulties. Multiband packet sniffers however are a lot harder to come by and when you do find them they can easily cost ten times as much as a singleband sniffer.

So the Mac then? Earlier when talking about logging BLE traffic for iOS I already mentioned PacketLogger. PacketLogger is a tool that comes with the [Additional Tools for XCode](https://developer.apple.com/download/more/?=xcode). It allows you to view a live log of all the Bluetooth traffic going to and from your Macbook. Various BLE protocol layers and message types are colored differently to easily detect certain messages and you can also filter the traffic in a few ways and save log files to view them later on. It is a great tool and I have used it a lot to analyse problems in BLE connections. But PacketLogger can only log traffic to and from your Mac, how will this help us log traffic between our application and another device? Well XCode allows you to create applications that run on your Mac and using CoreBluetooth, the BLE stack that Apple has created for iOS and MacOS applications, it is not that hard to create a small application that acts as a proxy between the peripheral and central in your BLE connection, provided you know some basic information about the peripheral you want to use this on. The information you need to know is:
- The name the peripheral advertises with.
- The password for the peripheral, if it uses a secure BLE connection.
- The Services offered by the peripheral, or at least the ones that you wish to use in your app.
- The Characteristics on the Services offered by the peripheral. Once again you would at least need to know about the ones you wish to use in your app.

If you plan on developing an application that communicates with your peripheral then you will probably have all of this information already.

## Let's get started
### Project setup
The first step is to create a new MacOS App project.

![alt text](./new_project.png "Create new project")

After going through the new project Wizard you should have something like this:

![alt text](./new_project_2.png "New project overview")

### Building the UI
This application does not need much of a UI, but its useful to have some. I bet we could use:
- A button to start the proxy
- A button to stop the proxy
- Some textfield where we can output useful logging
- A button to clear the logs

So let's open up the Main.storyboard of our project and add these components so that it looks like this:

![alt text](./app_layout.png "App UI layout")

Next connect the IBActions and IBOutlets for the views you added so that you get the following in the `ViewController` class.

```swift
import Cocoa

class ViewController: NSViewController {
    
    @IBOutlet weak var startProxyButton: NSButton!
    @IBOutlet weak var stopProxyButton: NSButton!
    @IBOutlet weak var logView: NSTextField!
    
    override func viewDidLoad() {
        super.viewDidLoad()

        // Do any additional setup after loading the view.
    }

    override var representedObject: Any? {
        didSet {
        // Update the view, if already loaded.
        }
    }

    @IBAction func startProxy(_ sender: Any) {
    }
    
    @IBAction func stopProxy(_ sender: Any) {
    }
    
    @IBAction func clearLog(_ sender: Any) {
    }
}

```

### Creating the BLECentral
Now that we have the project setup and UI done, the next thing we need is to create the central that will connect to your peripheral device.

So lets create a new file in our project that we call `BleCentral.swift`. In this file we will add two things, the BleCentral class and a delegate protocol for this class.

Let's start with the delegate protocol. The BleCentral that we are about to make needs to be able to communicate a couple of things to its delegate, these are:
- That it has connected to the peripheral
- That it has disconnected from the peripheral, including a reason why
- That it has received data from the peripheral
- That it has written data to the peripheral
- Log messages that could be useful to print in the logView

So we create a `BleCentralDelegate` protocol at the top of the `BleCentral.swift` that contains the following:
```swift
protocol BleCentralDelegate {
    func connected(services: [BleService])
    func disconnected(reason: String)
    func dataWritten(onCharacteristicWithUUID uuid: CBUUID, withResult result: CBATTError.Code)
    func dataReceived(data: Data, onCharacteristicWithUUID uuid: CBUUID)
    func logMessage(message: String)
}
```

Below this delegate we will create a new class `BleCentral`. This class will have a `var delegate` of the protocol we created. This class will also use `CBCentralManager` to scan for peripherals and connect to our peripheral device, so we create a `private var centralManager: CBCentralManager?` in the class for that. Don't forget to add `import CoreBluetooth` at the top of `BleCentral.swift` to be able to use the CoreBluetooth classes. 

In order for the centralmanager to be able to communicate back to this class we also need it to be an `NSObject` and implement `CBCentralManagerDelegate`. This gives our class an `init()` method where we can initialise the centralmanager var we added.

Also let's give this class public methods so that it can be told to:
- Connect to the peripheral
- Stop the connection to the peripheral
- Read data from a characteristic on the peripheral
- Write data to a characteristic on the peripheral
- Register for notifications on characteristic updates
- Unregister from notifications on characteristic updates

All of the above means we add the following code to `BleCentral.swift` below the `BleCentralDelegate` protocol:
```swift
class BleCentral: NSObject, CBCentralManagerDelegate {
    
    var delegate: BleCentralDelegate?

    private var centralManager: CBCentralManager?
    
    override init() {
        super.init()
        self.centralManager = CBCentralManager(delegate: self, queue: nil)
    }
    
    func connect() {
        
    }
    
    func disconnect() {
        
    }
    
    func readData(characteristicUUID: CBUUID) {
        
    }
    
    func writeData(characteristicUUID: CBUUID, data: Data, writeType: CBCharacteristicWriteType) {
        
    }

    func registerForNotifications(characteristicUUID: CBUUID) {

    }

    func unregisterFromNotifications(characteristicUUID: CBUUID) {

    }
    
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        
    }
}
```

The `centralManagerDidUpdateState` method was added as it is required for classes implementing `CBCentralManagerDelegate`.

The first thing the central needs to do when it is told to connect is to start scanning for BLE peripherals. However it is only allowed to do this when the centralmanager is in the `.poweredOn` state and we should take care not to start scanning when a scan is already running. To do all this, change the `connect` method to the following:
```swift
func connect() {
    self.peripheralName = BleConstants.DEVICE_NAME
    if self.centralManager?.state == .poweredOn && !(self.centralManager?.isScanning ?? false) {
        self.delegate?.logMessage(message: "Started scanning for BLE peripherals.")
        self.centralManager?.scanForPeripherals(withServices: nil, options: nil)
    } else {
        self.delegate?.logMessage(message: "BLE is not powered on (yet).")
    }
} 
```

With the code above, when the centralmanager is not in the `.poweredOn` state when the central is asked to connect, it will not start scanning for peripheral and therefore not connect it to one. However we did tell it to do so. To fix this, we can use the `centralManagerDidUpdateState` method that we were required to implement. Change this method so that it looks like this:
```swift
func centralManagerDidUpdateState(_ central: CBCentralManager) {
    if central.state == .poweredOn && self.peripheralName != nil && !central.isScanning {
        self.delegate?.logMessage(message: "BLE is powered on now, started scanning for BLE peripherals.")
        central.scanForPeripherals(withServices: nil, options: nil)
    }
}
```

Now the central will also start scanning for peripherals when the state of centralmanager changes to `.poweredOn`. It will do this only when `self.peripheralName` is not `nil` which indicates that the central was asked to connect and when it is not scanning already.

Wait a minute. The `BleCentral` class we have been working on so far has no `self.peripheralName` and what is `BleConstants.DEVICE_NAME` ? You are correct to ask such questions as we have not added these two bits yet. For the first, create a `private var peripheralName` of type `String?` just below the existing centralmanager var that we already had. For the second we will add a new file called `BLEConstants.swift` to our project and fill it with the following:
```swift
import Foundation

class BleConstants {
    static let DEVICE_NAME = "<the BLE advertising name of your BLE peripheral device>"
}
```

Before we continue the connection process, let's first make sure that we can also stop scanning when the `disconnect` method is called on the central. To do this, add the following:
```swift
func disconnect() {
    self.peripheralName = nil
    self.stopScanning()
}

private func stopScanning() {
    if (self.centralManager?.isScanning ?? false) {
        self.delegate?.logMessage(message: "Stopped scanning for BLE peripherals.")
        self.centralManager?.stopScan()
    }
}
```

The reason that we put some of these lines in a separate private function, is that we will need to call this function more often.

If the centralmanager has found BLE peripheral devices, our central can be notified of this by implementing one of the optional methods in `CBCentralManagerDelegate`, called `centralManager:didDiscoverPeripheral:advertisementData:RSSI`. This method is called for every BLE peripheral that is found. When it is called and the peripheral name matches the name of our peripheral device we want to stop scanning and connect to the peripheral. We do this by adding:
```swift
func centralManager(_ central: CBCentralManager, didDiscover peripheral: CBPeripheral, advertisementData: [String : Any], rssi RSSI: NSNumber) {
    guard let name = peripheral.name?.lowercased(), let gapName = (advertisementData[CBAdvertisementDataLocalNameKey] as? String)?.lowercased(), let peripheralName = self.peripheralName?.lowercased() else { return }
    self.delegate?.logMessage(message: "BLE peripheral found with names: [\(name),\(gapName)].")
    if peripheral.state != .connected && (name == peripheralName || gapName == peripheralName) {
        self.stopScanning()
        self.delegate?.logMessage(message: "Connecting to peripheral \(peripheral).")
        self.peripheral = peripheral
        self.peripheral?.delegate = self
        self.centralManager?.connect(peripheral, options: nil)
    }
}
```

Just as with `self.peripheralName` earlier, `self.peripheral` does not yet exist. To solve this add it as `private var peripheral: CBPeripheral?` below peripheralName var you added earlier.

The central we have created also does not yet implement the correct delegate for this peripheral, so let's add `CBPeripheralDelegate` to the protocols this class implements.

When the centralmanager has either connected or failed to connect to the peripheral device, the central can once again be notified of this by implementing optional `CBCentralManagerDelegate` methods. If the connection was a success, the next step is to start discovering services on the peripheral. If the connection fails there is not much to do but inform our delegate. Therefore below the previus method, add the following:
```swift
func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
    self.delegate?.logMessage(message: "Connected to peripheral \(peripheral), discovering services.")
    self.peripheral?.discoverServices(nil)
}

func centralManager(_ central: CBCentralManager, didFailToConnect peripheral: CBPeripheral, error: Error?) {
    self.delegate?.disconnected(reason: "Connection to peripheral \(peripheral) failed with error: \(error.debugDescription)")
}
```

Now that we can connect to the peripheral we should also make sure we can disconnect from it when asked to do so. To do this, change the earlier implementation of `disconnect` to this:
```swift
 func disconnect() {
    self.peripheralName = nil
    self.stopScanning()
    if let peripheral = self.peripheral {
        if peripheral.state == .connected || peripheral.state == .connecting {
            self.centralManager?.cancelPeripheralConnection(peripheral)
        }
    }
}
```

Also to inform our the delegate that we did indeed disconnect from the peripheral, add the following below the existing methods of the central:

```swift
func centralManager(_ central: CBCentralManager, didDisconnectPeripheral peripheral: CBPeripheral, error: Error?) {
    if let error = error {
        self.delegate?.disconnected(reason: "Connection to peripheral \(peripheral) closed with error: \(error).")
    } else {
        self.delegate?.disconnected(reason: "Connection to peripheral \(peripheral) closed.")
    }
}
```

Discovering services once again happens through a delegate method, although this time the method is an optional method on the `CBPeripheralDelegate`. Before we can start implementing this method however we need to create a way to track that all services and later characteristics were found. To do this, add `static var SERVICES_AND_CHARACTERISTICS` to your existing BleConstants class and initialise it as:
```swift
static var SERVICES_AND_CHARACTERISTICS = [
    // Services
    CBUUID(string: "<uuid of the service>"): [
        // Characteristics
        CBUUID(string: "<uuid of the characteristic>"),
        CBUUID(string: "<uuid of another characteristic>"),
        <additional characteristics>
    ],
    <additional services>
]
```

Create a new file called `BleService.swift` and give it the following contents:
```swift
import Foundation
import CoreBluetooth

class BleService: Hashable {
    init(uuid: CBUUID) {
        self.uuid = uuid
    }
    
    let uuid: CBUUID
    var service: CBService?
    var characteristics: [BleCharacteristic]?
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(uuid)
    }
    
    static func == (lhs: BleService, rhs: BleService) -> Bool {
        return lhs.uuid == rhs.uuid
    }
}
```

Create a new file called `BLECharacteristic.swift` and give it the following contents:

```swift
import Foundation
import CoreBluetooth

class BleCharacteristic {
    init(uuid: CBUUID) {
        self.uuid = uuid
    }
    
    let uuid: CBUUID
    var characteristic: CBCharacteristic?
}
```

Back in the central add `private var services: [BleService]?` to the existing vars. Also change the `connected` method of the `BleCentralDelegate` to be `connected(services: [BleService])` and add the following method to the BleCentral class:
```swift
private func setupServicesAndCharacteristics() {
    self.services = BleConstants.SERVICES_AND_CHARACTERISTICS.map({ (key: CBUUID, value: [CBUUID]) -> BleService in
        let service = BleService(uuid: key)
        service.characteristics = value.map({ (uuid) -> BleCharacteristic in
            return BleCharacteristic(uuid: uuid)
        })
        return service
    })
}
```

This method should be called in the `centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral` we already implemented, right before calling `self.peripheral?.discoverServices(nil)`. With this done, we can implement the `CBPeripheralDelegate` method `peripheral:(CBPeripheral *)peripheral didDiscoverServices:(nullable NSError *)error` with:
```swift
func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
    if let error = error {
        self.delegate?.disconnected(reason: "Peripheral \(peripheral) service discovery failed with error: \(error).")
        self.disconnect()
    } else if let services = peripheral.services, let expectedServices = self.services {
        for service in services {
            if let expectedService = expectedServices.first(where: { (expectedService) -> Bool in
                service.uuid == expectedService.uuid
            }) {
                expectedService.service = service
                self.delegate?.logMessage(message: "Service \(service.uuid.uuidString) found, discovering characteristics.")
                peripheral.discoverCharacteristics(nil, for: service)
            }
        }
    } else {
        self.delegate?.disconnected(reason: "Peripheral \(peripheral) has no services.")
        self.disconnect()
    }
}
```

In the implementation above we do a couple of things. First we check if there was an error and disconnect from the peripheral if this is the case. We also check if any services were found on the peripheral. If this is not the case it probably means something went wrong so we also disconnect from the peripheral. If there was no error and services were discovered, we check each of those services to see if they match the services we expected. When this is true we add the CBService instance to the BleService object and start discovering characteristics for the service.

Like with services, discovered characteristics are also receive though a callback on the `CBPeripheralDelegate`. In this case `peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(nullable NSError *)error`. Let's implement this method as well with the following contents:
```swift
func peripheral(_ peripheral: CBPeripheral, didDiscoverCharacteristicsFor service: CBService, error: Error?) {
    if let error = error {
        self.delegate?.disconnected(reason: "Peripheral \(peripheral) and service \(service.uuid.uuidString) characteristic discovery failed with error: \(error).")
        self.disconnect()
    } else if let characteristics = service.characteristics {
        if let expectedService = self.services?.first(where: { (expectedService) -> Bool in
            service.uuid == expectedService.uuid
        }) {
            for characteristic in characteristics {
                if let expectedCharacteristic = expectedService.characteristics?.first(where: { (expectedCharacteristic) -> Bool in
                    characteristic.uuid == expectedCharacteristic.uuid
                }) {
                    expectedCharacteristic.characteristic = characteristic
                }
            }
        }
        if let services = self.services, self.servicesAndCharacteristicsComplete(services) {
            self.delegate?.connected(services: services)
        }
    } else {
        self.delegate?.disconnected(reason: "Peripheral \(peripheral) and service \(service.uuid.uuidString) have no characteristics.")
        self.disconnect()
    }
}
```

Once again we first check if there was an error and disconnect if this is the case. We also check if we were able to discover any characteristics for the service and if not, we treat it as an error and disconnect. If there was no error and characteristics were discovered we check them to see if they match characteristics we expect. If so we add the CBCharacteristic instance to the BleCharacteristic object. In the end we check if `servicesAndCharacteristicsComplete()` returns true. If this is the case, we can notify our delegate that we are fully connected to the periperhal.

The method `servicesAndCharacteristicsComplete()` is one we do not yet have in the central, so lets add it:
```swift
private func servicesAndCharacteristicsComplete(_ services: [BleService]) -> Bool {
    return services.allSatisfy({ (bleService) -> Bool in
        return bleService.service != nil && bleService.characteristics?.allSatisfy({ (bleCharacteristic) -> Bool in
            return bleCharacteristic.characteristic != nil
        }) ?? false
    })
}
```

Now that our central is able to connect to the peripheral device, it is mostly done. You have probably noticed however that we have left a few of the public methods we created in the beginning unimplemented so far. So let's add implementations for these methods to finish up the central.

We start with reading data from a characteristic. To do this we added the public method `readData(characteristicUUID: CBUUID)` which we can implement like this:
``` swift
func readData(characteristicUUID: CBUUID) {
    if let peripheral = self.peripheral, let characteristic = findCharacteristic(characteristicUUID) {
        self.delegate?.logMessage(message: "Reading from peripheral on characteristic: \(characteristicUUID.uuidString)")
        peripheral.readValue(for: characteristic)
    }
}

private func findCharacteristic(_ characteristicUUID: CBUUID) -> CBCharacteristic? {
    if let services = self.services {
        for service in services {
            if let bleCharacteristic = service.characteristics?.first(where: { (bleCharacteristic) -> Bool in
                return bleCharacteristic.uuid == characteristicUUID
            }) {
                return bleCharacteristic.characteristic
            }
        }
    }
    return nil
}
```

We also have the option to write data to a characteristic, for which we added the method `writeData(characteristicUUID: CBUUID, data: Data, writeType: CBCharacteristicWriteType)`. Implement that as:
``` swift
func writeData(characteristicUUID: CBUUID, data: Data, writeType: CBCharacteristicWriteType) {
    if let peripheral = self.peripheral, let characteristic = findCharacteristic(characteristicUUID) {
        self.delegate?.logMessage(message: "Writing to peripheral on characteristic: \(characteristicUUID.uuidString) -> \(data.hexEncodedString())")
        peripheral.writeValue(data, for: characteristic, type: writeType)
    }
}
```

Adding this will give you a compiler error that `Value of type 'Data' has no member 'hexEncodedString'`. We can add that by create a new file called `Data+Hex.swift` and giving it the following contents:
```swift
import Foundation

extension Data {
    struct HexEncodingOptions: OptionSet {
        let rawValue: Int
        static let upperCase = HexEncodingOptions(rawValue: 1 << 0)
    }
    
    func hexEncodedString(options: HexEncodingOptions = []) -> String {
        let format = options.contains(.upperCase) ? "%02hhX" : "%02hhx"
        return map { String(format: format, $0) }.joined()
    }
}
```

In the case of a confirmed write in BLE, the delegate of our central needs to know that the write request was completed and what the result was. Our central can be informed of this through the `CBPeripheralDelegate` method `peripheral:(CBPeripheral *)peripheral didWriteValueForCharacteristic:(CBCharacteristic *)characteristic error:(nullable NSError *)error`, so let's implement that as well:
```swift
func peripheral(_ peripheral: CBPeripheral, didWriteValueFor characteristic: CBCharacteristic, error: Error?) {
    if let error = error {
        self.delegate?.disconnected(reason: "Writing data to peripheral on characteristic \(characteristic.uuid.uuidString) failed with error: \(error).")
        self.delegate?.dataWritten(onCharacteristicWithUUID: characteristic.uuid, withResult: CBATTError.unlikelyError)
    } else {
        self.delegate?.dataWritten(onCharacteristicWithUUID: characteristic.uuid, withResult: CBATTError.success)
    }
}
```

The last methods we need to fill in are `registerForNotifications(characteristicUUID: CBUUID)` and `unregisterFromNotifications(characteristicUUID: CBUUID)`, which we will implement as follows:
```swift
func registerForNotifications(characteristicUUID: CBUUID) {
    if let peripheral = self.peripheral, let characteristic = findCharacteristic(characteristicUUID) {
        if characteristic.properties.contains(.notify) || characteristic.properties.contains(.indicate) {
            peripheral.setNotifyValue(true, for: characteristic)
        } else {
            delegate?.logMessage(message: "registerForNotifications requested for characteristic \(characteristicUUID) that does not allow such actions.")
        }
    }
}

func unregisterFromNotifications(characteristicUUID: CBUUID) {
    if let peripheral = self.peripheral, let characteristic = findCharacteristic(characteristicUUID) {
        if characteristic.properties.contains(.notify) || characteristic.properties.contains(.indicate) {
            peripheral.setNotifyValue(false, for: characteristic)
        } else {
            delegate?.logMessage(message: "registerForNotifications requested for characteristic \(characteristicUUID.uuidString) that does not allow such actions.")
        }
    }
}
```

Just like we have already seen many times in this central, there is a `CBPeripheralDelegate` method that can give us feedback on the notification registration. In this case this is the method `peripheral:(CBPeripheral *)peripheral didUpdateNotificationStateForCharacteristic:(CBCharacteristic *)characteristic error:(nullable NSError *)error`. Apart from handling an error, the delegate of our central cannot really do much with this information, but we should implement the method to at least log the results:
``` swift
func peripheral(_ peripheral: CBPeripheral, didUpdateNotificationStateFor characteristic: CBCharacteristic, error: Error?) {
    if let error = error {
        self.delegate?.disconnected(reason: "Notification state update failed for characteristic \(characteristic.uuid.uuidString) with error: \(error).")
        self.disconnect()
    } else if (characteristic.isNotifying) {
        delegate?.logMessage(message: "Now receiving notifications for characteristic \(characteristic.uuid.uuidString)")
    } else {
        delegate?.logMessage(message: "No longer receiving notifications for characteristic \(characteristic.uuid.uuidString)")
    }
}
```

And with this, the central is done.

### Using the BLECentral
Now that we have created the class that implements the central for us, we need to integrate it into our application so that we can launch and test it. Open up `ViewController.swift` and change the contents to this:
```swift
import Cocoa
import CoreBluetooth

class ViewController: NSViewController, BleCentralDelegate {
    
    @IBOutlet weak var startProxyButton: NSButton!
    @IBOutlet weak var stopProxyButton: NSButton!
    @IBOutlet weak var logView: NSTextField!
    
    var bleCentral: BleCentral?
    
    override func viewDidLoad() {
        super.viewDidLoad()

        self.bleCentral = BleCentral()
        self.bleCentral?.delegate = self
    }

    @IBAction func startProxy(_ sender: Any) {
        startProxyButton.isEnabled = false
        stopProxyButton.isEnabled = true
        self.bleCentral?.connect()
    }
    
    @IBAction func stopProxy(_ sender: Any) {
        startProxyButton.isEnabled = true
        stopProxyButton.isEnabled = false
        self.bleCentral?.disconnect()
    }
    
    @IBAction func clearLog(_ sender: Any) {
        self.logView.stringValue = ""
    }
    
    func connected(services: [BleService]) {
        log("Connected to peripheral")
    }
    
    func disconnected(reason: String) {
        log("Stopped because: \(reason)")
    }
    
    func dataWritten(onCharacteristicWithUUID: CBUUID, withResult: CBATTError.Code) {
        log("Data written from central to peripheral on characteristic \(onCharacteristicWithUUID.uuidString) with result: \(withResult)")
    }
    
    func dataReceived(data: Data, onCharacteristicWithUUID: CBUUID) {
        log("Received \(data) from peripheral to pass to central on characteristic \(onCharacteristicWithUUID.uuidString)")
    }
    
    func logMessage(message: String) {
        log(message)
    }
    
    private func log(_ message: String) {
        logView.stringValue.append("\(message)\n")
    }
}
```

As you can see, we declare and initialise the BleCentral and we register the ViewController as delegate. At the moment we cannot yet do much when the delegate methods are called except log that it happens. What we can do is tell the BleCentral to connect when the proxy is started and disconnect when it is stopped and this means we can start our proxy for the first time to test if we can connect to the peripheral device.

Before we can do our first test run we need to set one more thing. Open the `.xcodeproj` for your project and on the tab `Signing and capabilities` under `App Sandbox` check `Bluetooth`. This allows our application to access the Bluetooth connection of your Mac when running from XCode. If we do no select this, Bluetooth will seem powered off for the app.

Now run the app. If you have specified the BLEConstants the right way for your peripheral device, you should see a series of log statements that ends with `Connected to peripheral`.

During testing of the app on my Macbook I noticed that the `centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral` method was called twice, even though my code only asked to connect to the peripheral once. This would be a problem further on in our app. If you have the same, we can easily make a little workaround for that. Add a new `private var connected: Bool = false` to the BleCentral class and add `self.connected = false` as the first line of the `connect()` method. Then change the implementation of the didConnect method to be:
```swift
func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
    if (!self.connected) {
        self.connected = true
        self.delegate?.logMessage(message: "Connected to peripheral \(peripheral), discovering services.")
        self.setupServicesAndCharacteristics()
        self.peripheral?.discoverServices(nil)
    }
}
```

### Creating the BLEPeripheral
At this point we have done most of the work for our proxy application already, as the BLEPeripheral is quite simple to implement compared to the BLECentral.

Lets create another new file in our project that we call `BLEPeripheral.swift`. Just like in the central file, we will add two things, the BlePeripheral class and a delegate protocol for this class.

Let's start with the delegate protocol again. The BlePeripheral that we are about to make needs to be able to communicate a couple of things to its delegate, these are:
- That it has started advertising
- That it has stopped advertising, including a reason why
- To read data was received from a central
- To write data was received from a central
- To register for notifications on characteristic updates
- To unregister from notifications on characteristic updates

So we create a `BlePeripheralDelegate` protocol at the top of the `BlePeripheral.swift` that contains the following:
```swift
protocol BlePeripheralDelegate {
    func advertisingStarted()
    func advertisingStopped(reason: String)
    func read(fromCharacteristicUUID uuid: CBUUID)
    func write(data: Data, toCharacteristicUUID uuid: CBUUID)
    func registerForNotifications(onCharacteristicWithUUID uuid: CBUUID)
    func unregisterFromNotifications(onCharacteristicWithUUID uuid: CBUUID)
}
```

Below this delegate we will create a new class `BlePeripheral`. This class will have a `var delegate` of the protocol we created. This class will also use `CBPeripheralManager` to advertise as a peripheral and offer services to centrals, so we create a `private var peripheralManager: CBPeripheralManager?` in the class for that. Don't forget to add `import CoreBluetooth` at the top of `BLEPeripheral.swift` to be able to use the CoreBluetooth classes. 

In order for the peripheralmanager to be able to communicate back to this class we also need it to be an `NSObject` and implement `CBPeripheralManagerDelegate`. This gives our class an `init()` method where we can initialise the peripheralmanager var we added.

Also let's give this class public methods so that it can be told:
- To start advertising
- To stop advertising
- To confirm a write request
- That data was received on a characteristic

All of the above means we add the following code to `BLEPeripheral.swift` below the `BlePeripheralDelegate` protocol:
```swift
class BlePeripheral: NSObject, CBPeripheralManagerDelegate {
    
    var delegate: BlePeripheralDelegate?
    
    private var peripheralManager: CBPeripheralManager?
    
    override init() {
        super.init()
        self.peripheralManager = CBPeripheralManager(delegate: self, queue: nil)
    }
    
    func startAdvertising(services: [BleService]) {
        
    }
    
    func stopAdvertising() {
        
    }
    
    func confirmWriteRequest(onCharacteristicUUID uuid: CBUUID, withResult result: CBATTError.Code) {
        
    }
    
    func dataReceived(data: Data, onCharacteristicUUID uuid: CBUUID) {
        
    }
    
    func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
        
    }
}
```

The `peripheralManagerDidUpdateState` method was added as it is required for classes implementing `CBPeripheralManagerDelegate`.

The first thing we set up in the BlePeripheral is advertising, but before we can advertise our peripheral and offer services on it, we need to create `CBMutableService` and `CBMutableCharacteristic` instances for all services and characteristics we wish to offer. So we add `private var services: [CBMutableService] = []` and `private var characteristics: [CBMutableCharacteristic] = []` and we create a few private methods that set this up for us:
```swift
private func initialiseServicesAndCharacteristics(services: [BleService]) {
    self.services = []
    for bleService in services {
        var characteristics:[CBMutableCharacteristic] = []
        if let bleCharacteristics = bleService.characteristics {
            for bleCharacteristic in bleCharacteristics {
                if let characteristic = bleCharacteristic.characteristic {
                    let mutableCharacteristic = CBMutableCharacteristic(type: characteristic.uuid, properties: characteristic.properties, value: nil, permissions: self.getPermissions(fromProperties: characteristic.properties))
                    characteristics.append(mutableCharacteristic)
                    self.allCharacteristics.append(mutableCharacteristic)
                }
            }
        }
        let service = CBMutableService(type: bleService.uuid, primary: bleService.service?.isPrimary ?? false)
        service.characteristics = characteristics
        self.services.append(service)
    }
}

private func getPermissions(fromProperties: CBCharacteristicProperties) -> CBAttributePermissions {
    if fromProperties.contains(.write) || fromProperties.contains(.writeWithoutResponse) {
        return .writeable
    } else {
        return .readable
    }
}
```

I must admit that the implementation of the `getPermissions` method is a bit of a guess on my end. The characteristics we have discovered in our central do not have permissions (that we can see), so we have to base them on the properties instead with some guesswork.  The possible options for permissions are:
- readable
- writeable
- readEncryptionRequired
- writeEncryptionRequired

Since we are making a debug tool, encryption seems unnecessary to me so the last two options can be disregarded. It is possible to only specify a single `CBAttributePermissions` value for a characteristic and it made sense to me that any allowed to write would also be allowed to read, which lead to the implementation above.

Now that the services and characteristics are ready, we can start advertising by changing the `startAdvertising` method to:
```swift
func startAdvertising(services: [BleService]) {
    self.initialiseServicesAndCharacteristics(services: services)
    self.peripheralManager?.startAdvertising([
        CBAdvertisementDataServiceUUIDsKey : self.services.map({ (service) -> CBUUID in
            return service.uuid
        }),
        CBAdvertisementDataLocalNameKey: BleConstants.DEVICE_NAME,
    ])
}
```

To get notified that advertising has started or that an error has occured during the call to start, we can add:
```swift
func peripheralManagerDidStartAdvertising(_ peripheral: CBPeripheralManager, error: Error?) {
    if let error = error {
        self.delegate?.advertisingStopped(reason: "There was an error: \(error)")
    } else {
        self.delegate?.advertisingStarted()
    }
}
```

To also make our services and characteristics known to the world, change `peripheralManagerDidUpdateState` to:
```swift
func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
    if peripheral.state != .poweredOn {
        self.delegate?.advertisingStopped(reason: "Bluetooth is turned off.")
        return
    }
    
    for service in self.services {
        self.peripheralManager?.add(service)
    }
}
```

To stop advertising and offering services when `stopAdvertising` is called, change that method to:
```swift
func stopAdvertising() {
    self.peripheralManager?.stopAdvertising()
    self.peripheralManager?.removeAllServices()
    self.delegate?.advertisingStopped(reason: "Stop requested")
}
```



With the above methods implemented our peripheral is already live and offering services and characteristics. The only problem is that when a central would connect and try to send requests, that central would be ignored. Let's fix that.

In order to receive requests to read data, we can implement the method `peripheralManager(_ peripheral: CBPeripheralManager, didReceiveRead request: CBATTRequest)` of `CBPeripheralManagerDelegate`. We can implement that as follows:
```swift
func peripheralManager(_ peripheral: CBPeripheralManager, didReceiveRead request: CBATTRequest) {
    self.openReadRequests.append(request)
    self.delegate?.read(fromCharacteristicUUID: request.characteristic.uuid)
}
```

The property `self.openReadRequests` is now yet know, so we should add it as `private var openReadRequests: [CBATTRequest] = []`. We should also add the line `self.openReadRequests = []` to the beginning of `startAdvertising` to ensure this array is cleared when we (re)start advertising our peripheral.

In order to receive requests to write data, we can implement the method `peripheralManager:(CBPeripheralManager *)peripheral didReceiveWriteRequests:(NSArray<CBATTRequest *> *)requests` of `CBPeripheralManagerDelegate`as follows:
```swift
func peripheralManager(_ peripheral: CBPeripheralManager, didReceiveWrite requests: [CBATTRequest]) {
    for request in requests {
        if let data = request.value {
            self.openWriteRequests.append(request)
            self.delegate?.write(data: data, toCharacteristicUUID: request.characteristic.uuid)
        }
    }
}
```

For `self.openWriteRequests`, do the same as we just did for `openReadRequests`.

With those steps we have registered the incoming requests and we have instructued our delegate to read and write when requests come in. What we have not yet done is reply to the requests, let's do that now.

Our peripheral should respond to write requests when its `confirmWriteRequest` method is called. Implement that as such:
```swift
func confirmWriteRequest(onCharacteristicUUID uuid: CBUUID, withResult result: CBATTError.Code) {
    let requestsToRespondTo = self.openWriteRequests.filter { (request) -> Bool in
        request.characteristic.uuid == uuid
    }
    for request in requestsToRespondTo {
        self.peripheralManager?.respond(to: request, withResult: result)
    }
    self.openWriteRequests.removeAll { (request) -> Bool in
        requestsToRespondTo.contains(request)
    }
}
```

The response to read requests should be sent when `dataReceived` is called. Implement that as such:
```swift
func dataReceived(data: Data, onCharacteristicUUID uuid: CBUUID) {
    if let characteristic = self.allCharacteristics.first(where: { (characteristic) -> Bool in
        characteristic.uuid == uuid
    }) {
        characteristic.value = data
        
        let requestsToRespondTo = self.openReadRequests.filter { (request) -> Bool in
            request.characteristic.uuid == uuid
        }
        for request in requestsToRespondTo {
            self.peripheralManager?.respond(to: request, withResult: .success)
        }
        self.openWriteRequests.removeAll { (request) -> Bool in
            requestsToRespondTo.contains(request)
        }
        
        self.peripheralManager?.updateValue(data, for: characteristic, onSubscribedCentrals: nil)
    }
}
```

This method does a little more than only respond to requests. It also ensure that the value is set onto the characteristic. Through the last line of the method, `self.peripheralManager?.updateValue(data, for: characteristic, onSubscribedCentrals: nil)`, it also ensures that any central that has subscribed to updates to this characteristics value gets notified.

The only thing that our peripheral is missing now is that it needs to tell the delegate register for notifications when it receives incoming registrations from centrals and unregister from notifications when the last central has unregistered. To get this last bit in, add `private var registrations: [CBCharacteristic: [CBCentral]] = [:]` to the class and add `self.registrations = [:]` to the beginning of `startAdvertising`. Finally, add the following two methods:
```swift
func peripheralManager(_ peripheral: CBPeripheralManager, central: CBCentral, didSubscribeTo characteristic: CBCharacteristic) {
    if var registrationsForCharacteristic = self.registrations[characteristic] {
        if (!registrationsForCharacteristic.contains(central)) {
            registrationsForCharacteristic.append(central)
        }
    } else {
        self.registrations[characteristic] = [central]
        self.delegate?.registerForNotifications(onCharacteristicWithUUID: characteristic.uuid)
    }
}

func peripheralManager(_ peripheral: CBPeripheralManager, central: CBCentral, didUnsubscribeFrom characteristic: CBCharacteristic) {
    if var registrationsForCharacteristic = self.registrations[characteristic] {
        registrationsForCharacteristic.removeAll { (centralRegistration) -> Bool in
            centralRegistration.identifier == central.identifier
        }
        if registrationsForCharacteristic.isEmpty {
            self.delegate?.unregisterFromNotifications(onCharacteristicWithUUID: characteristic.uuid)
        }
    }
}
```

With that, our peripheral should be ready to go.

### Using the BLEPeripheral
To use the peripheral we created means we have to update the ViewController. So open up `ViewController.swift` and change the contents to this:
```swift
import Cocoa
import CoreBluetooth

class ViewController: NSViewController, BleCentralDelegate, BlePeripheralDelegate {
    
    @IBOutlet weak var startProxyButton: NSButton!
    @IBOutlet weak var stopProxyButton: NSButton!
    @IBOutlet weak var logView: NSTextField!
    
    var bleCentral: BleCentral?
    var blePeripheral: BlePeripheral?
    
    override func viewDidLoad() {
        super.viewDidLoad()

        self.bleCentral = BleCentral()
        self.bleCentral?.delegate = self
        
        self.blePeripheral = BlePeripheral()
        self.blePeripheral?.delegate = self
    }

    @IBAction func startProxy(_ sender: Any) {
        startProxyButton.isEnabled = false
        stopProxyButton.isEnabled = true
        self.bleCentral?.connect()
    }
    
    @IBAction func stopProxy(_ sender: Any) {
        startProxyButton.isEnabled = true
        stopProxyButton.isEnabled = false
        self.bleCentral?.disconnect()
    }
    
    @IBAction func clearLog(_ sender: Any) {
        self.logView.stringValue = ""
    }
    
    func connected(services: [BleService]) {
        log("Connected to peripheral")
        self.blePeripheral?.startAdvertising(services: services)
    }
    
    func disconnected(reason: String) {
        log("Stopped because: \(reason)")
        self.blePeripheral?.stopAdvertising()
    }
    
    func advertisingStarted() {
        log("Started advertising peripheral")
    }
    
    func advertisingStopped(reason: String) {
        log("Stopped advertising because: \(reason)")
    }
    
    func read(fromCharacteristicUUID uuid: CBUUID) {
        log("Reading data from characteristic \(uuid.uuidString)")
        self.bleCentral?.readData(characteristicUUID: uuid)
    }
    
    func write(data: Data, toCharacteristicUUID uuid: CBUUID) {
        log("Writing data \(data) from central to peripheral on characteristic \(uuid.uuidString)")
        self.bleCentral?.writeData(characteristicUUID: uuid, data: data, writeType: .withResponse)
    }
    
    func registerForNotifications(onCharacteristicWithUUID uuid: CBUUID) {
        log("Registering for notifications on characteristic \(uuid.uuidString)")
        self.bleCentral?.registerForNotifications(characteristicUUID: uuid)
    }
    
    func unregisterFromNotifications(onCharacteristicWithUUID uuid: CBUUID) {
        log("Unregistering from notifications on characteristic \(uuid.uuidString)")
        self.bleCentral?.unregisterFromNotifications(characteristicUUID: uuid)
    }
    
    func dataWritten(onCharacteristicWithUUID uuid: CBUUID, withResult result: CBATTError.Code) {
        log("Data written from central to peripheral on characteristic \(uuid.uuidString) with result: \(result)")
        self.blePeripheral?.confirmWriteRequest(onCharacteristicUUID: uuid, withResult: result)
    }
    
    func dataReceived(data: Data, onCharacteristicWithUUID uuid: CBUUID) {
        log("Received \(data) from peripheral to pass to central on characteristic \(uuid.uuidString)")
        self.blePeripheral?.dataReceived(data: data, onCharacteristicUUID: uuid)
    }
    
    func logMessage(message: String) {
        log(message)
    }
    
    private func log(_ message: String) {
        logView.stringValue.append("\(message)\n")
    }
}
```