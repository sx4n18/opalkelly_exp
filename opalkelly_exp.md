c# Opal Kelly Experiment

This markdown logs down my learning curve for the board of opal kelly and also evetually how to do the chip measurement after.

I will take a different route as Steve and use python to do most of the backend/frontend.

The whole system works by the following logic:

```
+--------------------------------------------------+
|                      PC                          |
|--------------------------------------------------|
|  Python / C++ / MATLAB / GUI Application         |
|                                                  |
|  FrontPanel API                                  |
+-----------------------+--------------------------+
                        |
                        | USB cable
                        v
+--------------------------------------------------+
|              Opal Kelly FPGA Board               |
|--------------------------------------------------|
|                                                  |
|   +------------------------------------------+   |
|   | USB Host Interface Chip                  |   |
|   | (managed by Opal Kelly firmware)         |   |
|   +------------------+-----------------------+   |
|                      |                           |
|                okUH / okHU                      |
|                      |                           |
|   +------------------------------------------+   |
|   | FPGA                                      |  |
|   |------------------------------------------|  |
|   |                                          |  |
|   |  okHost                                  |  |
|   |                                          |  |
|   |  okWireIn                                |  |
|   |  okWireOut                               |  |
|   |  okTriggerIn                             |  |
|   |  okTriggerOut                            |  |
|   |  okPipeIn                                |  |
|   |  okPipeOut                               |  |
|   |                                          |  |
|   |  Your Custom Logic                       |  |
|   |  - LEDs                                  |  |
|   |  - FSMs                                  |  |
|   |  - DSP                                   |  |
|   |  - LVDS                                  |  |
|   |  - Compression Engine                    |  |
|   +------------------------------------------+   |
+--------------------------------------------------+
```



## First ever hello world project

**Date: 11 May 2026**


I will try to do a very simple hello world project using python script.

The hardware implemented in the FPGA will simply be a NAND gate, which takes 2 inputs from WireIn and output NAND result to LED.

This will be encapsulated at the top moudle with 2 okWireIn and 1 okHost.

The module looks like below:

```verilog
module Top(

// Front panel USB interface
    input   [4:0] okUH,  // USB to host
    output  [2:0] okHU,  // host back to USB
    inout   [31:0] okUHU, // host and USB's data exchange
    inout           okAA, 
    
    
    output  led // output of our LED

    );
    
    wire WireInA, WireInB;
    
    wire [112:0] okHostEP;
    wire [31:0] ep00wire;
    wire [31:0] ep01wire;
    
    
    wire okClk;
    
    
    assign WireInA = ep00wire[0];
    assign WireInB = ep01wire[0];
    
    okHost  okHOSTTOP
    (
    .okUH(okUH),
    .okHU(okHU),
    .okUHU(okUHU),
    .okAA(okAA),
    .okClk(okClk),
    .okHE(okHostEP)
    
    );
    
    
    
    okWireIn    EP00 (.okHE(okHostEP),                             .ep_addr(8'h00), .ep_dataout(ep00wire));
    okWireIn    EP01 (.okHE(okHostEP),                             .ep_addr(8'h01), .ep_dataout(ep01wire));
    
    // wire [65*2-1:0]  okEHx;
    // okWireOR # (.N(2)) wireOR (okEH, okEHx);
    // okWireOut    ep20 (.okHE(okHE), .okEH(okEHx[ 0*65 +: 65 ]), .ep_addr(8'h20), .ep_datain(ep20wire));
    // okWireOut    ep21 (.okHE(okHE), .okEH(okEHx[ 1*65 +: 65 ]), .ep_addr(8'h21), .ep_datain(ep21wire));
    
    
    NAND_2x1 simple_nand(
        .A(WireInA),
        .B(WireInB), 
        .O(led)
    );
    
    
    
endmodule

```


I then implemneted the logic above in vivado and then wrote a simple python script.

```python
import ok


## Create Front Panel object
dev = ok.okCFrontPanel()

# Open first detected device
if dev.OpenBySerial("") != 0:
    print("Failed to open device")
    exit()

print("Device opened")

# Configure FPGA (optional if already configured)
dev.ConfigureFPGA("Top.bit")

print("LED ON")

# Turn LED ON
dev.SetWireInValue(0x00, 0x0001)
dev.SetWireInValue( 0x01, 0x0001)

# Push WireIns into FPGA
dev.UpdateWireIns()

print("LED now should be off")

input("Press Enter to turn LED BACK ON...")

# Turn LED OFF
dev.SetWireInValue(0x00, 0x0000)
dev.UpdateWireIns()

print("LED should be back ON")

```


I then turned on the python debug mode to programme the FPGA board. The breakpoints were set at the following lines:

```python
# Turn LED ON
dev.SetWireInValue(0x00, 0x0001)

...


print("LED now should be off")

input("Press Enter to turn LED BACK ON...")

```

So at the first breakpoint, the board should have been programmed, and then by default, the LED should be ON.

Given that NAND's input A = 0, B = 0 --> O = 1, which should light up the LED.

![The 1st breakpoint Shown in the python script indicates that our FPGA board has been programmed](./img/First_breakpoint_of_the_first_project_LED_with_NAND_inputs.png)

However, the board is showing something different to the expectation with the LED

![The FPGA board has been programmed, but the LED light is off](./img/OpalKellyBoard_has_just_been_programmed.jpeg)

Then at the next breakpoint, both inputs have been set as 1, A = 1, B = 1 --> O = 0, which should turn off the LED.

![2nd breakpoint proceeds as expected](./img/2ND_breakpoint_of_the_first_LED_project_that_shows_LED_light_turned_on.png)

But the board is showing that the LED is turned ON.

![FPGA board at the 2nd breakpoint turned the LED ON](./img/At_the_second_breakpoint_for_the_FPGA_board.jpeg) 

So, in a way I have implemented the inverted logic, then I found that the correct LED drive should be a tristate assignment in the example.

```verilog
function [7:0] xem7310_led;
input [7:0] a;
integer i;
begin
    for(i=0; i<8; i=i+1) begin: u
        xem7310_led[i] = (a[i]==1'b1) ? (1'b0) : (1'bz);
    end
end
endfunction
```

So I should add an inverted logic with high-impedance to it.

Updating verilog now...

Okay, it is fixed now.


## PyQt6 GUI

**Date: 12-13 May 2026**

I then explored the possibility of having something similar to fonrtpanel's own profile interpretation.

Then I found the GUI package PyQt6 where everything can be configured as widgets and stacked in the window.

After a whole day of digging into this package, I now managed to recreate the firt FrontPanel SDK example.

This example can be found [here](https://docs.opalkelly.com/fpsdk/samples-and-tools/sample-first/).

But it seems that it has been taken down yesterday by OpalKelly.

To demonstrate, it looks something like this:

![Original First demo of FrontPanel](./img/Original_FrontPanel_design_of_the_first_example_looks_neat.png)

Where it has 8 buttons on the top, each would control a single LED separately.

Then followed by 4 LED display which is connected to a fixed value (in this case 0000).

Then it takes 2 16-bit numbers to do a simple addition and return the final sum.

All of these can be replicated using PyQt6 and opal kelly python package.

And it looks like this:

![Purely implemented with PyQt6 to reimplement First Demo of FrontPanel](./img/Purely_PYQT6_based_GUI_control_of_FPGA_of_the_first_FrontPanelSDK_example.png)

Probably the next step for Opal Kelly board experiment should be the inclusion of triggerIn/Out and PipeIn/Out.

I will try to design a very simple design to verify this.


## TriggerIn/Out and PipeIn/Out

**Date: 13 May 2026 ~**

Okay... it seems that opal kelly has dropped the new FrontPanel 6.0 and it has made changes to its python script about how it works.

Now I am reading its migration guide...

The biggest change so far is that it replaces all the wire trigger pipe and register calls.

These will be different with the code below:

```python


## Wires call are different now

# FP5:
   xem.SetWireInValue(0x00, value, mask)
   xem.UpdateWireIns()
   xem.UpdateWireOuts()
   out = xem.GetWireOutValue(0x20)

   # FP6:
   classic_data_port.SetWireInValue(0x00, value, mask)
   classic_data_port.UpdateWireIns()
   classic_data_port.UpdateWireOuts()
   out = classic_data_port.GetWireOutValue(0x20)


## Triggers are different
# FP5:
   xem.ActivateTriggerIn(0x40, 0)
   xem.UpdateTriggerOuts()
   if xem.IsTriggered(0x60, 1):
       ...

   # FP6:
   classic_data_port.ActivateTriggerIn(0x40, 0)
   classic_data_port.UpdateTriggerOuts()
   if classic_data_port.IsTriggered(0x60, 1):


## Pipes

 # FP5:
   xem.WriteToPipeIn(0x80, data)
   xem.ReadFromPipeOut(0xA0, data)
   xem.WriteToBlockPipeIn(0x80, blockSize, data)
   xem.ReadFromBlockPipeOut(0xA0, blockSize, data)

   # FP6:
   classic_data_port.WriteToPipeIn(0x80, data)
   classic_data_port.ReadFromPipeOut(0xA0, data)
   classic_data_port.WriteToBlockPipeIn(0x80, blockSize, data)
   classic_data_port.ReadFromBlockPipeOut(0xA0, blockSize, data)


## Registers

# FP5:
   value = xem.ReadRegister(addr)
   xem.WriteRegister(addr, data)

   # FP6:
   value = classic_data_port.ReadRegister(addr)
   classic_data_port.WriteRegister(addr, data)


## Reset FPGA

# FP5:
   xem.ResetFPGA()

   # FP6:
   classic_data_port.ResetFPGA()
```

I will have to stay with classic API for now...

But it would also mean that I probably do not have any where to refer to for API information.


## Chip3 overall Schematics

I will now organise what we have put on Chip3 so that we can design the test with Opal Kelly.

The following things will be included for at least my side of the chip:
+ Schematics
+ Chip pin order/name
+ Breakout on the PCB board <--> FPGA pin out


What I have put on the chip is basically a PLL configurator and PLL.


Pin out:

```

                  SDO  PLL_NMSX_SEL    <Corner>
                                        
                                        TCKO
                                        GNDK
                                        PLL_VCC18A
                                        PLL_VCC18D
                                        PLL_GND18A
                                        PLL_GND18D
                                        VCCK
                                        PLL_PDN
                                        FRANGE
                                        PLL_TEST

SDI PLL_NMSX_TCK    SE   PLL_FREF_TCKI <Corner>
```

PLL_config's verilog is very simple 

```
module PLL_config (

    input cfg_clk, // slow independent clock, I am expecting it to run at 10 MHz (100 ns)
    input PLL_NMSX_SEL, //SEL signal to mux to choose between default N/M or self customised N/M
    input SDI, // shift data in
    input SE, // shift enable
    input rst_n, // reset signal

    output SDO, // shift data out
    output [5:0] NSX, // final NSX output
    output [5:0] MSX  // final MSX output
    
);


reg [5:0] NS_SREG;
reg [5:0] MS_SREG;

always @(posedge cfg_clk or negedge rst_n)
begin
    if (!rst_n)
    begin 
        {NS_SREG, MS_SREG} <= 12'd0;    
    end
    else 
    begin 
        if(SE)
        begin 
            {NS_SREG, MS_SREG} <= {NS_SREG[4:0], MS_SREG, SDI};
        end
    end

end

assign SDO = NS_SREG[5];

assign {NSX, MSX} = PLL_NMSX_SEL? {NS_SREG, MS_SREG}: 12'b001111_000001; // this is either customised N/M or 15/1
```



**19 May 2026**

From the PCB board schematics Steve left me, I found the Pin-out to the chip.


```txt
----------- OUTPUT FROM FPGA --> INPUT TO CHIP -----------
B34_L21P -->  LVDS_ser_data_ctrl
B34_L19P -->  clk_LVDS
B34_L23P -->  ctrlB
B34_L15P -->  clk_config
B34_L13P -->  data0
B34_L11P -->  samphold
B34_L18P -->  NMSX_TCK
B34_L22P -->  SDI
B34_L6P  -->  PLL_test
B34_L5P  -->  SE
B34_L5N  -->  PLL_PDN
B34_L8N  -->  PLL_FREF_TCK
B13_L5N  -->  FRANGE
B13_L3N  -->  NMSX_SEL

----------- INPUT FROM FPGA --> OUTPUT TO CHIP -----------
B13_L16P <--  TCKO
B13_L1N  <--  SDO
B35_L19P <--  LVDS_P
B35_L19N <--  LVDS_N
```


And also this is what I designed for the counter 300M logic:

```verilog
`timescale 1ns/1ps
module CNT_300M (
    input clk_300m, 
    input rst_n,
    input [1:0] LVDS_SEL,

    output reg D_2_LVDS

    
);


reg [1:0] cnter;

always @(posedge clk_300m or negedge rst_n) begin : proc_cnter
    if(~rst_n) begin
        cnter <= 0;
    end else begin
        cnter <= cnter +1;
    end
end

always @(LVDS_SEL, cnter, clk_300m)
begin
    case (LVDS_SEL)
    2'b00: D_2_LVDS = clk_300m;
    2'b01: D_2_LVDS = cnter[0];
    2'b10: D_2_LVDS = cnter[1];
    2'b11: D_2_LVDS = cnter[0]||cnter[1];
        default : D_2_LVDS = 0;
    endcase
end


endmodule
```

Steve's Config scan chain looks something like:

```txt

on the pump of clk_LVDS, flops are chained up with the following:

data_LVDS --> ctrlA --> EN --> RS --> rst_n --> LVDS_set<1> --> LVDS_set<0> --> rst_n_pll
```

I should make a top level schematics based on my understanding.


