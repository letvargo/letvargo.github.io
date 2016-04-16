---
layout: static
published: true
comments: true
title: Core Audio and Swift Part 1 - A Gentle Introduction
---

Core Audio can be intimidating. The API is complex and the documentation is sparse. For someone like me who has no background in C, the low-level C style functions look like, well... a foreign language.

This post is meant to provide a gentle introduction to the most basic Core Audio operations. You'll learn how to open an audio file, examine its properties, and query it for information like album artwork, the artist's name, the duration of the audio, etc.

The examples in this post are based on an example in ["Learning Core Audio"](http://www.amazon.com/Learning-Core-Audio-Hands-On-Programming/dp/0321636848), available on Amazon. It is a great book, with many useful examples written in Objective-C. If you are interested in Core Audio, I recommend getting a copy.

## Opening an Audio File in Core Audio

We're going to be using the [Audio File Services](https://developer.apple.com/library/mac/documentation/MusicAudio/Reference/AudioFileConvertRef/) framework. Audio File Services is a part of the `AudioToolbox` framework. According to Apple, Audio File Services is...

>...a C programming interface that enables you to read or write a wide variety of audio data to or from disk or a memory buffer.

One of the simplest things you can do with Audio File Services is open an audio file and get information about the contents of the file. It makes for a good introduction to some of the peculiarities of how Core Audio works. 

To work with Audio File Services, you need to include the line,

    import AudioToolbox
    
at the top of any files that make use of it.

Let's jump right in with some code. The function that you use to open an audio file is called `AudioFileOpenURL`, and it looks like this:

    func AudioFileOpenURL(
          _ inFileRef: CFURL
        , _ inPermissions: AudioFilePermissions
        , _ inFileTypeHint: AudioFileTypeID
        , _ outAudioFile: UnsafeMutablePointer<AudioFileID> ) -> OSStatus
        
Take a look at the parameters:

 - `inFileRef`: The URL of the file you want to load.
 - `inPermissions`: A permission parameter, either `.ReadPermission`, `.WritePermission`, or `ReadWritePermission`.
 - `inFileTypeHint`: A hint to help identify the file type. Passing `0` (no hint) is fine at this stage of the game.
 - `outAudioFile`: A pointer to the audio file that will created.

The return value is an `OSStatus`, which is a `typealias` for `Int32`. The `OSStatus` code will be `0`, aka `noErr`, if there were no errors. Otherwise, it will be an `Int32` value that corresponds to a particular error. There will be more talk about error codes in the next few posts.

The first thing we need is a URL. Pick a file, any file, from your iTunes library and create a `String` to hold the file path and turn it into a `NSURL`. I'm going with some INXS, but it works for sucky music too:
    
    let filePath = "/Users/doofnugget/Music/iTunes/iTunes Media/Music/INXS/Kick/02 New Sensation.m4a"
    
    let url = NSURL(fileURLWithPath: filePath)
    
The `outAudioFile` parameter is an `UnsafeMutablePointer` to an `AudioFileID`. You pass the pointer in and Audio File Services fills it in with the information from the audio file that you are opening. To pass a pointer to an `AudioFileID` all you have to do is initialize one as a `var` and pass it in using `&`:

    var audioFile = AudioFileID()
    var status = AudioFileOpenURL(url, .ReadPermission, 0, &audioFile)
    
    if status != noErr { 
        print("An error occurred while opening the audio file: Error code \(status).") 
    }

`audioFile` is now a pointer to the file that we can use to get information about it.
    
Note that the `OSStatus` value that was returned from `AudioFileOpenURL` is used to determine if any errors occurred while the file was being opened. Error checking with Core Audio can be quite involved, but for now we'll just print a helpful message and the error code whenever an error occurs.

What we just did is a very common pattern when you are working with Core Audio. You initialize a data type like an `AudioFileID`, and then you pass a pointer to it into a Core Audio function. The function modifies the properties of the object, usually by copying in or modifying some piece of useful information. You can think of the `UnsafeMutablePointer` parameters like `inout` parameters. The object that is passed in is modified by the function.

## Getting Property Size Information

`AudioFileID` is a `typealias` for a `COpaquePointer`. The word "opaque" is fitting. You cannot directly access any of the `AudioFileID`s properties. For example, every `AudioFileID` has a property that represents the estimated duration of the audio file. But you can't access it the way you would normally access an object's properties. This doesn't work:

    let duration = audioFile.estimatedDuration
    
No, the process for getting property information is somewhat more complex. It has two parts: First, you have to get the size (a `UInt32` value that represents the property's size in bytes) of the property using `AudioFileGetPropertyInfo`. Only after you've gotten the size can you actually query the audio file for its properties.

`AudioFileGetPropertyInfo` looks like this:

    func AudioFileGetPropertyInfo(
          _ inAudioFile: AudioFileID
        , _ inPropertyID: AudioFilePropertyID
        , _ outDataSize: UnsafeMutablePointer<UInt32>
        , _ isWritable: UnsafeMutablePointer<UInt32> ) -> OSStatus
        
So let's do it. We're going to get the audio file's "info dictionary" property. The info dictionary is a `NSDictionary` that contains information like the artist's name, the tempo, the album name, the year that the song came out, etc.

Here is what the code looks like for getting the size of the property that we are going to access:

    var size: UInt32 = 0                         // 1
    status = AudioFileGetPropertyInfo(
          audioFile                              // 2
        , kAudioFilePropertyInfoDictionary       // 3
        , &size                                  // 4
        , nil )                                  // 5
    
    if status != noErr {                         // 6
        print("An error occurred while getting the info dictionary property size: Error code \(status).") 
    }
    
1. We create a `UInt32` value called `size` and initialize it with a dummy value of `0`. 
2. The first parameter of the function is the audio file that we want to query. 
3. The second parameter is a constant, `kAudioFilePropertyInfoDictionary`, that identifies the property that we are asking about. 
4. The third parameter is a pointer to the `size` variable that we created. When the function returns, `size` will contain the actual size of the property. 
5. The last parameter, `isWritable`, can be used to ask whether or not the property is read-only or if it can be written. We are not interested in that right now, and so we pass `nil`.
6. Last, we check to see if any errors occurred. If so, we print out a helpful message along with the error code.

## Accessing the Property

Now that we have the property's size stored away in our `size` variable, we can actually ask the audio file to give us the property data. The function for getting the property data is called `AudioFileGetProperty`, and it looks like this:

    func AudioFileGetProperty(
          _ inAudioFile: AudioFileID
        , _ inPropertyID: AudioFilePropertyID
        , _ ioDataSize: UnsafeMutablePointer<UInt32>
        , _ outPropertyData: UnsafeMutablePointer<Void> ) -> OSStatus
        
And this is the code for actually retrieving the dictionary information:

    var infoDictionary = NSDictionary()          // 1
    status = AudioFileGetProperty(
          audioFile                              // 2
        , kAudioFilePropertyInfoDictionary       // 3
        , &size                                  // 4
        , &infoDictionary )                      // 5
    
    if status != noErr {                         // 6
        print("An error occurred while getting the info dictionary property: Error code \(status).") 
    }
    
This should all be starting to look a little familiar by now. 

1. Create a dummy dictionary to hold the data.
2. Pass in the audio file that we are querying.
3. Pass in the property identifier constant.
4. Pass in a pointer to the size of the property
5. Pass in a pointer to the dummy dictionary.
    
When the function returns, `infoDictionary` will be populated with the data that we were looking for (artist name, album, song title, etc.) This is the output that I get when I print the info dictionary for the file that I opened:

    {
        album = Kick;
        "approximate duration in seconds" = "220.192";
        artist = INXS;
        genre = Rock;
        title = "New Sensation";
        "track number" = "2/12";
        year = 1988;
    }
    
Audio File Services contains a number of constants that represent the possible keys in this dictionary. For example, the key for retrieving the artist's name is `kAFInfoDictionary_Artist`. Use the keys to get specific information like this:

    if let artist = infoDictionary[kAFInfoDictionary_Artist] {
        print(artist)             // Output: "INXS"
    }
    
## Conclusion

That's a very simple introduction to Audio File Services. Getting and setting an audio file's properties is a common task. Once you get past the strangeness of these C functions it is pretty straightforward. (Strange, at least, to someone like me without a background in C.)

There are, of course, many more things you can do with it. There are functions for reading and writing data and creating new audio files, among other things. And `AudioToolbox` offers many other services as well, including Audio Queue Services which you can use to record and playback audio. But for today, we're done. Thanks for reading!
