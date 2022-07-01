## Shooter

Flag: UAD360{g0Od_g4mE_W3l1_Pl4yEd}

**Writeup**

El apk proporcionado es un juego hecho en Unity. Un juego en Unity puede estar compilado con Il2cpp o con Mono. Esto es fácilmente comprobable mirando la carpeta libs del apk donde aparecerán referencias a Il2cpp. Para empezar descomprimiremos el apk

```
unzip Shooter.apk -d Shooter
```

Como ya sabemos que el juego está compilado en Il2cpp usaremos [Il2cppDumper](https://github.com/Perfare/Il2CppDumper), el cual nos ayudará sobretodo en el tema del análisis con Ghidra/IDA. Para usar Il2cppDumper tendremos que proporcionarle el Native compilado (`libil2cpp.so`) y los metadatos  (`global-metadata.dat`) en `assets\bin\Data\Managed\Metadata`

```
Il2CppDumper.exe .\Shooter\lib\arm64-v8a\libil2cpp.so .\Shooter\assets\bin\Data\Managed\Metadata .\dump
```

Il2cppDumper creará distintos ficheros: `dump.cs` contendrá las cebazeras de las funciones con los nombres originales. `script.json` servirá si quieremos usar el plugin de Ghidra/IDA para cargar el .so del APK. il2cpp.h tiene las cabezeras de nuestro .so. Sin embargo, este último fichero no es compatible con ghidra. Por ello, se proporciona el script `il2cpp_header_to_ghidra.py` el cual creará las cabeceras correctas

```
└─$ python2 il2cpp_header_to_ghidra.py
Script started...
il2cpp.h opened...
il2cpp.h read...
il2cpp.h data fixed...
il2cpp.h closed.
il2cpp_ghidra.h opened...
header written...
fixed data written...
il2cpp_ghidra.h closed.
```

En este writeup se hará en Ghidra. Primero añadimos la cabecera `il2cpp_ghidra.h` desde File->Parse C source. A continuacion añadimos el script `ghidra_with_struct.py` proporcionado por `Il2cppDumper`.  Para añadir un nuevo script en Ghidra simplementa hay que ir al Script Manager (Icono circular verde con una flecha blanca) y pegar los archivos .py de Il2cppDumper en la carpeta `ghidra_scripts`. Recargamos la ventana de scripts y ejecutamos `ghidra_with_struct.py` y seleccionamos el script.json anteriormente generado. Esto tardará un buen rato.

Entre las funciones de `Enemy` podemos encontrar una funcion llamada `Enemy$$RegainHealthOverTime`. 

```cpp

System_Collections_IEnumerator_o * Enemy$$RegainHealthOverTime(Enemy_o *__this,MethodInfo *metho d)

{
  Il2CppObject *__this_00;
  
  if ((DAT_00b72ef6 & 1) == 0) {
    thunk_FUN_0034773c(&Enemy.<RegainHealthOverTime>d__13_TypeInfo);
    DAT_00b72ef6 = 1;
  }
  __this_00 = (Il2CppObject *)thunk_FUN_00348818(Enemy.<RegainHealthOverTime>d__13_TypeInfo);
  System.Object$$.ctor(__this_00,(MethodInfo *)0x0);
  *(undefined4 *)&__this_00[1].klass = 0;
  if (__this_00 != (Il2CppObject *)0x0) {
    __this_00[2].klass = (Il2CppClass *)__this;
    thunk_FUN_00381c20(__this_00 + 2,__this);
    return (System_Collections_IEnumerator_o *)__this_00;
  }
                    /* WARNING: Subroutine does not return */
  FUN_003a1168();
}

```

la cual es referenciada en la función `Update`

```cpp
void Enemy$$Update(Enemy_o *__this,MethodInfo *method)

{
  System_Collections_IEnumerator_o *routine;
  
  if (((__this->fields).health != (__this->fields).maxHealth) &&
     ((__this->fields).isRegenHealth == false)) {
    routine = Enemy$$RegainHealthOverTime(__this,method);
    UnityEngine.MonoBehaviour$$StartCoroutine
              ((UnityEngine_MonoBehaviour_o *)__this,routine,(MethodInfo *)0x0);
    return;
  }
  return;
}
```
Esto inica que cada vez que se ejecute el Update se va a llamar a la función que regenera vida al enemigo. Como tenemos la llamada a la función podríamos modificar el binario para que nunca se llame a tal función. En en ensamblador lo podemos ver así (Tener en cuenta que las instrucciones están en arm64)

```asm
                             LAB_007fed08                                    XREF[1]:     007fecf8 (j)   
        007fed08 e0  03  13       mov        x0 ,x19
                 aa
        007fed0c 07  00  00       bl         Enemy$$RegainHealthOverTime                      System_Collections_IEnumerator_o
                 94
        007fed10 fd  7b  41       ldp        x29 ,x30 ,[sp , #0x10 ]
                 a9
        007fed14 e1  03  00       mov        x1 ,x0
                 aa
        007fed18 e0  03  13       mov        x0 ,x19
                 aa

```

Al tener la direccion donde se hace la llamada (`007fed0c` en la instrucción `bl`) podemos ir a un editor hexadecimal a tal dirección y sustituir la instrucción por NOP. Para obtener el opcode de NOP podemos ir a [Armconverter](https://armconverter.com/?code=NOP). Por lo tanto habría que sustituir

```
07 00 00 94 => D5 03 20 1F   
```

Una vez parcheado el binario lo sustituimo por el original, comprimimos el APK como zip. Para versiones iguales o superiores a R, la compresión debe excuir el archivo `resources.arsc`.
```
zip -n "resources.arsc" -qr ../Shooter-mod.apk *
```

Finalmente lo firmamos. Para firmarlo podemos usar [Uber APK Signer](https://github.com/patrickfav/uber-apk-signer). Una vez firmado instalamos y ejecutamos

```
adb install Shooter-mod.apk
adb shell monkey -p com.uad360.shooter 1
```

Como la vida ya no se regenera, jugando un poco mataremos al enemigo y se nos monstrará la flag por pantalla

