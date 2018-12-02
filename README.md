# Multiplex M-LINK Protocol Specification

M-LINK is a proprietary wireless transmission protocol for the 2.4GHz ISM band developed by Multiplex Modellsport GmbH. It is typically used for remote controlled R/C models.

The M-LINK protocol has the following features:

- 2.4GHz ISM band
- FH-DSSS allowing for full-range communication with up to 100mW
- Up to 16 Channels with 12-bit channel resolution
- Fast (14ms) or normal (21ms) cycle time resulting in a 28ms or 42ms transmission rate
- Return channel for sensor telemetry

## Protocol Overview

M-LINK uses a Cypress CYRF6936 transceiver, which is configured in legacy DSSS SDR mode, allowing for a data rate of 15.625 kbp/s. The data is sent in packets.

The CYRF6936 frequencies range between 2400MHz and 2480MHz, where channels are 1MHz apart. Of the 80 available channels, M-LINK uses only even channels between 2 and 80, i.e up to 40 channels. The transmit channel is switched every transmission cycle using a pseudo-random sequence that is transmitted to the receiver during binding.

## M-LINK Data Packet

M-LINK data packets are always 8 bytes in size, and consist of a packet identifier byte (PID), 6 bytes payload (P0..P5), and a checksum byte (CRC):

  0 |  1 |  2 |  3 |  4 |  5 |  6 |  7
----|----|----|----|----|----|----|-----
PID | P0 | P1 | P2 | P3 | P4 | P5 | CRC

The CRC is a regular Dallas CRC8, with a random initial value that is transmitted during binding, offset by the current channel index.

## M-LINK Channel Data Packet

The channel data packet contains the R/C channel values, i.e. the pulse-width information for the servos.

In every channel data packet, three 16-bit channel values are transmitted. M-LINK can transmit up to sixteen channel values, for which six different PIDs are reserved. The channel values are encoded as follows:

  0 |  1  |  2  |  3  |  4  |  5  |  6  |  7
----|-----|-----|-----|-----|-----|-----|-----
PID | D1H | D1L | D2H | D2L | D3H | D3L | CRC

DxH/DxL -> 16-bit channel value (high byte/low byte) **Dx**.

The six PIDs and the channel indexes transmitted are shown in the table below:

 PID | D1 | D2 | D3
-----|----|----|----
0x09 | 11 |  9 |  7
0x01 | 12 | 10 |  8
0x0A |  - | 15 | 13
0x02 |  - | 16 | 14
0x88 |  5 |  3 |  1
0x80 |  6 |  4 |  2

The MSB of the channel data packet PID indicates the end of a cycle and a radio frequency change. The 16-bit channel value **Dx** contains a 12-bit value that encodes the PPM pulse width, which ranges between 800us and 2200us (receiver limits to 2200us).

## Channel Data Word

Of the 16-bit channel data word, the lower 12-bits specify the PPM pulse-width that is passed to the servos. The remaining 4-bits are unknown.

```text
Dx_value = (pulse_width - 800us) * 2 * 1.3824
```

The constant 1.3824 ticks/us is derived from the crystal frequency divided by 8, i.e. `11.0592MHz / 8`

Examples:

- 800us -> 0,
- 1000us ->  553 (0x229),
- 1500us -> 1936 (0x790),
- 1600us -> 2212 (0x8A4),
- 2000us -> 3318 (0xCF6)
- 2200us -> 3871 (0xF1F)
- 2281us -> 4095 (0xFFF)

## Packet Timing

M-LINK defines a normal mode and fast-response mode:

- Normal mode: 21ms cycle time, 42ms total transmit time for all 16 channel values
- Fast-response mode: 14ms cycle time, 28ms total transmit time for all 12 channel values

Both modes support a telemetry return channel. The time slot for a receiver telemetry packet is reserved in every 5th cycle.

### Normal mode (21ms cycle time, 16 channels)

In normal mode (without a telemetry cycle), the following channel data packets (PIDs) are transmitted repeatedly:

0x09, 0x0A, 0x88, 0x01, 0x02, 0x80

The six packets above are split in two groups of three packets.
Each group is transmitted in a 21ms transmission cycle.
After each transmission cycle, the transmit channel is changed.

To transmit all 16 channel values, 6 packets in 2 cycles (42ms) are needed.
Packets 0x09/0x01, 0x0A/0x02, and 0x88/0x80 alternate after each cycle.
Bit 7 of the PID indicates a radio frequency (channel) change, denoted by `Ch++`.

```text
|~0x09~~~~~~~~~~~~~|    |~0x0A~~~~~~~~~~~~~|    |~0x88~~~~~~~~~~~~~| Ch++
|~0x01~~~~~~~~~~~~~|    |~0x02~~~~~~~~~~~~~|    |~0x80~~~~~~~~~~~~~| Ch++
|-----------------------|-----------------------|------------------|------------|
|0ms                    |6ms                    |12ms              |~17ms       |21ms
```

### Normal mode cycle with telemetry

Every 5th cycle, a telemetry cycle is inserted. During a telemetry cycle,
the radio is switched to RX mode giving the receiver a window to transmit data. In a telemetry cycle, only the following packets (PID) of a certain group are transmitted:

0x09, 0x88 or 0x01, 0x80

Note that packets 0x0A/0x02 are omitted in a telemetry cycle, at the expense of channels 13 to 16. The time after packet 0x88/0x80 and before the packet 0x09/0x01 is the receive window, indicated by RX. The RX window is about 10ms long.

A normal cycle followed by a telemetry cycle is shown below:

```text
|~0x09/0x01~~~~~~~~|    |~0x0A/0x02~~~~~~~~|    |~0x88/0x80 ~~~~~~~| Ch++ [Rx~~~
|-----------------------|-----------------------|------------------|------------|
|0ms                    |6ms                    |12ms              |~17ms       |21ms

|~~~~~~~~~~~~~~~~~~Rx]  |~0x09/0x01~~~~~~~~|    |~0x88/0x80 ~~~~~~~~| Ch++ 
|-----------------------|-----------------------|-------------------------------|
|0ms                    |6ms                    |12ms                           |21ms
```

### Fast-response (14ms cycle time, 12 channels)

In fast-response mode, the 0x0A/0x02 packets are not included,
i.e. the following packets (PID) are transmitted repeatedly:

0x09, 0x88, 0x01, 0x80

The four packets above are split in two groups of two packets.
Each group is transmitted in a 14ms transmission cycle.
After each transmission cycle, the transmit channel is changed.

To transmit all 12 channels, 4 packets in 2 cycles (28ms) are needed.

```text
|~0x09~~~~~~~~~~~~~|    |~0x88~~~~~~~~~~~~~| Ch++ 
|~0x01~~~~~~~~~~~~~|    |~0x80~~~~~~~~~~~~~| Ch++ 
|-----------------------|------------------|------------|
|0ms                    |6ms               |11ms        |14ms
```

### Fast-response cycle with telemetry

Just like in normal mode, a telemetry cycle is inserted every 5th cycle. During a telemetry cycle, only the following packets (PID) are transmitted:

0x88 or 0x80

Note that packets 0x09/0x01 are omitted in a telemetry cycle, at the expense of channels 7 to 12.

```text
|~0x09/0x01~~~~~~~~|    |~0x88/0x80~~~~~~~~| Ch++ [Rx~~~
|-----------------------|------------------|------------|
|0ms                    |6ms               |11ms        |14ms

|~~~~~~~~~~~~~~~~~~Rx]  |~0x88/0x80~~~~~~~~| Ch++ 
|-----------------------|------------------|------------|
|0ms                    |6ms               |11ms        |14ms
```

## Binding Process

The M-LINK protocol uses a Two-Way Button Bind as outlined in [3].

The bind transmission is always performed on channel 1 using a lower output power.

The procedure involves sending a total of 26 bind packets with a payload of 5 bytes each, resulting in a total of 130 bytes.

This bind data packet have the following layout:

```text
  0 |  1 |  2 |  3 |  4 |  5 |  6 |  7
----|----|----|----|----|----|----|-----
PID | P0 | P1 | P2 | P3 | P4 | P5 | CRC

PID: 0x0F
P0 : Bind packet index (0..25)
P1 : Data byte 1 
P2 : Data byte 2
P3 : Data byte 3 
P4 : Data byte 4 
P5 : Data byte 5 
```

The first packet has the following data:

```text
P1 : 0x40 
P2 : 0x00
P3 : 0x01
P4 : 0x03 (normal) or 0x02 (fast)
P5 : 0xE3 (normal) or 0x9A (fast)
```

The following packets consists of a configuration structure, of which only 108 bytes are used, padded by 0xFF.

The data transmitted is shown in the following structure:

```c
struct mlink_bind_config
{
    uint8_t datacode[16];           // One of the golden PN codes
    uint8_t sopcode[8];             // Always 0
    uint8_t crc_init;               // Random
    uint8_t unknown;                // Perhaps 0x3E (SDR mode)?
    uint8_t channel_table[80];      // Only even channels, 2x 2..80, sometimes only odd channels 2x 3..81
    uint8_t used_channels;          // 0xEC = Every Channel, 0xFE = FrancE
    uint8_t telemetry_frequency;    // Always 5
};
```

## Telemetry

A telemetry data frame sent by the receiver uses the PID 0x13 and contains two sensor values with three bytes each, depicted as S1x and S2x. The high-nibble of the first byte SxA contains the 4-bit sensor address, the low-nibble of the first byte SxA contains a 4-bit unit type. The second and and third byte contains the 16-bit sensor data word, depicted as SxL and SxH. The LSB of the sensor data word is an alarm bit, the remaining 15 bits contain the signed sensor value.

  0 |  1  |  2  |  3  |  4  |  5  |  6  |  7
----|-----|-----|-----|-----|-----|-----|-----
PID | S1A | S1L | S1H | S2A | S2L | S2H | CRC

For more information on the sensor data format, see [4].

### Example

```text
  |   S1   |   S2   |
13 1A C8 00 01 60 00 XX
   || |  ^ Value MSB
   || ^ Value LSB
   |^ Unit
   ^ Address

Receiver LQI: 100%
Receiver Voltage: 4.8V
```

## Implementation details

The microprocessor of the transmitter (typically an ATmega324P) has a 11.0592MHz crystal.
The exact cycle time is a multiple of a fclk/1024 frequency, i.e. 21.111ms for normal mode, and 14.352ms in fast-response mode.
This corresponds to the magic values of 227+1 (0xE3 = normal) and 154+1 (0x9A = fast) found in the **0F 40** binding packet.

SPI transmission of packet takes about 150us.
RF Transmission of data packet takes about 4.5...4.8ms.

### Radio Configuration

- M-LINK uses the legacy SDR mode, i.e. all SOP and framing features are disabled, no 16-bit CRC is used.
- M-LINK uses up to 40 transmit channels. The channel changes every transmit cycle using a pseudo-random sequence that is unique for each transmitter. The pseudo-random sequence repeats every 78 cycles.

## References

- [1] [Cypress CYRF6936 Datasheet](http://www.cypress.com/file/126466/download)
- [2] [Cypress WirelessUSB LP: Technical Reference Manual](http://www.cypress.com/file/126466/download)
- [3] [Cypress AN6066: Wireless Binding Methodologies](http://www.cypress.com/file/126466/download)
- [4] [Multiplex Sensor Bus V2](https://www.multiplex-rc.de/Downloads/Multiplex/Schnittstellenbeschreibungen/beschreibung-sensor-bus-v2.pdf)
- [5] [SRXL-MULTIPLEX V2](https://www.multiplex-rc.de/Downloads/Multiplex/Schnittstellenbeschreibungen/srxl-multiplex-v2.pdf)

## Disclaimer

All product and company names are trademarks or registered trademarks of their respective holders. Use of them does not imply any affiliation with or endorsement by them. The author is not affiliated with Multiplex.

## License

SPDX-License-Identifier: GPL-3.0-or-later

Copyright (C) 2018 Marius Greuel
