# Opal Kelly Experiment

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




