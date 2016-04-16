---
layout: static
comments: true
published: true
title: A Swift OpenGL Tutorial
---

This post explains how to set up a simple OpenGL project using the Swift programming language and Apple's `NSOpenGLView` class. You can download the complete Xcode project containing everything in this tutorial and more from GitHub at [letvargo/GLTutorial_Swift](https://github.com/letvargo/GLTutorial_Swift).

All of the code in this tutorial is based on Tom Davies' excellent [Cocoa-GL-Tutorial](https://github.com/beelsebob/Cocoa-GL-Tutorial). I translated his code pretty much line-for-line to learn a little bit about how to get started with OpenGL and Swift. I learned a lot, and hopefully some of this code will be of use to others.

Among other things, the tutorial will cover:

 - How to initialize an `NSOpenGLView` with a custom `NSOpenGLPixelFormat`
 - How to store vertex data in a Swift `struct` and move it to an OpenGL buffer
 - How to load, compile and link a vertex shader and a fragment shader
 - How to create a display link and a display link callback function

It is not a tutorial on OpenGL or GLSL - I don't know enough about them to offer much insight. The OpenGL function calls and GLSL code are presented without explanation. Working through it, though, did give me my first glimmer of hope that I might be starting to understand something about how OpenGL works on the most basic level.

## Creating the `NSOpenGLView`

We're going to create a class called `GLTutorialController` that will create an `NSOpenGLView` and add it to the main window. Create a file called `GLTutorialController` and add the following:

    import Cocoa
    import OpenGL.GL3
    import CoreVideo.CVDisplayLink
    
    class GLTutorialController: NSObject {
    
        @IBOutlet var window: NSWindow!
        var view: NSOpenGLView!

        var displayLink: CVDisplayLink?
    
        var shaderProgram: GLuint = 0
        var vertexArrayObject: GLuint = 0
        var vertexBuffer: GLuint = 0

        var positionUniform: GLint = -1
        var colorAttribute: GLint = -1
        var positionAttribute: GLint = -1
    
        override func awakeFromNib() {
            createOpenGLView()
            createOpenGLResources()
            createDisplayLink()
        }
    }
    
You'll have to create a generic object in interface builder, change its class to `GLTutorialController`, and connect the `IBOutlet` to the main window.

Now add the `createOpenGLView` function:

    func createOpenGLView() {
        // 1 - Define a set of pixel attributes
        let pixelFormatAttributes: [NSOpenGLPixelFormatAttribute] =
            [ NSOpenGLPFAOpenGLProfile  , NSOpenGLProfileVersion3_2Core
            , NSOpenGLPFAColorSize      , 24
            , NSOpenGLPFAAlphaSize      , 8
            , NSOpenGLPFADoubleBuffer
            , NSOpenGLPFAAccelerated
            , NSOpenGLPFANoRecovery
            , 0 ]
            .map { UInt32($0) }
        
        // 2 - Create the pixel format
        let pixelFormat = NSOpenGLPixelFormat(attributes: pixelFormatAttributes)
        
        // 3 - Initialize the OpenGL view
        guard let contentView = window.contentView
            , let openGLView = NSOpenGLView(frame: contentView.bounds, pixelFormat: pixelFormat) else { return }
        
        // 4 - Assign it to the view property and add it as a subview
        view = openGLView
        
        contentView.addSubview(view)
    }
    
## Compiling and linking the shaders

The `awakeFromNib` function calls `createOpenGLResources` which looks like this:

    func createOpenGLResources() {
        view.openGLContext?.makeCurrentContext()
        loadShader()
        loadBufferData()
    }
    
As you can see, it actually handles two different tasks - it loads the shaders and loads the buffer data. We are going to start with the `loadShader` function:

    func loadShader() {
    
        guard let vShaderFile = NSBundle.mainBundle().pathForResource("Shader", ofType: "vsh")
            , let fShaderFile = NSBundle.mainBundle().pathForResource("Shader", ofType: "fsh")
            else { return }
        
        let vertexShader = compileShaderOfType(GLenum(GL_VERTEX_SHADER), file: vShaderFile)
        let fragmentShader = compileShaderOfType(GLenum(GL_FRAGMENT_SHADER), file: fShaderFile)
        
        guard vertexShader != 0 && fragmentShader != 0 else {
            print("Shader compilation failed.")
            return
        }
        
        shaderProgram = glCreateProgram()
        
        glAttachShader(shaderProgram, vertexShader)
        glAttachShader(shaderProgram, fragmentShader)
        glBindFragDataLocation(shaderProgram, 0, "fragColor")
        
        linkProgram(shaderProgram)

        positionUniform = glGetUniformLocation(shaderProgram, "p")
        colorAttribute = glGetAttribLocation(shaderProgram, "color")
        positionAttribute = glGetAttribLocation(shaderProgram, "position")

        glDeleteShader(vertexShader)
        glDeleteShader(fragmentShader)
    }
    
I have omitted a lot of debugging code from what I am posting here. I suggest taking a look at the original source file which contains a lot of diagnostic tools that will help you identify any problems that arise. I'm leaving it for the sake of brevity, but it was very useful to me in debugging my project. (And again, all credit goes to Tom Davies who wrote the original error diagnostics code in Objective-C.)

The `loadShader` function calls two additional functions: `compileShaderOfType:file:` and `linkProgram:`. These functions look like this (again, with debugging code omitted for brevity- but seriously, you should include it if you are doing this):
    
    func compileShaderOfType(type: GLenum, file: String) -> GLuint {
    
        var shader: GLuint = 0
        
        do {
            var source = try NSString(contentsOfFile: file, encoding: NSASCIIStringEncoding).cStringUsingEncoding(NSASCIIStringEncoding)
            
            shader = glCreateShader(type)
            glShaderSource(shader, 1, &source, nil)
            glCompileShader(shader)
        } catch {
            print("Failed to reader shader file.")
            print("\(error)")
        }
        return shader
    }
    
    func linkProgram(program: GLuint) {
    
        glLinkProgram(program)
    }
    
The shaders themselves are contained in two files, `Shader.vsh` (the vertex shader) and `Shader.fsh` (the fragment shader) which need to be created and included in your project's main bundle so that they can be loaded and compiled. The code for the shaders is available on the GitHub repository.

## Loading the buffer data

We are going to need to supply data to an OpenGL buffer. To organize the data, let's create a couple of `struct`s:

    struct Vertex {
        let position: (x: GLfloat, y: GLfloat, z: GLfloat, w: GLfloat)
        let color: (r: GLfloat, g: GLfloat, b: GLfloat, a: GLfloat)
    }

    struct Vertices {
        let v1: Vertex
        let v2: Vertex
        let v3: Vertex
        let v4: Vertex
    }

With those in place, we can write our vertex data in a way that is semi-human-readable. Let's look at the `loadBufferData` function that we called from `createOpenGLResources`:

    func loadBufferData() {
        
        var vertexData = Vertices(
            v1: Vertex( position:   (x: -0.5, y: -0.5, z:  0.0, w:  1.0),
                        color:      (r:  1.0, g:  0.0, b:  0.0, a:  1.0)),
            v2: Vertex( position:   (x: -0.5, y:  0.5, z:  0.0, w:  1.0),
                        color:      (r:  0.0, g:  1.0, b:  0.0, a:  1.0)),
            v3: Vertex( position:   (x:  0.5, y:  0.5, z:  0.0, w:  1.0),
                        color:      (r:  0.0, g:  0.0, b:  1.0, a:  1.0)),
            v4: Vertex( position:   (x:  0.5, y: -0.5, z:  0.0, w:  1.0),
                        color:      (r:  1.0, g:  1.0, b:  1.0, a:  1.0)) )
        
        glGenVertexArrays(1, &vertexArrayObject)
        glBindVertexArray(vertexArrayObject)

        glGenBuffers(1, &vertexBuffer)
        glBindBuffer(UInt32(GL_ARRAY_BUFFER), vertexBuffer);
        glBufferData(UInt32(GL_ARRAY_BUFFER), 4 * sizeof(Vertex), &vertexData, UInt32(GL_STATIC_DRAW))

        glEnableVertexAttribArray(GLuint(positionAttribute))
        glEnableVertexAttribArray(GLuint(colorAttribute))

        glVertexAttribPointer(GLuint(positionAttribute), 4, UInt32(GL_FLOAT), UInt8(GL_FALSE), GLsizei(sizeof(Vertex)), nil)

        let offset = UnsafePointer<Void>() + sizeofValue(vertexData.v1.position)
        glVertexAttribPointer(GLuint(colorAttribute), 4, UInt32(GL_FLOAT), UInt8(GL_FALSE), GLsizei(sizeof(Vertex)), offset)
    }
    
This is the section of code that gave me the most trouble in translation. I couldn't figure out how to set the offset in the last call to `glVertexAttribPointer`. Turns out that creating a pointer that represents an offset is easier than I imagined. Just create a null pointer and add the offset to it. That is what this line does:

    let offset = UnsafePointer<Void>() + sizeofValue(vertexData.v1.position)
    
## Creating the diplay link

A display link is used to synchronize the drawing with the display's normal refresh rate. Each time the screen refreshes, a C callback function is called that you use to call your drawing code.

First, we create the display link using the `createDisplayLink` function that was called from `awakeFromNib`:

    func createDisplayLink() {
        let displayID = CGMainDisplayID()
        let error = CVDisplayLinkCreateWithCGDisplay(displayID, &displayLink)
        
        guard let dLink = displayLink where kCVReturnSuccess == error else {
            NSLog("Display Link created with error: %d", error)
            displayLink = nil
            return
        }
        
        CVDisplayLinkSetOutputCallback(dLink, displayCallback, UnsafeMutablePointer<Void>(unsafeAddressOf(self)))
        CVDisplayLinkStart(dLink)
    }
    
Note how we pass a pointer to `self` (using `unsafeAddressOf`) as an `UnsafeMutablePointer<Void>` when the display link callback is set in this line:

    CVDisplayLinkSetOutputCallback(dLink, displayCallback, UnsafeMutablePointer<Void>(unsafeAddressOf(self)))

That pointer will get passed as a parameter to the callback function, and so that we can use it inside the callback. The callback function itself is called `displayCallback` and it looks like this:

    let displayCallback: CVDisplayLinkOutputCallback = { displayLink, inNow, inOutputTime, flagsIn, flagsOut, displayLinkContext in
        let controller = unsafeBitCast(displayLinkContext, GLTutorialController.self)
        controller.renderForTime(inOutputTime.memory)
        return kCVReturnSuccess
    }
    
The `displayLinkContext` parameter is the pointer to `self` that gets passed to the callback. In order to use it, we invoke `unsafeBitCast(displayLinkContext, GLTutorialController.self)` to cast it back to its type, `GLTutorialController`.

Now we've got a working display link. Every time the display refreshes, this line will be invoked from the callback:

    controller.renderForTime(inOutputTime.memory)
    
All of our drawing code will be included in that function, `renderForTime:`

## Drawing to the screen

The last thing we need to add to our `GLTutorialController` class is our `renderForTime:` function:

    func renderForTime(time: CVTimeStamp) {
    
        view.openGLContext?.makeCurrentContext()
        
        glClearColor(0.0, 0.0, 0.0, 1.0)
        glClear(UInt32(GL_COLOR_BUFFER_BIT))
        glUseProgram(shaderProgram)

        let timeValue = GLfloat(time.videoTime) / GLfloat(time.videoTimeScale)
        let p: [GLfloat] = [ 0.5 * sinf(timeValue), 0.5 * cosf(timeValue) ]
        glUniform2fv(positionUniform, 1, p)
        glDrawArrays(UInt32(GL_TRIANGLE_FAN), 0, 4)
        
        view.openGLContext?.flushBuffer()
    }
    
## Conclusion

Tom Davies' original project has a great `ReadMe` file that explains more about how a lot of this stuff works. I suggest reading it. He offers a lot of insight into some of the OpenGL concepts that are at work, and how they interect with `NSOpenGLView`.

I also highly recommend using the debugging code that is included in the source files available on the GitHub repo for this project. It is very useful, and reusable as well.
