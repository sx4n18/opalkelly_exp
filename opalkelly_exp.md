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

