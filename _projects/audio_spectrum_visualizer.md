---
layout: page
title: Audio Spectrum Visualizer
description: An FPGA system that streams audio off an SD card and draws it on an HDMI display in real time (ECE 385). The real work was getting audio playback, SD streaming, video, and USB input to share one processor without the sound falling apart.
img: assets/img/audio_hdmi_output.png
importance: 4
category: work
related_publications: false
---

ECE 385 is the digital systems lab, and the final project was open-ended, so we built an FPGA system that reads an audio file off an SD card, plays it back, and draws a live visualization of it on an HDMI monitor. The design pairs a MicroBlaze soft processor running C with custom SystemVerilog hardware, on an AMD Vivado and Xilinx Vitis toolchain. It was a team of two. I owned most of the system integration, the audio architecture, the buffering, and the visualization framework, which is also where most of the debugging ended up.

<div class="row justify-content-sm-center">
    <div class="col-sm-9 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/audio_hdmi_output.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    HDMI output during playback. The status line across the top is driven from C (mode, play state, volume), the bars are drawn from the audio currently coming out of the speaker, and the keys at the bottom switch modes and volume.
</div>

## The actual problem was concurrency

Basic audio playback was not the hard part. I had it working early, in isolation, and it sounded fine. The trouble started when the rest of the system showed up. Once HDMI frame updates and USB keyboard polling were running next to it, the audio that had been clean on its own turned choppy and drifted in speed, with clicks you could hear plainly. Audio is unforgiving about timing, because a slip of even a few sample periods is audible, and the software-driven playback loop was getting pushed off its schedule every time the display refreshed or a key got polled. The project quietly turned into a different one: making four things share a single processor and a single block of memory without any of them wrecking the sound.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/audio_block_diagram.png" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Top-level block diagram. The SD loader writes audio into BRAM through one port while the MicroBlaze reads the same BRAM through the other via the AXI BRAM controller, with the HDMI, PWM audio, and USB SPI blocks hanging off the processor bus.
</div>

## Streaming 3.47 MB through 8 KB of BRAM

The audio file is 3.47 MB of raw 8-bit PCM, and the on-chip BRAM set aside for buffering is 8 KB. You cannot load the file before playing it, so the SD card has to stream new audio into the buffer continuously while the playback hardware reads out of the same buffer. The first version played the opening correctly and then either froze or repeated garbage, because playback reached the end of the valid data in BRAM before the next chunk had arrived and the reader walked straight into stale samples. The fix was to run the 8 KB as a circular buffer. While one region plays, the SD streamer refills the region that was just consumed, so the same 8 KB gets reused for the entire 157-second file, a 424 to 1 ratio between file size and buffer size.

That only holds if the producer and consumer stay rate-matched. The SD card reads at roughly 80 KB/s while the CPU consumes audio at about 22 KB/s, so left alone the streamer laps the reader and overwrites samples that have not played yet, which comes out as corruption. Err the other way and you get underrun gaps. I added a delay state to the streaming state machine that waits about 22 ms between 512-byte sector reads, which pulls throughput down to around 23 KB/s, just barely ahead of consumption. The BRAM is configured true dual-port, so the SD streamer writes through Port B while the CPU reads through Port A, and neither side has to wait for the other to finish.

## Why the ring buffer is lock-free

Sharing one buffer between the SD writer and the playback reader is the standard producer-consumer arrangement, and the reflex is to put a lock on it. A lock does not survive contact with audio timing. Playback runs at 22,050 samples per second, one sample every 45.35 microseconds, and if the playback side ever stalls waiting to acquire a lock, the output rate wobbles and the wobble is audible. I made the ring buffer lock-free instead. The producer only ever writes the write pointer, the consumer only ever writes the read pointer, and neither side touches the other's pointer, so there is nothing to block on. Playback then runs at a constant rate regardless of what the SD streaming logic is doing at any given moment.

## Moving the sample clock into hardware

The lock-free buffer settled who owns which pointer, but the playback rate itself was still being generated in software, and that was the thing HDMI and USB activity kept disturbing. What finally made the system stable was to stop producing samples in the main loop and give that job to a dedicated AXI timer interrupt firing every 45.35 microseconds. The ISR pushes exactly one sample to the PWM output on each tick, so sample timing no longer depends on where the CPU happens to be in the display or keyboard code. With the sample clock living in hardware, the audio held its rate while video and input ran at the same time, which was the entire reason for the rebuild.

## The five visualization modes

The display is an HDMI text-mode output at 60 FPS, and the bars come from the same audio stream that is playing, kept in step through the lock-free ring buffer. There are five modes, switched from the keyboard. Audio mode is the one tied to the music: it groups recent samples into columns and sets each bar from the average deviation from center (128, since the samples are unsigned 8-bit), so the heights actually follow what you are hearing. The other four are generated patterns rather than analysis of the signal. Pulse mode expands a sine-enveloped wave outward from the center of the screen. Wave mode sums sines of several frequencies into a waveform that slides sideways. Random mode gives each column a fresh random height every frame, smoothed so it does not strobe. Symmetric mode sets height from distance to the center column so the pattern mirrors left and right.

One honest limitation: despite the name, Audio mode reacts to amplitude, not frequency. There is no FFT in the design, so it shows how loud the music is across time, not where the energy sits in the spectrum. A real frequency display would be the obvious next version, and fitting an FFT into the remaining fabric and timing budget is most of why we did not attempt it the first time around.

<hr>

[**View the full writeup (PDF)**]({{ '/assets/pdf/audio_spectrum_visualizer.pdf' | relative_url }}), with the full block diagrams, the Vivado schematics, the streaming state machine, and the file specifications.
