# PLL_OSU180nm_VSD
Documentation of work done for On-Chip clock multiplier / PLL workshop for OSU 180nm node by VSD for VSD Open 2021

---

## Introduction

- PLL : Phase Locked Loop
- Clock multiplier is a circuit that multiplies the frequency of a clock signal
- It is used in processors for generating variable clock frequencies
- Clock multipliers use PLLs with clock dividers in the feedback loop

---

## Tools used

- ```esim``` : Used to create spice netlists. Developed by FOSSEE group at IITB
- ```Magic``` : An open source layout editor for designing layouts and generating spice netlists from them
- ```ngspice``` : An open source circuit simulator

---

## Theory

### Phase Locked Loop

A phase locked loop is a closed loop feedback system. The output is generated from a voltage controlled oscillator. The phase difference between the oscillator output and required output is measured as the error. This error is low-pass filtered and fed into the VCO. This closed loop feedback system "locks" to the frequency of the input using the phase diffence, giving it the name

Below is an abstract block diagram of the PLL

![PLL block diagram](docs/pll.drawio.svg)

#### Clock multiplier using PLL

A PLL can be used to create a clock multiplier by adding a clock divider in the feedback loop as shown below.

![Clock multiplier block diagram](docs/clock_mul.drawio.svg)

#### Our implementation

Our design has some more blocks.

![Clock multiplier design](docs/design.drawio.svg)

It has a multiplexer in the feedback loop to provide the option of directly feeding a voltage to the VCO. The phase frequency detector 

#### Phase frequency detector

The phase detector measures the phase differencce between the output clock signal and the input clock signal. It is the error computing block from a control systems point of view.

![Phase detector](docs/phase_detector.png)

The circuit used here is :

![Phase detector circuit](docs/phase_detector_ckt.png)

The phase frequency detector has 2 outputs, ```UP``` and ```DOWN```. 

```UP``` being high means the clock is lagging with respect to reference and we need to increase clock frequency

Obviously, ```DOWN``` being high means the clock is leading with respect to reference and we need to decrease the clock frequency

#### Charge pump and RC filter

Charge pump converts the ```UP```, ```DOWN``` signals from the phase frequency detector to an actual voltage that is then low pass filtered and passed to the VCO

Charge pump circuit :

![Charge pump circuit](docs/charge_pump_ckt.png)

RC low pass filter circuit :

![LPF circuit](docs/lpf_ckt.png)

#### Voltage Controlled Oscillator

The VCO here is a ring oscillator. The control voltage is used to vary the supply voltage of the inverters which changes their delays and therefore the frequency of the ring oscillator.

![VCO](docs/vco_ckt.png)

---

# Pre-layout simulation

Pre-layout simulation is done using ngspice netlists generated from esim.

---

### 1) Inverter (example)

#### Esim netlist

The netlist generated from esim looks like this : 

File : [```pre_layout/esim/inv101.cir```](pre_layout/esim/inv101.cir)
```
* /home/paras/Desktop/udemypll/prelayout/esim/inv101.cir

* EESchema Netlist Version 1.1 (Spice format) creation date: Sun Jul 25 19:21:36 2021

* To exclude a component from the Spice Netlist add [Spice_Netlist_Enabled] user FIELD set to: N
* To reorder the component spice node sequence add [Spice_Node_Sequence] user FIELD and define sequence: 2,1,0

* Sheet Name: /
M2  vdd in out vdd mosfet_p		
M1  out in GND GND mosfet_n		

.end
```

#### Modifications

We need to modify this and add ```.control``` portion to do transient simulation. We also add voltage sources for power and input (V2 and V1 respectively). We also change the mosfet definitions for the PDK we use and specify the W and L parameters.

File : [```pre_layout/inv101.cir```](pre_layout/inv.cir)
```spice
****************************
*Inverter
***************************

.include osu018.lib

M1 out in GND GND nfet l=180n w=180n
M2 VDD in out VDD pfet l=180n w=360n

V1 in 0 PULSE 0 1.8 10p 50p 50p 100n 200n
v2 VDD 0 1.8


.control
tran 0.01ns 400ns
plot v(in)+2 v(out)
.endc

.end
```

#### Running simulation

We run the simulation by invkiing the command ```ngspice inv.cir``` where ```inv.cir``` is the name of the spice netlist. The output looks like :

![Inverter screenshot](docs/inv_out.png)

---

### 3) NAND gates

The phase detector requires NAND gate models. We simulate these too using esim and ngspice. The files are 

#### 2 input NAND

File : [pre_layout/nand101.cir](./pre_layout/nand101.cir)

Output : 

![NAND 101 output](docs/nand101_out.png)

#### 3 input NAND

File : [pre_layout/nand301.cir](./pre_layout/nand101.cir)

Output 

![NAND 301 output](docs/nand301_out.png)

#### 4 input NAND

File : [pre_layout/nand401.cir](pre_layout/nand401.cir)

Output

![NAND 401 output](docs/nand401_out.png)

---

### 3) Phase detector

The spice netlist for the phase detector is in this file : [```pre_layout/pfd.cir```](pre_layout/pll.cir)

It uses subcircuits definitions of NAND gates and inverters mentioned before. The subcircuit of the 

Output

![PFD output](docs/pfd_out.png)

Signals from top to bottom 

1) ```Reference clock```
2) ```VCO clock```
3) ```UP signal```
4) ```DOWN signal```

We can see that when the signal is lagging with respect to reference (at the start), ```UP``` signal becomes high, signaling the VCO to increase frequency to compensate. 

When the signal is leading with respect to reference (at the end), ```DOWN``` signal becomes high, signaling the VCO to reduce frequency to compensate.

Slight ringing noise can also be observed in the output ```UP``` and ```DOWN``` signals.

---

### 4) Phase detector with RC filter and charge pump

A simple RC low pass filter and charge pump is added to the PLL netlist. 

The charge pump is defined as a netlist which is made of MOSFETS and a voltage source

The RC filter part is :

```
C1 Vin_vco 0 200p
C2 N001 0 500p
R1 Vin_vco N001 500
```

The Phase detector from earlier part is also intergrated into a subcircuit : 

```
.subckt pfd_501 f_clk_in f_VCO up down
XX1 N001 N005 N002 vddd 0 nand101
XX2 N002 N008 N006 vddd 0 nand101
XX3 N006 N007 N008 vddd 0 nand101
XX4 N007 N010 N011 vddd 0 nand101
XX5 N011 N009 N010 vddd 0 nand101
XX6 N013 N012 N009 vddd 0 nand101
XX7 f_clk_in N005 vddd 0 inv101
XX8 f_VCO N013 vddd 0 inv101
XX9 N002 N003 vddd 0 inv101
XX10 N003 N004 vddd 0 inv101
XX11 N009 N014 vddd 0 inv101
XX12 N014 N015 vddd 0 inv101
XX13 N004 N006 N007 N001 vddd 0 nand301
XX14 N007 N010 N015 N012 vddd 0 nand301
XX15 N012 down vddd 0 inv101
XX16 N006 N002 N009 N010 vddd 0 N007 nand401
XX17 N001 up vddd 0 inv101
V1 vddd 0 1.8
.ends pfd_501
```

The phase detector consists of a voltage soure and nand gates and inverters. Refer theory part for the schematic

File : [```pre_layout/pfd_full.cir```](pre_layout/pfd_full.cir)

First, we comment out the RC filter to see only the charge pump output.

Output
![Charge pump output](docs/pfd_chargepump_out.png)

Then, we add the RC filter.

Output
![Full phase detector output](docs/pfd_full_out.png)

Signals in order from top to bottom

1) ```Reference clock```
2) ```VCO clock```
3) ```UP signal```
4) ```DOWN signal```
5) ```Input to VCO```

Here we can see how the UP and DOWN signals are converted to pulses by the charge pump and to a continuous waveform by the RC filter. This signal is then fed to the VCO.

---

### 5) Voltage Controlled Oscillator


