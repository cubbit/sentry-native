## Crashpad Architecture

Crashpad essentially has two parts, `crashpad_handler`, which is an out-of
process executable, and `crashpad_client`, which is a library that we link
against.

## Building

Our goal is to keep our workflows and build infrastructure as simple as
possible.

Upstream crashpad is built around `depot_tools` that fetches and links a number
of git repos, and also fetches the build tools `gn` and `ninja` for the users
platform.
We don’t want users of `sentry-native` to have to deal with all that.
There have been a number of attempts to build crashpad as a CMake project:

- https://github.com/unidentifieddeveloper/crashpad this apparently only works on windows
- https://developer.blender.org/D3576 adapts the below CMake files
- https://github.com/qedsoftware/crashpad

From these sources we can see that in order to build crashpad, we need 3
projects:

- `crashpad` itself
- `mini_chromium`
- `zlib`

We can consume those via git submodules, which avoids having to use
`depot_tools`, and should also be more stable, as we can pin specific known good
versions.

`zlib` is itself CMake project already. For `crashpad` and `mini_chromium`, we
created custom CMake files based on the inspiration above.

Right now, it is mostly a concatination of the CMake files taken from the
blender PR, but in the future, it might be a better idea to translate the
`BUILD.gn` files into CMake files instead.
