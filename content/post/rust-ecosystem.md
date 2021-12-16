---
title: "Bringing Rust to Space - Setting up a Rust ecosystem for the VA108XX MCU family"
#description: <descriptive text here>
date: 2021-12-15T23:31:00+01:00
draft: true
toc: false
image: ""
tags: []
categories: []
---

The last few weeks I have been busy diving into the Rust ecosystem and learning the language
through practical projects. I finishing the excellent
[Rust book](https://doc.rust-lang.org/book/) and the [Rust Embedded Book](https://docs.rust-embedded.org/book/)
first and then tinkered with Rust on some STM32 MCUs. As a next step, I was looking for practical
projects to further learn the language. I've tried to combine this with the usual activities at
the research institute I work now as well.

Working at an insitute which builds small satellites gives me access to unique hardware like
the radiation hardened Vorago MCUs which are usually not used by hobbyist because of their
price tag. 3-4 digits are very common even for basic radiation hardened components.
Some of these MCUs are used in our CubeSat projects and I was
curious whether I could set up Rust for some of the MCUs, namely the VA10820 Cortex-M0 based MCU.
Vorago sells some of their chips on a Vorago development board named REB1 and I was able to get my
hands on one.

Browsing the internet a bit, I quickly found out that nobody attempted to create supporting
libraries for these MCUs yet. I thought this was an excellent opportunity to learn a lot of
aspects of Embedded Rust. The Rust book had an excellent picture of what a ecosystem for a particular
MCU family or board might look like

![rust embedded ecosystem](/img/rust-ecosystem/crates.png)

I started with the lowest necessary level to build higher level abstractions:
The Peripheral Access Crate (PAC). After a few tweaks to the vendor supplied SVD file and the
`svd2rust` tool, I was able to generate my first own PAC.

I then set up the blinky application as the classical first application for a new MCU to test
my PAC. This is what it looks like

```rs
#![no_main]
#![no_std]

use cortex_m_rt::entry;
use panic_halt as _;
use va108xx as pac;

// REB LED pin definitions. All on port A
const LED_D2: u32 = 1 << 10;
const LED_D3: u32 = 1 << 7;
const LED_D4: u32 = 1 << 6;

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    // Enable all peripheral clocks
    dp.SYSCONFIG
        .peripheral_clk_enable
        .modify(|_, w| unsafe { w.bits(0xffffffff) });
    dp.PORTA
        .dir()
        .modify(|_, w| unsafe { w.bits(LED_D2 | LED_D3 | LED_D4) });
    dp.PORTA
        .datamask()
        .modify(|_, w| unsafe { w.bits(LED_D2 | LED_D3 | LED_D4) });
    for _ in 0..10 {
        dp.PORTA
            .clrout()
            .write(|w| unsafe { w.bits(LED_D2 | LED_D3 | LED_D4) });
        cortex_m::asm::delay(5_000_000);
        dp.PORTA
            .setout()
            .write(|w| unsafe { w.bits(LED_D2 | LED_D3 | LED_D4) });
        cortex_m::asm::delay(5_000_000);
    }
    loop {
        dp.PORTA
            .togout()
            .write(|w| unsafe { w.bits(LED_D2 | LED_D3 | LED_D4) });
        cortex_m::asm::delay(25_000_000);
    }
}
```

With the JLink tools installed and a few helper files in place you can find in my
[workspace](https://egit.irs.uni-stuttgart.de/rust/va108xx-workspace), I started the `JLinkGDBServer`
and then flashed the board with

```sh
cargo run -p va108xx-hal --example blinky-pac
```

![](/gif/rust-ecosystem/vor-blinky.gif)

Looking good!

Now, the next challenge was to write the next level of abstraction: A Hardware Abstraction
Layer (HAL) crate. I started browsing some of the HALs like the
[stm32f1xx-hal](https://github.com/stm32-rs/stm32f1xx-hal/blob/master/src/gpio.rs) and intially
felt overwhelmed by all the new syntax, macros and ways to write Rust code I have not seen before.
But I also thought that implementing this for a new MCU family would be an excellent way to learn
a lot.

You can find the first version I eventually was able to hack together
[here](https://github.com/robamu-org/va108xx-hal-rs/blob/v0.1.0/src/gpio.rs).
The most challenging task was to wrap around my head around the typesystem and the macro system.
Since then, I refactored the GPIO library to use even more type-level features based on the
excellent [atsamd HAL](https://docs.rs/atsamd-hal/latest/atsamd_hal/gpio/v2/index.html).

The blinky example using the hardware abstraction layer looks like this

```rs
#![no_main]
#![no_std]
use cortex_m_rt::entry;
use embedded_hal::digital::v2::ToggleableOutputPin;
use panic_halt as _;
use va108xx_hal::{gpio::PinsA, pac, prelude::*};
#[entry]
fn main() -> ! {
    let mut dp = pac::Peripherals::take().unwrap();
    let porta = PinsA::new(&mut dp.SYSCONFIG, Some(dp.IOCONFIG), dp.PORTA);
    let mut led1 = porta.pa10.into_push_pull_output();
    let mut led2 = porta.pa7.into_push_pull_output();
    let mut led3 = porta.pa6.into_push_pull_output();
    for _ in 0..10 {
        led1.set_low().ok();
        led2.set_low().ok();
        led3.set_low().ok();
        cortex_m::asm::delay(5_000_000);
        led1.set_high().ok();
        led2.set_high().ok();
        led3.set_high().ok();
        cortex_m::asm::delay(5_000_000);
    }
    loop {
        led1.toggle().ok();
        cortex_m::asm::delay(5_000_000);
        led2.toggle().ok();
        cortex_m::asm::delay(5_000_000);
        led3.toggle().ok();
        cortex_m::asm::delay(5_000_000);
    }
}
```

A lot more readble and neat that the low level PAC version.

Equipped with a lot more knowledge, I implemented a HAL for all other peripherals, using
what I saw in different HALs like the ATSAMD HAL or the STM32 HALs. Writing the basic blocking API
was mostly straighforward once I got more familiar with  writing HAL code. I still have to
write code which is able to use interrupts to efficiently clear the various FIFOs. Using these FIFOs
reduces CPU load is the only sensible for peripherals like the UART to efficiently receive all
arriving RX data.

As a final step, I also implemented a Board Support Package targeted towards the REB1 development
board. This also was an excellent way for me to test the I2C implementation because that board
was equiped with a simple ADT75 I2C temperature sensor device.

The example application looks like this now

```rs
#![no_main]
#![no_std]
use cortex_m_rt::entry;
use panic_rtt_target as _;
use rtt_target::{rprintln, rtt_init_print};
use va108xx_hal::{
    pac::{self, interrupt},
    prelude::*,
    timer::{default_ms_irq_handler, set_up_ms_timer, Delay},
};
use vorago_reb1::temp_sensor::Adt75TempSensor;

#[entry]
fn main() -> ! {
    rtt_init_print!();
    rprintln!("-- Vorago Temperature Sensor and I2C Example --");
    let mut dp = pac::Peripherals::take().unwrap();
    let tim0 = set_up_ms_timer(
        &mut dp.SYSCONFIG,
        &mut dp.IRQSEL,
        50.mhz().into(),
        dp.TIM0,
        interrupt::OC0,
    );
    let mut delay = Delay::new(tim0);
    unsafe {
        cortex_m::peripheral::NVIC::unmask(pac::Interrupt::OC0);
    }
    let mut temp_sensor = Adt75TempSensor::new(dp.I2CA, 50.mhz(), Some(&mut dp.SYSCONFIG))
        .expect("Creating temperature sensor struct failed");
    loop {
        let temp = temp_sensor
            .read_temperature()
            .expect("Failed reading temperature");
        rprintln!("Temperature in Celcius: {}", temp);
        delay.delay_ms(500);
    }
}

#[interrupt]
fn OC0() {
    default_ms_irq_handler();
}
```

In my opinion, all the abstractions made this code really neat a brief. In this example, one
can also see a few more features being used:

1. The RTT logger, which worked more or less out of the box. Really useful to get debugging output
2. An interrupt being used to get a MS timer tick. The Cortex-M0 does not really have
   a SysTick peripheral suitable for this task, but the 24 TIM peripherals can be set up to generate
   millisecond or second interrupts without issues. For something common like a millisecond counter,
   I set up default setup functions and IRQ handlers like seen in the code above.
3. A temperature sensor abstraction provided by the BSP, which makes this example application really
   concise.

And this is what the output looks like after flashing the board with

```rs
cargo run --example adt75-temp-sensor
```

![Reading a temperature sensor in Rust
](/img/rust-ecosystem/temp-sensor.png "Reading a temperature sensor in Rust")

I also want to write a bit about the development workflow I am using. It is possible doing
debugging using a graphical user interface which is important for any application which is more
complex in my opinion. I am using VS code for this and I'm really happy with the development
workflow. All that is required is the `Cortex-Debug` plugin.

An example `launch.json` file for VS code looks like this

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "cortex-debug",
            "request": "launch",
            "name": "Debug Blinky",
            "servertype": "jlink",
            "cwd": "${workspaceRoot}",
            "device": "VA10820",
            "svdFile": "../va108xx-rs/va108xx.svd",
            "preLaunchTask": "rust: cargo build minimal blinky",
            "executable": "${workspaceFolder}/target/thumbv6m-none-eabi/debug/examples/blinky-leds",
            "interface": "jtag",
            "runToMain": true,
        },
    ]
}
```

and an example `tasks.json` file like this

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "rust: cargo build blinky",
            "type": "shell",
            "command": "~/.cargo/bin/cargo", // note: full path to the cargo
            "args": [
                "build", "--example", "blinky-leds",
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        },
    ]
}
```

with this, we can soon use the Run & Debug tab of VS code to debug our applications

![Debugging with VS Code](/img/rust-ecosystem/vs-code.png)

Implementing a full Rust ecosystem for the VA10820 has been a challenging but satisfying task. I
think Rust offers excellent features which are really useful to write safe code for space
applications and I hope that some of the code I have written will be used to run awesome missions
soon.

In the meantime, I also managed to procure a
[PEB1 development board](https://www.voragotech.com/products/peb1va416x0-development-kit) for
the [VA416xx familyof devices](https://www.voragotech.com/products/va416xx/bga). I also want to
thank [Thales Alenia Space](https://www.thalesgroup.com/en/global/activities/space) for providing
me with this expensive hardware.
I am also trying to get the industry version of the
[SAMRH71](http://ww1.microchip.com/downloads/en/DeviceDoc/00003161A.pdf) MCU. Time to write more
Rust code!
