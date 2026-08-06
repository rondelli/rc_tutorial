# Comparativa con los shells típicos de Unix

A diferencia de la sintaxis limpia, elegante, y ortogonal de `rc`, los shells típicos de Unix, como `sh` ó `bash`---usualmente denominados shells POSIX---tienen una sintaxis historicamente derivada de ALGOL, y carecen de una estructura coherente.

La diferencia conceptual más importante reside en el modelo de datos. En `rc`, las
variables son listas de strings. En cambio, en un shell POSIX, todo es un
único string plano, donde cada variable debe procesarse e interpretarse internamente por el shell, lo que puede alterar los datos de formas no esperadas.

En los shells POSIX se usan reglas esotéricas de encomillado con comillas simples, y con comillas dobles. Los scripts en `bash` suelen ser muy frágiles, donde se penaliza la claridad y la simplicidad del código con sintaxis muy trucosa.

## Tabla comparativa: `rc` vs `bash`

| Concepto               | `rc`                      | `bash`                        |
|------------------------|-----------------------------|---------------------------|
|  Asignación              |  `x = 'hola'`               |  `x="hola"`                   |
|  Lectura                 |  `$x`                       |  `$x` o `"$x"`                |
|  Lista / array           |  `a = (a b c)`            |  `a=(a b c)`                |
|  Indexar                 |  `$a(1)`                  |  `${a[0]}`                  |
|  Largo de lista          |  `$#a`                    |  `${#a[@]}`                 |
|  Sustitución de comando  |  `` `{cmd} ``               |  `` `cmd` ``                  |
|  Código de error del comando        |  `$status`                  |  `$?`                         |
|  Definir función         |  `fn función { ... }`             |  `función() { ... }`                |
|  if                      |  `if(cmd) { ... }`          |  `if [...]; then ... fi`      |
|  PATH                    |  `path = ($home/bin $path)` |  `PATH="$HOME/bin:$PATH"`     |
|  Comillas para expansión  |  `$var` (siempre seguro)    |  `"$var"` (obligatorio)      |
