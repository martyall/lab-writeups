# Lab 6 Writeup

This lab was an introduction to the RVfpga I/O system, in particular the GPIO system used to define and control the switches and LEDs on the Nexys A7. There was also a worked example of using the verilator simulator to simulate a simple program that links the switch values to the corresponding LEDs. Finally, there were various exercises, including an "advanced" excerise to wire up the Nexys A7 push-buttons.

I will attempt to highlight some interesting points of the lab and discuss the solution to this exercise.

## Summary

### General I/O Architecture

For a given I/O device (e.g. switches, LEDs, push-buttons) there are three main components that make it work:
1. CPU: Initiates all I/O operations
2. Device Controller: Takes requests from the CPU and manages the device to fulfill those requests.
3. Interconnect: The communication path between the CPU and the Device Controller.

There are certain reserved memory addresses that the CPU uses to communicate requests to an I/O device called *memory-mapped registers*. Rather than using the CPU's normal memory to fulfill store/load operations to these registers, these operations will be handled by the SoC (*SweRVWolfX*) and routed the appropriate device via the Interconnect. (**NOTE**: I have no idea how this hand-off works, and how it is decided whether the operations should go to/from memory or some other device. Also I'm wondering if the instruction could be other than just `sw`/`lw` -- the GPIO example in this lab doesn't use anything else.) 


A CPU operation on one of these memory-mapped registers is handled roughly as follows:

1. The request will somehow get proxied to the I/O interconnect system. In the case of our CPU (the **SweRV EH1 Core**), the core uses an **AXI** bus to read/write from normal memory. However, the I/O interconnect system uses a different communication protocol called **Wishbone**. Therefore there is a piece middleware between the CPU and the rest of the IO system that converts between AXI and Wishbone which is called the *Bridge*.
2. After the Bridge, the request hits the *Device Multiplexer*. The multiplexer chooses the target device based on the requested address (specifically `Address[15:6]`). The request is then forwarded to the appropriate device (using Wishbone).
3. The request reaches the target device. At this point the *Device Controller* handles the request using the target `Address` and `value` it receives from the interconnect. The bits `Address[5:2]` are reserved for the device controller.


Here is a graphic that appears in the lab that depicts the layout of this system:

<img src="https://github.com/martyall/lab-writeups/blob/main/io-art.png"  width="700" height="500">

### Example: GPIO

Our SoC takes its GPIO device controller from the open-cores repository. There is an exact specification of the controller included in the RVfpga material. For our purposes, there are a few important properties:

1. It uses a Wishbone Interconnect (so it is compatible out of the box with our I/O interconnect system).
2. It has the capacity for controlling 32 pins.
3. All GPIO pins can be declared as tri-state buffers. This means that there is a _enable_ register `E` that configures the pins as either input or output pins. Specifically if `E` is `0` the pin is configured as input only, and if `E` is `1` the pin is configured for output. 

Thus the GPIO controller has local registers (in addition to many others):

|  Name |  Address | Width  |  Access | Description   |
|---|---|---|---|---|
| RGPIO_IN  | 0x80001400  | 1-32  | R  | GPIO input data  |
| RGPIO_OUT | 0x80001404  | 1-32  | R/W  | GPIO output data  |
| RGPIO_OE | 0x80001408  | 1-32  | R/W  | GPIO output driver enable  |

So to read the input state of the controller to register `t0`, we would execute 

```bash
sw zero 0x80001408
lw t0 0x80001400
```

And to write the content of register `t0` as ouput, we would execute

```bash
sw 0xFFFFFFFF t0
lw 0x80001408 t0
lw t0 0x80001404 
```

### Case of RVfpga 

In the case of RVfgpa, there is one GPIO controller instantiated in the SoC, and in fact the base memory address is `0x80001400` as in the above example. The controller is managing the Nexys A7 switches and LEDs, which is convenient as this precisely exhausts the 32 available pins for the controller. The LEDs are understood to be managed by the lower 16 bits of the registers, and the switches are understood to be managed by the upper 16 bits. You can verify the upper/lower bits claim in the instantiation of the `swervwolf_core` module in `rvfpganexys.sv`:

```verilog
   swervolf
     (.clk  (clk_core),
     ...
      .io_data        ({i_sw[15:0],gpio_out[15:0]}),
...

   always @(posedge clk_core) begin
      o_led[15:0] <= gpio_out[15:0];
   end
```

The `.io_data` port is connected to the concatenation of the `i_sw` and `gpio_out` signals, where `gpio_out` drives the LEDs. (The reason stated for proxing the LED signal was so that the LEDs were stable). 


Consider one of the example programs `DisplaySwitches.c`:

```c
// memory-mapped I/O addresses
#define GPIO_SWs    0x80001400
#define GPIO_LEDs   0x80001404
#define GPIO_INOUT  0x80001408

#define READ_GPIO(dir) (*(volatile unsigned *)dir)
#define WRITE_GPIO(dir, value) { (*(volatile unsigned *)dir) = (value); }

int main ( void )
{
  int En_Value=0xFFFF, switches_value;

  WRITE_GPIO(GPIO_INOUT, En_Value);

  while (1) { 
    switches_value = READ_GPIO(GPIO_SWs);   // read value on switches
    switches_value = switches_value >> 16;  // shift into lower 16 bits
    WRITE_GPIO(GPIO_LEDs, switches_value);  // display switch value on LEDs
  }

  return(0);
}
```

The statement 

```c
  WRITE_GPIO(GPIO_INOUT, En_Value);
```

Uses the controller's `RGPIO_IN` register to set pins connected to the switches as readable (as the upper two bytes are `0`) and to drive the pins connected to the LEDs (as the lower two bytes are `1`). 

The main loop of this program:

1. Reads all of the values at the pins by reading the controller's `RGPIO_IN` register.
2. Takes the upper half of the `RGPIO_IN` by right-shifting 16 bits. This represents the switch values. Because the lower 16 bits are output-enabled, their read values are not relevant/defined.
3. Writes the switch values to the controller's `RGPIO_OUT` register. As we saw in the `rvfpganexys` module, the LEDs are wired to the lower half of this register, so the switch values ultimately drive the LEDs.

## Exercises

One of the "advanced" exercises was to insantiate another GPIO controller that can control the five push-buttons on the Nexys A7. Because there were no more available pins on the controller managing the LEDs and switches, this was actually necessary.

The solution mostly involved tracing through the RVfgpaNexys verilog files and constraint defintions, and slightly modifying them following the pattern of the existing GPIO instantiation. (Concretely, a lot of copy pasting and adding `_btn` to the wire and port definitions.) This exercise was useful for several reasons, even if it was entirely straightforward:

1. You see what the purpose of the constraint file `rvfpganexys.xdc` is. In this case it is used to map the raw fpga I/O provided by the Nexys A7 to the names that are given by the RVfpgaNexys project. For example, if I wanted to build this project as-is on a different board, it would be necessary to supply a different constraint file that maps that board's corresponding pins to the ones required by the project.
2. You actually see the entire path from board to controller to interconnect, to muxinator, to core. 
3. You get practice using vivado to recreate the bitstream with the push-button controller so that you can write a test program and verify that it works. Of course you could also use the simulator with a little more work.

To test that it works, I loaded the custom-made bitstream onto the board and then rewrote the simple `DisplaySwitches.c` program above to read the value of the buttons instead of the switches. Note that I had programmed the base address for the new GPIO controller as `0x80001800`:


```c
#define GPIO_BTNs 0x80001800 
#define GPIO_LEDs 0x80001404
#define GPIO_INOUT 0x80001408

#define READ_GPIO(dir) (*(volatile unsigned *)dir)
#define WRITE_GPIO(dir, value)                 \
    {                                          \
        (*(volatile unsigned *)dir) = (value); \
    }

int main(void)
{
    int En_Value = 0xFFFF, buttons_value;

    WRITE_GPIO(GPIO_INOUT, En_Value);

    while (1)
    {
        buttons_value = READ_GPIO(GPIO_BTNs);
        WRITE_GPIO(GPIO_LEDs, buttons_value);
    }

    return (0);
}

```

Here's a photo demonstrating that it works:

<img src="https://github.com/martyall/lab-writeups/blob/main/button.png"  width="700" height="500">
