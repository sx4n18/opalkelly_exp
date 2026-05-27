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
B34_L21P -->  LVDS_ser_data_ctrl ====>      V9 
B34_L19P -->  clk_LVDS           ====>      V7  
B34_L23P -->  ctrlB              ====>      Y8
B34_L15P -->  clk_config         ====>      W6  (SC)
B34_L13P -->  data0              ====>      R4  (SC)
B34_L11P -->  samphold           ====>      Y4  (SC)
B34_L18P -->  NMSX_TCK           ====>      Y6 
B34_L22P -->  SDI                ====>      AA8 
B34_L6P  -->  PLL_test           ====>      U3 
B34_L5P  -->  SE                 ====>      W1 
B34_L5N  -->  PLL_PDN            ====>      Y1 
B34_L8N  -->  PLL_FREF_TCK       ====>      AB2 
B13_L5N  -->  FRANGE             ====>      AA14 
B13_L3N  -->  NMSX_SEL           ====>      AB13 

----------- INPUT FROM FPGA --> OUTPUT TO CHIP -----------
B13_L16P <--  TCKO              ====>      W15 
B13_L1N  <--  SDO               ====>      AA16 
B35_L19P <--  LVDS_P            ====>      N4 
B35_L19N <--  LVDS_N            ====>      N3 
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


## 20 May 2026

Here are the complete pin out for chip3:

```
<CornerTL>    VCCA   VCCD   OutP  OutN   GNDA   GNDD   BG   VCCK   GNDO   VCC3O   SDO  PLL_NMSX_SEL  <CornerTR>
data_LVDS                                                                                               TCKO
clk_LVDS                                                                                                VCCK
VCCK                                                                                                    
GNDK                                                                                                 PLL_VCC18A 
ctrlB                                                                                                PLL_VCC18D
VCC3O                                                                                                PLL_GND18A
GNDO                                                                                                 PLL_GND18D   
extres                                                                                                
VCCK                                                                                                 VCCK
clk_LVDS                                                                                             PLL_PDN                                                   
GNDK                                                                                                 FRANGE
srd                                                                                                  PLL_TEST
<CornerLL>    SH  Pw_out centre_pow lpowen LPF GNDK centre vddpow SDI PLL_NMSX_TCK SE PLL_FREF_TCKI <CornerLR>
```

I also also now ploted out the schematic for the testable of our chip3.

![The schematics of what we have implemented on Chip3](./img/Chip3_overall_schematics_to_be_tested.png)

I will now start looking at the opal kelly board design, and try to at least make the scan chain work.


## 21 May 2026

From the PCB prints and schematics, it looks that the power and ground has been sorted by the board.

So I do not have to consider about that but simply how to drive and observe the actual signal.

To start with, I would like to simply just test the scan chain.

I have now finished the hardware design, with the following address information added for the hardware:

```text

############## WireIn endpoint 0x00 ###############

[11:0] -->  PARA_FEED
[12]   -->  PARAIN_EN
[13]   -->  OK_SCAN_IN
[14]   -->  rstn_ok
[15]   -->  PLL_PDN
[16]   -->  PLL_TEST
[17]   -->  FRANGE
[18]   -->  PLL_NMSX_SEL
[19]   -->  LVDS_ser_data_ctrl
[20]   -->  ctrlB


############## WireOut endpoint 0x20 ###############

OK_LED_WIREOUT_EP  -->  [11:0]
SDO_from_chip      -->  [12]
OK_SCAN_OUT        -->  [13]


############## TriggerIn endpoint 0x40 ###############
# @ FPGA divided clock of 20MHZ

[0]   -->  OK_SCAN_ENABLE
[1]   -->  SE
```
The schematics for the developed hardware on OpalKelly for the first round test can be found below:

![First scan chain test schematics developed on the opal kelly board](./img/First_opal_kelly_scan_chain_test_schematics.png)

I will now proceed to develop a simple python script to test the hardware implementation first.


## 22 May 2026

I implemented a simple script to test the implemented circuit on the opal kelly board, and built a quick GUI interface based on it.

But it appears that the "set scan in high" button and "scan chain shift" did not work properly.


### Symptoms

By default all the LED will be lit and reset works fine.

when scan in set high is pressed after reset, it will lit up the last 3 LEDs and when de-asserted, the last 2 LEDs will be lit.

Scan chain shift does not work at all.

### Diagnose

I decided to build the text box to display what is the actual WireIn values.

Because all 10 signals were twisted into the same 32-bit word, it can be easily mistaken.

Then I realised that in the set_scan_in method, I set the wrong wire in value:

when scan_in needs to be set 1, I did:

```python
 dev.SetWireInValue(0x00, 0x1FE00)  # [11:0] = 00, [15:12] = 1110 = E, [20:16] = 11111 = 1F
```

which produces 4-bit less than needed value, which means Para_feed was set "0xE00", and parain_en = 1. This disabled the scan functionality for RTL on the FPGA.

Now it is fixed as below:

```python
 def set_scan_in(self):
        if self.scan_in_button.isChecked():
            ## only change the ok_scan_in bit to 1, keep other bits unchanged

            dev.SetWireInValue(0x00, 0x1FE000)  # [11:0] = 000, [15:12] = 1110 = E, [20:16] = 11111 = 1F
        else:
            ## only change the ok_scan_in bit to 0, keep other bits unchanged

            dev.SetWireInValue(0x00, 0x1FC000)  # [11:0] = 000, [15:12] = 1100 = C, [20:16] = 11111 = 1F
        dev.UpdateWireIns()
```

Now the behaviour is correct:

![Updated scan chain test for opal kelly board](./img/Fixed_scan_shift_behaviour_for_designed_RTL_on_FPGA.png)

This also inspired me to probably implement individual buttons for needed WireIn instead.


## 26 May 2026

Long weekend, also high heat....

Will now add the buttons and a set box to set up the wire in values

Okay, now I have added textboxes and check boxes to programme the WireIn values.

But the update WireIn button needs to be clicked before these values are sent in.

Also I realised that the MSB LED has been placed on the right hand most and LSB has been placed on the left hand most, which is counter intuitive.

I have updated the LED arrangement to make it more readable as well.


## 27 May 2026

Now I want to do the proper testing in the lab, but the problem is that I could only carry my mac into the lab.

And FrontPanel has released their 6.0 version, which caused some changes on the python API.

So I decided to make a new branch and update all my code in this new branch in 6.0 feature.

Migration complete.

Now I can start the test by plugging FPGA and the PCB together.

And observe the chip socket through oscilloscope. 

Ok, I think I have just made the first step for chip measurement.

Thanks to the helpful test PCB design from Steve, I could easily observe the signal coming out of our FPGA pin.

![The side by side of the PCB schematics and my designed GUI for testing](./img/My_designed_GUI_togehter_with_PCB_signal_monitor_connector.png)

As the PCB schematic suggests, punch-hole of 6,9,16 should be driven by the same 20 MHz clock.

Combined with the actual PCB layout displayed below:

![the actual connector layout on the PCB](./img/Connector_plug_pin_out_on_the_PCB_actual_layout.png)

We could do preliminary test for it.

So I tested pin 16 of the connector, aka clk_LVDS.

![The probe clutching on to the punch hole 16 which connects to clk_LVDS](./img/probe_of_the_oscilloscope_connected_to_pin_hole_16_of_J1.jpeg)

And correspondingly, 20MHz clock was observed from the scope

![What was measured and displayed on the oscilloscope for this measurement](./img/20MHz_observed_from_punchhole_16_on_J1_connector_plug.jpeg)

From the GUI I have set up for this test, it can be seen that FRANGE/NMSX_SEL has been set to 1, therefore, 3.3 V should be observed on punch-hole 3/4.

![scope probe hook has been connected to punch hole of number 4, which corresponds to NMSX_SEL](./img/probe_of_the_scope_has_been_moved_to_punchhole_number_4_which_corresponds_to_NMSX_SEL.jpeg)

And I am very lucky that 3.3 V has been observed from the scope

![3.3V was observed from the oscilloscope from the setting of the pin NMSX_SEL](./img/Presset_T_is_3.3V_which_mathces_the_output_of_FPGA_NMSX_SEL.jpeg)

This has been verified to be true when I turned it off on the GUI, then 0V was observed.

