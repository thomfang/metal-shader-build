# border-beam-shader

Metal `colorEffect` shader used by the **Border Beam** script for the Scripting app.
Ported from [ismnoiet/border-beam-ios](https://github.com/ismnoiet/border-beam-ios) (MIT).

CI compiles `BorderBeam.metal` into `BorderBeam.iphoneos.metallib` on a macOS runner and
publishes it as a release asset (`shader-build`).

`effects.json` is the Scripting shader-library manifest describing the bound arguments.
