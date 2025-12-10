# oresat-at86rf215-driver
Microchip AT86RF215 Rust driver and utilities

[Datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/Atmel-42415-WIRELESS-AT86RF215_Datasheet.pdf) (PDF)
Initial [design thoughts](https://docs.google.com/document/d/1zBbb-4qnycPR2GkD_XNbqpXR3gw3rPBdjSQARGNUBZ0/edit?tab=t.0)


# PSU Rust Class information

The beginnings of a radio driver for OreSat.
The initial layout in `radio.rs` was inspired by the previous OreSat radio driver: [ax5043 driver](https://github.com/oresat/oresat-ax5043-driver/tree/master).

Instead of custom macros to define the registers as seen in ax5043, I decided to use the [bitfield_struct](https://github.com/wrenger/bitfield-struct-rs) crate. One of the main benefits is that it can define read and write permissions for each subfield of a register. For instance in the BBCn_PMUC register (Datasheet pages 162-163) the SYNC field is read only, but the remaining fields are read/write. `bitfield_struct` will generate commands to write to all the fields except the SYNC field.

The bulk of defining the bitfield registers was done with AI.
I created a markdown version of the Datasheet with [marker-pdf](https://pypi.org/project/marker-pdf/), and passed that on to Claude. This made all registers single `u8` as described in the Datasheet, but I later conglomerated the multi-registers like the 32-bit timestamp counter (BBCn_CNT0 through BBCn_CNT3) since they would always be read together anyway.
* **Note**: I have verified most registers are correct, but have not looked through them all yet.

The driver will generate write and read commands, but the actual SPI communication will be passed on to whatever tool the user desires. This will be `spidev` in our case.
This is nice for me at the moment since I don't have the actual radio hardware. Even without the hardware tests can still be made for all command generation.

The radio is capable of Block Access Mode (Datasheet pages 17-18). Since SPI is a relatively slow operation it is best to limit the number of commands sent. I created bulk read and writing structs that will check if any pending operations are contiguous and consolidate them.

I want to simplify the implementations for the Readable and Writable traits.
Right now the implementations are a ton of repeated code starting at line 107 in `registers.rs`.
From what I understand, since the bitfield registers are different sizes there needs to be different implementations to handle `Into<u8>` (single byte register) through `Into<u64>` (eight byte register). But I need to look into this more and verify. It would be nice to cut down the boilerplate if possible.
I tried only doing `Into<u64>`, then converting to a vector of `u8`s and slicing the required number of bytes, but that did not work. It complained about the wrong usize in the impl block if I wasn't using an 8 byte register.

I wasn't sure about what to do for the bitfield name `mod` since this is a keyword in the Rust language for a module. I wanted all the bitfield names to match what is in the Datasheet, and match the naming conventions for variables. For now, I set it to `mod_` and added `#[allow(non_snake_case)]` to git rid of the warnings.


## Next steps
* See if I can simplify the `Into<usize>` stuff.
* Write tests for the bulk operations.
* Start on adding TOML based config loading.