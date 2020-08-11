
# PCB Design


[Software of choice: kiCAD](https://kicad-pcb.org/)
Reasoning: Its free and well supported.

## Learning PCB Design

[Digi-Key's Intro to kiCAD:](https://www.youtube.com/watch?v=vaCVh2SAZY4/i) this series alone taught me kiCAD. A heads up the version you use will likely be different. If an icon isn't where it appears in the video look in the comments and someone has posted where it got moved to. Also its a 10 video series I recommend 1.75X speed.
[Importing custom graphics into kiCAD](https://www.youtube.com/watch?v=w_7iRCyau7w) - If you want to fancy up your boards.
[PCB Trace Width Calculator](https://www.4pcb.com/trace-width-calculator.html) - To help with determine trace sizes.
 
## kiCAD Project Files

Files go here

## AC creepage and clearance considerations
Useful technical documentation:

[High Voltage PCB Design Creepage and Clearance Distance](https://resources.altium.com/p/high-voltage-pcb-design-creepage-and-clearance-distance)
[Advanced Circuits PCB trace width calculator](https://www.4pcb.com/trace-width-calculator.html)
[Creepage calculator](https://www.smps.us/pcbtracespacing.html)
[Grounding considerations](https://www.autodesk.com/products/eagle/blog/8-pcb-grounding-rules/)
[CTI and PTI](https://db-electronic.com/en/pcb-manufacturing_s56.htm)
[CTI Wikipedia page](https://en.wikipedia.org/wiki/Comparative_Tracking_Index)

Clearance - is the shortest distance through air between two conductors. Environmental conditions have a significant effect on this, primarily humidity changes.

Creepage - measures distances between two conductors along the surface of a high voltage PCB.  This requirement varies with the material of your PCB and can be increaed by slotting or routing air gaps between parts.

Our high voltage components
Our circuit has a small number of high voltage AC traces. Namely the spade terminals where AC power comes onto the boards, our HiLink Mains to 5V PSU, the protection circuitry for the PSU, and the relay contactor pins. These traces will operate at either 120V or 240V depending on the tool being controlled (the HiLink is capable of stepping down both). 

![d0400a78639494adf883cf8e1f101b76.png](:/419b35bbdf2444b29c5b0c16f7a64f89)
Source: [High Voltage PCB Design Creepage and Clearance Distance](https://resources.altium.com/p/high-voltage-pcb-design-creepage-and-clearance-distance)



<u>Determining creepage spacing</u>

**Pollution degree**: Degree 1 - our circuit will be enclosed, dust proof, and indoors therefore falls under only dry non-conductive however, the fabrication shop does get very cold in the winter and can experience temperature swings over the course of the day so condensation is a possibility.
**PCB material**: FR4
**FR4 CTI**: 175 therefore material group IIIb.  
**Max current (Signal lines)**: 40mA DC (ESP32 GPIO limit).
**Max current (5V lines)**: 1A (DC limit of our PSU).
**Max current (AC lines)**: 0.5A (AC limit of our PSU/Fuse threshold).

Given that our system could have either 120V or 240V supply we must plan for clearance and creepage at 240V. Additionally although our deployment mostly falls under pollution degree 1 using the spacing recommendations for spacing level 3 (where possible) will help us guard against edge cases.

Clearance calculation at 240V
![dcfa7dfef7616ca484b450242055016c.png](:/c80cf014d720440f8b6b7ad06b1333c6)
Source: https://www.smps.us/pcbtracespacing.html

Creepage calculation at 240V pollution degree 1 and 3
![9c36e94cb88f419afeb9491e59f662be.png](:/a69857e221cf4c25875aa11e6f0d5e93)
Source: https://pcbdesign.smps.us/creepage.html.
Units: mm.

With these numbers in mind all AC traces were kept >1.25mm clear of one another and 5mm slots were routed between the high voltage AC side of the board and the low voltage DC side to achieve creepage distances of >4mm.


## Mounting Solutions

## BOM

