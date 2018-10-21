## Calling Cardano-Rust from iOS App

Following on from the first post about [Calling Cardano-Rust from C code](https://hm999.github.io/cardano-rust-c-example-doc/).

And similar to the previous post about [Calling Cardano-Rust from Android App](https://hm999.github.io/cardano-rust-android-example-doc/).

We will today invoke the Cardano-Rust functions from an iOS app.

Source code discussed below is [here](https://github.com/HM999/cardano-rust-ios-example), but with binaries and local config omitted.

The Rust code compiles to a C callable library, C is callable from iOS/Swift, therefore we can call the code from Swift. We have already written the code for the Rust-to-C side, by making the special bridging functions, as seen in the first post. We now need to compile those bits for iOS hardware platforms and make the C-to-Swift part of the bridge.

### The Rust Side

First we need to build a version of the library for the various iOS hardware platforms, however we do not need separate libraries because the iOS compiler can create a static “universal” library. 

You will need an Apple Mac with xcode, and an apple developer account if you want to deploy to a physical iPhone.

The version of xcode on your system must be pointing at the developer version, check:

```
pusheen: xcode-select -p
/Applications/Xcode.app/Contents/Developer
```

In addition to cargo, we need cargo lipo to be installed, which is the universal library builder for iOS. 

Do the build:

```
cargo lipo
```

Or “cargo lipo –release“ for optimised version. Under the target directory there will be a number of iOS subdirectories with a static library in them:

```
pusheen: find *ios* -name "*.a" | grep -v dep | grep debug
aarch64-apple-ios/debug/libtest.a
armv7-apple-ios/debug/libtest.a
armv7s-apple-ios/debug/libtest.a
i386-apple-ios/debug/libtest.a
x86_64-apple-ios/debug/libtest.a
```

But we are not interested in them, we want the one in the “universal” subdirectory:

```
pusheen: nm libtest.a 2>/dev/null | grep my_b58_
0000000000000180 T _my_b58_decode
0000000000000000 T _my_b58_encode
```

### The iOS Side

We open xcode and create a new “single view” project. I rename the library to libcardano_funcs.a and add it under “Linked Framework and Libraries” in Xcode. Also click the “+” icon and add libresolv.tbd.

We need a header file containing the C function definitions for linking, File / New / File / Header / cardano_funcs.h and then add the following between the default lines:

```
int8_t my_b58_encode( const uint8_t *bytes, unsigned int size, char *encoded );
int8_t my_b58_decode( const char *encoded, uint8_t *bytes );
```

and add this above them:

```
#include <inttypes.h>
```

NB: although uint8_t is just unsigned char, and almost everyone uses it, but it's not out of the box

And we need a bridging header; again File / New / File / Header / cardano_funcs_bridge and then add:

```
#import "cardano_funcs.h"
```

Under the project Build Settings 

1. Set Objective-C Bridging Header for debug and release to:

```
$(PROJECT_DIR)/cardano_funcs_bridge.h
```

2. To Library Search Paths, add the directory containing the libcardano_funcs.a

3. Under Build Options, set Enable Bitcode = No.

4. Optional: set deployment device orientation: Portrait only.

NB: If you get a generate-pch error, that means it can't find the bridging header. Look at the command it spews out, see where it is looking for the header – amend entry accordingly, or move files around. If Linking fails, it means it can't find the libcardano_funcs.a

We need to call our code, so we add a Swift file to the project, called base58.swift:

```
import Foundation

class CardanoFuncs {
    static func encode(stringToEncode: String) -> String {

        let byteArrayToEncode: [UInt8] = Array(stringToEncode.utf8)
        
        let encoded = UnsafeMutableRawPointer.allocate(bytes: 1000, alignedTo: MemoryLayout<UInt8>.alignment)
        let opaquePtr = OpaquePointer(encoded)
        let contextPtr = UnsafeMutablePointer<Int8>(opaquePtr)
        
        let status = my_b58_encode(byteArrayToEncode,UInt32(byteArrayToEncode.count), contextPtr)
        
        var encodedString = "*** ERROR ***"
        
        if status >= 0 {
            encodedString = String(cString: contextPtr)
        }
        
        encoded.deallocate(bytes: 1000, alignedTo: MemoryLayout<UInt8>.alignment)
        
        return encodedString
    }
    
    static func decode(stringToDecode: String) -> String {
        
        let raw = UnsafeMutableRawPointer.allocate(bytes: 1000, alignedTo: MemoryLayout<UInt8>.alignment)
        let typedPtr = raw.initializeMemory(as: UInt8.self, at: 0, count: 1000, to: 0)
        
        let cstr = stringToDecode.cString(using: String.Encoding.utf8)
        
        let status = my_b58_decode(cstr, typedPtr)
        
        var decodedString = "*** ERROR ***"
        
        if status >= 0 {
            decodedString = String(cString: typedPtr)
        }
        
        raw.deallocate(bytes: 1000, alignedTo: MemoryLayout<UInt8>.alignment)
        
        return decodedString
    }
}
```

To call this code we need to edit the Main.storyboard using the xcode visual editor. Make a couple of TextFields (one for text to encode, one for text to decode) and a couple of buttons (encode, decode). Then you have to do a weird thing specific to xcode/Swift development, you drag these widgets into the ViewController.swift source code file, which creates references to them. After that we add code to them, calling our encode and decode functions.

My ViewController.swift file looks like this:

```
import UIKit

class ViewController: UIViewController {
    
    @IBOutlet weak var encodeLabel: UILabel!
    @IBOutlet weak var decodeLabel: UILabel!
    
    @IBOutlet weak var encodeText: UITextField!
    @IBOutlet weak var decodeText: UITextField!
    
    func alert( msg: String ) {
        let alert = UIAlertController(title: nil, message: msg, preferredStyle: UIAlertControllerStyle.alert)
        alert.addAction(UIAlertAction(title: "Ok", style: UIAlertActionStyle.default, handler: nil))
        self.present(alert, animated: true, completion: nil)
    }
    
    @IBAction func encodeButton(_ sender: UIButton) {
        if let str: String = encodeText.text {
            let encoded = CardanoFuncs.encode(stringToEncode: str)
            alert(msg: encoded)
        }
    }
    
    @IBAction func decodeButton(_ sender: UIButton) {
        if let str: String = decodeText.text {
            let decoded = CardanoFuncs.decode(stringToDecode: str)
            alert(msg: decoded)
        }
    }
    
    override func viewDidLoad() {  // default func created by xcode
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
    }

    override func didReceiveMemoryWarning() { // default func created by xcode
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }
}
```

At this point the project looks like this:

[Cardano-Rust iOS Project](https://raw.githubusercontent.com/HM999/cardano-rust-ios-example-doc/master/images/cardano-rust-ios-project.png)

Everything should build without error. I attach an iPhone by USB cable and run the app on it:

[Cardano-Rust iOS App](https://raw.githubusercontent.com/HM999/cardano-rust-ios-example-doc/master/images/cardano-rust-ios-app.jpg)

Source code is [here](https://github.com/HM999/cardano-rust-ios-example), but with binaries and local config omitted.
