---
layout: static
published: false
title: Core Audio and Swift Part Two
---

## Error Handling

This post describes a technique for simplifying error checking in Core Audio using Swift's error handling mechanism.

This is a piece of code from my last post:

    var audioFile = AudioFileID()
    var status = AudioFileOpenURL(url, .ReadPermission, 0, &audioFile)

    if status != noErr { 
        print("An error occurred while opening the audio file: Error code \(status).") 
    }
    
This is somewhat tedious. Almost every Core Audio function returns an `OSStatus` value that needs to be checked. It would be nice to have some custom utility functions that throw errors that can be handled in a `catch` block.

Aside from the tediousness, the biggest problem is that each `OSStatus` is an `Int32` which isn't very helpful by itself. What do you do if you get an error code of `2003334207`? What does that even mean? One of the goals here is to turn these error codes into something more decipherable.

### Creating a custom Audio File Services `ErrorType` `enum`

One improvement would be to translate these error codes into an `enum` that conforms to `ErrorType` so that we can throw any errors that we encounter. The Audio File Services API names 16 different possible error codes. Let's take those 16 values and put them in an `enum` called `AudioFileError`:

    enum AudioFileError: ErrorType, CustomStringConvertible {
    
        case Unspecified(status: OSStatus, message: String)
        case UnsupportedFileType(status: OSStatus, message: String)
        case UnsupportedDataFormat(status: OSStatus, message: String)
        
        init(status: OSStatus, message: String) {
            switch status {
            case kAudioFileUnsupportedFileTypeError:
            	   self = .UnsupportedFileType(status: s, message: m)
            case kAudioFileUnsupportedDataFormatError:
                self = .UnsupportedDataFormatError(status: s, message: m)
            default:
            	   self = .Unspecified(status: s, message: m)
            }
        }
    }
    
These are just a few of the 16 possible error codes defined by Audio File Services. To keep the code brief, I haven't included all 16. But as you can see, an `AudioFileError` is initialized with an `OSStatus` value and a message that can be passed in when the error is thrown.

Those associated values are helpful, but when you are debugging, more information is always better. Let's add a computed property called `errorCategory` that tells us that this error is a member of the Audio File Services API:

    public var errorCategory: String {
        return "Audio File Services Error"
    }
    
The `errorCategory` property is useful because later on we may want to create a similar `enum` for error codes that are defined in another API, like Audio Queue Services. Just knowing which API the error came from is helpful.

Next, we can add another computed property, `errorDescription`:

    public var errorDescription: String {
        switch self {
        case .UnsupportedFileType(_):
            return "Unsupported file type."
        case .UnsupportedDataFormat(_):
            return "Unsupported data format."
        case .Unspecified(_):
        	  return "Unspecified error."
        }
    }
    
These descriptions provide a clear, concise description of what the error code actually means.
    
What we want, eventually, is a nicely formatted error that contains as much useful information as possible. Let's write a separate function called `composeError` that will take all of the information that we've collected and return it as a `String`:

    public func composeError(message: String, code: OSStatus) -> String {
        return  "\(errorCategory): \(errorDescription)\n\tMessage: \(message)\n\tError code: \(code)"
    }
    
Later on we'll improve on `composeError` even more. But first, let's make `AudioFileError` conform to `CustomStringDebuggable`, using `composeError` to generate a nicely formatted error message:

    public var description: String {
        
        switch self {
        case .UnsupportedFileType(let err):
            return composeError(err.message, code: err.status)
        case .UnsupportedDataFormat(let err):
            return composeError(err.message, code: err.status)
        case .Unspecified(let err):
            return composeError(err.message, code: err.status)
        }
    }
    
Now we can see what it looks like when we print an error:

    let error = AudioFileError(status: kAudioFileUnsupportedFileTypeError, message: "This is a test.")

    print(error)
    
    // Output:
    // Audio File Services Error: Unsupported file type.
    //     Message: This is a test.
    //     Error code: 1954115647 

That's not too bad. 

### Writing an error checking function

Next let's write a function that will actually use our new `AudioFileError` to check the result of any Audio File Services functions that we might call. We'll make it a `static` function and call it `check`:

    public static func check(status: OSStatus, message: String) throws {
        guard status == noErr else { 
            throw AudioFileError(status: status, message: message) 
        }
    }
    
Now we can see what it looks like in action. In my last post I intruduced the `AudioFileOpenURL` function. Call it inside our new `check` function and if there is a problem it will throw an `AudioFileError`:

    try AudioFileError.check(
          AudioFileOpenURL(url, .ReadPermission, 0, &audioFile)
        , message: "Failed to open audio file.")
        
If, for example, you try to open a text file, you will get an error that, when printed, looks like this:

    Audio File Services Error: Unsupported file type.
        Message: Failed to open audio file.
        Error code: 1954115647 

That's quite a bit more helpful than just getting back the error code `1954115647`. Now you know that error code is defined by Audio File Services, that the reason is because the file type is unsupported, and that the error was thrown while trying to open the audio file.

There's just a couple more things that we can do to improve on what we have.

### Create a `CodedErrorType` protocol and move some of our functions into an extension on it

As I mentioned earlier, we may want to create another `ErrorType` for other Core Audio frameworks, like Audio Queue Services. Let's generalize some of our utility functions and properties into a protocol. We'll call it `CodedErrorType`, because it represents an error based on a status code.

    public protocol CodedErrorType: ErrorType {

        var errorCategory: String { get }
        var errorDescription: String { get }
        
        static func check(status: OSStatus, message: String) throws
        
        func composeError(message: String, code: OSStatus) -> String
        
        init(status: OSStatus, message: String)
    }
    
Both the `composeError` and `check` functions can be moved into an extension on `CodedErrorType` so that we don't have to include those functions in every new error type that we create:

    extension CodedErrorType {
        
        public func composeError(message: String, code: OSStatus) -> String {
            return "\(errorCategory): \(errorDescription)\n\tMessage: \(message)\n\tError code: \(code)"
        }
        
        public static func check(status: OSStatus, message: String) throws {
            guard status == noErr else { 
                throw self.init(status: status, message: message) 
            }
        }
    }
    
If we declare that `AudioFileError` conforms to `CodedErrorType` we can use the default implementations defined in the extension and delete them from `AudioFileError` itself.

### Getting the four-character error code as a `String`

There's one more improvement that we can make. `OSStatus` error codes sometimes represent four character strings that are somewhat descriptive. For example, the `kAudioFileUnsupportedFileTypeError` error represents the four character string `'typ?'`.

You may wonder why these codes are helpful if we already have a more complete description of the errors that are being thrown. Why do we need a four-character `'typ?'` code when we already get `"Invalid file type"` as an error description.

It is important to know that just about any Core Audio function can return an error code from any of the APIs in the Core Audio framework. In other words, when you call a function that is defined in Audio File Services, you may get an error code that is defined in Audio Queue Services. If that happens, initializing an `AudioFileError` with a code from another API results in an `AudioFileError.Unspecified`, and the `errorDescription` property will just read `"Unspecified error."`

The only way to eliminate this problem would be to create an `enum` that included every error code in the entire Core Audio framework. It would be hundreds, if not thousands, of entries long. I don't know if anyone even knows how many error codes there are altogether, but it would be a nightmare.

Rather than go down that road, we'll settle for getting an `.Unspecified` error now and then, but we will check to see if there is a four-character code that might still give us some information that will be helpful in tracking down the error code in the documentation.

So where do these codes come from? Each `OSStatus` value is four bytes long. (`OSStatus` is just an `Int32`, remember?). Each of those bytes can represent a single ASCII character. So if you examine the individual bytes in the `Int32` value and each of those bytes represents a single ASCII character, you can throw all four of those characters into a string, and presto, you've got a four-character error code.

To do this, let's create a protocol called `CodeStringConvertible`. We'll give it one property, `codeString: String?`:

    protocol CodeStringConvertible {
        var codeString: String? { get }
    }
    
And now we can create an extension that does what we need:
    
    extension CodeStringConvertible {
        
        public var codeString: String? {
            
            let size = sizeof(self.dynamicType)             // 1

            func parseBytes(value: UnsafePointer<Void>)     // 2
                    -> [UInt8]? {
                
                let ptr = UnsafePointer<UInt8>(value)       // 3
                var bytes = [UInt8]()                       // 4
                
                for index in (0..<size).reverse() {         // 5
                    
                    let byte = ptr.advancedBy(index).memory // 6
                    
                    if (32..<127).contains(byte) {          // 7
                        bytes.append(byte)
                    } else {
                        return nil
                    }
                }
                
                return bytes                                // 8
            }
            
            if  let bytes = parseBytes([self]),             // 9
                let output = NSString(                      // 10
                      bytes: bytes
                    , length: size
                    , encoding: NSUTF8StringEncoding) as? String {
                    
                return output                               // 11
            }
            
            return nil
        }
    }

1. First, we get the size of the value that we are parsing in bytes. Though the purpose of this exercise is to parse an `Int32`, there's no reason other types couldn't conform to the protocol, and they may have different sizes. An `Int16`, for example, would only be two bytes long.
2. Define a nested function, `parseBytes` that takes a void pointer to the first byte that we will parse and returns an `Optional` array of `UInt8` values.
3. Convert the void pointer that was passed in to an `UnsafePointer<UInt8>`. By advancing this pointer we can examine specific bytes.
4. Create an empty array of `UInt8` values that will hold the bytes.
5. Start a `for-in` loop. The `index` will start at `3` and increment backwards to `0`. This is because the order of the bytes in the `Int32` is backwards from what you might expect. The first character is stored in the 4th byte, and the last character is stored in the 1st byte.
6. Inside the loop, advance the pointer by the specified distance and get the `UInt8` value that is stored in that byte.
7. Check each byte to make sure it is in the range of ASCII characters that could possibly make up a valid string. Everything before `32` is a control code of some kind. `127` is `Delete` (which we want to ignore) and we will ignore the "extended ASCII codes" that start at `128`. If the byte is valid, append it to `bytes`. Otherwise return `nil`
8. Return `bytes` which will contain exactly four `UInt8` values that each represent a valid ASCII character code.
9. Call `parseBytes`, passing `self` inside an array, and unwrap the result. I did not know you could do this before I wrote this code, but anytime you have a function that takes an `UnsafePointer<Void>` as a parameter, you can pass an array literal that contains immutable values. Handy.
10. Attempt to initialize an `NSString` with `bytes` and cast it to a `String`.
11. If successful, return the `String`. Otherwise, return `nil`.

I'm not convinced that this is the most elegant way to accomplish this task, but it works perfectly.

Before we can use it we have to declare that `Int32` conforms to our new protocol.

    extension Int32: CodeStringConvertible { }

Now we can test it out:

    if let err = kAudioFileUnsupportedFileType.codeString {
        print(err) 
    }
    
    // Output:
    // fmt?
    
Last, we'll incorporate this `codeString` property into our `composeError` function. That way, if all else fails and we end up with an `.Unspecified` error, we might at least get lucky and be able to check the `codeString`:

    public func composeError(message: String, code: OSStatus) -> String {
        var base = "\(errorCategory): \(errorDescription)\n\tMessage: \(message)\n\tError code: \(code)"
        if let codeString = code.codeString {
            base.appendContentsOf(" ('\(codeString)')")
        }
        return base
    }

Let's try it out:

    let err = AudioFileError.UnsupportedFileType(status: kAudioFileUnsupportedFileTypeError, message: "This is a test.")
    
    print(err)
    
    // Output:
    // Audio File Services Error: Unsupported file type.
    //     Message: This is a test.
    //     Error code: 1954115647 ('fmt?')
    
And that, my friends, is as complete as I could get.

The code in this post is all available on GitHub. Slowly but surely, I'm plugging away at a framework called `SwiftCoreAudio` which makes it easier to work with these APIs in Swift. The complete code for `AudioFileError` is there with all 16 Audio File Services error codes. Also there is the code for the `CodedErrorType` and `CodeStringConvertible` protocols.

Hopefully you found this interesting. I intend to keep digging into Core Audio and posting about it as I go. Thanks for reading!