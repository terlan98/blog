---
title: "\u2615 Java Everywhere: How to Run it on iOS Devices"
description: "Ever wondered if you could use Java in your iOS app?"
date: "2024-05-10"
FRtags: ["ios", "java", "html", "themes"]
aliases: ["java-on-ios"]
ShowToc: true
TocOpen: false
---


<img src="../images/mobivm-article.png#center" width=35%>

## The Backstory

Recently, I encountered a challenge while working on an iOS project that required a specific functionality which proved to be highly complex to implement. My search led me to a Java library that had the exact features I needed. However, there was one major roadblock: it was only available in Java. At first, I tried to find a native alternative that could be easily integrated into my iOS project via SPM, CocoaPods, or other dependency managers. Unfortunately, none of the available options matched the functionality of the Java library. As I explored methods for integrating Java into iOS, I found that many existing solutions were not up-to-date, lacked sufficient documentation, and had limited support, which made them less effective for my needs. Just when I was about to give up, I stumbled upon MobiVM (formerly known as RoboVM).

## What is MobiVM?

MobiVM is a fork of the last open-source release of RoboVM, a project that was discontinued after being acquired by Microsoft. It is basically an AOT (ahead-of-time) compiler for Java bytecode, targeting iOS, macOS and Linux. This implies that no interpretation is ever done and the code is run natively on the target CPU. MobiVM also provides bindings for iOS-specific APIs, enabling developers to access native iOS features directly from Java.

With MobiVM, you can create a Java project and convert it into an .xcframework, supporting various architectures, such as arm64 and x86_64. You can use this framework in your Swift and Objective-C projects.

## Setting up your environment

Before diving into coding your Java framework, you should complete the following steps:

1. Install an IDE supporting Java (IntelliJ IDEA, Android Studio, Eclipse)
2. Install the MobiVM plugin
    - You can download it from [here](https://plugins.jetbrains.com/plugin/14440-mobivm) if you’re using IntelliJ
    - For other IDEs, please follow this [guide](https://mobivm.github.io/dev/) by MobiVM

## “Hello from Java!”

Now that the environment is set up, let's start exploring MobiVM through a straightforward demo project. This project consists of a single Java class (DemoClass) with a small function that returns “Hello from Java!”.

The first step is to create a new MobiVM project. The plugin makes this process pretty straightforward. You need to open the new project dialog in your IDE and select RoboVM → RoboVM iOS Framework. I’ve already done it for you and it can be accessed [here](https://github.com/terlan98/mobivm-demo).

- If you’d like to create the project on your own, check out [this](https://dkimitsa.github.io/2018/01/16/tutorial-writing-framework-improved/#creating-framework-step-by-step) guide. By default, the created project contains some code which I’ve removed to keep it as simple as possible.

As you can see, the `DemoClass` is annotated with the `@CustomClass` annotation and contains a `hello()` method, annotated with the `@Method` annotation.

```java
@CustomClass("DemoClass")
public class DemoClass extends NSObject {
  @Method(selector = "hello")
  public static String hello() {
    return "Hello from Java!";
  }
}
```

The annotations help MobiVM wire things up in the background. Without them, your Java classes and methods won’t be accessible from Swift.

In addition to the annotations, we need an Objective-C header file (`headers/MyFramework.h`) containing the interface for the `DemoClass`:

```objectivec
#import <Foundation/Foundation.h>

@interface DemoClass: NSObject
+(NSString*)hello;
@end
```

The header must accurately represent the function’s signature. Otherwise, it won’t behave as expected when called. Since the `hello()` function returns a `String`, we use the corresponding `NSString*` return type in our header.

In addition, the header must be exposed via the modulemap located at `modules/module.modulemap` as follows:

```objectivec
framework module MyFramework {
  umbrella header "MyFramework.h"

  export *
  module * { export * }
}
```

In order to create an `xcframework` from your code, run the following terminal command in the project directory:

```bash
mvn -Drobovm.enableBitcode=true compile robovm:install
```

<br>

This command will generate an xcframework file under `target/robovm`. In order to test the code, you should drag and drop this file into your iOS project.
<img src="../images/xcode-filetree.png" width=30%>

Once added to the project, the framework can be imported and used as follows. You can also find a demo iOS app [here](https://github.com/terlan98/mobivm-demo). 

```swift
import SwiftUI
import MyFramework // <- Importing the Java framework

struct ContentView: View {
    @State private var someText = ""
    
    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "cup.and.saucer.fill")
                .imageScale(.large)
                .foregroundStyle(.tint)
            Text(someText)
        }
        .padding()
        .onAppear {
            // Calling the Java method
            someText = DemoClass.hello() ?? someText
        }
    }
}
```

<br>
As you can see below, the function call succeeded 🎉:
<img src="../images/iphone-success.png#center" width=40%>

If you’d like to specify the architectures that the framework should be built for, modify the `robovm.xml` file:

```xml
<config>
    <!-- The framework targets iOS. -->
    <os>ios</os>

    <!-- Build a fat binary containing 32-bit and 64-bit machine code for both
    devices and the simulator. -->
    <arch>arm64</arch>
    <arch>arm64-simulator</arch>
    <arch>x86_64</arch>
    
    <!-- We're building an xcframework containing binaries for multiple architectures. -->
    <target>xcframework</target>

...
```

## Issues
Getting MobiVM to work with my Java code wasn't straightforward as I encountered some challenges mentioned below.

### EXC_BAD_ACCESS

Adding a dependency to the Java project may sometimes lead to an EXC_BAD_ACCESS crash in your app. If this happens, you’ll have to add the missing package (or class) names under the `<forceLinkClasses>` tag of your `robovm.xml` file. Here’s an example:

```xml
<forceLinkClasses>
    <!-- Your Java package must be included -->
    <pattern>com.mycompany.myframework.**</pattern>
    
    <!-- An example dependency -->
    <pattern>com.example.dependency.**</pattern>

</forceLinkClasses>
```

> ⚠️  Force linking too many classes is not a good idea as it increases the size of your framework. As an optimization, you can replace the wildcard imports with more specific ones, linking only the classes that your code depends on.

### Duplicate symbol ‘_OBJC_CLASS_$_NSUserActivity'

When I attempted to build my project for the `arm64-simulator` architecture, the build failed with the following error:

```
[ERROR] ld: building exports trie: duplicate symbol '_OBJC_CLASS_$_NSUserActivity'
[ERROR] clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Updating Xcode to the latest version solved the problem.

### Java 8-related NoClassDefFoundError

At the time of writing this post, MobiVM is compatible with Java 7. If your code or dependencies require Java 8 or above, you’ll get a NoClassDefFoundError when you call your Java methods. To resolve this issue, consider using an [experimental build](https://github.com/robovmx/robovmx/releases) of MobiVM.

## Things to Keep in Mind

While MobiVM is incredibly helpful in situations where coding in Java is unavoidable, it also comes with certain drawbacks. It is definitely a good idea to look for native Swift solutions before considering this approach. 

### Development Process

The development process gets complicated since now you have to maintain two separate codebases. The typical development process involves updating the Java code, converting it into an `xcframework`, moving the framework into an iOS project and testing the changes. 

### Debugging

Even if you do your best to handle Java exceptions properly in your project, some unchecked exceptions might slip through. When this happens, the app crashes, and it’s not always easy to determine the cause via Xcode. This difficulty extends to the debugging process in general. Although the Xcode debugger allows placing breakpoints and observing code execution step-by-step, this only works until you call a Java method. I often found myself adding ugly `System.out.println` statements in my Java code for debugging.

### Lack of resources

While investigating ways to execute Java on iOS, I realized that this is a niche need among developers, which is why there are few alternatives to MobiVM available right now. Not being an expert in MobiVM, I consistently rely on the GitHub community for support whenever I’m stuck. MobiVM has an awesome [discussions page](https://github.com/MobiVM/robovm/discussions) where you can quickly get your questions answered. Besides that, support options are limited, with Stack Overflow and ChatGPT providing minimal help.

## Conclusion
Despite the challenges, MobiVM made it possible to incorporate an important feature that brought high business value to the iOS project I was working on. This post is my way of paying it forward, hoping it proves beneficial to you as well.

Cheers! ✨