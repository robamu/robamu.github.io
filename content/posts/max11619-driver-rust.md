---
title: "Rust driver for the 10-bit MAX11619 ADCs"
#description: <descriptive text here>
date: 2021-12-16T22:51:15+01:00
draft: false
toc: false
# Unfortunately, the preview image looks too large in my opinion..
# Need to adapt beautifulhugo for that
image: ""
# image: "/img/max11619-driver/setup-rotated.jpg"
tags: ["rust", "driver", "adc", "max11619"]
categories: ["rust", "embedded", "coding"]
---

In my [last blog post](https://robamu.github.io/post/rust-ecosystem/), I described how I set up a
small Rust ecosystem for the Vorago REB1 development board. This development board also
has a MAX11619 10-bit ADC device by Maxim devices. I thought this was a good opportunity
to develop my first device driver crate because there isn't one for this device yet.

The REB1 development board also has a 2K potentiometer connected directly to a channel of the ADC
which makes testing convenient.

<center>
{{< figure
	src="/img/max11619-driver/setup-rotated.jpg"
	alt="Setup with a Digilent Oscilloscope"
	caption="Setup with a Digilent Oscilloscope"
>}}
</center>

<center>
{{< figure
	src="/img/max11619-driver/pot-schem.png"
	alt="Board schematic for ADC and Potentiometer"
	caption="Board schematic for ADC and Potentiometer"
>}}
</center>

From the schematic, I saw that the CNVST pin is not connected so I could not really test
any of the ADC functions using that pin, but everything else should work.

I used the knowledge I gained from programming type-safe APIs for the VA108xx HAL and BSP to
code a type-safe API for the MAX116xx device. The ADC device has different modes and configuration
options to specify the clock source and the voltage reference source. Some examples:

1. Use the external SPI clock for acquisiton and conversion and use an external voltage reference
2. Use the SPI interface to start the acquisiton but use the internal oscillator for the conversions.
   The End-Of-Conversion (EOC) pin is used to check whether the conversion is complete.
   Use an internal voltage reference which is off after acquisition, so a 65 microseconds wake-up
   delay becomes necessary.

There are a lot other configurations, and the Rust typesystem prevents using a wrong API for a
given configuration. [`Max116xx10Bit`](https://docs.rs/max116xx-10bit/latest/max116xx_10bit/struct.Max116xx10Bit.html)
is now initially created as an externally clocked device with an external voltage reference.
This is also a valid configuration to read the ADC on the REB1 board,
as the reference is pin is tied to the system voltage.

If another configuration is desired, the device struct needs to be converted into a different
configuration using the `into_*()` API common to Rust. The [driver docs](https://docs.rs/max116xx-10bit/latest/max116xx_10bit/)
specify some of these functions. To achieve the second configuration shown above, one would
use the [`into_int_clkd_int_timed_through_ser_if_with_wakeup`
](https://docs.rs/max116xx-10bit/latest/max116xx_10bit/struct.Max116xx10Bit.html#method.into_int_clkd_int_timed_through_ser_if_with_wakeup)
function.

There are also some helper constructor functions for each ADC family derivative. Some of these
derivatives have different channel numbers, and the constructors set the highest channel number
correct automatically, which can prevent some errors like specifying an invalid channel number
as well.

This is the example function to use the first shown configuration and using different API
options to read the channels. The full example can be found
[here](https://egit.irs.uni-stuttgart.de/rust/vorago-reb1/src/branch/main/examples/max11619-adc.rs).
Provided that a `JLinkGDBServer` is running, flashing the software can be done with this simple
command:

```sh
cargo run --example max11619-adc --release
```

I also used `release` here because I checked the correct timing and an optimized build is best for
that.

```rs
/// Use the SPI clock as the conversion clock
fn adc_example_externally_clocked(spi: SpiBase<SPIB>, mut delay: Delay) -> ! {
    let mut adc = max11619_externally_clocked_no_wakeup(spi)
        .expect("Creating externally clocked MAX11619 device failed");
    if READ_MODE == ReadMode::AverageN {
        adc.averaging(
            AveragingConversions::FourConversions,
            AveragingResults::FourResults,
        )
        .expect("Error setting up averaging register");
    }
    let mut cmd_buf: [u8; 32] = [0; 32];
    let mut counter = 0;
    loop {
        rprintln!("-- Measurement {} --", counter);

        match READ_MODE {
            ReadMode::Single => {
                rprintln!("Reading single potentiometer channel");
                let pot_val = adc
                    .read_single_channel(&mut cmd_buf, POTENTIOMETER_CHANNEL)
                    .expect("Creating externally clocked MAX11619 ADC failed");
                rprintln!("Single channel read:");
                rprintln!("\tPotentiometer value: {}", pot_val);
            }
            ReadMode::Multiple => {
                let mut res_buf: [u16; 4] = [0; 4];
                adc.read_multiple_channels_0_to_n(
                    &mut cmd_buf,
                    &mut res_buf.iter_mut(),
                    POTENTIOMETER_CHANNEL,
                )
                .expect("Multi-Channel read failed");
                print_res_buf(&res_buf);
            }
            ReadMode::MultipleNToHighest => {
                let mut res_buf: [u16; 2] = [0; 2];
                adc.read_multiple_channels_n_to_highest(
                    &mut cmd_buf,
                    &mut res_buf.iter_mut(),
                    AN2_CHANNEL,
                )
                .expect("Multi-Channel read failed");
                rprintln!("Multi channel read from 2 to 3:");
                rprintln!("\tAN2 value: {}", res_buf[0]);
                rprintln!("\tAN3 / Potentiometer value: {}", res_buf[1]);
            }
            ReadMode::AverageN => {
                rprintln!("Scanning and averaging not possible for externally clocked mode");
            }
        }
        counter += 1;
        delay.delay_ms(500);
    }
}
```

There is also an example mode which uses the averaging functionality of the ADC. This can be used
for something like filtering a noisy signal.

The `SpiBase<SPIB>` struct is VA10820 specific, but any SPI instance which implements the
[`embedded-hal`](https://docs.rs/embedded-hal/latest/embedded_hal/) can be used to
instantiate an ADC struct.

<center>
{{< figure
	src="/gif/max11619-driver/vor-pot.gif"
	alt="Operating the potentiometer"
>}}
</center>

<center>
{{< figure
	src="/gif/max11619-driver/vor-rtt.gif"
	alt="ADC channel output"
	caption="ADC channel output, AN1 tied to 3.3V"
>}}
</center>

I also checked the SPI signals to make fully sure that my the HAL SPI driver was correctly
functioning concerning properties like timing. To check the signals directly, I was able to
multiplex some pins to gain access to the SPI signals. Unfortunately, this did not really work for
the MISO line, but the received values are
definitely valid: When the pontentiometer is at the lowest resistance, the full system voltage
is tied to the analog channel. For a 10-bit ADC, a value close to 2 to the power of 10 (1023)
makes sense here. The Digilent Oscilloscope also has a really neat decoder function
to analyze common peripherals.

<center>
{{< figure
	src="/img/max11619-driver/adc-single-read.png"
	alt="ADC single read signal"
	caption="ADC single readout signals"
>}}
</center>

Finally, I also checked whether the timing was correctly when using a mode with a 65 us
wake-up delay after initiating the conversion by sending one byte:

<center>
{{< figure
	src="/img/max11619-driver/adc-with-delay.png"
	alt="ADC single readout signals with wake-up delay"
	caption="ADC single readout signals with wake-up delay"
>}}
</center>

Good signals, and the RTT viewer was displaying correct ADC chanel values as well!

I really like how Rust allows library and device driver developers to write safe APIs which can
prevent a lot of errors at compile time. I think this has a lot of potential for satellite
software development, where it is common to forbid certain operations for different software modes.
Encoding something like that at compile time would make the software a lot safer.

In some of our projects, we also use the [MAX1227 12-bit ADCs
](https://www.maximintegrated.com/en/products/analog/data-converters/analog-to-digital-converters/MAX1227.html)
which have a lot of similarities to the MAX116xx 10-bit devices. I might look into writing
a device driver crate for those as well soon.
