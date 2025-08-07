---

title: A Hacky Way to Use raylib in Zig
layout: post
author: Kyle Cooper
category: blog 
excerpt_separator: 
---

HACK: Getting raylib and Zig to work together
<!--more-->

Lets say you want to use the [raylib](https://www.raylib.com/) with Zig. Well, at least as of writing, there is a build.zig  
in the rayib project. 

You will need the following versions of raylib and Zig:

*Version of Zig*:0.14.0
*Version of rayib*: [5.5](https://github.com/raysan5/raylib/tree/5.5)
*OS: Ubuntu 24 LTS*

There is also a method to do that with the zon package manager, I have not tried this. 

Lets say you have the following directory structure for your Zig project: 

```txt
src/
    main.zig 
third_party/raylib/[all things raylib in this directory]
build.zig
```
raylib is within the third_party directory, you can add it here however you like: copy paste or as a git submodule.


Your projects build.zig file (the one at the root level) will look something like this:

```zig
const std = @import("std");

const rayLib = @import("third_party/raylib/build.zig");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "tmp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // this is where we start to build the zig as a library and our project:
    const raylib = try rayLib.compileRaylib(b, target, optimize, rayLib.Options.getOptions(b));

    // You will need to call out the headers that ray lib provides. 
    // I found it more reliable to all them out specifically:
    raylib.installHeader(b.path("third_party/raylib/src/raylib.h"), "raylib.h");
    raylib.installHeader(b.path("third_party/raylib/src/raymath.h"), "raymath.h");
    raylib.installHeader(b.path("third_party/raylib/src/rlgl.h"), "rlgl.h");

    exe.addIncludePath(b.path("third_party/raylib/src"));

    // link it and build into our zig project:
    exe.linkLibrary(raylib);
    b.installArtifact(exe);

    // the rest of these just allow for tests and you to use zig build run to run your own project
    const runCmd = b.addRunArtifact(exe);
    runCmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| {
        runCmd.addArgs(args);
    }
    const runStep = b.step("run", "Run the app");
    runStep.dependOn(&runCmd.step);
    const unitTests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // unit tests
    const runUnitTests = b.addRunArtifact(unitTests);
    const testStep = b.step("test", "Run unit tests");
    testStep.dependOn(&runUnitTests.step);
}
```

However, this will not work without modifying the build.zig file at the root of the raylib directory. 

## The hacky raylib build.zig

Overall you need to update some methods to use the newer Zig build functionality and then hack together some dynamic linking, and paths.

Your not going to be happy but here is the diff: 

```zig
diff --git a/build.zig b/../zig-game/third_party/raylib/build.zig
index 5c2ac77c..127243da 100644
--- a/build.zig
+++ b/../zig-game/third_party/raylib/build.zig
@@ -2,7 +2,7 @@ const std = @import("std");
 const builtin = @import("builtin");
 
 /// Minimum supported version of Zig
-const min_ver = "0.13.0";
+const min_ver = "0.14.0";
 
 comptime {
     const order = std.SemanticVersion.order;
@@ -11,13 +11,20 @@ comptime {
         @compileError("Raylib requires zig version " ++ min_ver);
 }
 
+// It looks like dynamic linking works around an issue with the zig build system, similar to:
+// https://github.com/ziglang/zig/issues/20476
+const dynLinkOpts: std.Build.Module.LinkSystemLibraryOptions = .{
+    .preferred_link_mode = .dynamic,
+    .search_strategy = .mode_first,
+};
+
 fn setDesktopPlatform(raylib: *std.Build.Step.Compile, platform: PlatformBackend) void {
-    raylib.defineCMacro("PLATFORM_DESKTOP", null);
+    raylib.root_module.addCMacro("PLATFORM_DESKTOP", "");
 
     switch (platform) {
-        .glfw => raylib.defineCMacro("PLATFORM_DESKTOP_GLFW", null),
-        .rgfw => raylib.defineCMacro("PLATFORM_DESKTOP_RGFW", null),
-        .sdl => raylib.defineCMacro("PLATFORM_DESKTOP_SDL", null),
+        .glfw => raylib.root_module.addCMacro("PLATFORM_DESKTOP_GLFW", ""),
+        .rgfw => raylib.root_module.addCMacro("PLATFORM_DESKTOP_RGFW", ""),
+        .sdl => raylib.root_module.addCMacro("PLATFORM_DESKTOP_SDL", ""),
         else => {},
     }
 }
@@ -76,7 +83,7 @@ const config_h_flags = outer: {
     break :outer flags[0..i].*;
 };
 
-fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.builtin.OptimizeMode, options: Options) !*std.Build.Step.Compile {
+pub fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.builtin.OptimizeMode, options: Options) !*std.Build.Step.Compile {
     var raylib_flags_arr = std.ArrayList([]const u8).init(b.allocator);
     defer raylib_flags_arr.deinit();
 
@@ -148,31 +155,32 @@ fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.
     }
 
     var c_source_files = try std.ArrayList([]const u8).initCapacity(b.allocator, 2);
-    c_source_files.appendSliceAssumeCapacity(&.{ "src/rcore.c", "src/utils.c" });
+    c_source_files.appendSliceAssumeCapacity(&.{ "third_party/raylib/src/rcore.c", "third_party/raylib/src/utils.c" });
 
     if (options.raudio) {
-        try c_source_files.append("src/raudio.c");
+        try c_source_files.append("third_party/raylib/src/raudio.c");
     }
     if (options.rmodels) {
-        try c_source_files.append("src/rmodels.c");
+        try c_source_files.append("third_party/raylib/src/rmodels.c");
     }
     if (options.rshapes) {
-        try c_source_files.append("src/rshapes.c");
+        try c_source_files.append("third_party/raylib/src/rshapes.c");
     }
     if (options.rtext) {
-        try c_source_files.append("src/rtext.c");
+        try c_source_files.append("third_party/raylib/src/rtext.c");
     }
     if (options.rtextures) {
-        try c_source_files.append("src/rtextures.c");
+        try c_source_files.append("third_party/raylib/src/rtextures.c");
     }
 
     if (options.opengl_version != .auto) {
-        raylib.defineCMacro(options.opengl_version.toCMacroStr(), null);
+        //                                                                was null
+        raylib.root_module.addCMacro(options.opengl_version.toCMacroStr(), "");
     }
 
     switch (target.result.os.tag) {
         .windows => {
-            try c_source_files.append("src/rglfw.c");
+            try c_source_files.append("third_party/raylib/src/rglfw.c");
             raylib.linkSystemLibrary("winmm");
             raylib.linkSystemLibrary("gdi32");
             raylib.linkSystemLibrary("opengl32");
@@ -181,19 +189,20 @@ fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.
         },
         .linux => {
             if (options.platform != .drm) {
-                try c_source_files.append("src/rglfw.c");
+                try c_source_files.append("third_party/raylib/src/rglfw.c");
 
                 if (options.linux_display_backend == .X11 or options.linux_display_backend == .Both) {
-                    raylib.defineCMacro("_GLFW_X11", null);
-                    raylib.linkSystemLibrary("GLX");
-                    raylib.linkSystemLibrary("X11");
-                    raylib.linkSystemLibrary("Xcursor");
-                    raylib.linkSystemLibrary("Xext");
-                    raylib.linkSystemLibrary("Xfixes");
-                    raylib.linkSystemLibrary("Xi");
-                    raylib.linkSystemLibrary("Xinerama");
-                    raylib.linkSystemLibrary("Xrandr");
-                    raylib.linkSystemLibrary("Xrender");
+                    raylib.root_module.addCMacro("_GLFW_X11", "");
+
+                    raylib.linkSystemLibrary2("GLX", dynLinkOpts);
+                    raylib.linkSystemLibrary2("X11", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xcursor", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xext", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xfixes", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xi", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xinerama", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xrandr", dynLinkOpts);
+                    raylib.linkSystemLibrary2("Xrender", dynLinkOpts);
                 }
 
                 if (options.linux_display_backend == .Wayland or options.linux_display_backend == .Both) {
@@ -204,35 +213,43 @@ fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.
                         , .{});
                         @panic("`wayland-scanner` not found");
                     };
-                    raylib.defineCMacro("_GLFW_WAYLAND", null);
+
+                    std.log.info("BUILD {} ENVS", .{options.linux_display_backend});
+
+                    raylib.root_module.addCMacro("_GLFW_WAYLAND", "");
                     raylib.linkSystemLibrary("EGL");
                     raylib.linkSystemLibrary("wayland-client");
                     raylib.linkSystemLibrary("xkbcommon");
-                    waylandGenerate(b, raylib, "wayland.xml", "wayland-client-protocol");
-                    waylandGenerate(b, raylib, "xdg-shell.xml", "xdg-shell-client-protocol");
-                    waylandGenerate(b, raylib, "xdg-decoration-unstable-v1.xml", "xdg-decoration-unstable-v1-client-protocol");
-                    waylandGenerate(b, raylib, "viewporter.xml", "viewporter-client-protocol");
-                    waylandGenerate(b, raylib, "relative-pointer-unstable-v1.xml", "relative-pointer-unstable-v1-client-protocol");
-                    waylandGenerate(b, raylib, "pointer-constraints-unstable-v1.xml", "pointer-constraints-unstable-v1-client-protocol");
-                    waylandGenerate(b, raylib, "fractional-scale-v1.xml", "fractional-scale-v1-client-protocol");
-                    waylandGenerate(b, raylib, "xdg-activation-v1.xml", "xdg-activation-v1-client-protocol");
-                    waylandGenerate(b, raylib, "idle-inhibit-unstable-v1.xml", "idle-inhibit-unstable-v1-client-protocol");
+
+                    const waylandDir = "third_party/raylib/src/external/glfw/deps/wayland";
+
+                    waylandGenerate(b, raylib, "wayland.xml", "wayland-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "xdg-shell.xml", "xdg-shell-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "xdg-decoration-unstable-v1.xml", "xdg-decoration-unstable-v1-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "viewporter.xml", "viewporter-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "relative-pointer-unstable-v1.xml", "relative-pointer-unstable-v1-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "pointer-constraints-unstable-v1.xml", "pointer-constraints-unstable-v1-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "fractional-scale-v1.xml", "fractional-scale-v1-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "xdg-activation-v1.xml", "xdg-activation-v1-client-protocol", waylandDir);
+                    waylandGenerate(b, raylib, "idle-inhibit-unstable-v1.xml", "idle-inhibit-unstable-v1-client-protocol", waylandDir);
+
+                    raylib.addIncludePath(b.path("third_party/raylib/src/external/glfw/include/"));
                 }
 
                 setDesktopPlatform(raylib, options.platform);
             } else {
                 if (options.opengl_version == .auto) {
                     raylib.linkSystemLibrary("GLESv2");
-                    raylib.defineCMacro("GRAPHICS_API_OPENGL_ES2", null);
+                    raylib.root_module.addCMacro("GRAPHICS_API_OPENGL_ES2", "");
                 }
 
                 raylib.linkSystemLibrary("EGL");
                 raylib.linkSystemLibrary("gbm");
                 raylib.linkSystemLibrary2("libdrm", .{ .use_pkg_config = .force });
 
-                raylib.defineCMacro("PLATFORM_DRM", null);
-                raylib.defineCMacro("EGL_NO_X11", null);
-                raylib.defineCMacro("DEFAULT_BATCH_BUFFER_ELEMENT", "2048");
+                raylib.root_module.addCMacro("PLATFORM_DRM", "");
+                raylib.root_module.addCMacro("EGL_NO_X11", "");
+                raylib.root_module.addCMacro("DEFAULT_BATCH_BUFFER_ELEMENT", "2048");
             }
         },
         .freebsd, .openbsd, .netbsd, .dragonfly => {
@@ -283,9 +300,9 @@ fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.
                 raylib.addIncludePath(dep.path("upstream/emscripten/cache/sysroot/include"));
             }
 
-            raylib.defineCMacro("PLATFORM_WEB", null);
+            raylib.root_module.addCMacro("PLATFORM_WEB", "");
             if (options.opengl_version == .auto) {
-                raylib.defineCMacro("GRAPHICS_API_OPENGL_ES2", null);
+                raylib.root_module.addCMacro("GRAPHICS_API_OPENGL_ES2", "");
             }
         },
         else => {
@@ -301,11 +318,18 @@ fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.
     return raylib;
 }
 
-pub fn addRaygui(b: *std.Build, raylib: *std.Build.Step.Compile, raygui_dep: *std.Build.Dependency) void {
+pub fn addRaygui(
+    b: *std.Build,
+    raylib: *std.Build.Step.Compile,
+    //raygui_dep: *std.Build.Dependency,
+    raygui_dep: *std.Build,
+    ) void {
     var gen_step = b.addWriteFiles();
     raylib.step.dependOn(&gen_step.step);
 
     const raygui_c_path = gen_step.add("raygui.c", "#define RAYGUI_IMPLEMENTATION\n#include \"raygui.h\"\n");
+    //const basePath = "./third_party/raylib/";
+
     raylib.addCSourceFile(.{ .file = raygui_c_path });
     raylib.addIncludePath(raygui_dep.path("src"));
 
@@ -327,7 +351,7 @@ pub const Options = struct {
 
     const defaults = Options{};
 
-    fn getOptions(b: *std.Build) Options {
+    pub fn getOptions(b: *std.Build) Options {
         return .{
             .platform = b.option(PlatformBackend, "platform", "Choose the platform backedn for desktop target") orelse defaults.platform,
             .raudio = b.option(bool, "raudio", "Compile with audio support") orelse defaults.raudio,
@@ -403,8 +427,8 @@ fn waylandGenerate(
     raylib: *std.Build.Step.Compile,
     comptime protocol: []const u8,
     comptime basename: []const u8,
+    comptime waylandDir: []const u8,
 ) void {
-    const waylandDir = "src/external/glfw/deps/wayland";
     const protocolDir = b.pathJoin(&.{ waylandDir, protocol });
     const clientHeader = basename ++ ".h";
     const privateCode = basename ++ "-code.h";
@@ -415,6 +439,7 @@ fn waylandGenerate(

```

## A summary of the changes:
- Update the version of zig.
- Re-add some dynamic linking
- Update how macros are defined in the new zig build system
- Needing to change some of the file paths for c files to understand the third_party directory it now lives in 
- Update some of the paths for the Wayland code
- Changing visibility of methods so that our build.zig can call into raylibs build.zig


## There is still an issue BUT IT WORKS...
We get some linker warnings
```
install
└─ install tmp
   └─ zig build-exe tmp Debug native failure
error: warning(link): unexpected LLD stderr:
ld.lld: warning: [..]/.zig-cache/o/9d5dde655db8f68af92a2145049ec451/libraylib.a: archive member '/usr/lib/x86_64-linux-gnu/libGLX.so' is neither ET_REL nor LLVM bitcode
ld.lld: warning: [..]/.zig-cache/o/9d5dde655db8f68af92a2145049ec451/libraylib.a: archive member '/usr/lib/x86_64-linux-gnu/libX11.so' is neither ET_REL nor LLVM bitcode
ld.lld: warning: [..]/.zig-cache/o/9d5dde655db8f68af92a2145049ec451/libraylib.a: archive member '/usr/lib/x86_64-linux-gnu/libXcursor.so' is neither ET_REL nor LLVM bitcode

....
```
For now things build, and raylib works well. This is a future issue to fix


## You can now use raylib in zig
You can now import raylib with something like the following:
```zig
const rLib = @cImport({
    @cInclude("raylib.h");
    @cInclude("raymath.h");
});

... your zig code here.. 

```

You can also import each header alone like this:

```zig
const rMath = @cImport({
    @cInclude("raymath.h");
});
const rLib = @cImport({
    @cInclude("raylib.h");
});
```

However, this can cause issues with how #defined structs (and more) are defined inside of raylib. For instance a rMath.Vector3 is now now the same as a rLib.Vector3. Something to keep in mind.


# The full code for raylibs to work in your project

```zig
// backup of the zig build
const std = @import("std");
const builtin = @import("builtin");

/// Minimum supported version of Zig
const min_ver = "0.14.0";

comptime {
    const order = std.SemanticVersion.order;
    const parse = std.SemanticVersion.parse;
    if (order(builtin.zig_version, parse(min_ver) catch unreachable) == .lt)
        @compileError("Raylib requires zig version " ++ min_ver);
}

// It looks like dynamic linking works around an issue with the zig build system, similar to:
// https://github.com/ziglang/zig/issues/20476
const dynLinkOpts: std.Build.Module.LinkSystemLibraryOptions = .{
    .preferred_link_mode = .dynamic,
    .search_strategy = .mode_first,
};

fn setDesktopPlatform(raylib: *std.Build.Step.Compile, platform: PlatformBackend) void {
    raylib.root_module.addCMacro("PLATFORM_DESKTOP", "");

    switch (platform) {
        .glfw => raylib.root_module.addCMacro("PLATFORM_DESKTOP_GLFW", ""),
        .rgfw => raylib.root_module.addCMacro("PLATFORM_DESKTOP_RGFW", ""),
        .sdl => raylib.root_module.addCMacro("PLATFORM_DESKTOP_SDL", ""),
        else => {},
    }
}

fn createEmsdkStep(b: *std.Build, emsdk: *std.Build.Dependency) *std.Build.Step.Run {
    if (builtin.os.tag == .windows) {
        return b.addSystemCommand(&.{emsdk.path("emsdk.bat").getPath(b)});
    } else {
        return b.addSystemCommand(&.{emsdk.path("emsdk").getPath(b)});
    }
}

fn emSdkSetupStep(b: *std.Build, emsdk: *std.Build.Dependency) !?*std.Build.Step.Run {
    const dot_emsc_path = emsdk.path(".emscripten").getPath(b);
    const dot_emsc_exists = !std.meta.isError(std.fs.accessAbsolute(dot_emsc_path, .{}));

    if (!dot_emsc_exists) {
        const emsdk_install = createEmsdkStep(b, emsdk);
        emsdk_install.addArgs(&.{ "install", "latest" });
        const emsdk_activate = createEmsdkStep(b, emsdk);
        emsdk_activate.addArgs(&.{ "activate", "latest" });
        emsdk_activate.step.dependOn(&emsdk_install.step);
        return emsdk_activate;
    } else {
        return null;
    }
}

/// A list of all flags from `src/config.h` that one may override
const config_h_flags = outer: {
    // Set this value higher if compile errors happen as `src/config.h` gets larger
    @setEvalBranchQuota(1 << 20);

    const config_h = @embedFile("src/config.h");
    var flags: [std.mem.count(u8, config_h, "\n") + 1][]const u8 = undefined;

    var i = 0;
    var lines = std.mem.tokenizeScalar(u8, config_h, '\n');
    while (lines.next()) |line| {
        if (!std.mem.containsAtLeast(u8, line, 1, "SUPPORT")) continue;
        if (std.mem.startsWith(u8, line, "//")) continue;
        if (std.mem.startsWith(u8, line, "#if")) continue;

        var flag = std.mem.trimLeft(u8, line, " \t"); // Trim whitespace
        flag = flag["#define ".len - 1 ..]; // Remove #define
        flag = std.mem.trimLeft(u8, flag, " \t"); // Trim whitespace
        flag = flag[0 .. std.mem.indexOf(u8, flag, " ") orelse continue]; // Flag is only one word, so capture till space
        flag = "-D" ++ flag; // Prepend with -D

        flags[i] = flag;
        i += 1;
    }

    // Uncomment this to check what flags normally get passed
    //@compileLog(flags[0..i].*);
    break :outer flags[0..i].*;
};

pub fn compileRaylib(b: *std.Build, target: std.Build.ResolvedTarget, optimize: std.builtin.OptimizeMode, options: Options) !*std.Build.Step.Compile {
    var raylib_flags_arr = std.ArrayList([]const u8).init(b.allocator);
    defer raylib_flags_arr.deinit();

    try raylib_flags_arr.appendSlice(&[_][]const u8{
        "-std=gnu99",
        "-D_GNU_SOURCE",
        "-DGL_SILENCE_DEPRECATION=199309L",
        "-fno-sanitize=undefined", // https://github.com/raysan5/raylib/issues/3674
    });

    if (options.shared) {
        try raylib_flags_arr.appendSlice(&[_][]const u8{
            "-fPIC",
            "-DBUILD_LIBTYPE_SHARED",
        });
    }

    if (options.config.len > 0) {
        // Sets a flag indiciating the use of a custom `config.h`
        try raylib_flags_arr.append("-DEXTERNAL_CONFIG_FLAGS");

        // Splits a space-separated list of config flags into multiple flags
        //
        // Note: This means certain flags like `-x c++` won't be processed properly.
        // `-xc++` or similar should be used when possible
        var config_iter = std.mem.tokenizeScalar(u8, options.config, ' ');

        // Apply config flags supplied by the user
        while (config_iter.next()) |config_flag|
            try raylib_flags_arr.append(config_flag);

        // Apply all relevant configs from `src/config.h` *except* the user-specified ones
        //
        // Note: Currently using a suboptimal `O(m*n)` time algorithm where:
        // `m` corresponds roughly to the number of lines in `src/config.h`
        // `n` corresponds to the number of user-specified flags
        outer: for (config_h_flags) |flag| {
            // If a user already specified the flag, skip it
            config_iter.reset();
            while (config_iter.next()) |config_flag| {
                // For a user-specified flag to match, it must share the same prefix and have the
                // same length or be followed by an equals sign
                if (!std.mem.startsWith(u8, config_flag, flag)) continue;
                if (config_flag.len == flag.len or config_flag[flag.len] == '=') continue :outer;
            }

            // Otherwise, append default value from config.h to compile flags
            try raylib_flags_arr.append(flag);
        }
    }

    const raylib = if (options.shared)
        b.addSharedLibrary(.{
            .name = "raylib",
            .target = target,
            .optimize = optimize,
        })
    else
        b.addStaticLibrary(.{
            .name = "raylib",
            .target = target,
            .optimize = optimize,
        });
    raylib.linkLibC();

    // No GLFW required on PLATFORM_DRM
    if (options.platform != .drm) {
        raylib.addIncludePath(b.path("src/external/glfw/include"));
    }

    var c_source_files = try std.ArrayList([]const u8).initCapacity(b.allocator, 2);
    c_source_files.appendSliceAssumeCapacity(&.{ "third_party/raylib/src/rcore.c", "third_party/raylib/src/utils.c" });

    if (options.raudio) {
        try c_source_files.append("third_party/raylib/src/raudio.c");
    }
    if (options.rmodels) {
        try c_source_files.append("third_party/raylib/src/rmodels.c");
    }
    if (options.rshapes) {
        try c_source_files.append("third_party/raylib/src/rshapes.c");
    }
    if (options.rtext) {
        try c_source_files.append("third_party/raylib/src/rtext.c");
    }
    if (options.rtextures) {
        try c_source_files.append("third_party/raylib/src/rtextures.c");
    }

    if (options.opengl_version != .auto) {
        //                                                                was null
        raylib.root_module.addCMacro(options.opengl_version.toCMacroStr(), "");
    }

    switch (target.result.os.tag) {
        .windows => {
            try c_source_files.append("third_party/raylib/src/rglfw.c");
            raylib.linkSystemLibrary("winmm");
            raylib.linkSystemLibrary("gdi32");
            raylib.linkSystemLibrary("opengl32");

            setDesktopPlatform(raylib, options.platform);
        },
        .linux => {
            if (options.platform != .drm) {
                try c_source_files.append("third_party/raylib/src/rglfw.c");

                if (options.linux_display_backend == .X11 or options.linux_display_backend == .Both) {
                    raylib.root_module.addCMacro("_GLFW_X11", "");

                    raylib.linkSystemLibrary2("GLX", dynLinkOpts);
                    raylib.linkSystemLibrary2("X11", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xcursor", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xext", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xfixes", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xi", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xinerama", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xrandr", dynLinkOpts);
                    raylib.linkSystemLibrary2("Xrender", dynLinkOpts);
                }

                if (options.linux_display_backend == .Wayland or options.linux_display_backend == .Both) {
                    _ = b.findProgram(&.{"wayland-scanner"}, &.{}) catch {
                        std.log.err(
                            \\ `wayland-scanner` may not be installed on the system.
                            \\ You can switch to X11 in your `build.zig` by changing `Options.linux_display_backend`
                        , .{});
                        @panic("`wayland-scanner` not found");
                    };

                    std.log.info("BUILD {} ENVS", .{options.linux_display_backend});

                    raylib.root_module.addCMacro("_GLFW_WAYLAND", "");
                    raylib.linkSystemLibrary("EGL");
                    raylib.linkSystemLibrary("wayland-client");
                    raylib.linkSystemLibrary("xkbcommon");

                    const waylandDir = "third_party/raylib/src/external/glfw/deps/wayland";

                    waylandGenerate(b, raylib, "wayland.xml", "wayland-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "xdg-shell.xml", "xdg-shell-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "xdg-decoration-unstable-v1.xml", "xdg-decoration-unstable-v1-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "viewporter.xml", "viewporter-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "relative-pointer-unstable-v1.xml", "relative-pointer-unstable-v1-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "pointer-constraints-unstable-v1.xml", "pointer-constraints-unstable-v1-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "fractional-scale-v1.xml", "fractional-scale-v1-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "xdg-activation-v1.xml", "xdg-activation-v1-client-protocol", waylandDir);
                    waylandGenerate(b, raylib, "idle-inhibit-unstable-v1.xml", "idle-inhibit-unstable-v1-client-protocol", waylandDir);

                    raylib.addIncludePath(b.path("third_party/raylib/src/external/glfw/include/"));
                }

                setDesktopPlatform(raylib, options.platform);
            } else {
                if (options.opengl_version == .auto) {
                    raylib.linkSystemLibrary("GLESv2");
                    raylib.root_module.addCMacro("GRAPHICS_API_OPENGL_ES2", "");
                }

                raylib.linkSystemLibrary("EGL");
                raylib.linkSystemLibrary("gbm");
                raylib.linkSystemLibrary2("libdrm", .{ .use_pkg_config = .force });

                raylib.root_module.addCMacro("PLATFORM_DRM", "");
                raylib.root_module.addCMacro("EGL_NO_X11", "");
                raylib.root_module.addCMacro("DEFAULT_BATCH_BUFFER_ELEMENT", "2048");
            }
        },
        .freebsd, .openbsd, .netbsd, .dragonfly => {
            try c_source_files.append("rglfw.c");
            raylib.linkSystemLibrary("GL");
            raylib.linkSystemLibrary("rt");
            raylib.linkSystemLibrary("dl");
            raylib.linkSystemLibrary("m");
            raylib.linkSystemLibrary("X11");
            raylib.linkSystemLibrary("Xrandr");
            raylib.linkSystemLibrary("Xinerama");
            raylib.linkSystemLibrary("Xi");
            raylib.linkSystemLibrary("Xxf86vm");
            raylib.linkSystemLibrary("Xcursor");

            setDesktopPlatform(raylib, options.platform);
        },
        .macos => {
            // Include xcode_frameworks for cross compilation
            if (b.lazyDependency("xcode_frameworks", .{})) |dep| {
                raylib.addSystemFrameworkPath(dep.path("Frameworks"));
                raylib.addSystemIncludePath(dep.path("include"));
                raylib.addLibraryPath(dep.path("lib"));
            }

            // On macos rglfw.c include Objective-C files.
            try raylib_flags_arr.append("-ObjC");
            raylib.root_module.addCSourceFile(.{
                .file = b.path("src/rglfw.c"),
                .flags = raylib_flags_arr.items,
            });
            _ = raylib_flags_arr.pop();
            raylib.linkFramework("Foundation");
            raylib.linkFramework("CoreServices");
            raylib.linkFramework("CoreGraphics");
            raylib.linkFramework("AppKit");
            raylib.linkFramework("IOKit");

            setDesktopPlatform(raylib, options.platform);
        },
        .emscripten => {
            // Include emscripten for cross compilation
            if (b.lazyDependency("emsdk", .{})) |dep| {
                if (try emSdkSetupStep(b, dep)) |emSdkStep| {
                    raylib.step.dependOn(&emSdkStep.step);
                }

                raylib.addIncludePath(dep.path("upstream/emscripten/cache/sysroot/include"));
            }

            raylib.root_module.addCMacro("PLATFORM_WEB", "");
            if (options.opengl_version == .auto) {
                raylib.root_module.addCMacro("GRAPHICS_API_OPENGL_ES2", "");
            }
        },
        else => {
            @panic("Unsupported OS");
        },
    }

    raylib.root_module.addCSourceFiles(.{
        .files = c_source_files.items,
        .flags = raylib_flags_arr.items,
    });

    return raylib;
}

pub fn addRaygui(
    b: *std.Build,
    raylib: *std.Build.Step.Compile,
    //raygui_dep: *std.Build.Dependency,
    raygui_dep: *std.Build,
    ) void {
    var gen_step = b.addWriteFiles();
    raylib.step.dependOn(&gen_step.step);

    const raygui_c_path = gen_step.add("raygui.c", "#define RAYGUI_IMPLEMENTATION\n#include \"raygui.h\"\n");
    //const basePath = "./third_party/raylib/";

    raylib.addCSourceFile(.{ .file = raygui_c_path });
    raylib.addIncludePath(raygui_dep.path("src"));

    raylib.installHeader(raygui_dep.path("src/raygui.h"), "raygui.h");
}

pub const Options = struct {
    raudio: bool = true,
    rmodels: bool = true,
    rshapes: bool = true,
    rtext: bool = true,
    rtextures: bool = true,
    platform: PlatformBackend = .glfw,
    shared: bool = false,
    linux_display_backend: LinuxDisplayBackend = .Both,
    opengl_version: OpenglVersion = .auto,
    /// config should be a list of space-separated cflags, eg, "-DSUPPORT_CUSTOM_FRAME_CONTROL"
    config: []const u8 = &.{},

    const defaults = Options{};

    pub fn getOptions(b: *std.Build) Options {
        return .{
            .platform = b.option(PlatformBackend, "platform", "Choose the platform backedn for desktop target") orelse defaults.platform,
            .raudio = b.option(bool, "raudio", "Compile with audio support") orelse defaults.raudio,
            .rmodels = b.option(bool, "rmodels", "Compile with models support") orelse defaults.rmodels,
            .rtext = b.option(bool, "rtext", "Compile with text support") orelse defaults.rtext,
            .rtextures = b.option(bool, "rtextures", "Compile with textures support") orelse defaults.rtextures,
            .rshapes = b.option(bool, "rshapes", "Compile with shapes support") orelse defaults.rshapes,
            .shared = b.option(bool, "shared", "Compile as shared library") orelse defaults.shared,
            .linux_display_backend = b.option(LinuxDisplayBackend, "linux_display_backend", "Linux display backend to use") orelse defaults.linux_display_backend,
            .opengl_version = b.option(OpenglVersion, "opengl_version", "OpenGL version to use") orelse defaults.opengl_version,
            .config = b.option([]const u8, "config", "Compile with custom define macros overriding config.h") orelse &.{},
        };
    }
};

pub const OpenglVersion = enum {
    auto,
    gl_1_1,
    gl_2_1,
    gl_3_3,
    gl_4_3,
    gles_2,
    gles_3,

    pub fn toCMacroStr(self: @This()) []const u8 {
        switch (self) {
            .auto => @panic("OpenglVersion.auto cannot be turned into a C macro string"),
            .gl_1_1 => return "GRAPHICS_API_OPENGL_11",
            .gl_2_1 => return "GRAPHICS_API_OPENGL_21",
            .gl_3_3 => return "GRAPHICS_API_OPENGL_33",
            .gl_4_3 => return "GRAPHICS_API_OPENGL_43",
            .gles_2 => return "GRAPHICS_API_OPENGL_ES2",
            .gles_3 => return "GRAPHICS_API_OPENGL_ES3",
        }
    }
};

pub const LinuxDisplayBackend = enum {
    X11,
    Wayland,
    Both,
};

pub const PlatformBackend = enum {
    glfw,
    rgfw,
    sdl,
    drm,
};

pub fn build(b: *std.Build) !void {
    // Standard target options allows the person running `zig build` to choose
    // what target to build for. Here we do not override the defaults, which
    // means any target is allowed, and the default is native. Other options
    // for restricting supported target set are available.
    const target = b.standardTargetOptions(.{});
    // Standard optimization options allow the person running `zig build` to select
    // between Debug, ReleaseSafe, ReleaseFast, and ReleaseSmall. Here we do not
    // set a preferred release mode, allowing the user to decide how to optimize.
    const optimize = b.standardOptimizeOption(.{});

    const lib = try compileRaylib(b, target, optimize, Options.getOptions(b));

    lib.installHeader(b.path("src/raylib.h"), "raylib.h");
    lib.installHeader(b.path("src/raymath.h"), "raymath.h");
    lib.installHeader(b.path("src/rlgl.h"), "rlgl.h");

    b.installArtifact(lib);
}

fn waylandGenerate(
    b: *std.Build,
    raylib: *std.Build.Step.Compile,
    comptime protocol: []const u8,
    comptime basename: []const u8,
    comptime waylandDir: []const u8,
) void {
    const protocolDir = b.pathJoin(&.{ waylandDir, protocol });
    const clientHeader = basename ++ ".h";
    const privateCode = basename ++ "-code.h";

    const client_step = b.addSystemCommand(&.{ "wayland-scanner", "client-header" });
    client_step.addFileArg(b.path(protocolDir));
    raylib.addIncludePath(client_step.addOutputFileArg(clientHeader).dirname());

    const private_step = b.addSystemCommand(&.{ "wayland-scanner", "private-code" });
    private_step.addFileArg(b.path(protocolDir));

    raylib.addIncludePath(private_step.addOutputFileArg(privateCode).dirname());

    raylib.step.dependOn(&client_step.step);
    raylib.step.dependOn(&private_step.step);
```
