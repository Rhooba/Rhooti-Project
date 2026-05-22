

```

ORG 100h

; ============================================================
; Rhooti Smart Traffic Light Controller - Phase 2
; CS 240 | Dr. Dube | Cuesta College
; ============================================================
; PORT 4 BIT MAP (12-bit device):
;   East  side: Bit 0=Red,  Bit 1=Yellow, Bit 2=Green
;   North side: Bit 3=Red,  Bit 4=Yellow, Bit 5=Green
;   West  side: Bit 6=Red,  Bit 7=Yellow, Bit 8=Green
;   South side: Bit 9=Red,  Bit 10=Yellow,Bit 11=Green
;
; HEX STATES:
;   NS Green  + EW Red  = 030Ch
;   NS Yellow + EW Red  = 028Ah
;   ALL RED             = 0249h
;   EW Green  + NS Red  = 0861h  <-- also used for PEDESTRIAN
;   EW Yellow + NS Red  = 0451h
;
; BH FLAG:
;   BH = 0 -> normal cycle
;   BH = 1 -> pedestrian waiting
;   BH = 2 -> emergency
; ============================================================


MOV BH, 0 ; clear pedestrian flag, no crossing requested yet

; ============================================================
; MAIN PROC
; Normal cycle: NS_GREEN -> NS_YELLOW -> ALL_RED ->
;               EW_GREEN -> EW_YELLOW -> ALL_RED -> repeat
;
; After every phase, CHECK_INPUT runs once (non-blocking).
; Then flag is checked immediately — if set, jump now.
; No waiting through the next phase.
; ============================================================

MAIN PROC
        MAIN_LOOP:
            CALL NS_GREEN   ; run North-South green phase 
            CALL CHECK_INPUT ; check once after phase
            CMP BH, 2       ; check for emergency? 
            JE HANDLE_EMERGENCY ; yes, go to emergency procedure
            
            CALL NS_YELLOW  ; run North-South yellow phase 
            CALL CHECK_INPUT 
            CMP BH, 2        
            JE HANDLE_EMERGENCY 
            
            CALL ALL_RED    ; transition and check pedestrian flag
            CALL CHECK_INPUT 
            CMP BH, 2       
            JE HANDLE_EMERGENCY 
            
            CALL EW_GREEN   ; run East-West green phase
            CALL CHECK_INPUT 
            CMP BH, 2       
            JE HANDLE_EMERGENCY 
            
            CALL EW_YELLOW  ; run East-West yellow phase
            CALL CHECK_INPUT 
            CMP BH, 2       
            JE HANDLE_EMERGENCY 
            
            CALL ALL_RED    ; transition and check pedestrian flag again
            CALL CHECK_INPUT 
            CMP BH, 2       
            JE HANDLE_EMERGENCY 
            
        JMP MAIN_LOOP       ; wall prevents falling into handlers 
        
        HANDLE_PEDESTRIAN:
            MOV BH, 0       ; clear flag before acting on it
            CALL PEDESTRIAN_MODE
            JMP MAIN_LOOP
            
        HANDLE_EMERGENCY:
            MOV BH, 0           ; clear emergency flag
            CALL EMERGENCY_MODE ; handle it
            JMP MAIN_LOOP       ; restart fresh cycle
MAIN ENDP 


; ============================================================
; CHECK_INPUT PROC
; Non-blocking keyboard check using DOS service 06h.
; DL=FFh puts service 06h into "check" mode (not output mode).
; INT 21h sets ZF if no key is waiting — we jump to NO_KEY.
; If a key is ready, AL holds the ASCII code.
; Sets BH=1 for P/p, BH=2 for E/e.
; ============================================================

CHECK_INPUT PROC
    PUSH AX                 ; save AX, for when we need input service
    
    MOV AH, 06h             ; service 06h, direct console input, no blocks
    MOV DL, 0FFh            ; DL = FFh means check for input waiting
    INT 21h                 ; AL = key if pressed, ZF set if no key
    JZ NO_KEY               ; no key waiting, skip
    
    CMP AL, 'E'             ; emergency, uppercase
    JE SET_EMERGENCY    
    CMP AL, 'e'             ; emergency, lowercase
    JE SET_EMERGENCY
    
    CMP AL, 'P'             ; pedestrian, uppercase
    JE SET_PEDESTRIAN
    CMP AL, 'p'             ; pedestrian, lowercase
    JE SET_PEDESTRIAN         
    
    JMP NO_KEY              ; unrecognized key, so ignore 
    
 SET_EMERGENCY:
    MOV BH, 2               ; flag 2=emergency, caller will handle pronto
    JMP NO_KEY              ; flag set, fall through return
    
 SET_PEDESTRIAN:
    MOV BH, 1               ; flag 1=pedestrian, extend next all red
    JMP NO_KEY              ; fall through to return

 NO_KEY:
    POP AX                  ; restore AX before returning   
    
    RET   
    
CHECK_INPUT ENDP  


; ============================================================
; NS_GREEN PROC
; NS = Green, EW = Red
; Bits ON: 2(NS Green E) + 8(NS Green W) + 3(EW Red N) + 9(EW Red S)
; = 4 + 256 + 8 + 512 = 780 = 030Ch
; ============================================================  

NS_GREEN PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as LOOP counter
        
    MOV AX, 030Ch       ; NS green + EW red combined values
    OUT 4, AX           ; send to traffic light port
        
    MOV CX, 4000        ; green duration, full timing
        
 NS_G_LOOP:   
    LOOP NS_G_LOOP      ; no, continue timing loop
      
 NS_G_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX 
       
    RET 
    
NS_GREEN ENDP 


; ============================================================
; NS_YELLOW PROC
; NS = Yellow, EW = Red
; Bits ON: 1(NS Yel E) + 7(NS Yel W) + 3(EW Red N) + 9(EW Red S)
; = 2 + 128 + 8 + 512 = 650 = 028Ah
; ============================================================


NS_YELLOW PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as LOOP counter
    
    MOV AX, 028Ah       ; NS yellow + EW red combined values
    OUT 4, AX           ; send to traffic light port
    
    MOV CX, 2000        ; yellow duration is half of green
    
 NS_Y_LOOP: 
    LOOP NS_Y_LOOP      ; no, continue timing loop
    
 NS_Y_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX
    
    RET  
    
NS_YELLOW ENDP


;===============================================
;ALL_RED PROC
; Sets all lights to RED, safe state.
; Bits ON: 0(E Red) + 6(W Red) + 3(N Red) + 9(S Red)
; = 1 + 64 + 8 + 512 = 585 = 0249h 
;================================================

ALL_RED PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as LOOP counter
    
    MOV AX, 0249h       ; NS red + EW red all lights red
    OUT 4, AX           ; send to traffic light port 
    
    MOV CX, 1000        ; normal transition duration
  
 AR_LOOP:
    LOOP AR_LOOP        ; no continue timing loop
 
 AR_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX 
    
    RET  
    
ALL_RED ENDP   


;===============================================
;EW_GREEN PROC
; Sets East-West green, North-South stays red.
; Bits ON: 5(EW Green N) + 11(EW Green S) + 0(NS Red E) + 6(NS Red W)
; = 32 + 2048 + 1 + 64 = 2145 = 0861h
;===============================================

EW_GREEN PROC 
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used in loop counter
    
    MOV AX, 0861h       ; EW green + NS red combined value
    OUT 4, AX           ; send to traffic light port
    
    MOV CX, 4000        ; green duration, full timing 
    
 EW_G_LOOP:
    LOOP EW_G_LOOP      ; no, continue timing loop
    
 EW_G_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX
    
    RET   
    
EW_GREEN ENDP
    

;================================================
;EW_YELLOW PROC
; Sets East-West to yellow, North-South stays red.
; Bits ON: 4(EW Yel N) + 10(EW Yel S) + 0(NS Red E) + 6(NS Red W)
; = 16 + 1024 + 1 + 64 = 1105 = 0451h
; Half duration of green, is a warning phase only.
;================================================

EW_YELLOW PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as loop counter

    MOV AX, 0451h       ; EW yellow + NS red combined values
    OUT 4, AX           ; send to traffic light port  
    
    MOV CX, 2000        ; yellow duration is half of green
 
 EW_Y_LOOP:
    LOOP EW_Y_LOOP      ; no continue the loop
 
 EW_Y_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX
    
    RET
 
EW_YELLOW ENDP  


; ============================================================
; PEDESTRIAN_MODE PROC
; Realistic crosswalk: NS traffic stops, EW traffic continues.
; Only TWO lights go red (NS-facing: East + West).
; EW traffic keeps moving — same state as EW_GREEN phase.
; Hex = 0861h (NS Red + EW Green), held for extended duration.
; ============================================================

PEDESTRIAN_MODE PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as loop counter 
    
    MOV AX, 0249h       ; brief ALL RED flash, visual que
    OUT 4, AX
    MOV CX, 1000        ; just long enough to see it
    PM_FLASH:
        LOOP PM_FLASH   
    
    MOV AX, 0861h       ;NS=Red, EW=Green, 2 lights red only
    OUT 4, AX    
    MOV CX, 8000        ;extended crossing duration (~3t for demo)
    
  PM_LOOP:
    LOOP PM_LOOP
    
  PM_DONE:
    POP CX              ; restore CX
    POP AX              ; restore AX
                        
    RET
    
PEDESTRIAN_MODE ENDP    
   
   
;===================================================
;EMERGENCY_MODE PROC
; Called from the MAIN_LOOP when BH=2.
; Forces ALL lights to RED immediately.
; Regardless of what phase is currently running.
; Holds for extended duration.
; Emergency vehicle needs time to clear intersection.
; Clears emergency flag before returning.
; Returns to top of MAIN_LOOP, which starts fresh cycle.
;====================================================

EMERGENCY_MODE PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as loop counter

    MOV AX, 0249h       ; ALL RED is safest state for emergency
    OUT 4, AX           ; send to traffic light port pronto
    
    MOV CX, 10000       ; extended hold, vehicle needs time to pass
 
 E_LOOP:
    LOOP E_LOOP         ; no input check during emergency, when triggered
                            ; it will run for its full duration 
 
 E_DONE:    
    POP CX              ; restore CX
    POP AX              ; restore AX  
    
    RET
EMERGENCY_MODE ENDP


END                     ; end program
```