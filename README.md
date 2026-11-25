![mpv logo](https://raw.githubusercontent.com/mpv-player/mpv.io/master/source/images/mpv-logo-128.png)

# mpv


* [About this fork](#about-this-fork)
* [External links](#external-links)
* [Overview](#overview)
* [System requirements](#system-requirements)
* [Downloads](#downloads)
* [Changelog](#changelog)
* [Compilation](#compilation)
* [Release cycle](#release-cycle)
* [Bug reports](#bug-reports)
* [Contributing](#contributing)
* [License](#license)
* [Contact](#contact)


## About this fork


This is a personal fork of [mpv](https://github.com/mpv-player/mpv) for watching
video with **animated ASS/SSA subtitles that move at the refresh rate of the
display**, such as scrolling comments (danmaku), moving signs and karaoke. It is
not intended to be merged upstream; it is kept in sync with upstream `master`
and these changes are rebased/re-applied on top.

### What it adds

**1. `--sub-animation-fps`: subtitle animations on every refresh of the display**

Upstream mpv renders subtitles once per video frame, for the time of that frame.
An animated subtitle (`\move`, `\t`, `\fad`, karaoke, ...) therefore advances at
the frame rate of the video, which looks choppy with 24 or 30 fps video on a
display that refreshes faster.

```text
--sub-animation-fps=no        # disabled (default; upstream behavior)
--sub-animation-fps=display   # animate on every refresh of the display
--sub-animation-fps=30        # animate at most this many times per second
```

It requires a display synced `--video-sync` mode (e.g. `display-resample`), the
same as `--interpolation`. It applies to ASS/SSA subtitles drawn by `--vo=gpu`,
`--vo=gpu-next` and the libmpv render API (cocoa-cb on macOS), and has no effect
with `--blend-subtitles`. See `--sub-animation-fps` in the manual.

**2. Playback speed changes without a hitch (display synced modes)**

When the playback speed changes, the audio that is already buffered by the audio
output keeps playing at the old speed for a moment (0.2 s by default). Upstream
changes the speed of the video at once and then repairs the resulting A/V
difference in one go: the video stands still after speeding up, and skips ahead
after slowing down. Here the video changes its speed at the moment the audio of
the new speed is heard, so audio and video stay in sync through the change and
nothing has to be repaired.

### How it works

Subtitle animations:

* In the display synced modes every video frame is put on the vsyncs around its
  time, and the player knows where between the video frames each vsync is: it is
  what `--interpolation` mixes the frames by. The VO passes that position along
  with every draw of a frame (`vo_frame.sub_anim_offset`, `video/out/vo.c`).
  No clock, timer or extra redraw is involved, so the animation advances by
  exactly the same amount on every vsync, and follows the video through speed
  changes, dropped or repeated vsyncs and filters that change the frame timing
  (inverse telecine, deinterlacing).
* The renderers hand the time to the OSD before they draw it
  (`osd_set_sub_anim_pts()`; `video/out/gpu_next/video.c`,
  `video/out/gpu/video.c`), from where it reaches the subtitle decoder
  (`sub/osd.c`, `sub/dec_sub.c`).
* Which subtitle events are shown is still decided by the time of the video
  frame alone, only the animation within an event uses the finer time. libass
  takes a single time for both, so the frame is rendered at the animation time
  and the few events the two times disagree about have their start moved for
  the call (`render_frame()` in `sub/sd_ass.c`). An event therefore appears and
  disappears on exactly the video frame it was timed to, as upstream.

Playback speed changes:

* Every audio frame knows the speed it was made at. Where that speed jumps in
  the audio written to the audio output is remembered until it has played
  (`player/audio.c`). With that, the position of the audio that is heard is
  known through a speed change (`playing_audio_pts()`), and so is the speed it
  plays at (`playing_speed_after()`).
* Display sync shows each video frame at the speed of the audio heard with it
  (`handle_display_sync_frame()` in `player/video.c`).

### Files changed vs upstream

`options/options.{c,h}`, `video/out/vo.{c,h}`, `video/out/gpu_next/video.c`,
`video/out/gpu/video.c`, `sub/osd.{c,h}`, `sub/osd_state.h`,
`sub/dec_sub.{c,h}`, `sub/sd.h`, `sub/sd_ass.c`, `player/audio.c`,
`player/video.c`, `player/core.h`, `DOCS/man/options.rst`, `common/meson.build`
(version suffix).

### Syncing with upstream

```sh
git remote add upstream https://github.com/mpv-player/mpv.git   # once
git fetch upstream
git rebase upstream/master      # re-apply the fork commits on top
```


## External links


* [Wiki](https://github.com/mpv-player/mpv/wiki)
* [User Scripts](https://github.com/mpv-player/mpv/wiki/User-Scripts)
* [FAQ][FAQ]
* [Manual](https://mpv.io/manual/master/)


## Overview


**mpv** is a free (as in freedom) media player for the command line. It supports
a wide variety of media file formats, audio and video codecs, and subtitle types.

There is a [FAQ][FAQ].

Releases can be found on the [release list][releases].

## System requirements

- A not too ancient Linux (usually, only the latest releases of distributions
  are actively supported), Windows 10 1607 or later, or macOS 10.15 or later.
- A somewhat capable CPU. Hardware decoding might help if the CPU is too slow to
  decode video in realtime, but must be explicitly enabled with the `--hwdec`
  option.
- A not too crappy GPU. mpv's focus is not on power-efficient playback on
  embedded or integrated GPUs (for example, hardware decoding is not even
  enabled by default). Low power GPUs may cause issues like tearing, stutter,
  etc. On such GPUs, it's recommended to use `--profile=fast` for smooth playback.
  The main video output uses shaders for video rendering and scaling,
  rather than GPU fixed function hardware. On Windows, you might want to make
  sure the graphics drivers are current. In some cases, ancient fallback video
  output methods can help (such as `--vo=xv` on Linux), but this use is not
  recommended or supported.

mpv does not go out of its way to break on older hardware or old, unsupported
operating systems, but development is not done with them in mind. Keeping
compatibility with such setups is not guaranteed. If things work, consider it
a happy accident.

## Downloads


For semi-official builds and third-party packages please see
[mpv.io/installation](https://mpv.io/installation/).

## Changelog


There is no complete changelog; however, changes to the player core interface
are listed in the [interface changelog][interface-changes].

Changes to the C API are documented in the [client API changelog][api-changes].

The [release list][releases] has a summary of most of the important changes
on every release.

Changes to the default key bindings are indicated in
[restore-old-bindings.conf][restore-old-bindings].

Changes to the default OSC bindings are indicated in
[restore-osc-bindings.conf][restore-osc-bindings].

## Compilation


Compiling with full features requires development files for several
external libraries. Mpv requires [meson](https://mesonbuild.com/index.html)
to build. Meson can be obtained from your distro or PyPI.

After creating your build directory (e.g. `meson setup build`), you can view a list
of all the build options via `meson configure build`. You could also just simply
look at the `meson_options.txt` file. Logs are stored in `meson-logs` within
your build directory.

Example:

    meson setup build
    meson compile -C build
    meson install -C build

For libplacebo, meson can use a git check out as a subproject for a convenient
way to compile mpv if a sufficient libplacebo version is not easily available
in the build environment. It will be statically linked with mpv. Example:

    mkdir -p subprojects
    git clone https://code.videolan.org/videolan/libplacebo.git --depth=1 --recursive subprojects/libplacebo

Essential dependencies (incomplete list):

- gcc or clang
- X development headers (xlib, xrandr, xext, xscrnsaver, xpresent, libvdpau,
  libGL, GLX, EGL, xv, ...)
- Audio output development headers (libasound/ALSA, pulseaudio)
- FFmpeg libraries (libavutil libavcodec libavformat libswscale libavfilter
  and either libswresample or libavresample)
- libplacebo
- zlib
- iconv (normally provided by the system libc)
- libass (OSD, OSC, text subtitles)
- Lua (optional, required for the OSC pseudo-GUI and youtube-dl integration)
- libjpeg (optional, used for screenshots only)
- uchardet (optional, for subtitle charset detection)
- nvdec and vaapi libraries for hardware decoding on Linux (optional)

Libass dependencies (when building libass):

- gcc or clang, nasm on x86 and x86_64
- fribidi, freetype, fontconfig development headers (for libass)
- harfbuzz (required for correct rendering of combining characters, particularly
  for correct rendering of non-English text on macOS, and Arabic/Indic scripts on
  any platform)

FFmpeg dependencies (when building FFmpeg):

- gcc or clang, nasm on x86 and x86_64
- OpenSSL or GnuTLS (have to be explicitly enabled when compiling FFmpeg)
- libx264/libmp3lame/libfdk-aac if you want to use encoding (have to be
  explicitly enabled when compiling FFmpeg)
- For native DASH playback, FFmpeg needs to be built with --enable-libxml2
  (although there are security implications, and DASH support has lots of bugs).
- AV1 decoding support requires dav1d.
- For good nvidia support on Linux, make sure nv-codec-headers is installed
  and can be found by configure.

Most of the above libraries are available in suitable versions on normal
Linux distributions. For ease of compiling the latest git master of everything,
you may wish to use the separately available build wrapper ([mpv-build][mpv-build])
which first compiles FFmpeg libraries and libass, and then compiles the player
statically linked against those.

If you want to build a Windows binary, see [Windows compilation][windows_compilation].


## Release cycle

Once or twice a year, a release is cut off from the current development state
and is assigned a 0.X.0 version number. No further maintenance is done, except
in the event of security issues.

The goal of releases is to make Linux distributions happy. Linux distributions
are also expected to apply their own patches in case of bugs.

Releases other than the latest release are unsupported and unmaintained.

See the [release policy document][release-policy] for more information.

## Bug reports


Please use the [issue tracker][issue-tracker] provided by GitHub to send us bug
reports or feature requests. Follow the template's instructions or the issue
will likely be ignored or closed as invalid.

Questions can be asked in the [discussions][discussions] or on IRC (see
[Contact](#Contact) below).

## Contributing


Please read [contribute.md][contribute.md].

For small changes you can just send us pull requests through GitHub. For bigger
changes come and talk to us on IRC before you start working on them. It will
make code review easier for both parties later on.

You can check [the wiki](https://github.com/mpv-player/mpv/wiki/Stuff-to-do)
or the [issue tracker](https://github.com/mpv-player/mpv/issues?q=is%3Aopen+is%3Aissue+label%3Ameta%3Afeature-request)
for ideas on what you could contribute with.

## License

GPLv2 "or later" by default, LGPLv2.1 "or later" with `-Dgpl=false`.
See [details.](https://github.com/mpv-player/mpv/blob/master/Copyright)

## History

This software is based on the MPlayer project. Before mpv existed as a project,
the code base was briefly developed under the mplayer2 project. For details,
see the [FAQ][FAQ].

## Contact


Most activity happens on the IRC channel and the GitHub issue tracker.

- **GitHub issue tracker**: [issue tracker][issue-tracker] (report bugs here)
- **Discussions**: [discussions][discussions]
- **User IRC Channel**: `#mpv` on `irc.libera.chat`
- **Developer IRC Channel**: `#mpv-devel` on `irc.libera.chat`

[FAQ]: https://github.com/mpv-player/mpv/wiki/FAQ
[releases]: https://github.com/mpv-player/mpv/releases
[mpv-build]: https://github.com/mpv-player/mpv-build
[issue-tracker]:  https://github.com/mpv-player/mpv/issues
[discussions]: https://github.com/mpv-player/mpv/discussions
[release-policy]: https://github.com/mpv-player/mpv/blob/master/DOCS/release-policy.md
[windows_compilation]: https://github.com/mpv-player/mpv/blob/master/DOCS/compile-windows.md
[interface-changes]: https://github.com/mpv-player/mpv/blob/master/DOCS/interface-changes.rst
[api-changes]: https://github.com/mpv-player/mpv/blob/master/DOCS/client-api-changes.rst
[restore-old-bindings]: https://github.com/mpv-player/mpv/blob/master/etc/restore-old-bindings.conf
[restore-osc-bindings]: https://github.com/mpv-player/mpv/blob/master/etc/restore-osc-bindings.conf
[contribute.md]: https://github.com/mpv-player/mpv/blob/master/DOCS/contribute.md
