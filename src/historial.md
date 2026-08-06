# Caso de uso: Implementación del Historial

En esta sección vamos a analizar un caso de uso que demuestra la composición de comandos, y el uso de scripts para implementar fácilmente nuevas funcionalidades.

`rc` no guarda un historial de los comandos usados previamente como otros shells. En `rc`, no existe el comando `history`, ni `Ctrl-P`, ni `$home/.rc_history`. Esto es una decisión consciente de diseño. En Plan 9, el historial *es* simplemente el texto visible en la ventana de la terminal. _Lo que ya está en pantalla no necesita guardarse aparte._

Lo que sigue es un conjunto de tres scripts que implementan búsqueda, y re-ejecución de comandos componiendo herramientas pequeñas. Son un ejemplo concreto de la filosofía Unix: en lugar de complicar el shell, se construye con lo que ya existe.

---

## `wintext` muestra el texto de la ventana actual

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

## `"` buscar en el historial

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

## `""` buscar y ejecutar

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

## El ejemplo completo

El algoritmo sería:

1. `wintext` para obtener el texto de los comandos usados.
1. `" comando` para buscar en el historial el `comando`.
1. `"" comando` para ejecutar el último uso del `comando`.

Los tres scripts juntos implementan un historial completo sin ninguna estructura de datos especial ni mecanismo interno del shell. El "historial" es simplemente el texto que ya está en pantalla, procesado con `sed`, `grep` y `sort`.

Cualquiera puede reemplazar `wintext` por otra fuente de texto---un archivo de log, la salida de otro comando, etc---y los otros dos scripts siguen funcionando sin cambios.
