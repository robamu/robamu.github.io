+++
disableShare = true
comments = false
hideMeta = true
+++

## sat-rs

I am currently working on the
[sat-rs library](https://absatsw.irs.uni-stuttgart.de/projects/sat-rs) as part of my PhD thesis.

In large parts, this will be a Rust port of the [FSFW](#flight-softwre-framework-fsfw-contributor), but the goal
will also be to leverage the incredible language features and the ecosystem of Rust to alleviate
the weak spots of the FSFW.

## SOURCE satellite project

I worked on the on-board software of thesis[SOURCE student satellite project](https://git.ksat-stuttgart.de/source/sourceobsw)
for a few years and still regularly contribute to the
project. 

## EIVE satellite project

I worked and I am still working on the
[EIVE satellite project](https://www.irs.uni-stuttgart.de/en/research/satellitetechnology-and-instruments/smallsatelliteprogram/EIVE/)
and the corresponding [EIVE On-Board Software](https://egit.irs.uni-stuttgart.de/eive/eive-obsw/).
I wrote large parts of the software which are now running on the EIVE satellite launched in June 2023.

## Flight Software Framework (FSFW) Contributor

I contributed a lot to the [FSFW](https://absatsw.irs.uni-stuttgart.de/projects/fsfw.html), a
framework which is used for both the [SOURCE](https://www.ksat-stuttgart.de/en/our-projects/source/)
and [EIVE](https://www.irs.uni-stuttgart.de/en/research/satellitetechnology-and-instruments/smallsatelliteprogram/EIVE/) project.
A lot of the architecture and design of the FSFW will also be transferred to the `sat-rs` Rust
framework which is part of my PhD thesis.

## Spacepackets and tmtccmd libraries

I wrote the [Python spacepackets](https://github.com/us-irs/spacepackets-py) and the
[Rust spacepackets](https://github.com/us-irs/spacepackets-rs) library, which offer a convenient
API for various CCSDS and ECSS packet
standards.

I have also written a Python program which allows satellite software developers to quickly
test TMTC handling in their software. The program was refactored into a format which makes it
easier to adapt it to other missions.
