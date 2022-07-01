# Cursed Python

Descripción: ¿Quién dijo que no se podría hacer un crackme en python? ¡Pues ahi va uno!

Flag: UAD360{n0th1nG_l1k3_CrAck1nG_with_s0urC3_cOd3}

## Writeup

Se nos da un fichero llamado crackme.py, aún siendo este un ELF. Si lo ejecutamos se nos pide una contraseña. Podemos ver si en strings podemos ver la clave en texto plano o algo que ayude a identificar de qué se trata.

Con strings no encontramos la clave pero sí muchas referencias a python. Esto indica que el fichero .py ha sido compilado en bytecode y después incluido en un ELF.
La manera más simple de hacer eso es usando [pyinstaller](https://pyinstaller.readthedocs.io/en/stable/). Buscando un poco podemos decubrir que [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) extrae el contenido de un ejecutable creado con PyInstaller.

```
python3 pyinstxtractor.py ../crackme.py
```

Se nos generará una carpeta con muchos ficheros. El más interesante es crackme.pyc ya que si lo ejecutamos directamente con python se ejecuta el mismo programa. Para decompilar bytecode podemos usar [uncompyle6](https://pypi.org/project/uncompyle6/). uncompyle6 imprimirá por la salida estándar el código fuente de bytecode.

**Importante: Debido que uncompyle6 y pyinstxtractor no son compatibles con las últimas versiones de python es recomendable usar python3.7**

Una vez ya tenemos el código fuente podemos ver que el programa hace 5 comprobaciones. Si todas son correctas la clave es válida.

En el check1 solo tenemos que despejar un sistema de ecuaciones simple con sumas y restas
```
key[0] + key[3] == 214                      => key[0] = 110
key[0] - key[5] == 0                        => key[5] = 110
59 + key[3] == 163                          => key[3] = 104
key[2] + key[4] == 165                      => key[2] = 116
key[4] + key[0] == 159                      => key[4] = 49
key[6] + key[6] == 142                      => key[6] = 71
key[1] + key[2] + key[3] + key[4] == 317    => key[1] = 48

110 48 116 104 49 110 71 => n0th1nG
```
`n0th1nG`


El check2 es muy parecido al check1. Sin embargo tiene multiplicaciones y divisiones
```
key[5] * key[1] * key[4] == 523260
key[1] * 2 - 2 == key[3] * 2            => key[1] = 108
key[3] * key[6] // key[6] + 1 == 108    => key[3] = 107
key[5] // 5 * key[4]  == 969            => key[4] = 51
key[0] * key[4] == 4845                 => key[0] = 95
key[2] * 11 // 11 * 2 // 2 == 49        => key[2] = 49
key[6] * key[2] == 3283                 => key[6] = 67
key[0] * key[5] == 9025                 => key[5] = 95

95 108 49 107 51 95 67 => _l1k3_C
```
`_l1k3_C`

En cuanto al check3 tenemos un XOR donde tenemos el output y la clave. Con estos datos podemos obtener la cadena original.
Podemos hacerlo fácilmente con [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')XOR(%7B'option':'Hex','string':'4f%202b%2072%209b%201f%20aa%2071'%7D,'Standard',false)&input=M2QgNmEgMTEgZjAgMmUgYzQgMzY)

`rAck1nG`

El check4 es una implementación muy simple del arlgoritmo ROT13, donde cada letra se rota 13 posiciones. Poderemos decodificarlos con la biblioteca codecs
```python
import codecs
codecs.decode('_jvgu_f', 'rot_13')
'_with_s'
```
`_with_s`

La última comprobación es un AES ECB donde tenemos el ciphertext y la clave. Simplemente tendremos que descifrarlo y ya tendremos la última parte de la flag
```python
from Crypto.Cipher import AES

ct = b'yJ\x9a\x91:\xac%\xb9\xb2\xfdR\xe1\xf0\x10\xd4\r'
keyAES = "sUp3r_s3crEt_k3y"
AES.new(keyAES, AES.MODE_ECB)

cipher.decrypt(ct)
b'0urC3_cOd3\x00\x00\x00\x00\x00\x00'
```
`0urC3_cOd3`

La clave completa por lo tanto sería: n0th1nG_l1k3_CrAck1nG_with_s0urC3_cOd3
