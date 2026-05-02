
``
```

ORG 100h

MOV BH, 0 ; clear pedestrian flag, no crossing requested yet

MAIN PROC
        MAIN_LOOP:
            CALL NS_GREEN   ; run North-South green phase 
            CALL CHECK_INPUT ; check once after phase
            CMP BH, 2       ; check for emergency? 
            JE HANDLE_EMERGENCY ; yes, go to emergency procedure
            
            CALL NS_YELLOW  ; run North-South yellow phase 
            CALL CHECK_INPUT ; check once
            CMP BH, 2       ; check for emergency? 
            JE HANDLE_EMERGENCY ; yes, go to emergency procedure
            
            CALL ALL_RED    ; transition and check pedestrian flag
            CALL CHECK_INPUT ; check once 
            CMP BH, 2       ; check for emergency?
            JE HANDLE_EMERGENCY ; yes, go to emergency procedure
            
            CALL EW_GREEN   ; run East-West green phase
            CALL CHECK_INPUT ; check once
            CMP BH, 2       ; check for emergency? 
            JE HANDLE_EMERGENCY ; yes, go to emergency procedure
            
            CALL EW_YELLOW  ; run East-West yellow phase
            CALL CHECK_INPUT ; check once
            CMP BH, 2       ; check for emergency? 
            JE HANDLE_EMERGENCY ; yes, go to emergency
            
            CALL ALL_RED    ; transition and check pedestrian flag again
            CALL CHECK_INPUT ;check once
            CMP BH, 2       ; check for emergency?
            JE HANDLE_EMERGENCY ;yes, go to emergency procedure
            
        JMP MAIN_LOOP       ; normal cycle complete, repeats forever 
            
        HANDLE_EMERGENCY:
            MOV BH, 0           ; clear emergency flag
            CALL EMERGENCY_MODE ; handle it
            JMP MAIN_LOOP       ; restart fresh cycle
MAIN ENDP 


;=============================================
;CHECK_INPUT PROC
; Called between phases in MAIN_LOOP. 
; This will use a block input.
; It does NOT jump mid-procedure.
; But rather:
; Sets BH=1 if a pedestrian key is pressed.      
; or Sets BH=2 if an emergency key is pressed. 
; Caller checks BH after the LOOP completes. 
;============================================== 

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


;=============================================== 
;NS_GREEN PROC
; Sets North-South to green, East-West to red.
; Combined bit value: 030Ch (NS green bits are 8+2, 
; EW red bits 3+9).
;===============================================   

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


;================================================
;NS_YELLOW PROC
; Sets North-South to yellow, East-West stays red.
; Combined bit value: 028Ah (NS yellow bits 7+1, 
; EW red bits 3+9).
; Half the duration for green 
;================================================

NS_YELLOW PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as LOOP counter
    
    MOV AX, 028Ah         ; NS yellow + EW red combined values
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
; TRansition state between NS and EW phases.
; Sets all lights to RED, safe state.
; Combined bit value: 0249h (NS red + EW red).
; If pedestrian flag set (BH=1), extends hold
; for crossing duration then it clears the flag.
;================================================

ALL_RED PROC
    PUSH AX             ; save AX, used for OUT instruction
    PUSH CX             ; save CX, used as LOOP counter
    
    MOV AX, 0249h       ; NS red + EW red all lights red
    OUT 4, AX           ; send to traffic light port 
    CMP BH, 1           ; pedestrian flag set?
    JE EXTENDED_RED     ; yes, extend light for crossing
    
    MOV CX, 1000        ; normal transition duration
    JMP AR_LOOP         ; skip extended timing
 
 EXTENDED_RED:
    MOV CX, 6000        ; extended duration for ped crossing
    MOV BH, 0           ; clears ped flag/crossing handled
 
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
; Combined bit value: 0861h (EW green bits 5+B, 
; NS red bits 6+0)
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
; Combined bit value 0451h (EW yellow bits 4+A,
; NS red bits 6+0).
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
    
    MOV BH, 0           ; clear emergency flag before the loop same as ped
    
    MOV CX, 10000       ; extended hold, vehicle needs time to pass
 
 E_LOOP:
    LOOP E_LOOP         ; no input check during emergency, when triggered
                            ; it will run for its full duration
    
    POP CX              ; restore CX
    POP AX              ; restore AX 
    RET
EMERGENCY_MODE ENDP


END                     ; end program
```