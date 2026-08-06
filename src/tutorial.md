# Tutorial de `rc` — El shell de Plan 9
Copyright © 2026 Hernán Rondelli. All rights reserved.

---

## Qué es `rc`?

`rc` es el shell del sistema operativo *Plan 9 from Bell Labs*---que diseñado por los mismos creadores de Unix. Es un shell más simple, consistente, y elegante que `sh`. El shell `rc` elimina inconsistencias históricas acumuladas en los shells POSIX (e.g., `sh`, `ksh`, `bash`, `zsh`).

En `rc` hay una sola regla principal de la que se deriva casi todo:

> [!IMPORTANT]
> En `rc` **todo** es una lista: los comandos reciben listas y producen listas.

---

## Comandos básicos

Los comandos y programas clásicos de Unix funcionan igual que en cualquier otro shell:

```rc
% echo 'Aguante Boca'
% ls -l /etc
% cat /etc/fstab
```

---

## Variables

El símbolo `$` se usa cuando se *lee* el valor, en la asignación no usa `$`:

```rc
% nombre = 'Diego Armando Maradona'  # se crea la variable, y se le asigna un valor
% echo $nombre                       # → Diego Armando Maradona
```

> [!NOTE]
> `rc` ignora todo lo que se escriba después de un símbolo `#` hasta el fin de esa línea.

---

## Listas

La lista es la estructura de datos principal en `rc`. Los elementos de la lista se escriben separados por un espacio:

```rc
% colores = (azul amarillo verde negro)
% echo $colores
azul amarillo verde negro
% echo $colores(1)        # se empieza a contar desde el 1
azul
% echo $colores(1 2 1)
azul amarillo azul
% echo $#colores          # longitud de la lista
4
% arquero = 'Juan Carlos Olave'
% echo $arquero
Juan Carlos Olave
% echo $#arquero          # longitud de la lista
1
```

En el último ejemplo, podemos ver que en `rc` **todo** es una lista, i.e. la variable `arquero` es una lista que contiene un solo string.

---

## El operador `^` de concatenación

En `rc` la concatenación es explícita: se usa el operador `^`. Esto elimina toda una categoría de errores de _quoting_ que existen en `sh`, donde la concatenación es implícita y depende del contexto.

### Concatenación básica

```rc
% echo 'Aguante'^'Belgrano'
AguanteBelgrano
% echo 'Aguante'^' Belgrano'
Aguante Belgrano
% nombre = 'Juan Carlos Olave'
% echo 'Sr. ' ^ $nombre
Sr. Juan Carlos Olave

# Dos variables
% extension = 'txt'
% nombre_del_archivo = 'notas'
% echo $nombre_del_archivo^'.'^$extension
notas.txt
```

> [!NOTE]
> **Por qué `^` es necesario?**
>
> Sin el operador `^`, dos tokens separados por espacios son argumentos distintos para el comando. El operador `^` indica explícitamente "esto es una sola cosa":
>
> ```rc
> % nombre_del_archivo = 'notas'
> % echo $nombre_del_archivo '.txt'
> notas .txt                           # → dos argumentos: 'notas' y '.txt'
> % echo $nombre_del_archivo ^ '.txt'
> notas.txt                            # → un argumento: 'notas.txt'
> % echo $nombre_del_archivo'.txt'
> notas.txt                            # → un argumento: 'notas.txt'
> ```

### Concatenación de listas

Cuando uno de los operandos es una lista, y el otro es un elemento simple, `^` distribuye la concatenación sobre cada elemento de la lista:

```rc
% archivos = (main funciones data)
% echo $archivos ^ '.c'
main.c funciones.c data.c

% echo 'test_' ^ $archivos ^ '.c'
test_main.c test_funciones.c test_data.c
```

Esto es útil para generar nombres de archivos, flags, ó cualquier conjunto de strings que comparten un prefijo, ó sufijo.

Cuando ambos operandos son listas de la misma longitud, la concatenación es _elemento a elemento_:

```rc
% prefijos = (src/ include/ test/)
% sufijos = (main.c config.h run.c)
% echo $prefijos ^ $sufijos
src/main.c include/config.h test/run.c
```

Si las listas tienen distinta longitud, `rc` produce un error.

---

## Quoting

En `rc` las comillas simples se usan para literales, sin ninguna expansión.

```rc
# Comillas simples: literal, sin ninguna expansión
% echo 'El valor es $nombre'    # → El valor es $nombre


Las comillas dobles unen los elementos de una lista en un único string separado con espacios:
% num1 = (uno dos tres)
% num2 = $"num1         # $"var: une los elementos de la lista en un único string
% echo $#num1
3
% echo $#num2
1
% echo $num1
uno dos tres
% echo $num2
uno dos tres
```

No hay comillas dobles para expansión como en `bash`. La expansión siempre ocurre con `$`, sin importar el contexto.

---

## Globbing

El _globbing_ es la expansión de nombres de archivo que realiza el shell antes de ejecutar un comando. Los patrones de glob son más simples que las expresiones regulares.

> [!WARNING]
> Los patrones de glob, y las expresiones regulares, **no** son la misma cosa, aunque se parecen bastante---y en concepto, son la misma idea.

### `*` cero o más caracteres

Coincide con cualquier secuencia de caracteres, excepto `/` dado que es el separador de directorios:

```rc
ls *.c               # todos los archivos .c
ls notas-*.txt       # archivos que empiezan con 'notas-' y que terminen con `.txt`
ls src/*.h           # archivos .h dentro de src/
```

> [!NOTE]
> `*` no es recursivo, i.e no entra en subdirectorios.

### `?` exactamente un único carácter

El operador `?` coincide con exactamente **un** carácter, excepto `/`.

```rc
ls archivo?.c        # archivo1.c, archivoa.c, archivo_.c,...
ls ???.txt           # exactamente 3 caracteres antes de .txt
```

### `[...]` clase de caracteres

Coincide con exactamente un carácter del conjunto:

```rc
ls archivo[123].c       # archivo1.c, archivo2.c o archivo3.c
ls [Aa]rchivo.txt       # Archivo.txt, archivo.txt
ls letra_[a-z].txt      # letra_a.txt, letra_b.txt, ..., letra_z.txt
ls test_[0-9].sh        # test_0.sh, test_1.sh, ..., test_9.sh
ls [a-zA-Z]*.c          # archivos .c que empiezan con una letra
```

Se pueden combinar rangos y caracteres sueltos dentro de `[...]`:

```rc
ls archivo[0-9a-f].txt  # un dígito o una letra hexadecimal
```

### `[~a-z]` clase de caracteres negada

En `rc`, la negación dentro de `[...]` se indica con `~` (ojo, no es con `!` ni `^` como en otros shells):

```rc
ls archivo[~0-9].c      # archivoa.c, archivob.c... pero NO archivo1.c
ls [~aeiou]*.txt        # .txt que NO empiezan con vocal
ls [~.]*.c              # .c que no empiezan con punto
ls [~a-zA-Z]*.sh        # .sh que no empiezan con una letra
```

### El globbing produce listas

La expansión de un patrón produce una lista, consistente con el modelo central de rc:

```rc
source = (*.c)
wc -l $source        # cantidad de líneas en cada archivo .c encontrado

for(f in *.txt) {
    echo 'Procesando el archivo: '^$f
}
```

---

## Pipes

Un _pipe_ en Unix es un mecanismo de _Inter-Process Communication (IPC)_, y es uno de los conceptos más importantes, y poderosos para la composición de operaciones de distintos programas.

Un _pipe_ es un _canal unidireccional de comunicación entre procesos_ que permite que la salida estándar de un programa se conecte directamente a la entrada estándar de otro programa, sin necesidad de archivos intermedios.

```rc
% ls | sort
% cat compras_del_super.txt | grep detergente
% ls | grep '.txt' | wc -l
```

---

## Redireccionamiento

El redireccionamiento permite cambiar la salida ó la entrada de un programa:

```rc
% echo 'El Diego' > jugadores.txt     # crea ó sobreescribe el archivo de salida 'jugadores.txt'
% echo 'Messi' >> jugadores.txt       # crea ó le agrega al archivo de salida 'jugadores.txt'
% cat < entrada.txt
```

### File descriptors

- **0 - stdin**: entrada estándar, el teclado.
- **1 - stdout**: salida estándar, el display.
- **2 - stderr**: salida de error, el display.

### Operadores

A continuación se muestra la sintaxis de los operadores.

| Operador | Acción                                | Ejemplo                       |
|:--------:|---------------------------------------|-------------------------------|
| `>`      | Redirige stdout (sobreescribe)        | `ls > archivos.txt`           |
| `>[1]`   | Idem anterior                         | `ls >[1] archivos.txt`        |
| `>>`     | Redirige stdout (agrega al final)     | `echo 'hello' >> log.txt`     |
| `>>[1]`  | Idem anterior                         | `echo 'hello' >>[1] log.txt`  |
| `<`      | Redirige stdin                        | `sort < lista.txt`            |
| `<[0]`   | Idem anterior                         | `sort <[0] lista.txt`         |
| `>[2]`   | Redirige stderr                       | `comando >[2] errores.log`    |
| `>[2=1]` | Redirige stderr a la salida de stdout | `comando > salida.txt >[2=1]` |
| `>[2=]`  | Redirige stderr a null                | `comando >[2=]`               |

Ejemplo combinando pipes con redireccionamiento:

```rc
% cat archivo.log | grep error >errores_encontrados.txt >[2]/dev/null
```

---

## Sustitución de comandos

En `sh` se usa `` `comando` ``, en `rc` se usa `` `{comando} ``:

```rc
% fecha = `{date +%F} # (suponiendo Unix date)
% echo $fecha
2011-06-26
% archivos = `{ls /etc/}
% echo $#archivos       # cantidad de archivos en /etc
```

El resultado es una lista, dividida por los caracteres en `$ifs` (by default: espacio, tab, y newline).

---

## Aritmética con `hoc`

`rc` no tiene operadores aritméticos incorporados. Para hacer cálculos numéricos, en `rc` hay que usar un programa externo llamando `hoc` _(High Order Calculator)_, que es la calculadora de Plan 9, introducida por Kernighan y Pike en *The Unix Programming Environment* (1984).

### Uso interactivo de hoc

`hoc` es una calculadora simple de línea de comandos con soporte para floats:

```rc
% hoc
3 * 5
15
27 / 3
9
28 / 3
9.333333333333334
sqrt(16)
4
2 ^ 10
1024
PI
3.141592653589793
E
2.718281828459045
```

### Guardar el resultado en variables

Se puede combinar `hoc` con sustitución de comandos para guardar el resultado en una variable en `rc`:

```rc
% x = `{hoc -e '3 * 5'}
% echo $x
15
% raíz = `{hoc -e 'sqrt(4)'}
% echo $raíz
2
```

### Usar variables de `rc` en expresiones en `hoc`

Se construye la expresión como string usando `^`, y se la pasa a hoc:

```rc
% radio = 5
% area = `{hoc -e $radio ^ '^2 * PI'}
% echo $area
78.53981633974483
```

Esta línea tiene un detalle importante: aparecen dos `^` con **significados distintos**.

- El primer `^` es el _operador de concatenación_ en `rc`, y concatena `$radio` (que vale `5`) con el string `'^2 * PI'`, produciendo la expresión `5^2 * PI`.
- Dentro de esa expresión, el `^` que queda es el _operador de exponenciación_ en hoc.

El string `5^2 * PI` se pasa a `hoc`, que lo evalúa como 25 × π.

Otro ejemplo con dos variables:

```rc
% a = 28
% b = 3
% cociente = `{hoc -e $a ^ '/' ^ $b}
% echo $cociente
9.333333333333334
```

### Operaciones en `hoc`

A continuación se muestra la sintaxis de los operadores.

| Operación        | Sintaxis       | Uso                        |
|------------------|----------------|----------------------------|
| suma             | `a + b`        | `2 + 3` → `5`              |
| resta            | `a - b`        | `10 - 4` → `6`             |
| multiplicación   | `a * b`        | `6 * 7` → `42`             |
| división         | `a / b`        | `22 / 7` → `3.1428...`     |
| módulo           | `a % b`        | `17 % 5` → `2`             |
| exponenciación   | `a ^ b`        | `2^10` → `1024`            |
| raíz cuadrada    | `sqrt(x)`      | `sqrt(2)` → `1.4142...`    |
| valor absoluto   | `abs(x)`       | `abs(-5)` → `5`            |
| parte entera     | `int(x)`       | `int(3.9)` → `3`           |
| logaritmo        | `log(x)`       | `log(E)` → `1`             |
| exponencial      | `exp(x)`       | `exp(1)` → `2.7182...`     |
| seno             | `sin(x)`       | (en radianes)              |
| coseno           | `cos(x)`       | (en radianes)              |
| tangente         | `tan(x)`       | (en radianes)              |
| arcotangente     | `atan(x)`      | (en radianes)              |
| número π         |`PI`            | 3.141592653589793          |
| número _e_       | `E`            | 2.718281828459045          |

---

## Control de flujo

### if

La condición es un comando. Si devuelve código de error `0`, i.e comando exitoso, se ejecuta el bloque.

```rc
if(test -f /etc/passwd) {
    echo 'El archivo existe.'
}

if(test -f /etc/passwd) {
    echo 'El archivo existe.'
} if not {
    echo 'El archivo **no** existe.'
}
```

### while

```rc
i = 1
while(test $i -le 10) {
    echo $i
    i = `{hoc -e $i' + 1'}
}
```

### for

```rc
for(c in azul amarillo azul) {
    echo $c
}

# Con una variable lista:
colores = (azul amarillo azul)
for(c in $colores) {
    echo $c
}

# Sobre archivos:
for(f in `{ls *.txt}) {
    echo 'Procesando el archivo: '^$f
}

# un "for range" de 1 a 10
for(i in `{seq 10}) {
	echo $i
}
```

### switch

```rc
input = 'hola'
switch($input) {
case hola
    echo 'Dijiste hola'
case chau
    echo 'Dijiste chau'
case *
    echo 'No sé qué dijiste'
}
```

---

## El operador `~` match

`rc` incluye el operador `~` para comparar strings contra patrones de globbing:

```rc
# Comparación exacta
if(~ $nombre Juan) {
    echo 'El nombre es Juan.'
}

# Con patrón de globbing
if(~ $archivo *.txt) {
    echo 'El archivo es un archivo de texto.'
}

# Múltiples patrones (cualquiera puede coincidir)
if(~ $jugador Cavenaghi Ortega Gallardo 'Ramón Díaz') {
	echo 'Es un jugador de la B.'
}
```

---

## Funciones

Las funciones se definen con `fn` y el nombre de la función, y los parámetros de la función se especifican por posición, en este caso, la función recibe un único parámetro `$1`:

```rc
     1	fn say_hello {
     2		echo 'Hello, ' ^ $1
     3	}
```

Los argumentos a la función se pasan posicionalmente, en este caso, el primer argumento es `$1`:

```rc
% say_hello 'Diego Armando Maradona'
Hello, Diego Armando Maradona
```

La lista de todos los argumentos se especifican con la variable `$*`. Además, se pueden especificar por posición, donde el primer argumento es `$1`, el segundo argumento es `$2`, etc:

```rc
     1	fn mostrar_argumentos {
     2	    echo 'La cantidad de argumentos de la función es ' ^ $#* ^ '.'
     3		echo 'Los argumentos son:'
     4		for(arg in $*) {
     5			echo '-' $arg
     6		}
     7	}
     8
     9	mostrar_argumentos azul amarillo azul
```

> [!TIP]
> Si bien se pueden definir las funciones interactivamente, es más habitual y conveniente, escribir las funciones en un archivo y luego ejecutar ese archivo. Es el tema que se trata a continuación.

---

## Scripts

La primera línea de un script `rc` es el _shebang_:

```rc
#!/bin/rc
```

Ejemplo completo:

```rc
     1	#!/bin/rc
     2
     3	# Verifica que se pasó un argumento
     4	if(~ $#* 0) {
     5	    echo 'Uso: '$0' <directorio>'
     6	    exit 1
     7	}
     8
     9	dir = $1
    10
    11	if(! test -d $dir) {
    12	    echo $dir 'no es un directorio'
    13	    exit 1
    14	}
    15
    16	echo 'Archivos en '$dir':'
    17	for(f in `{ls $dir}) {
    18	    echo '  ' $f
    19	}
```

---

## Argumentos en scripts

Cuando se invoca un script así:

```
./script arg1 arg2 arg3
```

los argumentos quedan disponibles dentro del script a través de varias variables especiales.

### `$*` la lista completa de argumentos

En `rc`, `$*` es una lista con todos los argumentos.

```rc
     1	#!/bin/rc
     2
     3	echo 'Argumentos recibidos:' $*
     4	echo 'Cantidad de argumentos:' $#*
```

Invocado como `./script uno dos tres`:

```rc
Argumentos recibidos: uno dos tres
Cantidad de argumentos: 3
```

### `$1`, `$2`, `$3`, ... acceso posicional

Son equivalentes a `$*(1)`, `$*(2)`, `$*(3)`: índices sobre la lista `$*`, con base 1.

```rc
     1	#!/bin/rc
     2
     3	echo 'Primero:' $1
     4	echo 'Segundo:' $2
     5	echo 'Tercero:' $3
```

Si se intenta acceder a un índice que no existe (por ejemplo `$3` cuando solo se pasaron dos argumentos), `rc` devuelve una lista vacía sin producir un error.

### `$0` nombre del script

`$0` contiene el nombre con el que fue invocado el script. Es útil para mensajes de uso:

```rc
     1	#!/bin/rc
     2
     3	echo 'Este script se llama:' $0
```

```
Este script se llama: ./script
```

### `$#*` cantidad de argumentos

El operador `$#` seguido de un nombre de lista devuelve su longitud. Para los argumentos del script, la lista se llama `*`, así que la sintaxis es `$#*`:

```rc
echo $#*       # cantidad de argumentos
```

### Validar argumentos

El patrón más común es verificar que se recibió la cantidad correcta de argumentos antes de hacer cualquier otra cosa:

```rc
     1	#!/bin/rc
     2
     3	# Verificar que se pasó exactamente un argumento
     4	if(~ $#* 0) {
     5	    echo 'Uso: '$0' <nombre>'
     6	    exit 1
     7	}
     8
     9	echo 'Hola,' $1
```

Para verificar un mínimo de argumentos:

```rc
if(test $#* -lt 2) {
    echo 'Uso: '$0' <origen> <destino>'
    exit 1
}
```

### Iterar sobre los argumentos

Como `$*` es una lista, el `for` funciona directamente:

```rc
#!/bin/rc

for(arg in $*) {
    echo 'procesando:' $arg
}
```

Invocado como `./script boca belgrano olave`:

```rc
procesando: boca
procesando: belgrano
procesando: olave
```

---

## Variables especiales

En la siguiente tabla, se muestran las variables especiales de `rc`.

| Variable  | Significado                                               |
|-----------|-----------------------------------------------------------|
| `$*`      | Todos los argumentos de la función, ó del script          |
| `$#*`     | Cantidad de argumentos                                    |
| `$1`, `$2`... | Argumentos posicionales                               |
| `$status` | Código de salida del último comando (equivale a `$?`)     |
| `$pid`    | PID del proceso `rc` actual                               |
| `$home`   | Home directory (en `sh` es `$HOME`)                       |
| `$path`   | Lista de directorios donde se encuentran los binarios     |
| `$ifs`    | Separadores para sustitución de comandos                  |

Ejemplos:

```rc
% path = ($home/bin /usr/local/bin /usr/bin /bin)
%
% # Agregar un directorio al frente:
% path = ($home/mis_scripts $path)
```

## Herramientas del ecosistema: historial sin historial

`rc` no guarda un historial de los comandos usados previamente como otros shells. En `rc`, no existe el comando `history`, ni `Ctrl-P`, ni `$home/.rc_history`. Esto es una decisión consciente de diseño. En Plan 9, el historial *es* simplemente el texto visible en la ventana de la terminal. _Lo que ya está en pantalla no necesita guardarse aparte._

Lo que sigue es un conjunto de tres scripts que implementan búsqueda y re-ejecución de comandos componiendo herramientas pequeñas. Son un ejemplo concreto de la filosofía del ecosistema: en lugar de complicar el shell, se construye con lo que ya existe.

---

### `wintext` muestra el texto de la ventana actual

`wintext` obtiene el contenido de la ventana activa. Soporta tres entornos: Acme, 9term y `tmux`.

```rc
     1	#!/usr/local/plan9/bin/rc
     2
     3	if(~ $winid [0-9]*) {
     4		9p read acme/$winid/body
     5		exit 0
     6	}
     7	if(~ $text9term 'unix!'*) {
     8		dial -e $text9term < /dev/null
     9		exit 0
    10	}
    11	if(~ $TMUX ?*) {
    12		tmux capture-pane -p
    13		exit 0
    14	}
    15
    16	echo 'no running window found' >[2=1]
    17	exit 1
```

En cada caso se usa una variable que el entorno correspondiente setea automáticamente:

- **`$winid`**: Acme lo define en cada ventana `win` con el ID numérico del buffer. `9p read acme/$winid/body` lee el contenido de ese buffer a través del servidor 9P. El patrón `[0-9]*` verifica que la variable esté definida y contenga un número.

- **`$text9term`**: `9term` (la terminal de plan9port) lo define con la dirección ip de la terminal, siempre con el prefijo `unix!` (notación de Plan 9 para sockets Unix). `dial -e` conecta a esa dirección y lee su contenido.

- **`$TMUX`**: `tmux` lo define cuando se está dentro de una sesión. El patrón `?*` verifica que no esté vacía. `tmux capture-pane -p` imprime el contenido visible del pane actual.

El fallback usa `>[2=1]`, redirige stderr a stdout.

---

### `"` buscar en el historial

El nombre del script es literalmente el carácter `"`. En `rc`, casi cualquier string puede ser el nombre de un script.

```rc
     1	#!/usr/local/plan9/bin/rc
     2
     3	. 9.rc
     4
     5	PROMPT='[^ 	]*[%;$#][ 	]+'
     6
     7	fn cmds {
     8		wintext | sed -n 's/^'$PROMPT'([^"])/	\1/p'
     9	}
    10
    11	switch($#*) {
    12	case 0
    13		cmds | tail -1
    14	case *
    15		cmds | grep -n '^	'^$"* | tail -r |
    16			sort -u +1 | sort -n |
    17			sed 's/^[0-9]+: //'
    18	}
```

`. 9.rc` ejecuta ese archivo en el contexto actual. `PROMPT` es una expresión regular que describe el aspecto de un prompt típico: caracteres sin espacio, seguidos de `%`, `;`, `$` o `#`, seguidos de espacios. Captura prompts como `%`, `$`, `user$`, etc.

La función `cmds` pasa el texto de la ventana por `sed`, que extrae todo lo que aparece después de un prompt---es decir, los comandos que le usuarie ejecutó---siempre que no empiecen con `"`. Ese filtro `[^"]` evita que el script se liste a sí mismo en el historial.

Sin argumentos, devuelve el último comando ejecutado (`tail -1`). Con argumentos, busca comandos que los contengan: `grep -n` numera las líneas para preservar el orden cronológico, `tail -r` invierte, `sort -u +1` deduplica por contenido ignorando el número de línea, `sort -n` restaura el orden, y `sed` limpia los prefijos numéricos.

Notar `$"*`: convierte la lista de argumentos `$*` en un único string con espacios, para usarlo como patrón en `grep`. Es el mismo operador `$"var` del tutorial.

---

### `""` buscar y ejecutar

El nombre del script es `""`. Usa `"` para buscar y ejecuta el resultado:

```rc
     1	#!/usr/local/plan9/bin/rc
     2
     3	cmd = `{quote1 $* | tail -1}
     4	if (~ $#cmd 0) {
     5		echo no such command found >[1=2]
     6		exit notfound
     7	}
     8
     9	echo '	' $cmd >[1=2]
    10	rc -c $"cmd
```

`quote1` es un auxiliar que llama a `"` y formatea la salida con quoting correcto para rc, de modo que el resultado pueda pasarse a `rc -c` sin problemas. `| tail -1` toma el match más reciente.

Tres detalles que vale notar:

**`>[1=2]`** redirige stdout hacia stderr, que es útil para mensajes que no deben mezclarse con la salida del comando ejecutado.

**`exit notfound`**: en `rc`, `$status` puede ser un string arbitrario, `exit notfound`, `exit 'permiso denegado'`, `exit ok` son todos válidos. Esto es una particularidad de `rc` y Plan 9 que no existe en `bash`.

**`rc -c $"cmd`**: arranca un nuevo proceso `rc` con `-c`, que toma el comando a ejecutar directamente de un string. `$"cmd` convierte la lista `cmd` en ese string.

---

### El sistema completo

El algoritmo sería:

1. `wintext` para obtener el texto de los comandos usados.
1. `" comando` para buscar en el historial el `comando`.
1. `"" comando` para ejecutar el último uso del `comando`.

Los tres scripts juntos implementan un historial completo sin ninguna estructura de datos especial ni mecanismo interno del shell. El "historial" es simplemente el texto que ya está en pantalla, procesado con `sed`, `grep` y `sort`.

Cualquiera puede reemplazar `wintext` por otra fuente de texto---un archivo de log, la salida de otro comando, etc---y los otros dos scripts siguen funcionando sin cambios.
