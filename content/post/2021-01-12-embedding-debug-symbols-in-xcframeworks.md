---
title: "Embedding Debug Symbols in XCFrameworks"
subtitle: ""
date: 2021-01-12T07:00:00-06:00
tags: ["xcframework"]
---

When creating and distributing a binary, a [dSYM](https://developer.apple.com/documentation/xcode/building_your_app_to_include_debugging_information) and BCSymbolMaps are generated (if [Bitcode](https://pspdfkit.com/guides/ios/current/faq/bitcode/) is enabled) and these artifacts contain debug information about the code you just compiled. Prior to Xcode 12, when creating an [XCFramework](https://developer.apple.com/videos/play/wwdc2019/416/), these additional files needed to be distributed separately or manually added into the XCFramework. Now, `xcodebuild` has a new option when executing the `-create-xcframework` command: `-debug-symbols`.

However, the use of this option is not straightforward. The first gotcha is that the path passed to the flag must be an absolute path (not relative). This is unfortunately not documented in the help output. This is ok for dSYMs as you always know where they are (e.g. `"$(pwd -P)"/build/artifacts/iOS.xcarchive/dSYMs/MyFramework.framework.dSYM`), BCSymbolMaps are a different story as they contain a random UUID for a name. Therefore, you need use `find` (or another solution) to get a list of the files to add.

The second thing to note is that you need to use the `-debug-symbols` option multiple times to add all of the dSYMs and BCSymbolMaps to the generated XCFramework. But, they need to be associated with the right target. This means that after passing `-framework`, you need to pass all of the debug symbols associated with that particular input as the command does not know how to associate those symbols to the right target; i.e. you can't shove them all in as the last argument after passing in all of the framework inputs. If you don't do this, the debug symbols are added to the wrong output or the command errors out with a duplicate file name error for the dSYMs. 

Putting this altogether, the process for creating an XCFramework for iOS (as an example) looks like:

1. Archive the framework for devices.
2. Archive the framework for simulators.
3. Archive the framework for Mac Catalyst.
4. Start making an `xcodebuild` command with the `-create-xcframework` option passing in the device framework, its dSYM, and its BCSymbolMap using the appropriate options. Note: there will be two BCSymbolMaps instead of one if you force the arm64e architecture.
5. Add the simulator framework and its dSYM to the command above.
6. Add the Mac Catalyst framework and its dSYM to the command above.
7. Execute the command and verify the output is a valid XCFramework.

---

XCFrameworks are very convenient methods of distributing code across multiple platforms and architectures, but I wish it was easier to do. Having to create a script to ensure that all of the pieces are defined rather than having the command know how to search directories is inconvenient and easy to get wrong. Perhaps in a newer version of Xcode, `xcodebuild` will get smarter and do the work for us.
