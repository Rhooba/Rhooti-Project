
1. **Start program.**  
2. **Initialize system.**  
- Set a flag that is triggered by a pedestrian request.  
- Set a variable to hold the current traffic‑light state number.   
3. **Cycle through phases:**	  
- **Phase 1: North-South Green & East-West Red**  
  - Set lights: North-South Green and East-West Red.  
  - Continuously check for emergency or pedestrian input.  
  - Run for T duration, then move to phase 2\.  
- **Phase 2: North-South Yellow & East-West Red**  
  - Set lights: North-South Yellow and East-West Red.  
  - Continuously check for emergency or pedestrian input.  
  - Run for t/2 duration, then move to phase 3\.  
- **Phase 3: Transition (All Red)**  
  - Set lights: All Red.  
  - Continuously check for emergency or pedestrian input.  
  - Run for t/4 duration, and check the pedestrian flag.  
    - If the pedestrian flag is triggered, extend the all‑red duration to 10t.   
      - Clear the pedestrian flag.  
  - Move to phase 4\.  
- **Phase 4: East-West Green & North-South Red**  
  - Set lights: North-South Red and East-West Green.  
  - Continuously check for emergency or pedestrian input.  
  - Run for T duration, then move to phase 5\.  
- **Phase 5: East-West Yellow & North-South Red**  
  - Set lights: North-South Red and East-West Yellow.  
  - Continuously check for emergency or pedestrian input.  
  - Run for t/2 duration, then move to phase 6\.  
- **Phase 6: Transition (All Red)**  
  - Set lights: All Red.  
  - Run for t/4 duration, and check the pedestrian flag.  
    - If the pedestrian flag is triggered, extend the all‑red duration to 10t.   
      - Clear the pedestrian flag.  
  - Jump back to phase 1\.  
* **Input Interrupts (during any phase):**  
- If 'P' or 'p' is pressed, set the pedestrian request flag to 1 and continue timing.  
- If 'E' or 'e' is pressed: Go to **Step 5** (Emergency Procedure).  
4. **Pedestrian procedure:**		  
- When the user presses P or p, it activates the pedestrian flag  
- Do not interrupt the current cycle and wait for the next All Red phase.  
- Hold all lights at RED for an extended 10t duration to allow safe crossing.  
- Clear the pedestrian flag.  
- Return to the active cycle.  
5. **Emergency procedure (using Stack):**  
- Force all lights to RED immediately.  
- Hold all lights at RED for an extended 10t duration to allow safe crossing to allow the vehicle to clear the intersection.  
- Resume the normal cycle.  
6. **Repeat the entire cycle indefinitely.**
