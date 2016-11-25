;
; display.asm
;
; Created: 23/9/2016 2:16:36 p. m.
; Author : waral
;
	

;para usar el i2c no hay que inicializar nada de los puertos... 
;lo que yo puse fue para encender un led que me diga que esta todo ok

;PB1 entrada, PC0 salida
	.def TEMP = R16
	.def TEMP2=R17
	.def RETARDO = R18
	.def DISPVAR = R19
	.def CONTADOR = R20
	.def TOTAL=R21
	.def IMPRIMO=R22
	.def CONTROL=R23

	.equ TWI_RATE = 0xF8
	.equ STARTi = 0x08
	.equ MT_SLA_ACK = 0x20
	.equ MT_DATA_ACK= 0x28
	.equ SL_ADD = 0b01001110

	.org 0x0
	jmp start

	.org 0x0016
	jmp T1_B_ISR

;para usar el i2c no hay que inicializar nada de los puertos... 
;lo que yo puse fue para encender un led que me diga que esta todo ok
	.org 0x100
start:
	
	
	call InicI2C			; esta funcion inicializa el display, si o si tiene que ir. no hace falta modificarle nada
	call InicDisplay		; lo mismo que la anterior
	Call DisplayCocktail	; Mando "Cocktail" al display
	call DisplayEnter		; Mando "Enter" al display
	call DisplayWelcome		; Mando "Welcome" al display

	;call displayClear
	

	ldi TEMP, HIGH(RAMEND)
	out SPH,TEMP
	ldi R20,LOW(RAMEND)
	out SPL,TEMP

	ldi TEMP,0x00
	out ddrb,TEMP
	out ddrd,TEMP
	sbi ddrc,0
	

start1:	
	cli
	sbis pind,5
	jmp contin
	;rjmp start1
contin:
	sbi portc,0
	ldi TEMP,0                    ;;;;;;;;;;;;;;;;;;,
	sts OCR1AH,TEMP;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
	ldi TEMP,140
	sts OCR1AL,TEMP


	ldi TEMP,0x00
	sts TCCR1A,TEMP
	;ldi TEMP,0b00001001  ; CTC INTERNAL clock
	ldi TEMP,0b00001110  ; CTC External clock
	sts TCCR1B,TEMP
	ldi TEMP,(1<<1)  ; Interrupts enabled, compare match b
	sts TIMSK1, TEMP
	ldi TOTAL,0x01
	ldi CONTROL,0x00
	sei
loop4:
	
	sbrc CONTROL, 0
	jmp ahora
	lds TOTAL,TCNT1L
	call imprimototal
	call retardo50ms
	rjmp loop4
ahora:	 	
	cli
	call DisplayClear
	call DisplayEnter		; Mando "Enter" al display
	call DisplayWelcome		; Mando "Welcome" al display
loop6:
	jmp loop6



L1:
	lds TOTAL,0xf0
	call imprimototal
	in r20,TIFR1
	sbrs r20,OCF1A
	RJMP L1
	ldi r20,1<<OCF1A
	out tifr1,r20

	lds TOTAL,TCNT1L
	call imprimototal
	;ldi r17,0x01
	;lds r16,portc+0x20
	;eor r16,r17
	;sts portc+0x20,r16
	

	rjmp start1


	call I2CStop			; cuando finaliza el programa hay que ponerle stop al i2c
loop:
	rjmp loop

;----------------------------SUBRUTINAS-------------------------------

;________________________________________________--
;				 Caudalimetro
;__________________________________________________


;-----numero a string-----
ImprimoTotal:
		call DisplayClear

		ldi IMPRIMO,8
contimprtot:
		sbrs TOTAL,7
		rjmp CeroTotal
		rjmp UnoTotal

CeroTotal: 
		ldi DISPVAR,'0'
		call DisplayChar
		dec IMPRIMO
		breq finImprimo
		lsL TOTAL
		rjmp contimprtot
UnoTotal:
		ldi DISPVAR,'1'
		call DisplayChar
		dec IMPRIMO
		breq finImprimo
		lsL TOTAL
		rjmp contimprtot
FinImprimo:
ret


;_____________________________________________________
;;;;;;;;;;;;;;;;;DISPLAY;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;|
;_____________________________________________________|

;------- inicializacion i2c-------
InicI2C:	
	
	ldi TEMP, TWI_RATE
	sts TWBR,TEMP

	ldi TEMP, (1<<TWINT)|(1<<TWSTA)|(1<<TWEN)   
	sts TWCR, TEMP								;envia condicion de start

wait1:
	lds TEMP,TWCR
	sbrs TEMP,TWINT								;espera flag de start ok
	rjmp wait1

	lds TEMP,TWSR
	andi TEMP, 0xF8								;si el estado en el registro TWI es distinto de START se va a error
	cpi TEMP, STARTi
	brne error_A
	jmp continuo
error_A:
	call ERROR1
	
continuo:
	ldi TEMP, SL_ADD								
	sts TWDR, TEMP								;Carga direccion del esclavo en el registro TWDR, limpia bit TWINT para empezar la transmision de la direccion
	ldi TEMP, (1<<TWINT) | (1<<TWEN)
	sts TWCR, TEMP								;envio direccion del esclavo

wait2:
	lds TEMP,TWCR
	sbrs TEMP,TWINT								;espera seteo de TWINT para confirmar transmision ok
	rjmp wait2

	lds TEMP,TWSR
	andi TEMP, 0xF8								;chequea el registro TWI, salta a error si no se transmitio bien
	cpi TEMP, MT_SLA_ACK
	breq error_B
	jmp continuo2
error_B:
	call ERROR1
continuo2:
	ret
	
;----------------------------------Fin inicializacion i2c----------------------------------------;

;.................................incialización display, envio de a 4bits........................;
InicDisplay:	
	
	call retardo50ms
	
	ldi TEMP, 0x30
	ldi r17,0x30
	sts TWDR, TEMP								; Carga DATA en twdr, limpia twint para empezar la transmision
	ldi TEMP, (1<<TWINT) |(1<<TWEN)
	sts TWCR, TEMP								
	call WaitDataI2c

	call DisplayEnable

	call retardo5ms

	call DisplayEnable

	call retardo1ms
												; todo esto te lo pide que hagas la hoja de datos del display
	call DisplayEnable

	call retardo5ms

	ldi r16, 0x28								;set 4bit mode
	ldi r17,0x28
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c
	
	call DisplayEnable

	call retardo1ms

	ldi TEMP,0x28								;0x28_H
	ldi r17,0x28
	sts TWDR, TEMP								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, TEMP								
	call WaitDataI2c

	call DisplayEnable

	ldi TEMP,0x88								;0x28_L
	ldi r17,0x88
	sts TWDR, r16								
	ldi TEMP, (1<<TWINT) |(1<<TWEN)
	sts TWCR, TEMP								
	call WaitDataI2c

	call DisplayEnable

	ldi TEMP, 0x08								;0x08_H
	ldi r17,0x08
	sts TWDR, TEMP								
	ldi TEMP, (1<<TWINT) |(1<<TWEN)
	sts TWCR, TEMP								
	call WaitDataI2c

	call DisplayEnable							

	ldi TEMP,0x88								;0x08_L				
	ldi r17,0x88
	
	sts TWDR, TEMP								
	ldi TEMP, (1<<TWINT) |(1<<TWEN)
	sts TWCR, TEMP								
	call WaitDataI2c

	call DisplayEnable

	ldi r16,0x08								;0x01_H	
	ldi r17,0x08
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable

	ldi r16,0x18								;0x01_L	
	ldi r17,0x18
		
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable

	ldi r16,0x08								;0x0F_H		
	ldi r17,0x08

	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable
	
	ldi r16,0xF8								;0x0F_L	
	ldi r17,0xF8
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable

	ldi r16,0x08							;0x06_H
	ldi r17,0x08
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)				
	sts TWCR, r16
	call WaitDataI2c

	call DisplayEnable

	call retardo5ms
	
	ldi r16, 0x68							;0x06_L
	ldi r17, 0x68
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)				
	sts TWCR, r16
	call WaitDataI2c

	call DisplayEnable

	ret

;-----------Fin Inicialización display--------------

;--DISPLAY : DATA I2C OK---;

WaitDataI2c:

wait_twint:
	lds r16,TWCR
	sbrs r16,TWINT								; Espera TWINT para confirmar que se envió ok
	rjmp wait_twint

	lds r16,TWSR
	andi r16, 0xF8
	cpi r16, MT_DATA_ACK
	brne error_data1
	ret
error_data1:
	jmp error_data
;------DISPLAY : CHAR------;                  Con esta Funcion le enviamos un CHAR al display
DisplayChar:

	mov r16,DISPVAR							  ;En DISPVAR tiene que estar el CHAR
	andi r16,0xF0							  ;Envio DISPVAR_H
	ori r16,0x09
	mov r17,r16
	sts TWDR, r16								; Carga DATA en twdr, limpia twint para empezar la transmision
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16

	call WaitDataI2c
	call DisplayEnable

	mov r16,DISPVAR							  ;Envio DISPVAR_L
	lsl r16
	lsl r16
	lsl r16
	lsl r16
	ori r16,0x09
	mov r17,r16
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)				
	sts TWCR, r16
	call WaitDataI2c

	call DisplayEnable

	ret

;------Display :ENTER----------

DisplayEnter:

	ldi r16,0xC8								;0x08_H
	ldi r17,0xC8
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable							

	ldi r16,0x08								;0x08_L				
	ldi r17,0x08
	
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable
ret

;----Display :Clear------
DisplayClear:
	
	ldi r16,0x08								;0x01_H	
	ldi r17,0x08
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable

	ldi r16,0x18								;0x01_L	
	ldi r17,0x18
		
	sts TWDR, r16								
	ldi r16, (1<<TWINT) |(1<<TWEN)
	sts TWCR, r16								
	call WaitDataI2c

	call DisplayEnable
ret
;----DISPLAY : STOP------

I2CStop:
	ldi r16, (1<<TWINT)|(1<<TWEN)| (1<<TWSTO)	;Transmite bit de STOP
	sts TWCR, r16
			ret

error_data:
	call ERROR1




retardo10us:
	push RETARDO
	ldi RETARDO,58
loop_ret_10:
	dec RETARDO
	brne loop_ret_10
	pop RETARDO
	ret

retardo1ms:
	push RETARDO
	ldi RETARDO,100
loop_ret_1m:
	call retardo10us
	dec RETARDO
	
	brne loop_ret_1m
	pop RETARDO
	ret

retardo50ms:
	push RETARDO
	ldi RETARDO,50
loop_ret_50m:
	call retardo1ms
	dec RETARDO
	brne loop_ret_50m
	pop RETARDO
	ret

retardo5ms:
	ldi RETARDO,5
loop_ret_5m:
	call retardo1ms
	dec RETARDO
	brne loop_ret_5m
	ret

retardo3s:

	ldi RETARDO, 60
loop_ret_3s:
	call retardo50ms
	dec RETARDO
	brne loop_ret_3s
	ret

DisplayEnable:
	call retardo1ms
	
	ori r17, 0x04
	sts TWDR, r17								
	ldi r16, (1<<TWINT) |(1<<TWEN)				
	sts TWCR, r16
	call WaitDataI2c
;	call retardo1ms

	andi r17, 0b11111011
	sts TWDR, r17								
	ldi r16, (1<<TWINT) |(1<<TWEN)				
	sts TWCR, r16
	call WaitDataI2c

	;call retardo1ms
	ret


;-----------Error----------------------------------

ERROR:
	
	;ldi R16,0x02
	;out PORTB,R16
	rjmp error

ERROR1:
	
	;ldi R16,0x02
	;out PORTB,R16
	ldi r16, (1<<TWINT)|(1<<TWEN)| (1<<TWSTO)	;Transmite bit de STOP
	sts TWCR, r16
	rjmp error

;....................Display welcome............................
DisplayWelcome:
	
	LDi ZH, High(2*T_Welcome)
	LDI ZL, LOW(2*T_Welcome)
	ldi CONTADOR,13
	
DisplayWelcome_cont:
	lpm DISPVAR, Z+
	call DisplayChar

	dec  CONTADOR
	brne DisplayWelcome_cont
	
	ret

DisplayCocktail:
	
	LDI ZH, High(2*T_Cocktail)
	LDI ZL, LOW(2*T_Cocktail)
	ldi CONTADOR,15
	
DisplayCocktail_cont:
	lpm DISPVAR, Z+
	call DisplayChar

	dec  CONTADOR
	brne DisplayCocktail_cont
	
	ret

;....................TABLAS (display)...........................

T_Welcome:
	.Db 'B','I','E','N','V','E','N','I','D','O','S','!','!','0'

T_COCKTail:
	.DB '*','*','*','C','O','C','K','-','T','A','I','L','*','*','*','0'
	
	.org 0x300
T1_B_ISR:
	cli
	ldi CONTROL, 0x01
	cbi portc,0
	reti