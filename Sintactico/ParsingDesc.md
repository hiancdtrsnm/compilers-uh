## Parsing Descendiente

Hasta el momento hemos visto el proceso de compilación a grandes razgos, y hemos definido que la primera fase consiste en el análisis sintáctico. Esta fase a su vez la hemos divido en 2 procesos secuenciales: el análisis lexicográfico (*tokenización* o *lexing*), y el análisis sintáctico en sí (*parsing*). En esta sección nos concentraremos en este segundo proceso.

Recordemos que el proceso de parsing consiste en analizar una secuencia de tokens, y producir un árbol de derivación, lo que es equivalente a producir una derivación extrema izquierda o derecha de la cadena a reconocer. Tomemos la siguiente gramática, que representa expresiones aritméticas (sin ambigüedad):

    E = T
      | T + E

    T = int * T
      | int
      | (E)

Tomemos la cadena siguiente:

    2 * (3 + 5)

Que una vez procesada por la fase lexicográfica produce la siguiente secuencia de *tokens*.

    int  *  (  int  +  int  )

Intentemos producir una derivación (en principio, extrema izquierda, por comodidad) de esta cadena. Como sabemos, si existe una derivación extrema izquierda, es porque se cumple que:

    [1]  E -*-> int * ( int + int )

Preguntémonos entonces, ¿de cuántas formas pudiera `E` derivar en la cadena? Evidentemente, hay exactamente dos formas en que `E` es capaz de producir esta cadena, es decir, hay solamente dos producciones posibles que pudieran seguir en la derivación extrema izquierda. O bien `E -> T + E` o bien `E -> T`. Probemos entonces con la primera de ellas:

    [2]  E -*-> T -*-> int * ( int + int )

Como estamos produciendo una derivación extrema izquierda, tenemos que expandir a continuación el símbolo `T`. Nuevamente, hay varias opciones, probemos con la primera:

    [3]  E -*-> int * T -*-> int * ( int + int )

De momento parece que vamos por buen camino, pues hemos logrado producir los primeros 2 *tokens* de la cadena. Expandimos entonces nuevamente el no-terminal más a la izquierda con la primera producción posible `T -> int * T`:

    [4]  E -*-> int * int * T -*-> int * ( int + int )

En este punto, podemos darnos cuenta de que hemos tomado el camino equivocado. Cómo estamos produciendo una derivación extrema izquierda, y los terminales no derivan en ningún símbolo, sabemos que todo lo que esté a la izquierda del primer no-terminal no va a cambiar en el futuro. Luego, es evidentemente que ya no seremos capaces de generar la cadena, pues estos terminales (`int * int *`) no son prefijo de la cadena a reconocer. Deshagamos entonces la última producción (volviendo al paso 3) y probemos otro camino (`T -> int`):

    [4]  E -> int * int -*-> int * ( int + int )

Nuevamente, hemos generado una secuencia de *tokens* que no es prefijo de la cadena. Probamos de nuevo:

    [4]  E -*-> int * ( E ) -*-> int * ( int + int )

Parece que en este punto hemos hecho un avance, pues logramos reconocer tres *tokens* de prefijo. Expandimos entonces el nodo `E -> T` nuevamente, y de ahí probamos la próxima producción (`T -> int * T`), que también falla:

    [5]  E -*-> int * ( T ) -*-> int * ( int + int )
    [6]  E -*-> int * ( int * T ) -*-> int * ( int + int )

Probando con cualquiera de las producciones de `T` nunca podremos generar el *token* `+` que falta, por lo tanto eventualmente volveremos a probar con `E -> T + E`:

    [5]  E -*-> int * ( T + E ) -*-> int * ( int + int )

Una vez llegados a este punto, ya podemos hacernos una idea de cómo funciona este proceso. Eventualmente, tendremos que derivar `T -> int` y `E -> T -> int` para lograr producir la cadena final. Finalmente obtenemos la derivación extrema izquierda siguiente:

    E -> T
      -> int * T
      -> int * ( E )
      -> int * ( T + E )
      -> int * ( int + E )
      -> int * ( int + T )
      -> int * ( int + int )

Básicamente, lo que hemos hecho ha sido probar todas las posibles derivaciones extrema izquierda, de forma recursiva, podando inteligentemente cada vez que era evidente que habíamos producido una derivación incorrecta. Tratemos de formalizar este proceso.

### Parsing Recursivo Descendente

De forma general, tenemos tres tipos de operaciones o situaciones que analizar:
* La expansión de un no-terminal en la forma oracional actual.
* La prueba recursiva de cada una de las producciones de este no-terminal.
* La comprobación de que un terminal generado coincide con el terminal esperado en la cadena.

Para cada una de estas situaciones, vamos a tener un conjunto de métodos. Para representar la cadena a reconocer, necesitamos mantener un estado global que indique la parte reconocida de la cadena. Diseñemos una clase para ello:

```csharp
interface IParser {
    bool Parse(Token[] tokens);
}

class RecursiveParser : IParser {
    Token[] tokens;
    int nextToken;

    //...
}
```

Para reconocer un terminal, tendremos un método cuya función es a la vez decidir si se reconocer el terminal, y avanzar en la cadena:

```csharp
bool Match(Token token) {
    return tokens[nextToken++] == token;
}
```

A cada no-terminal vamos a asociar un método recursivo cuya función es determinar si el no-terminal correspondiente genera una sub-cadena "adecuada" del lenguaje. Por ejemplo, para el caso de la gramática anterior, tenemos los siguientes métodos:

```csharp
bool E() {
    // Parsea un no-terminal E
}

bool T() {
    // Parsea un no-terminal T
}
```

La semántica de cada uno de estos métodos es que devuelven `true` si y solo si el no-terminal correspondiente genera una parte de la cadena, comenzando en la posición `nextToken`. Tratemos de escribir el código del método `E`. Para ello, recordemos que el símbolo `E` deriva en 2 producciones: `E -> T` y `E -> T + E`. Por tanto, de forma recursiva podemos decir que `E` genera esta cadena si y solo si la genera a partir de una de estas dos producciones. El primer caso es fácil: `E` genera la cadena a partir de derivar en `T` si y solo si `T` a su vez genera dicha cadena, y ya tenemos un método para eso:

```csharp
bool E1() {
    // E -> T
    return T();
}
```

Para el segundo caso, notemos que la producción `E -> T + E` básicamente lo que dice es: necesitamos generar una cadena a partir de `E`, de forma tal que primero se genere una parte con `T`, luego se genere un `+` y luego se genere otra parte con `E`. Dado que ya tenemos todos los métodos necesarios:

```csharp
bool E2() {
    // E -> T + E
    return T() && Match(Token.Plus) && E();
}
```

Aprovechamos el operador `&&` con cortocircuito para podar lo antes posible el intento de generar la cadena, de forma que el primero de estos tres métodos que falle ya nos permite salir de esa rama recursiva. Ahora que tenemos métodos para cada producción, podemos finalmente develar el cuerpo del método `E`:

```csharp
bool E() {
    int currToken = nextToken;
    if (E1()) return true;

    nextToken = currToken;
    if (E2()) return true;

    return false;
}
```

Este método simplemente prueba cada una de las producciones en orden, teniendo cuidado de retornar `nextToken` a su valor original tras cada llamado recursivo fallido. Así mismo, podemos escribir el método asociado al símbolo `T`, basado en los métodos correspondientes a cada producción:

```csharp
bool T1() {
    // T -> int * T
    return Match(Token.Int) && Match(Token.Times) && T();
}

bool T2() {
    // T -> int
    return Match(Token.Int);
}

bool T3() {
    // T -> (E)
    return Match(Token.Open) && E() && Match(Token.Closed)
}
```

Y el método general para `T` queda así:

```csharp
bool T() {
    int currToken = nextToken;
    if (T1()) return true;

    nextToken = currToken;
    if (T2()) return true;

    nextToken = currToken;
    if (T3()) return true;

    return false;
}
```

Es posible hacer estos métodos más compactos introduciendo un nuevo método auxiliar:

```csharp
bool Reset(int pos) {
    nextToken = pos;
    return true;
}
```

Este método nos permite reescribir cada método con una sola expresión, haciendo uso del operador `||` con cortocircuito:

```csharp
bool E() {
    int n = nextToken;
    return E1() || Reset(n) && E2();
}

bool T() {
    int n = nextToken;
    return T1() || Reset(n) && T2() || Reset(n) && T3();
}
```

Finalmente, para reconocer la cadena completa, solo nos queda garantizar que se hayan consumido todos los *tokens*:

```csharp
bool Parse(Token[] tokens) {
    return E() && nextToken == tokens.Length;
}
```

Esta metodología para crear parsers recursivos descendentes puede ser aplicada fácilmente a cualquier gramática libre del contexto. Sin embargo, no todas las gramáticas pueden ser reconocidas de esta forma. Según la estructura de la gramática, es posible que es parser definido no funcione correctamente.

Por ejemplo, para gramáticas ambigüas, el parser (si termina) dará alguna de las derivaciones extrema izquierda posibles, en función del orden en que hayan sido definidas las producciones. Esto se debe al uso de operadores con cortocircuito. Es posible modificar este tipo de parsers fácilmente para generar no el primero sino todas las derivaciones extrema izquierda disponibles, simplemente reemplazando los operadores `||` más externos.

Consideremos ahora la siguiente gramática:

    S -> Sa | b

Para esta gramática, el parser recursivo descendente queda de la siguiente forma (simplificada):

```csharp
bool S() {
    int c = nextToken;
    return S() && Match(Token.A) || Reset(n) && Match(Token.B);
}
```

El problema evidente con este parser es que al intentar reconocer el símbolo `S` el algoritmo cae en una recursión infinita. Este tipo de gramáticas se denominan gramáticas con recursión izquierda, que definiremos así:

> **Definición**: Una gramática libre del contexto G=<S,N,T,P> se dice recursiva izquierda si y solo si `S -*-> Sw` (donde `w` es una forma oracional).

La forma más sencilla de las gramáticas recursivas izquierdas es cuando existe directamente una producción `S -> Sw`. A este caso le llamamos *recursión izquierda directa*. Para este caso, es posible eliminar la recursión izquierda de forma sencilla. Tomemos nuevamente la gramática anterior:

    S -> Sa | b

Es fácil ver que esta gramática general el lenguaje `ba*`. Otra gramática que genera dicho lenguaje sin recursión izquierda es:

    S -> bX
    X -> aX | epsilon

Aún cuando `a` y `b` son formas oracionales en general, y no simplemente terminales, el patrón anterior es válido. De forma general, si una gramática tiene recursión izquierda de la forma:

    S -> Sa1 | Sa2 | ... San | b1 | b2 | ... | bm

Es posible eliminar la recursión izquierda con la transformación:

    S -> b1X | b2X | ... | bmX
    X -> a1X | a2X | ... | anX | epsilon

Para el caso más general de recursión izquierda indirecta, también existe un algoritmo para su eliminación, que no presentaremos aquí :(

### Parsing Predictivo Descendente

### Gramáticas LL(1)