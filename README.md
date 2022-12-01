
# Examples of iOS/XCode project with rust libraries

# Backgound
This Basic work is based on the following articles, some pieces were takes from each one to get to the final result:

1. (older piece about xcode from Mozilla)[https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-06-rust-on-ios.html]
2. (more contemporary piece about xcode integration with a simpler shared lib directory structure)[https://blog.mozilla.org/data/2022/01/31/this-week-in-glean-building-and-deploying-a-rust-library-on-ios/#fn1]
3. (Another great but little outdated medium post from 2019 where I found while looking for a way to automatically generate the c header file and found `cbindgen`)[https://medium.com/visly/rust-on-ios-39f799b3c1dd)]

# iOS

1. make sure you have the xcode command line `xcode-select --install`
   
## Rust

1. Make sure you have rustup `curl https://sh.rustup.rs -sSf | sh`
2. Add libs and targets: `rustup target add aarch64-apple-ios-sim armv7s-apple-ios x86_64-apple-ios i386-apple-ios`
3. Add `cargo install cbindgen`.

## Xcode

To link our rust library to XCode we need to follow the following steps:

1. Link function signatures to XCode/Clang in the form of a header (`.h`) file.
2. Link swift to this `.h` file by specifying a bridging header. 
3. Run a script as part of the XCode build phase that executes the rust command based matching the rust `profile` with a build variant (`release`/`debug`) and architecture (provided as env var `$ARCH`).
4. Sepcify a static linked lib (`*.a`) as to be linked at build time
5. Match XCode's library search paths so it can find the right (rust) build artifact (`*.a`) for each architecture and build variants (i.e `$(PROJECT_DIR)/target/aarch64-apple-ios/debug`)
6. Make sure all the above happen before XCode starts compiling the `swift` / `objective-c` files. (make sure the order build phase makes sense).


### Linking Headers
The header is the actual C interface declaration that xcode/clang will use to validate and link function calls from withing the iOS app. That happens in the linking phase of compilation. Here clang does not need to know about architecture but, C macros might be applied conditionally based on the variants (i.e `#ifdef DEBUG`)

1. To create a C header that includes the `extern "C"` functions signatures, and tell clang with XCode to bridge these functions to swift as well.and swift/objective c 
2. create the header file: `cbindgen shipping-rust-ffi/src/lib.rs -l c > rust.h` in the root folder. or, add this command as a build phase: 1. in xcode=>target=>build phases add Run Script Phase, make sure it is at the bare start of the ordered list of phases. see example in the project itself calling `bin/create-header.sh`.
3. Add `rust.h` to the Headers in xcode=>target=>build phases=> as a project header.
4. (Create a bridging header in xcode)[https://developer.apple.com/documentation/swift/importing-objective-c-into-swift]. withing the bridging header, import `rust.h`. make sure the target's build configuration bridging header is specified and the directory is the root direction like such:`$(PROJECT_DIR)/BridgingHeader.h`

### Linking Library
The library is the actual binary that xcode/clang will use to generate it's build artifacts (multiple platforms and variant) to create iOS app, for simulator/device based on arch (arm/intel) and variant (release/debug).

Add `libshipping_rust_ffi.a` to the xcode => target => build phases => link binary with libraries. Then open `<ProjectName>.xcodeproj/project.xcproj` and search for `LIBRARY_SEARCH_PATHS` add and replace for `Debug` configuration with:
```javascript
    "LIBRARY_SEARCH_PATHS[sdk=iphoneos*][arch=arm64]" = "$(PROJECT_DIR)/target/aarch64-apple-ios/debug";
    "LIBRARY_SEARCH_PATHS[sdk=iphonesimulator*][arch=arm64]" = "$(PROJECT_DIR)/target/aarch64-apple-ios-sim/debug";
    "LIBRARY_SEARCH_PATHS[sdk=iphonesimulator*][arch=x86_64]" = "$(PROJECT_DIR)/target/x86_64-apple-ios/debug";
```
and the following for `Release` configuration:
```javascript
				"LIBRARY_SEARCH_PATHS[sdk=iphoneos*][arch=arm64]" = "$(PROJECT_DIR)/target/aarch64-apple-ios/release";
				"LIBRARY_SEARCH_PATHS[sdk=iphonesimulator*][arch=arm64]" = "$(PROJECT_DIR)/target/aarch64-apple-ios-sim/release";
				"LIBRARY_SEARCH_PATHS[sdk=iphonesimulator*][arch=x86_64]" = "$(PROJECT_DIR)/target/x86_64-apple-ios/release";
```

# XCode Compile
1. Make sure everything is ok on the rust side and it build with no errors: `cargo build`.
2. 'Add User Defined Setting' in xcode=>target=>build setting, call it `buildvariant` and input `release` under the `Release` configuration and 'debug' under the 'Debug' configuration.
3. Add a script to the xcode=>target=>build phase, name it `Build Rust library`. This will call a script that will run build the rust library. the script will be: `bash ${PROJECT_DIR}/bin/compile-library.sh shipping-rust-ffi $buildvariant` This will use the xcode build configuration (variant, arch etc)
4. Make sure the build phase order makes sense.
5. Run the compile with any of the device targets or variants.