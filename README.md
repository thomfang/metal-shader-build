# metal-shader-build

Compile Metal (MSL) shaders for the **[Scripting](https://scripting.fun)** app in CI — no Mac needed
on the device side.

SwiftUI shader effects in Scripting (`colorEffect`, `layerEffect`, `distortionEffect`) can only be driven
by a `ShaderLibrary` built from a **precompiled `.metallib` on disk**. iOS *can* compile MSL at runtime
(`MTLDevice.makeLibrary(source:)`), but there is no public API to hand that `MTLLibrary` to SwiftUI and
none to serialize it back into a `.metallib`. So the library has to be produced by Apple's `metal`
toolchain — which this repo runs for you on a macOS runner.

## Layout

```
shaders/     *.metal sources — one library per file
manifests/   Scripting shader manifests (argument signatures), one per library
build/       compiled *.metallib + index.json (auto-committed by CI, do not edit)
```

## Use it

1. Add `shaders/Foo.metal` and push.
2. CI compiles it to `build/Foo.iphoneos.metallib` (plus `Foo.iphonesimulator.metallib` when the
   simulator target builds). Failures show up as normal workflow errors.
3. Download the artifact straight from `main`:

   ```
   https://raw.githubusercontent.com/thomfang/metal-shader-build/main/build/Foo.iphoneos.metallib
   ```

   `build/index.json` lists size and sha256 of everything so you can verify what you got.
4. Put the `.metallib` next to your manifest inside the script project (Scripting resolves manifest
   `libraries` entries relative to the manifest's folder), then:

   ```tsx
   const lib = await ShaderLibrary.load(Path.join(Script.directory, "shaders", "effects.json"))
   const shader = lib.shader("foo", [
     Shader.Argument.boundingRect,
     Shader.Argument.float(0.5),
   ])
   ```

## Manifest

```json
{
  "version": 1,
  "libraries": {
    "iphoneos": "Foo.iphoneos.metallib",
    "iphonesimulator": "Foo.iphonesimulator.metallib"
  },
  "functions": [
    {
      "name": "foo",
      "effect": "colorEffect",
      "arguments": ["boundingRect", "float", "float4"]
    }
  ]
}
```

- Argument types: `float`, `float2`, `float3`, `float4`, `color`, `boundingRect`, `image`.
- `position` and the source colour are supplied by SwiftUI — do **not** list them.
- The list must match the Metal signature one to one, in order. A mismatch fails at
  `ShaderLibrary.load` with `invalidSignature`.
- Limits: manifest ≤ 32 KiB, metallib ≤ 8 MiB, ≤ 32 functions, ≤ 16 bound arguments per function.
- `effect` is `colorEffect` for these; `layerEffect` / `distortionEffect` need `maxSampleOffset` at the
  call site and one extra `image` argument.

## Local compile

```sh
xcrun -sdk iphoneos metal -target air64-apple-ios17.0 -c shaders/Foo.metal -o Foo.air
xcrun -sdk iphoneos metal -target air64-apple-ios17.0 Foo.air -o Foo.iphoneos.metallib
```

## Runtime requirements

- iOS 18 or later, script running in the foreground of the Scripting app.
- `ShaderLibrary.getCapabilities().projectLibraries === true`. When it is `false`, `ShaderLibrary.load`
  rejects with `unsupported` **before touching the file** — that gate lives in the host app, nothing on
  the script side can work around it.

## Included example

`shaders/BorderBeam.metal` + `manifests/BorderBeam.json` — a `colorEffect` port of
[ismnoiet/border-beam-ios](https://github.com/ismnoiet/border-beam-ios) (MIT). An animated comet that
orbits a rounded rectangle / capsule / circle: signed-distance field, perimeter parameterisation by arc
length, 24 Gaussian blobs for the glow, one GPU pass. Palette arrives as five `float4` RGBA values
because the manifest format has no float array.
