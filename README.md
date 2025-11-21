## AIM:

To implement and design synthesis and simulation for uart using cadence - genus and innovus.

## TOOLS REQUIRED:

Functional Simulation: Incisive Simulator (ncvlog, ncelab, ncsim) Synthesis: Genus Floorplanning: Innovus

## PROCEDURE:

Step 1: Synthesis requires the following files as follows: ◦ Liberty Files (.lib) ◦ VerilogFiles (.v )

### Add Verilog  (.v ) file

```
`timescale 1ns / 1ps

module uart (
    input reset,
    input txclk,
    input ld_tx_data,
    input [7:0] tx_data,
    input tx_enable,
    output reg tx_out,
    output reg tx_empty,
    input rxclk,
    input uld_rx_data,
    output reg [7:0] rx_data,
    input rx_enable,
    input rx_in,
    output reg rx_empty
);

// Internal Variables
reg [7:0] tx_reg;
reg tx_over_run;
reg [3:0] tx_cnt;

reg [7:0] rx_reg;
reg [3:0] rx_sample_cnt;
reg [3:0] rx_cnt;
reg rx_frame_err;
reg rx_over_run;
reg rx_d1;
reg rx_d2;
reg rx_busy;

// UART RX Logic
always @(posedge rxclk or posedge reset)
begin
    if (reset) begin
        rx_reg <= 0;
        rx_data <= 0;
        rx_sample_cnt <= 0;
        rx_cnt <= 0;
        rx_frame_err <= 0;
        rx_over_run <= 0;
        rx_empty <= 1;
        rx_d1 <= 1;
        rx_d2 <= 1;
        rx_busy <= 0;
    end 
    else begin
        rx_d1 <= rx_in;
        rx_d2 <= rx_d1;

        if (uld_rx_data) begin
            rx_data <= rx_reg;
            rx_empty <= 1;
        end

        if (rx_enable) begin
            if (!rx_busy && !rx_d2) begin
                rx_busy <= 1;
                rx_sample_cnt <= 1;
                rx_cnt <= 0;
            end

            if (rx_busy) begin
                rx_sample_cnt <= rx_sample_cnt + 1;

                if (rx_sample_cnt == 7) begin
                    if ((rx_d2 == 1) && (rx_cnt == 0)) begin 
                        rx_busy <= 0;
                    end 
                    else begin
                        rx_cnt <= rx_cnt + 1;

                        if (rx_cnt > 0 && rx_cnt < 9) begin 
                            rx_reg[rx_cnt - 1] <= rx_d2;
                        end

                        if (rx_cnt == 9) begin
                            rx_busy <= 0;
                            rx_empty <= 0;
                            rx_over_run <= (rx_empty) ? 0 : 1;
                        end
                    end
                end
            end
        end

        if (!rx_enable) begin
            rx_busy <= 0;
        end
    end
end

// UART TX Logic
always @(posedge txclk or posedge reset)
begin
    if (reset) begin
        tx_reg <= 0;
        tx_empty <= 1;
        tx_over_run <= 0;
        tx_out <= 1;
        tx_cnt <= 0;
    end 
    else begin
        if (ld_tx_data) begin
            if (!tx_empty) begin
                tx_over_run <= 1;
            end 
            else begin
                tx_reg <= tx_data;
                tx_empty <= 0;
            end
        end

        if (tx_enable && !tx_empty) begin
            tx_cnt <= tx_cnt + 1;
            if (tx_cnt == 0)
                tx_out <= 0; // Start bit
            else if (tx_cnt > 0 && tx_cnt < 9)
                tx_out <= tx_reg[tx_cnt - 1]; // Data bits
            else if (tx_cnt == 9) begin
                tx_out <= 1; // Stop bit
                tx_cnt <= 0;
                tx_empty <= 1;
            end
        end

        if (!tx_enable) begin
            tx_cnt <= 0;
        end
    end
end

endmodule
```

### Add Testbench file
```
`timescale 1 ns / 1 ps
 module uart_tb; 
// Inputs
reg reset; 
reg txclk;
reg ld_tx_data; 
reg [7:0] tx_data; 
reg tx_enable; 
reg rxclk;
reg uld_rx_data; 
reg rx_enable; 
reg rx_in;
// Outputs
wire tx_out;
wire tx_empty;
wire [7:0] rx_data;
wire rx_empty;

uart uut (
.reset(reset),
.txclk(txclk),
.ld_tx_data(ld_tx_data),
.tx_data(tx_data),
.tx_enable(tx_enable),
.tx_out (tx_out),
.tx_empty(tx_empty), 
.rxclk(rxclk),
.uld_rx_data(uld_rx_data),
.rx_data(rx_data),
.rx_enable(rx_enable),
.rx_in(rx_in),
.rx_empty(rx_empty) );
//generate a master clk
reg clk;
//setup clocks
initial clk=0;
always #10 clk = ~clk; // this speed is somewhat arbitrary for the purposes of this sim..
//generate rxclk and txclk so that txclk is 16 times slower than rxclk 
reg [3:0] counter;
initial begin
rxclk=0;
txclk=0;
counter=0;
end
always @(posedge clk) begin
counter<=counter+1;
if (counter == 15) 
txclk <= ~txclk;
rxclk<= ~rxclk;
end
//setup loopback
always@ (tx_out)
 rx_in=tx_out;
initial begin
// Initialize Inputs
reset = 1;
ld_tx_data = 0;
tx_data = 0;
tx_enable = 1;
uld_rx_data = 0;
rx_enable = 1;
rx_in = 1;
// Wait 100 ns for global reset to finish
#500;
reset = 0;
// Send data using tx portion of UART and wait until data is recieved 
tx_data=8'b0111_1111;
#500;
wait (tx_empty==1); //make sure data can be sent
ld_tx_data = 1; //load data to send
wait (tx_empty==0); //wait until data loaded for send
$display("Data loaded for send");
ld_tx_data = 0;
wait (tx_empty==1); //wait for flag of data to finish sending 
$display ("Data sent");
wait (rx_empty==0); //wait for
$display("RX Byte Ready");
uld_rx_data = 1;
wait (rx_empty==1);
$display("RX Byte Unloaded: %b", rx_data);
#100;
$finish;
end
endmodule
```

### Add run.tcl file
```
read_libs /cadence/install/FOUNDRY-01/digital/90nm/dig/lib/slow.lib
read_hdl uart.v

elaborate

read_sdc uart_constraints.sdc

syn_generic
report_area

syn_map
report_area

syn_opt
report_area 

report_area > uart_area.txt
report_power > uart_power.txt
report_timing > uart_timing.txt

write_hdl > uart_netlist.v
write_sdc > uart_output_cons.sdc
gui_show
```

SDC (Synopsis Design Constraint) File (.sdc)

In your terminal type “gedit input_constraints.sdc” to create an SDC File if you do not have one.


The SDC File must contain the following commands; 

### Add SDC file

i→ Creates a Clock named “clk” with Time Period 2ns and On Time from t=0 to t=1. 

ii, iii → Sets Clock Rise and Fall time to 100ps. 

iv → Sets Clock Uncertainty to 10ps. 

v, vi → Sets the maximum limit for I/O port delay to 1ps.

•	The Liberty files are present in the library path,

•	The Available technology nodes are 180nm ,90nm and 45nm.

•	In the terminal, initialise the tools with the following commands if a new terminal is being used. ◦ csh ◦ source /cadence/install/cshrc

•	The tool used for Synthesis is “Genus”. Hence, type “genus -gui” to open the tool.

•	Genus Script file with .tcl file Extension commands are executed one by one to synthesize the netlist. Step 2 : Creating an SDC File Step 3 : Performing Synthesis
![WhatsApp Image 2025-11-21 at 11 31 47_909578b8](https://github.com/user-attachments/assets/0fd2c5f0-b6e0-4d23-811d-dc7fe4b638f8)

### Fig 1: RTL Simulation:
<img width="1478" height="765" alt="image" src="https://github.com/user-attachments/assets/2fa20d08-6f3e-4568-abff-7edf3193c22d" />


### Fig 2: Synthesis RTL Schematic:
![WhatsApp Image 2025-11-21 at 11 39 26_a8ff81e1](https://github.com/user-attachments/assets/28976bcc-1339-4b0e-b665-8319649fa482)


### Fig 3: Area Report:
![WhatsApp Image 2025-11-17 at 09 52 24_7c1c8722](https://github.com/user-attachments/assets/04f65234-f0d3-4ac4-978e-b48dc45b5682)


### Fig 4: Power Report:
![WhatsApp Image 2025-11-17 at 09 52 54_d1258e2d](https://github.com/user-attachments/assets/b78daca4-d687-40ec-bb44-90cdafdd2597)

### Fig 5: Timing Report:
![WhatsApp Image 2025-11-17 at 09 53 12_ad28624f](https://github.com/user-attachments/assets/e7e0fad9-d0a7-4387-8d00-9ac61d30d19a)


### Fig 6: UART APR:

<img width="1560" height="812" alt="image" src="https://github.com/user-attachments/assets/ac0388f3-5d5c-427d-9306-f919979a67a7" />


## RESULT:

The functionality of the uart was successfully verified using a testbench and simulated with the nclaunch tool and genus and for innovus creating the physical design.
