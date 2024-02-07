 ;*****************************************************************************
 ; Universidad del Valle de Guatemala
 ; IE2023: Programacion de Microcontroladores
 ; Autor: Pablo Garcia
 ; Proyecto: Laboratorio 1
 ; Archivo: Laboratorio1.asm
 ; Hardware: ATMEGA328P
 ; Created: 30/01/2024
 ;*****************************************************************************
 ; ENCABEZADO
 ;*****************************************************************************
 .include "M328PDEF.inc"
 .cseg
 .org 0x0000

 ;*****************************************************************************
 ; STACK
 ;*****************************************************************************
	LDI		R16, LOW(RAMEND)
	OUT		SPL, R16
	LDI		R16, HIGH(RAMEND)
	OUT		SPH, R16

 ;*****************************************************************************
 ; SETUP
 ;*****************************************************************************
 SETUP:
 ; CLOCK
	LDI		R16, (1 << CLKPCE)
	STS		CLKPR, R16		// Enable changeS of the CLKPS bits
	LDI		R16, 0x04
	STS		CLKPR, R16		// Preescaler divider: 16	

 ; INPUTS
 ; Counter 1: [+A0,-A1], Counter 2: [+A2,-A3], Sum: A4
	LDI		R16, 0x20
	OUT		DDRC, R16		// PORTC pins as input
	NOP
	LDI		R16, 0x1F		
	OUT		PORTC, R16		// PORTC[PC4:PC0] pins with pull-up
	NOP
	LDI		R19, 0x1F		// Save State[PC4:PC0] of input pins

 ; OUTPUTS	Carry: D13
 ; Counter 1: [D12,D11,D10,D9], Counter 2: [D8,D7,D6,D5], Sum: [D4,D3,D2,A5]
	LDI		R16, 0x3F
	OUT		DDRB, R16		// PORTB[PB5:PB0] pins as output
	NOP
	LDI		R16, 0xFC
	OUT		DDRD, R16		// PORTD[PD7:PD2] pins as output
	NOP
	LDI		R16, 0x00
	OUT		PORTB, R16		// Clear PORTB
	OUT		PORTD, R16		// Clear PORTD
	CBI		PORTC, PC5		// Clear PC5
	NOP

 ;*****************************************************************************
 ; LOOP
 ;*****************************************************************************
 LOOP:
	IN		R20, PINC		// Read PINC and load it in R20

	SBRS	R20, PC0		// If PINC[PC0] is 0 then UPDATE
	RJMP	UPDATE

	SBRS	R20, PC1		// else if PINC[PC1] is 0 then UPDATE
	RJMP	UPDATE

	SBRS	R20, PC2		// else if PINC[PC2] is 0 then UPDATE
	RJMP	UPDATE

	SBRS	R20, PC3		// else if PINC[PC3] is 0 then UPDATE
	RJMP	UPDATE

	SBRS	R20, PC4		// else if PINC[PC4] is 0 then UPDATE
	RJMP	UPDATE

	CP		R20, R19
	BREQ	LOOP			// If nothing has changed then LOOP
	RJMP	RETENTION		// else RETENTION
;*****************************************************************************
UPDATE:
	IN		R20, PINC		// Read PINC and load it in R20

	SBRS	R20, PC0		// If PINC[PC0] == 0 then
	IN		R19, (0<<PC0)	// update State[PC0] of input pins

	SBRS	R20, PC1		// else if PINC[PC1] == 0 then
	IN		R19, (0<<PC1)	// update State[PC1] of input pins

	SBRS	R20, PC2		// else if PINC[PC2] == 0 then
	IN		R19, (0<<PC2)	// update State[PC2] of input pins

	SBRS	R20, PC3		// else if PINC[PC3] == 0 then
	IN		R19, (0<<PC3)	// update State[PC3] of input pins

	SBRS	R20, PC4		// else if PINC[PC4] == 0 then
	IN		R19, (0<<PC4)	// update State[PC4] of input pins

	RJMP	LOOP			// anyway return to loop

RETENTION:
	IN		R20, PINC		// Read PINC and load it in R20
	MOV		R16, R20
	MOV		R17, R19		; Using registers as operands
	COM		R17
	AND		R16, R17		; PINC[PCn] && ~State[PCn]

	SBRC	R16, PC0		// If PINC[PCO] is 1 and State[PCO] is 0
	RJMP	INCCOUNT1		// then INCCOUNT1, maybe CALL

	SBRC	R16, PC1		// else if PINC[PC1] is 1 and State[PC1] is 0
	RJMP	DECCOUNT1		// then DECCOUNT1, maybe CALL

	SBRC	R16, PC2		// else if PINC[PC2] is 1 and State[PC2] is 0
	RJMP	INCCOUNT2		// then INCCOUNT2, maybe CALL

	SBRC	R16, PC3		// else if PINC[PC3] is 1 and State[PC3] is 0
	RJMP	DECCOUNT2		// then DECCOUNT2, maybe CALL

	SBRC	R16, PC4		// else if PINC[PC4] is 1 and State[PC4] is 0
	RJMP	SUMCOUNTS		// then SUMCOUNTS, maybe CALL

	RJMP	LOOP			// else return to loop
;*****************************************************************************
; Counter 1 register: R21
INCCOUNT1:
	IN		R19, (1<<PC0)	// Update State[PC0] of input pins
	INC		R21
	SBRC	R21, 4			; Upper limit
	CLR		R21
	RJMP	COUNTER1		// Refresh counter 1 outputs

DECCOUNT1:
	IN		R19, (1<<PC1)	// Update State[PC1] of input pin
	DEC		R21
	SBRC	R21, 4			; Lower limit
	LDI		R21, 0x0F
	RJMP	COUNTER1		// Refresh counter 1 outputs
;*****************************************************************************
; Counter 2 register: R22
INCCOUNT2:
	IN		R19, (1<<PC2)	// Update State[PC2] of input pins
	INC		R22
	SBRC	R22, 4			; Upper limit
	CLR		R22
	RJMP	COUNTER2		// Refresh counter 1 outputs

DECCOUNT2:
	IN		R19, (1<<PC3)	// Update State[PC3] of input pin
	DEC		R22
	SBRC	R22, 4			; Lower limit
	LDI		R22, 0x0F
	RJMP	COUNTER2		// Refresh counter 1 outputs

; Moving R21[3:0] register to [PB4:PB1] output
COUNTER1:
	SBRC	R21, 3
	SBI		PORTB, PB4
	SBRS	R21, 3
	CBI		PORTB, PB4
	SBRC	R21, 2
	SBI		PORTB, PB3
	SBRS	R21, 2
	CBI		PORTB, PB3
	SBRC	R21, 1
	SBI		PORTB, PB2
	SBRS	R21, 1
	CBI		PORTB, PB2
	SBRC	R21, 0
	SBI		PORTB, PB1
	SBRS	R21, 0
	CBI		PORTB, PB1
	RJMP	LOOP

; Moving R22[3:0] register to [PB0, PD7:PD5] output
COUNTER2:
	SBRC	R22, 3
	SBI		PORTB, PB0
	SBRS	R22, 3
	CBI		PORTB, PB0
	SBRC	R22, 2
	SBI		PORTD, PD7
	SBRS	R22, 2
	CBI		PORTD, PD7
	SBRC	R22, 1
	SBI		PORTD, PD6
	SBRS	R22, 1
	CBI		PORTD, PD6
	SBRC	R22, 0
	SBI		PORTD, PD5
	SBRS	R22, 0
	CBI		PORTD, PD5
	RJMP	LOOP
;*****************************************************************************
SUMCOUNTS:
	IN		R19, (1<<PC4)	// Update State[PC4] of input pins
	MOV		R18, R21		// Copy counter 1 in R18
	ADD		R18, R22		// Sum counter 1 with counter 2
	SBRC	R18, 4
	SBI		PORTB, PB5
	SBRS	R18, 4
	CBI		PORTB, PB5

; Moving R17[3:0] register to [PD4:PD2, PC5] output

	SBRC	R18, 3
	SBI		PORTD, PD4
	SBRS	R18, 3
	CBI		PORTD, PD4
	SBRC	R18, 2
	SBI		PORTD, PD3
	SBRS	R18, 2
	CBI		PORTD, PD3
	SBRC	R18, 1
	SBI		PORTD, PD2
	SBRS	R18, 1
	CBI		PORTD, PD2
	SBRC	R18, 0
	SBI		PORTC, PC5
	SBRS	R18, 0
	CBI		PORTC, PC5

	RJMP	LOOP
;*****************************************************************************
  


