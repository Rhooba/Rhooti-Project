# Rhooti-Project
Smart Traffic Light Controller
CS 240 — Micro Architecture & Assembly Language
Cuesta College | Dr. Dube | EMU8086

Overview
An 8086 assembly program that simulates a smart traffic light controller using the EMU8086 Traffic Light virtual device. The controller handles three distinct scenarios: a standard timed cycle, a pedestrian crossing request, and an emergency vehicle override.

How It Works
StateTriggerBehaviorNormal CycleAutomaticLoops Green → Yellow → Red with timed delaysPedestrian ModePress P or pHolds Red for an extended duration to allow safe crossingEmergency ModePress E or eForces Red immediately and holds for emergency vehicle passage

Port & Bit Values
The program communicates with the traffic light hardware via Port 4 using the OUT instruction. Each color maps to a specific bit in the byte sent to the port:
LightBitValueRedBit 01YellowBit 12GreenBit 24

Non-Blocking Input
Keyboard input uses a two-step approach so the timer never freezes:

AH=0Bh — checks if a key is waiting (returns FFh = yes, 00h = no)
AH=01h — reads the key only if one is available


Procedures
ProcedureDescriptionMAINEntry point, runs the main cycle loopGREEN_LIGHTActivates green light with timed delayYELLOW_LIGHTActivates yellow light with timed delayRED_LIGHTActivates red light with timed delayCHECK_INPUTNon-blocking keyboard check, routes to correct modePEDESTRIAN_MODEHolds Red for pedestrian crossingEMERGENCY_MODEForces immediate Red for emergency vehicle
