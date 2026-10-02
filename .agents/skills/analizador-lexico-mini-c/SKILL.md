---
name: analizador-lexico-mini-c
description: Especificación del analizador léxico de Mini-C definida por PalicoMocca. Úsala cuando debas implementar, probar o corregir el lexer de Mini-C: indica alfabeto, palabras reservadas, patrones por tipo de token, políticas de separación, prioridades y formato de salida.
---

# Analizador léxico de Mini-C

Especificación elaborada por PalicoMocca (grupo 1SF-134) en el Taller N°6 de Lenguajes Formales y Autómatas (UTP-FISC). Implementa el analizador exactamente como se describe aquí. Si algo no está definido, pregunta antes de asumir; no inventes tokens, reglas ni excepciones.

## 1. Alfabeto Σ

Sensible a mayúsculas. Cualquier carácter fuera de este conjunto es un error léxico.

espacio, `!`, `(`, `)`, `+`, `-`, `0`, `1`, `2`, `3`, `4`, `5`, `6`, `7`, `8`, `9`, `;`, `=`, `A`, `B`, `C`, `D`, `E`, `F`, `G`, `H`, `I`, `J`, `K`, `L`, `M`, `N`, `O`, `P`, `Q`, `R`, `S`, `T`, `U`, `V`, `W`, `X`, `Y`, `Z`, `_`, `a`, `b`, `c`, `d`, `e`, `f`, `g`, `h`, `i`, `j`, `k`, `l`, `m`, `n`, `o`, `p`, `q`, `r`, `s`, `t`, `u`, `v`, `w`, `x`, `y`, `z`, `{`, `}`

## 2. Palabras reservadas

- `int`
- `while`

Todo nombre que no esté en esta lista se reconoce con el patrón de identificador.

## 3. Tipos de token y patrones

Notación: `"abc"` literal · `[a-z]` clase · `|` unión · `*` cero o más · `#` una o más (equivale a `+` de regex) · `< >` agrupación (equivale a paréntesis) · `""` cadena vacía = fin de entrada.

| Tipo de token | Patrón |
|---|---|
| `ASSIGN` | `"="` |
| `EOF` | `""` |
| `EQUAL_EQUAL` | `"=="` |
| `KW_INT` | `"int"` |
| `KW_WHILE` | `"while"` |
| `LBRACE` | `"{"` |
| `RBRACE` | `"}"` |
| `LPAREN` | `"("` |
| `RPAREN` | `")"` |
| `NOT_EQUAL` | `"!="` |
| `PLUS` | `"+"` |
| `MINUS` | `"-"` |
| `INTEGER_LITERAL` | `[0-9]#` |
| `SEMICOLON` | `";"` |
| `IDENTIFIER` | `[A-Za-z_][A-Za-z0-9_]*` |

No existen otros tipos de token.

## 4. Políticas de separación en lexemas

1. Los blancos (espacio, \t, \r, \n) separan lexemas y se consumen sin emitir token.
2. Máxima coincidencia: siempre se toma el lexema más largo posible (whilex es un solo lexema).
3. Primero se lee el nombre completo y después se consulta la tabla de palabras reservadas (int2 es IDENTIFIER).
4. Los operadores dobles (==, !=) se prueban antes que los simples (=).
5. Un número seguido de letras se separa en dos lexemas: 12abc → 12 y abc.
6. -5 se separa en MINUS y INTEGER_LITERAL; el signo lo resuelve la gramática, no el léxico.
7. Un carácter fuera del alfabeto genera LEX001, se consume y el análisis continúa.
8. Al terminar la entrada se emite exactamente un token EOF.

## 5. Prioridad cuando dos patrones coinciden

- Grupo 1 (de mayor a menor prioridad): `KW_INT` = `KW_WHILE` > `IDENTIFIER`
- Grupo 2 (de mayor a menor prioridad): `EQUAL_EQUAL` > `ASSIGN`

Sin conflicto de prioridad: `INTEGER_LITERAL`, `LPAREN`, `RPAREN`, `RBRACE`, `LBRACE`, `SEMICOLON`, `PLUS`, `MINUS`, `NOT_EQUAL`, `EOF`.

## 6. Estructura del token y diagnósticos

```python
Token(type, lexeme, literal, line, column)
Diagnostic(code, severity, message, line, column)
```

- `literal`: valor `int` solo para literales enteros (`"007"` → `7`); `None` en los demás.
- `line` y `column`: posición donde empieza el lexema, desde 1. Solo `\n` abre nueva línea.
- Error léxico: `Diagnostic("LEX001", "error", "Carácter no reconocido: '<c>'", línea, columna)`.
- Salida en terminal: `TIPO 'lexema' línea columna`, un token por línea.

## 7. Casos de prueba que el agente debe verificar

Fuente:
```c
int2 = 12abc;
whilex == -5
```

Salida esperada según la especificación del estudiante:
```text
IDENTIFIER 'int2' 1 1 
ASSIGN '=' 1 6
INTEGER_LITERAL '12' 1 8
IDENTIFIER 'abc' 1 10
SEMICOLON ';' 1 13
IDENTIFIER 'whilex' 2 1
EQUAL_EQUAL '==' 2 8
MINUS '-' 2 11
INTEGER_LITERAL '5' 2 12
EOF '' 2 13
```

Fuente con errores:
```c
int x = @;
x ! = 0; // fin
```

Tokens esperados:
```text
KW_INT 'int' 1 1
IDENTIFIER 'x' 1 5
ASSIGN '=' 1 7
SEMICOLON ';' 1 10
IDENTIFIER 'x' 2 1
ASSIGN '=' 2 5
INTEGER_LITERAL '0' 2 7
SEMICOLON ';' 2 8
IDENTIFIER 'fin' 2 13
EOF '' 2 16
```

Diagnósticos esperados:
```text
LEX001 error 1:9 Carácter no reconocido: '@'
LEX001 error 2:3 Carácter no reconocido: '!'
LEX001 error 2:10 Carácter no reconocido: '/'
LEX001 error 2:11 Carácter no reconocido: '/'
```

## 8. Instrucciones para el agente

1. Implementa el analizador siguiendo las secciones 1–6, en ese orden de precedencia.
2. Ejecuta los casos de la sección 7 y compara la salida. Si difiere, informa qué regla de esta especificación produce la diferencia en lugar de cambiar la regla por tu cuenta.
3. Si detectas una contradicción o un vacío en la especificación (por ejemplo, un símbolo usado en un patrón que no está en Σ), señálalo antes de continuar.

## 9. Integración con el parser existente (obligatorio)

Este lexer NO es un ejercicio aislado: su salida debe alimentar directamente
a `minic.parser.Parser` tal como está en el repositorio, sin modificarlo.
Por lo tanto:

1. No definas tus propias clases `Token` o constantes de tipo de token.
   Importa y usa exactamente:
       from minic.lexer.token import Token
       from minic.lexer.token_type import TokenType
   Un token construido por tu implementación debe ser una instancia real
   de `Token` (o estructuralmente idéntica: mismos 5 campos, en ese
   orden, mismos tipos), y su `.type` debe ser una de las 15 constantes
   de `TokenType`, comparadas por IGUALDAD DE STRING (`TokenType.KW_INT`
   es literalmente `"KW_INT"`), porque `Parser.check()` compara así.

2. `scan()` SIEMPRE debe llegar hasta el final de la entrada y devolver
   la lista COMPLETA de tokens hasta el `EOF`, exista o no un `LEX001`
   en el camino. Los diagnósticos se acumulan aparte; nunca detienen el
   recorrido. (Esto ya lo exige la regla 8 de la sección 4: "al terminar
   la entrada se emite exactamente un EOF" — solo puede cumplirse si el
   lexer no se detiene antes.)

3. Reglas de posición, sin ambigüedad:
   - Un carácter de tabulación (\t) cuenta como 1 columna, igual que
     cualquier otro carácter — NO se alinea a un ancho de tabulación.
   - Solo `\n` abre una línea nueva y reinicia la columna en 1. Un `\r`
     suelto (sin `\n`) es un blanco más: consume una columna, no abre
     línea.

4. Formato de diagnóstico en terminal (ver corrección aplicada en la
   sección 7):
       CÓDIGO severidad línea:columna mensaje
   Ejemplo real: `LEX001 error 1:9 Carácter no reconocido: '@'`

5. Antes de dar por terminado el lexer, corre:
       python -m pytest tests/lexer/
   Esas pruebas ya verifican, contra el lexer de referencia, las
   posiciones, la máxima coincidencia y la recuperación con LEX001. Tu
   implementación debe pasarlas SIN modificarlas.
