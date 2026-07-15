# Método de edición masiva sobre un sitio en producción

Este es el protocolo con el que se editaron **840 páginas HTML sin romper una sola**.
Cero JSON-LD rotos, cero archivos corrompidos, cero rollbacks. Pégaselo entero a la
otra sesión.

---

## LAS 4 REGLAS QUE NO SE NEGOCIAN

**1. Tú no pusheas. Nunca.**
Carlos ejecuta todo desde su PowerShell en Windows. Tú construyes el script, se lo das,
él lo corre, te pega la salida, tú la lees. Él decide cuándo hace `git commit` y
`git push`. Si tocas `.git` desde el sandbox dejarás un `index.lock` colgado y le
bloquearás el commit.

**2. El mount de bash MIENTE.**
El sandbox de Linux sirve copias truncadas o rancias de los archivos recién escritos.
Llegó a reportar "804 archivos truncados" y "git index corrupt" — **todo falso**.
- Para **leer el repo**: usa las herramientas `Read` y `Grep` (van contra Windows).
- Para la **verdad histórica**: `git show HEAD:ruta/archivo`.
- Para **verificar de verdad**: que Carlos corra el script en su Python de Windows.
- `bash` sirve para computar y para analizar salidas, **no para leer el repo**.

**3. Mide antes de tocar. Siempre.**
Nada de "asumo que todos los archivos tienen la misma estructura". Cuéntalos. Grepea
el patrón exacto. Mira 2 o 3 con tus ojos. Si el ancla no es idéntica en el 100% de
los casos, **no es un ancla**.

**4. Sin sed, sin awk, sin reemplazos a ciegas.**
Solo Python con blindajes. Quirúrgico, no a machetazos.

---

## EL FLUJO, PASO A PASO

### Paso 1 — Reconocimiento
Grepea el patrón objetivo. Cuenta ocurrencias **por archivo**. Busca las excepciones:
casi siempre hay 2 o 3 archivos que no siguen la regla, y **esos son los que te rompen
el sitio**. Encuéntralos antes, no después.

### Paso 2 — Encuentra el ancla
Un ancla es una cadena **exacta, única e idéntica** en todos los archivos objetivo.
Ejemplo real que funcionó:

```
{"@context": "https://schema.org", "@graph": [{"@type": ["TechArticle", "Article"], "@id"
```

Si el `@type` es un zoo de 8 variantes distintas, el ancla no es el `@type`: es
**"el primer nodo del `@graph`"**. Piensa en estructura, no en texto.

### Paso 3 — Escribe el script con `--hazlo`
Por defecto **DRY RUN**: imprime lo que haría y no escribe nada.
Solo escribe si recibe `--hazlo`. Sin excepciones.

```python
HAZLO = "--hazlo" in sys.argv
...
if HAZLO:
    with open(p, "w", encoding="utf-8", newline="") as f:
        f.write(despues)
```

### Paso 4 — LOS BLINDAJES. Esto es el corazón del método.

**Si un solo blindaje falla, ese archivo NO se toca, y el script dice por qué.**
Nunca se escribe "por si acaso". Nunca se fuerza.

```python
# 1. La cadena objetivo solo puede vivir DENTRO de la región que quieres cambiar.
#    Si aparece en un href, en texto visible o en un <script> de JS -> se salta.
bloques = RE_LD.findall(antes)
dentro = sum(b.count(objetivo) for b in bloques)
if dentro != antes.count(objetivo):
    saltar("hay ocurrencias FUERA del ld+json")

# 2. El JSON tiene que parsear ANTES.  (Si ya estaba roto, no es culpa tuya:
#    detéctalo y no lo empeores.)
j_antes = [json.loads(b) for b in bloques]

# 3. El JSON tiene que parsear DESPUÉS, y el número de bloques no puede cambiar.
j_desp = [json.loads(b) for b in RE_LD.findall(despues)]

# 4. NADA fuera de la región objetivo puede cambiar. Comparación exacta.
if RE_LD.sub("", antes) != RE_LD.sub("", despues):
    saltar("cambiaría algo FUERA del ld+json")

# 5. EL BLINDAJE QUE LO CAZA TODO — comparación semántica.
#    Normaliza el JSON original aplicando EXACTAMENTE los cambios que pretendes,
#    y exige que el resultado sea idéntico al JSON del archivo modificado.
#    Si tu regex tocó una coma de más, un campo vecino o un nodo equivocado,
#    esto lo detecta. Es el que te salva la vida.
if [norm(j) for j in j_antes] != j_desp:
    saltar("el JSON no coincide — cambiaría ALGO MÁS")
```

**El blindaje 5 es el que hace que el método funcione.** No confía en tu regex:
verifica el *resultado*. Con eso, una regex mala no rompe nada — solo hace que el
archivo se salte y te lo diga.

### Paso 5 — Dry-run, y lee la salida de verdad
Cuenta los archivos. **Los saltados no son ruido: son la información.**

En la sesión real, los saltados revelaron:
- Un `<script>` de JSON minificado en una línea que la regex no contemplaba
  → **190 archivos saltados**, regex arreglada, 0 saltados en el siguiente intento.
- Un `@type` que era una lista y no un string → `TypeError`, una línea de arreglo.
- Un delimitador de nodos que escaneaba hacia atrás y estaba mal
  → **15 de 15 saltados**, reescrito hacia delante, 15 de 15 OK.

**Ninguno de esos bugs escribió un solo byte.** Ese es el punto.

### Paso 6 — `--hazlo`, y luego un VERIFICADOR aparte
Un segundo script que **solo lee**. Nunca escribe, jamás. Debe comprobar:
- lo que hiciste (¿está el cambio?)
- lo que NO podía romperse (¿JSON válido? ¿0 bytes NUL? ¿0 restos del valor viejo?)
- **lo que prometiste no tocar** (esto es tan importante como lo anterior)

Ejemplo real de la salida del verificador:

```
  JSON-LD ROTO            : 0     <- tiene que ser 0
  Restos del @id viejo    : 0     <- tiene que ser 0
  Enlaces a Wikipedia     : 43    <- deben seguir ahí (prometí no tocarlos)
  Wikidata temáticos vivos: 9     <- deben seguir ahí
```

### Paso 7 — Auditoría EN VIVO contra el DOM real
El código no es la verdad. **La verdad es lo que sirve el servidor.**
Tras el push, con las herramientas de Chrome: haz `fetch` de una muestra de páginas,
parsea el JSON-LD del DOM real, y cuenta. Si tocaste algo con lógica (una calculadora),
**ejecuta su autotest** y comprueba que sigue pasando.

---

## PARA OPERACIONES DESTRUCTIVAS (reescribir historial, borrar en masa)

1. **Copia de seguridad primero.** `Copy-Item -Recurse`. Que Carlos confirme con
   `Test-Path` que existe **antes** de que corras nada.
2. **Verifica localmente antes de tocar el remoto.** Nunca un `--force` sin haber
   comprobado el resultado en local.
3. **Di la verdad completa sobre lo que NO se consigue.** Ejemplo: purgar el historial
   de git no elimina los objetos huérfanos que GitHub cachea, ni los clones ajenos.
   Dilo una vez, claro, antes — no después.

---

## LO QUE HACE QUE ESTO FUNCIONE

**No es que aciertes a la primera. Es que fallar sea gratis.**

El dry-run y los blindajes convierten cada error en un mensaje en pantalla en vez de en
840 archivos corrompidos. Se hicieron 3 o 4 intentos en varios scripts. Ninguno escribió
nada hasta que la salida cuadró al 100%.

**Y cuando te equivoques —te vas a equivocar— dilo de frente.** En la sesión real hubo
una falsa alarma ("¡la calculadora de Trail está en CC BY!" — era el logo, no la
calculadora). Se dijo en cuanto se supo, sin rodeos. La confianza vale más que parecer
infalible: si no puede fiarse de tus alarmas, no puede fiarse de tus verificaciones.
