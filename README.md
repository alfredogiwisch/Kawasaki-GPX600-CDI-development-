# Kawasaki-GPX600-CDI-development-

CDI development with programable advance spark curve

Digital electronic ignition units are commonly in use in cars and motorcycles. The most important feature of a digital igniter is the processor accuracy on timing calculations, the programmable spark advance curve and the over RPMs cutoff for engine protection.

There are two main types of electronic igniter's on the market: TCI (transistor conmuted ignition) and CDI (capacitive discharge ignition). In TCI ignition a low voltage is conmuted (typically the 12V battery voltage) and applied to the spark coil at ignition time. In the case of CDI igniters a capacitor is charged at high voltage using a DC-DC converter stage (typically 200 to 400 volts) and discharged at ignition time. CDI igniters are much better in performance because coil high output voltage remains at high RPMs but one disadvantage is the need of an additional DC-DC inverter stage. 

I have build the CDI igniter unit in assembler language for my old Kawasaki (GPX600) sportbike because the factory igniter unit started to fail. In the picture above the external black transformer forms part of the low frequency DC-DC inverter stage that increase the voltage from 12 volts to approximately 300 volts. For testing purposes and initial development phase it worked as expected using a transformer but the selected power rate is to small to sustain the spark energy at high RPMs. To reach a better spark and improve the efficiency the right choice on next design would be to implement a high frequency switching DC-DC converter stage. Anyway the CDI project worked well on my Kawasaki GPX600 bike !!!

The entire project was developed in assembler language for 16F84 microcontrollers. The advance timing table is user programmable and follows GPX600 manufacturer specifications: 10º before TDC at 1000 to 40º before TDC at 4000 rpm. Over RPMs spark cut-off is set at 12500 rpm. For the RPM instruments panel a tachometer output is available.

Before running the CDI unit on the bike I tested the circuit board externally measuring the advance timings with an oscilloscope to avoid damages in case of failures.  A common CDI failure is the triggering of the spark out of ignition time and such events can lead to mechanical failures. Testing phase is very important and more on the development of a CDI unit.

Below is the assembler code used on the project. Please be aware that I don't have maintained this program after Nov 2004, so in case of any damages on the engine is your own responsibility. The advance timing table is calculated for a 8Mhz oscillator following original Kawasaki GPX600 values (10° at 1000 RPMs to 40° at 4000).
----------------------------------------------------------------------------------------------------------------------------------

;==========================================================================
;   Kawasaki GPX 600 CDI igniter unit project
;   
;   Final release 11/23/2004 for 16F84 microcontroller 
;==========================================================================
;Advance timing values:
;    10° before TDC at 1000 RPMs
;    40° before TDC at 4000 RPMs
;    Spark over-rev cutoff at 12500 RPMs
;    8 Mhz master clock frequency
;==========================================================================

;==========================================================================
;
;       Directives
;
;==========================================================================
        list p=16f84a        ; processor type
        ;
        __config  B'11111111110010'
        ; No code protect, PWRT on, no watchdog, Quartz 4to20Mhz
        ;
        ; Page 0: 0-FF page1: 100-1FF page2: 200-2FF page3: 300-3FF
;==========================================================================
;
;       Register Definitions
;
;==========================================================================

W                            EQU     H'0000'
F                            EQU     H'0001'

;----- Register Files------------------------------------------------------

INDF                         EQU     H'0000'
TMR0                         EQU     H'0001'
PCL                          EQU     H'0002'
STATUS                       EQU     H'0003'
FSR                          EQU     H'0004'
PORTA                        EQU     H'0005'
PORTB                        EQU     H'0006'
EEDATA                       EQU     H'0008'
EEADR                        EQU     H'0009'
PCLATH                       EQU     H'000A'
INTCON                       EQU     H'000B'

OPTION_REG                   EQU     H'0081'
TRISA                        EQU     H'0085'
TRISB                        EQU     H'0086'
EECON1                       EQU     H'0088'
EECON2                       EQU     H'0089'

;----- STATUS Bits --------------------------------------------------------

IRP                          EQU     H'0007'
RP1                          EQU     H'0006'
RP0                          EQU     H'0005'
NOT_TO                       EQU     H'0004'
NOT_PD                       EQU     H'0003'
Z                            EQU     H'0002'
DC                           EQU     H'0001'
C                            EQU     H'0000'

;----- INTCON Bits --------------------------------------------------------

GIE                          EQU     H'0007'
EEIE                         EQU     H'0006'
T0IE                         EQU     H'0005'
INTE                         EQU     H'0004'
RBIE                         EQU     H'0003'
T0IF                         EQU     H'0002'
INTF                         EQU     H'0001'
RBIF                         EQU     H'0000'

;----- OPTION Bits --------------------------------------------------------

NOT_RBPU                     EQU     H'0007'
INTEDG                       EQU     H'0006'
T0CS                         EQU     H'0005'
T0SE                         EQU     H'0004'
PSA                          EQU     H'0003'
PS2                          EQU     H'0002'
PS1                          EQU     H'0001'
PS0                          EQU     H'0000'

;==========================================================================
;
;       Define
;
;==========================================================================

#DEFINE rotor14    PORTA,2     ; [pin1]= sensor 1-4
#DEFINE rotor23    PORTA,3     ; [pin2]= sensor 2-3
#DEFINE rpmmax    PORTB,1     ; [pin7]= Min and Max RPM led
#DEFINE led    PORTB,7     ; [pin13]= pickup led

;==========================================================================
;
;       RAM Definition
;
;==========================================================================

rpmhi    equ 0Ch        ; RPM counter register high
rpmlo    equ 0Dh        ; RPM counter register low
dwell    equ 0Eh       ; dwell time counter
rtdset    equ 0Fh      ; retard value holding register
math    equ 11h       ; calculation register
coilflg    equ 12h        ; coil status
spkflg    equ 13h      ; flag after one spark
rotflg    equ 14h      ; rotor sense flag
retard    equ 15h      ; new retard value
coilor    equ 16h         ; coil order flag

;==========================================================================
;
;       Main Program
;
;==========================================================================


    org 0            ; start adress 0
start   bsf    STATUS,RP0      ; go to bank 1
        movlw  H'FF'           ; set all portA..
        movlw  TRISA        ; ..as inputs
        movlw  H'00'         ; set all portB..
        movwf  TRISB        ; ..as outputs
    ;                   
        movlw b'11000001'    ; no pull-up,rising edge int,internal timer,1:4
        movwf OPTION_REG
        bcf  STATUS,RP0        ; come back into bank 0
        ;
        clrw                   ; clear all variables
        clrf PORTA             ; clear variables
        clrf PORTB
        clrf math
        clrf rpmlo
        clrf coilflg
    clrf coilor   
        movlw 0xff             ; rpmhi = FF initially
        movwf rpmhi
        movwf spkflg        ; no spark until pickup is high
        movwf rotflg        ; and no spark loop if pickup stay up
    movwf retard        ; retard value = 25ms initially
    ;
        ;
        ;       100uS loop time
        ;       4MHz clock / 4 = 1MHz instruction cycle
        ;       cycle time = 1/1MHz  =  100uS
        ;       100uS/100uS  =  100 cycles
        ;       prescaler set to divide by 4
        ;       preset TMR0 = 231, when = 0  =  100uS
        ;       231 to 255 = 25 x 4(prescaler) = 100
        ;
        movlw .231    ;movlw .206 for a 8MHz clock
        movwf TMR0
        ;
        ;
mnloop  movf TMR0,W
        btfss STATUS,Z        ; is TMR0=0 ?
        goto mnloop         ; no, main delay loop
        ;
        movlw .231        ; yes, re-preset TMR0 (240max)
        movwf TMR0
        ;
    incf rpmlo,F        ; increment rpmlo
        btfsc STATUS,Z      ; is rpmlo = 0 ?
        incf rpmhi,F        ; yes, increment rpmhi
        ;
    ;       if rpm count > 1400h, 5120 x 100us=0.5 Secs, then it is assumed that the
        ;       motor has stopped. Reinitialize system.
        movlw 0x14
        subwf rpmhi,W
        btfsc STATUS,Z         ; rpmhi - 14 = 0 ?
        goto start             ; yes
        ;
       
        btfss rotor14        ; + is rotor high or low?
    goto rot23
        movlw B'00000100'
    movwf coilor
        goto rotchk
rot23    btfss rotor23       
        goto rnowlw             ; low
        movlw B'00001000'
        movwf coilor
        ;
rotchk  movf rotflg,W       ; high, is it flagged high also? rotflg=FF?
        btfss STATUS,Z
        goto coilck         ; yes
        ;
        movlw 0xff         ; no
        movwf rotflg         ; flag rotor as up
        bsf led            ; turn ignition pickup led on
    clrf spkflg              ; reset flag for one spark
        goto rpmcalc        ; calculate RPM
    ;
   
rnowlw  movf rotflg,W       ; is it flagged low already ? rotflg=0?
        btfsc STATUS,Z
        goto coilck             ; yes, wasn't flagged high
        ;
       
dolow   bcf led               ; no, turn ignition pickup led off
        clrf rotflg         ; flag rotor as down
        goto mnloop
        ;

decret    decf retard,F        ; no, decrement retard then wait
    goto mnloop
    ;
                   
coilck  movf coilflg,W           ; is coil flagged high? coilflg=FF?
        btfss STATUS,Z
        goto upcoil             ; yes, then decrement dwell count
        movf retard,W        ; is retard = 0 ?
        btfss STATUS,Z
        goto decret        ; no, then decrement retard
        ;
    movf spkflg,W           ; yes, a spark have been done? spkflg=FF?
        btfss STATUS,Z
        goto mnloop             ; yes, no more spark
        ;
    movf coilor,W
    btfss PORTB,1       
        movwf PORTB        ; no, go on for the first spark
        bcf PORTB,1
        movlw 0xff          ; set flag for high coil
        movwf coilflg
        movwf spkflg        ; set flag for one spark
   
        ;   ****************** YOU CAN MODIFY THE DWELL VALUE BELLOW ********
        movlw .10        ; 10 X 100uS  = 1mS
        movwf dwell        ; set dwell time
        ;
        goto mnloop
    ;
   
upcoil  ;               if dwell # 0 then dwell - 1
        ;            if dwell now = 0, turn on coil
        ;
        decfsz dwell,F      ; dwell = dwell - 1
        goto mnloop         ; dwell <> 0, wait
    bcf PORTB,.2        ; dwell = 0, turn coil off
        bcf PORTB,.3
        ;bsf revcoil          ; dwell = 0, turn revcoil on
        clrf coilflg              ; set flag for low coil
        clrf coilor
        goto mnloop
        ;

rpmcalc ;       engine RPM <= 1000 at maximum retard, = 60ms
    ;   
        ;       60mS = 600 X 100us loops, = 258Ch
        ;       ( 1000RPM for a 1cyl - 4stroke with one spark loose )
        ;       ( 1000RPM for a 1cyl - 2stroke )
        ;       (  500RPM for a 4cyl - 4stroke = 16Hz )
        ;
        ;    engine RPM >= 8000 at maximum advance, = 7.5mS
        ;       4.8= 48 X 100uS loops, = 30h
    ;    4.8ms = 50 x 100us = 37h
        ;       ( 8000RPM for a 1cyl - 4stroke with one spark loose )
        ;       ( 8000RPM for a 1cyl - 2stroke )
        ;       (  4000RPM for a 4cyl - 4stroke = 133Hz )
        ;
        ;       this routine determines whether the engine RPM
        ;       value is below 75 loop counts - max advance, or above
        ;       600 loop counts - max retard, or some where in between.
        ;      
        ;      
        ; high RPM:         ; > 8000RPM
        ;
        movf rpmhi,W
        btfss STATUS,Z      ; is rpmhi = 0?
        goto bigfv          ; no
        ;
        movf rpmlo,W
        xorlw 0x30              ; yes, is rpmlo = 4Bh ?
        btfsc STATUS,Z
        goto maxadv         ; yes, do max advance
        ;
        movlw 0x30
        subwf rpmlo,W       ; no, is rpmlo < 4Bh ?
        btfss STATUS,C
        goto maxadv         ; yes, do max advance
        ;
        ; medium RPM:
        goto caladv         ; no, calculate new advance
        ;
        ;
        ; low RPM:          ; < 1000RPM
bigfv   movf rpmhi,W
        xorlw .2            ; is rpmhi = 2 ?
        btfsc STATUS,Z
        goto resfv          ; yes, see if rpmlo = 58h
        ;
        movlw .2
        subwf rpmhi,W       ; no, is high byte > 2 ?
        btfsc STATUS,C
        goto maxret         ; yes, do max retard
        ;
        goto caladv         ; no, calculate new advance
        ;
resfv   movf rpmlo,W
        xorlw 0x58          ; rpmhi = 2, does rpmlo = 58h ?
        btfsc STATUS,Z
        goto maxret         ; yes, do max retard
        ;
        movlw 0x58
        subwf rpmlo,W       ; no, is rpmlo > 58h ?
        btfsc STATUS,C
        goto maxret         ; yes, do max retard
        ;
        ;
        ; the formula to get the data stored in the map is as follows
        ;
        ;                 rpmhi/lo count  -  48
        ;    138   -    ---------------------------
        ;                           4
        ;
caladv  movlw d'48'        ; rpmhi/lo - 48
        subwf rpmlo,F        ; rpmlo = rpmlo - 48
        btfss STATUS,C
        decf rpmhi,F
        ;
        ; divide result by 4
        ;
        bcf STATUS,C
        rrf rpmhi,F
        rrf rpmlo,F         ; / 2
        ;
        bcf STATUS,C
        rrf rpmhi,F
        rrf rpmlo,F         ; / 4
        ;
        movlw .138          ; 138 entries in map list
        movwf math
        ;
        movf rpmlo,W
        subwf math,W        ; W = 138 - result
        ;
        bcf PCLATH,0        ; be sure to go h'200'
        bsf PCLATH,1        ; where is the map.
        call map        ; read map
        movwf rtdset        ; come back with new retard value
        ;
rpmset  clrf rpmhi        ; clear RPM counters
        clrf rpmlo
    movf rtdset,W        ; transfert advance value
        movwf retard
        goto mnloop
        ;
maxret  movlw 0x64
        movwf rtdset         ; retard value = 32h (5ms)   258=600=16hz
        bcf rpmmax         ; turn off maxrpm led
        goto rpmset
        ;
maxadv  clrf rtdset          ; retard value = 0ms    4B=75=124hz
        bsf rpmmax         ; turn on maxrpm led
        goto rpmset
        ;
map     org h'200'        ; store map at 200h
    addwf PCL,1        ; add W + PCL
    ;  *********** INSERT YOUR OWN VALUES HERE *******************
        retlw 64h     ;5,00ms   1000rpm 600L  
        retlw 64h     ;4,98ms   1003rpm 598L
        retlw 63h     ;4,93ms   1010rpm 594L
        retlw 62h     ;4,89ms   1016rpm 590L
        retlw 61h     ;4,85ms   1023rpm 586L
        retlw 60h     ;4,80ms   1030rpm 580L
        retlw 5Fh     ;4,75ms   1038rpm 576L
    retlw 5Eh     ;4,71ms    1045rpm 574L
        retlw 5Dh     ;4,67ms   1052rpm 570L
    retlw 5Ch     ;4,62ms   1060rpm 566L       
        retlw 5Ch     ;4,58ms   1067rpm 562L
        retlw 5Bh     ;4,53ms   1075rpm 558L
        retlw 5Ah     ;4,48ms   1083rpm 554L
        retlw 5Ah     ;4,44ms   1090rpm 550L
        retlw 59h     ;4,40ms   1098rpm 546L
        retlw 58h     ;4,35ms   1107rpm 542L
        retlw 57h     ;4,31ms   1115rpm 538L
        retlw 56h     ;4,26ms   1123rpm 534L
        retlw 55h     ;4.22ms   1132rpm 530L
        retlw 54h     ;4,18ms   1140rpm 526L
        retlw 53h     ;4,13ms   1149rpm 522L
        retlw 52h     ;4,09ms   1158rpm 518L
        retlw 51h     ;4,04ms   1167rpm 514L
        retlw 50h     ;4,00ms   1176rpm 510L
        retlw 4Fh     ;3,95ms   1185rpm 506L
        retlw 4Eh     ;3,91ms   1195rpm 502L
        retlw 4Dh     ;3,86ms   1204rpm 498L
        retlw 4Ch     ;3,82ms   1214rpm 494L
        retlw 4Bh     ;3,77ms   1224rpm 490L
        retlw 4Bh     ;3,73ms   1234rpm 486L
        retlw 4Ah     ;3,68ms   1244rpm 482L
        retlw 49h     ;3,65ms   1255rpm 478L
        retlw 48h     ;3,60ms   1265rpm 474L
        retlw 47h     ;3,55ms   1276rpm 470L
        retlw 46h     ;3,51ms   1287rpm 466L
        retlw 45h     ;3,46ms   1298rpm 462L
        retlw 44h     ;3,42ms   1310rpm 458L
        retlw 44h     ;3,38ms   1321rpm 454L
        retlw 43h     ;3,33ms   1333rpm 450L
        retlw 42h     ;3,28ms   1345rpm 446L
        retlw 41h     ;3,25ms   1357rpm 442L
        retlw 40h     ;3,20ms   1369rpm 438L
        retlw 3Fh     ;3,15ms   1382rpm 434L
        retlw 3Eh     ;3,10ms   1395rpm 430L
        retlw 3Dh     ;3,05ms   1408rpm 426L
        retlw 3Ch     ;3,02ms   1421rpm 422L
        retlw 3Ch     ;2,98ms   1435rpm 418L
        retlw 3Bh     ;2,93ms   1449rpm 414L
        retlw 3Ah     ;2,89ms   1463rpm 410L
        retlw 39h     ;2,84ms   1477rpm 406L
        retlw 38h     ;2,79ms   1492rpm 402L
        retlw 37h     ;2,75ms   1507rpm 398L
        retlw 36h     ;2,71ms   1522rpm 394L
        retlw 35h     ;2,66ms   1538rpm 390L
        retlw 34h     ;2,62ms   1554rpm 386L
        retlw 33h     ;2,57ms   1570rpm 382L
        retlw 33h     ;2,53ms   1587rpm 378L
        retlw 32h     ;2,49ms   1604rpm 374L
        retlw 31h     ;2,44ms   1621rpm 370L
        retlw 30h     ;2,40ms   1639rpm 366L
        retlw 2Fh     ;2,35ms   1657rpm 362L
        retlw 2Eh     ;2,31ms   1675rpm 358L
        retlw 2Dh     ;2,26ms   1694rpm 354L
        retlw 2Ch     ;2,22ms   1714rpm 350L
        retlw 2Ch     ;2,18ms   1734rpm 346L
        retlw 2Bh     ;2,13ms   1754rpm 342L
        retlw 2Ah     ;2,09ms   1775rpm 338L
        retlw 29h     ;2,05ms   1796rpm 334L
        retlw 28h     ;2,00ms   1818rpm 330L
        retlw 27h     ;1,95ms   1840rpm 326L
        retlw 26h     ;1,91ms   1863rpm 322L
        retlw 25h     ;1,86ms   1886rpm 318L
        retlw 24h     ;1,82ms   1910rpm 314L
        retlw 23h     ;1,77ms   1935rpm 310L
        retlw 22h     ;1,73ms   1960rpm 306L
        retlw 22h     ;1,69ms   1986rpm 302L
        retlw 21h     ;1,65ms   2013rpm 298L
        retlw 20h     ;1,59ms   2040rpm 294L
        retlw 1Fh     ;1,55ms   2068rpm 292L
        retlw 1Eh     ;1,51ms   2097rpm 286L
        retlw 1Dh     ;1,46ms   2127rpm 282L
        retlw 1Ch     ;1,42ms   2158rpm 278L
        retlw 1Ch     ;1,37ms   2189rpm 274L
        retlw 1Bh     ;1,33ms   2222rpm 270L
        retlw 1Ah     ;1,29ms   2255rpm 266L
        retlw 19h     ;1,25ms   2290rpm 262L
        retlw 18h     ;1,20ms   2325rpm 258L
        retlw 17h     ;1,15ms   2362rpm 254L
        retlw 16h     ;1,11ms   2400rpm 250L
        retlw 15h     ;1,06ms   2439rpm 246L
        retlw 14h     ;1,02ms   2479rpm 242L
        retlw 13h     ;0,97ms   2521rpm 238L
        retlw 12h     ;0,93ms   2564rpm 234L
        retlw 12h     ;0,88ms   2608rpm 230L
        retlw 11h     ;0,84ms   2654rpm 226L
        retlw 10h     ;0,79ms   2702rpm 222L
        retlw 0Fh     ;0,75ms   2752rpm 218L
        retlw 0Eh     ;0,71ms   2803rpm 214L
        retlw 0Dh     ;0,66ms   2857rpm 210L
        retlw 0Ch     ;0,62ms   2912rpm 206L
        retlw 0Bh     ;0,57ms   2970rpm 202L
        retlw 0Bh     ;0,53ms   3030rpm 198L
        retlw 0Ah     ;0,48ms   3092rpm 194L
        retlw 09h     ;0,44ms   3157rpm 190L
        retlw 08h     ;0,39ms   3225rpm 186L
        retlw 07h     ;0,35ms   3296rpm 182L
        retlw 06h     ;0,31ms   3370rpm 178L
        retlw 05h     ;0,26ms   3448rpm 174L
        retlw 04h     ;0,22ms   3529rpm 170L
        retlw 03h     ;0,17ms   3614rpm 166L
        retlw 02h     ;0,13ms   3703rpm 162L
        retlw 02h     ;0,08ms   3797rpm 158L
        retlw 01h     ;0,04ms   3896rpm 154L
        retlw 0h      ;0,00ms   4000rpm 150L
        retlw 0h      ;0,00ms   4109rpm 146L
        retlw 0h      ;0,00ms   4225rpm 142L
        retlw 0h      ;0,00ms   4347rpm 138L
        retlw 0h      ;0,00ms   4477rpm 134L
        retlw 0h      ;0,00ms   4615rpm 130L
        retlw 0h      ;0,00ms   4761rpm 126L
        retlw 0h      ;0,00ms   4918rpm 122L
        retlw 0h      ;0,00ms   5084rpm 118L
        retlw 0h      ;0,00ms   5263rpm 114L
        retlw 0h      ;0,00ms   5454rpm 110L
        retlw 0h      ;0,00ms   5660rpm 106L
        retlw 0h      ;0,00ms   5882rpm 102L
        retlw 0h      ;0,00ms   6122rpm  98L
        retlw 0h      ;0,00ms   6382rpm  94L
        retlw 0h      ;0,00ms   6666rpm  90L
        retlw 0h      ;0,00ms   6976rpm  86L
        retlw 0h      ;0,00ms   7317rpm  82L
        retlw 0h      ;0,00ms   7692rpm  78L
        retlw 0h      ;0,00ms   8108rpm  74L
        retlw 0h      ;0,00ms   8571rpm  70L
        retlw 0h      ;0,00ms   9090rpm  66L
        retlw 0h      ;0,00ms   9677rpm  62L
        retlw 0h      ;0,00ms  10344rpm  58L
        retlw 0h      ;0,00ms  11111rpm  54L
        retlw 0h      ;0,00ms  12000rpm  50L
        retlw 0h      ;0,00ms  12500rpm  48L ignition shut-down for higher rpm
        retlw 0h      ;0,00ms  12500rpm  overlap value
        retlw 0h      ;0,00ms  12500rpm  overlap value
           
        ; *********** END OF YOUR OWN VALUES ******************************
        ; line559
        ;
        ;       reset vector
        ;
        ;
        org 300h        ; if the program is loose,
        ;
        ;
        ;
        goto start        ; It goes back home.
        ;
        ;
        ;
        end
