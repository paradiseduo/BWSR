# BWSR
Inline Hooking Library for Arm64 devices across multiple platforms.

Platform Support:
- iOS (arm64/arm64e).
- Android (arm64).
- Linux (arm64).
- OSX (arm64/arm64e).

BWSR is a versatile inline hooking library implemented entirely in `C`, designed to facilitate dynamic code modification and interception on `Linux`, `OSX`, `iOS`, and `Android`.

## Building
Building `iOS` and `MacOSX` requires a Mac machine with `X-Code` and `Command-Line-Tools` installed.

Building `Android` requires a recent AndroidNDK downloaded and have the `ANDROID_NDK` environmental set to the path of the NDK.

Building `Linux` requires that `build-essential` and `clang` are installed.

To make the hooking library and all the example programs simply run:
```sh
make
```

To make only the hooking library
```sh
make hooklibs
```

To make the example program
```sh
make examples
```

## Symbol Resolver (Locating the Address of a Function)

### Symbol Resolution When the Image IS Known
This will return the address of the first occurence encountered of only the mach header matching the specific image name and the specific function name.
```c
uintptr_t function_address = 0;

// iOS
retVal = BWSR_ResolveSymbol( "AudioUnitProcess", "AudioToolbox", &function_address );

// Android
retVal = BWSR_ResolveSymbol( "open", "/apex/com.android.runtime/lib64/bionic/libc.so", &function_address );
```

### Symbol Resolution When the Image Is **NOT** Known
This will return the address of the first occurence encounted in any mach header for the matching function name. This is usually fine but if the function name is common enough you may get the address of the wrong function.
```c
uintptr_t function_address = 0;

// iOS/Mac
retVal = BWSR_ResolveSymbol( "AudioUnitProcess", NULL, &function_address );

// Android/Linux
retVal = BWSR_ResolveSymbol( "open", NULL, &function_address );
```

## Inline Hooking
Hooks and code page backups are handled internally, so there is no need to worry about reverting hooks or memory leaks.

### Simple Inline Hook
The easiest and most user friendly way to hook
```c
// Function receiving the printf calls
int hook_printf( const char* text, ... ) {
    return -1;
}

// A simple hook of printf
BWSR_InlineHook( printf, hook_printf, NULL, NULL, NULL );

// Cleanup the hook
BWSR_DestroyHook( printf );
```

### Original Function Callback
It may be benefitial to have a pointer to where the original function to use inside of the hooked function for recording or altering parameters.
```c
// Pointer to the original creat()
void* original_creat = NULL;

// The version replacing creat()
int hcreat( const char * files, mode_t modes ) {
    BWSR_DEBUG( LOG_CRITICAL, "SUCCESS! Caught creat()\n" );
    if( NULL != original_creat )
        return ((__typeof(creat)*)original_creat)("/some/new/location", modes);
    return 0;
}

BWSR_InlineHook( creat, hcreat, &original_creat, NULL, NULL );
```


### Codesign Friendly
On iOS it may be benefitial to know the address of the page where the hook is employed before or after the hook is written out. In the snippet below, is an example of a callback triggered before and after the modification of the code page is done.
```c
void BeforePageWriteCallbackFn( uintptr_t PageAddress ) {
    BWSR_DEBUG( LOG_CRITICAL, "PageAddress: %" PRIuPTR "\n", PageAddress );
    /// Do something before the write occurs.
}

void AfterPageWriteCallbackFn( uintptr_t PageAddress ) {
    BWSR_DEBUG( LOG_ALERT, "PageAddress: %" PRIuPTR "\n", PageAddress );
    // Do something after the write occurs...
    // Update CDHashes, etc.
}

BWSR_InlineHook( printf, hook_printf, NULL, BeforePageWriteCallbackFn, AfterPageWriteCallbackFn );
```

### Simple Hook Cleanup
It may be benefitial to undo all hooks and release all memory associated with hooks at once. This can be done at any time through one simple call.
```c
BWSR_DestroyAllHooks();
```

## Memory Tracker
> [!IMPORTANT]
> The memory tracker is only used when `DEBUG_MODE` is defined. It is not used for release builds.

The memory tracker was built to ensure all memory leaks are resolved. In order for the memory tracker to successfully track allocations the calls BWSR allocation and free calls be used. The current allocations/leaked memory at any time with a simple call.
```c
size_t leaks = 0;

leaks = MemoryTracker_CheckForMemoryLeaks();

if( leaks > 0 )
{
    // Uh-oh!
}
```

## TODO
The list of items that needs to be done is far longer than this list, but these are these are the next important goals:
- Dynamic entitlements for iOS.

## ACKNOWLEDGMENTS
- https://github.com/kubo/plthook/
- https://github.com/jmpews/Dobby
- https://github.com/asLody/whale