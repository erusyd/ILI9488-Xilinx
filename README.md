# ILI9488 TFT display library for Xilinx FPGA and SoC

This is the port of Jaret Burkett's [ILI9488 Arduino library](https://github.com/jaretburkett/ILI9488) to AMD Xilinx FPGA and SoC.

I removed all Arduino-specific code, optimized SPI writes for the DMA capabilities
of Xilinx SPI libraries and did other modifications.

I must also port the [Adafruit_GFX library](https://github.com/adafruit/Adafruit-GFX-Library).

Tested on AMD Xilinx Zynq-7000 SoC and Artix-7 FPGA (using MicroBlaze CPU) with
the following display: [3.5" SPI Module ILI9488 SKU:MSP3520](http://www.lcdwiki.com/3.5inch_SPI_Module_ILI9488_SKU:MSP3520) (can be purchased [on Amazon](https://www.amazon.com/Hosyond-Display-Compatible-Mega2560-Development/dp/B0BWJHK4M6/ref=sr_1_1?crid=1JH7HIUGZU29J&keywords=3%2C5%22%2B480x320%2BSPI%2BTFT%2BILI9488&qid=1691653179&sprefix=3%2C5%2B480x320%2Bspi%2Btft%2Bili9488%2Caps%2C212&sr=8-1&th=1) or [on AliExpress](https://www.aliexpress.com/item/32995839609.html)).

**TODO:**

- Přečíslovat pin CS na Zynq.

- PS SPI: The code using the library is responsible to select/deselect slaves. Library doesn't do it.

- For MicroBlaze AXI Quad SPI FIFO 256 for performance reasons. **Transaction width must be 8 bits.**
  Don't enable Performance Mode AXI-Lite has to be used (AXI on the MicroBlaze is AXI-Lite).
  Must reduce AXI peripheral frequency to 90 MHz (AXI Quad SPI wasn't working at 100 MHz).

- Increase stack size for the MicroBlaze.

- Add folders src/Adafruit_GFX and src/Adafruit_GFX/Fonts as include paths. Without it the IDE is not able to generate Outlines of the source files.  
  Properties of the application project, C/C++ General, Paths and Symbols, Includes. Specify it as a directory within the workspace.

- Peripherals are overclocked in the demo design:
  
  - AXI GPIO max AXI-lite clock on Artix-7 (slowest speed grade) is 120 MHz in [PG144](https://docs.xilinx.com/v/u/en-US/pg144-axi-gpio), [Table 2-1](https://docs.xilinx.com/pdf-viewer?file=https%3A%2F%2Fdocs.xilinx.com%2Fapi%2Fkhub%2Fdocuments%2F0c0ItRCmnYkoHpcYUCPkEA%2Fcontent%3FFt-Calling-App%3Dft%252Fturnkey-portal%26Ft-Calling-App-Version%3D4.2.26%26filename%3Dpg144-axi-gpio.pdf#G5.306784).
  - AXI Quad SPI max AXI-lite clock on Artix-7 (slowest speed grade) is 120 MHz in [PG153](https://docs.xilinx.com/r/en-US/pg153-axi-quad-spi), chapter [Performance](https://docs.xilinx.com/r/en-US/pg153-axi-quad-spi/Performance).
  - AXI UART lie max AXI-lite clock on Artix-7 (slowest speed grade) is 120 MHz in [PG142](https://docs.xilinx.com/v/u/en-US/pg142-axi-uartlite), [Table 2-1](https://docs.xilinx.com/pdf-viewer?file=https%3A%2F%2Fdocs.xilinx.com%2Fapi%2Fkhub%2Fdocuments%2FdB1MAeh~uLG7FE62a5_QbA%2Fcontent%3FFt-Calling-App%3Dft%252Fturnkey-portal%26Ft-Calling-App-Version%3D4.2.26%26filename%3Dpg142-axi-uartlite.pdf#G5.309065).

- Two warnings when compiling MicroBlaze demo in Release configuration are result of the compiler's optimization.

- Compiler optimization matters.

# Performace

ILI9488 is not very fast.

The performance measurements revealed that for AXI SPI, the use of the high-level function [XSpi_Transfer](https://xilinx.github.io/embeddedsw.github.io/spi/doc/html/api/group__spi.html#ga4c44c082ef2d2b9cf4ba3db8bcebd954) significantly decreases the library's overall performance.

The library only writes over the SPI; we do not read any data back. However, function XSpi_Transfer always reads the receive FIFO buffer from the AXI SPI IP (even when you provide NULL as the value of the receive buffer).  
It means that when you send 100 B of data to the display, 200 B of data are transferred in total over a relatively slow AXI bus.

For AXI SPI, I, therefore, implemented the private method [ILI9488::writeToSPI](ILI9488-Xilinx_library/ILI9488_Xil.cpp) [ILI9488::writeToSPI](ILI9488-Xilinx_library/ILI9488_Xil.cpp#L406) using low-level SPI functions (e.g., [XSpi_WriteReg](https://xilinx.github.io/embeddedsw.github.io/spi/doc/html/api/group__spi.html#ga32e741800118678aa060ef2a13661e31)). The implementation doesn't read any data back from the receive FIFO buffer from the AXI SPI IP.  
The fact that  ILI9488::writeToSPI has similar performance on both slow 160 MHz MicroBlaze and fast 667 MHz Zynq-7000 tells me that it's efficient and the performance bottleneck is the 20 MHz SPI clock of the ILI9488 controller.

For PS SPI on Zynq-7000, the method ILI9488::writeToSPI just calls the function XSpiPs_PolledTransfer. XSpiPs_PolledTransfer also always reads the content of the receive FIFO, but that is very fast on Zynq-7000, and, I, therefore, didn't invest time into low-level SPI implementation for PS SPI.

For all tests listed below, the app was compiled with the highest gcc compiler optimization (flag -O3).

#### Fill display 320x480 performance

The durations listed in the table are the durations of the call `display.fillRect( 0, 0, 480, 320, ILI9488_BLUE );` (measured using a GPIO pin and an oscilloscope).

| HW                                              | SW implementation                                                                                                              | FIFO length | Duration |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------- | -------- |
| MicroBlaze&nbsp;160&nbsp;MHz,&nbsp;AXI&nbsp;SPI | low-level SPI functions used                                                                                                   | 256 B       | 185.1 ms |
| Zynq-7000 667 MHz, PS SPI                       | function XSpiPs_PolledTransfer used                                                                                            | 128 B       | 203.3 ms |
| Zynq-7000 667 MHz, AXI SPI                      | low-level SPI functions used                                                                                                   | 256 B       | 185.0 ms |
| Zynq-7000 667 MHz, AXI SPI                      | function XSpi_Transfer used<br/>based on this measurement, I decided to use low-level SPI functions in the library for AXI SPI | 256 B       | 462.7 ms |

#### Fill rectangle 50x50 performance

The durations listed in the table are the durations of the call `display.fillRect( 0, 0, 50, 50, ILI9488_BLUE );` (measured using a GPIO pin and an oscilloscope).

| HW                                              | SW implementation                                                                                                              | FIFO length | Duration |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | ----------- | -------- |
| MicroBlaze&nbsp;160&nbsp;MHz,&nbsp;AXI&nbsp;SPI | low-level SPI functions used                                                                                                   | 256 B       | 3.041 ms |
| Zynq-7000 667 MHz, PS SPI                       | function XSpiPs_PolledTransfer used                                                                                            | 128 B       | 3.324 ms |
| Zynq-7000 667 MHz, AXI SPI                      | low-level SPI functions used                                                                                                   | 256 B       | 3.019 ms |
| Zynq-7000 667 MHz, AXI SPI                      | function XSpi_Transfer used<br/>based on this measurement, I decided to use low-level SPI functions in the library for AXI SPI | 256 B       | 7.665 ms |
