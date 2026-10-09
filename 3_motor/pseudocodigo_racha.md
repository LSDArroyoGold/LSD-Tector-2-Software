# Pseudocódigo: ventanas, decisión por racha y salidas

Este documento baja a pseudocódigo la sección 4 y 5 del [README del motor](README.md): qué hace el hilo de
clasificación con cada evento. Lo que pasa adentro de la red en cada ventana está en
[`../4_red/README.md`](../4_red/README.md); acá se la trata como una caja que devuelve logits.

## Parámetros conceptuales

| Parámetro | Significado | Valor |
|---|---|---|
| `VENTANA` | largo de la entrada de la red | 3 s |
| `PASO` | avance entre ventanas | 1 s (solapamiento de 2 s) |
| `UMBRAL` | confianza mínima de la racha ganadora | 0,5 |
| `SENS` | factor de sensibilidad de la sigmoide | ajustado con un barrido sobre audio de campo |
| `RACHA_DEBIL` | largo máximo de una racha considerada evidencia débil | 2 ventanas |

## Preparado una sola vez, al arrancar

```
etiquetas      ← lista de clases del modelo ("nombre científico_nombre común")
es_no_ave[c]   ← verdadero si la clase c está en la lista de no-aves (101 clases)
                  (si la lista nombra una clase inexistente: error de arranque)
Δ[c]           ← término regional de la provincia del equipo, uno por clase
                  (0 si no hay datos; ver 4_red/filtro_regional.md)
red            ← intérprete del .tflite (ya trae las neuronas corregidas)
```

## Una ventana

```
función PUNTAJES(ventana_de_audio):
    x ← ventana, completada con ceros hasta VENTANA si es más corta
    z ← red(x)                              # un logit por clase
    z ← z + Δ                               # filtro regional, sumado al logit
    p ← sigmoide(SENS_escala · z)           # cada clase por separado, independiente de las otras
    p[c] ← 0 para toda c con es_no_ave[c]   # las no-aves nunca ganan
    devolver p
```

## Todas las ventanas de un evento

```
función VENTANAS(evento):
    resultado ← lista vacía
    para inicio = 0, PASO, 2·PASO, ... mientras quede al menos ~1 s de audio por delante:
        p ← PUNTAJES(evento[inicio : inicio + VENTANA])
        ganadora ← argmax(p)
        agregar (ganadora, p[ganadora]) a resultado
    devolver resultado
```

No se aplica ningún umbral por ventana: el umbral se aplica una sola vez, al final, sobre la racha.

## Decisión por racha

Una **racha** es un tramo maximal de ventanas consecutivas con la misma especie ganadora.

```
función DECIDIR(ventanas):
    mejor ← diccionario vacío            # especie → (confianza de su mejor racha, largo)
    i ← 0
    mientras i < largo(ventanas):
        especie ← ventanas[i].ganadora
        j ← i
        mientras j < largo(ventanas) y ventanas[j].ganadora = especie: j ← j + 1
        L ← j − i
        # promedio ponderado por posición: la k-ésima ventana de la racha pesa k
        conf ← Σ_{k=1..L} k · ventanas[i+k−1].confianza  /  Σ_{k=1..L} k
        si especie no está en mejor, o conf > mejor[especie].confianza:
            mejor[especie] ← (conf, L)
        i ← j

    si mejor está vacío: devolver "sin ventanas analizables"
    ganadora ← la especie con mayor confianza en mejor
    (conf, L) ← mejor[ganadora]
    si conf < UMBRAL: devolver NO_DETECTADO(ganadora, conf)          # sólo se registra
    débil ← (L ≤ RACHA_DEBIL) y (L < cantidad de ventanas del evento)
    devolver DETECTADO(ganadora, conf, largo_racha = L, débil)
```

### Ejemplo cualitativo

```
ventana:   1      2      3      4      5      6      7
ganadora:  A      B      B      B      B      A      C
            \___/ \__________________/ \___/  \___/
racha:      A,1        B,4             A,1    C,1
```

La racha de B (cuatro ventanas) se resume dando más peso a sus últimas ventanas; la de A no se suma con la
otra A porque no son consecutivas. Si B supera 0,5, gana B con evidencia firme. Si en cambio hubiera ganado
una racha de una sola ventana dentro de este evento de siete, sería evidencia débil.

### Por qué así

- **Rachas y no votos.** Un ave cantando sostiene la misma respuesta en ventanas solapadas; una confusión
  puntual (un golpe, otra especie que pasa, un fragmento ambiguo) suele aparecer en una ventana aislada.
- **Ponderar por posición.** Premia que la racha se sostenga: el promedio de una racha larga queda dominado por
  su tramo maduro, no por la primera ventana, que muchas veces agarra sólo el comienzo del canto.
- **Umbral al final.** Un umbral por ventana descartaría información antes de juntarla.
- **Evidencia débil y no descarte.** Una racha corta en un evento largo es dudosa pero no inútil: el audio se
  guarda y se sube (alguien lo puede escuchar en Tector Hub), pero no se publica como identificación firme.
  Si el evento entero tiene una o dos ventanas, la racha corta es todo lo que hay y no se marca como débil.

## Procesar un evento (hilo de clasificación)

```
procedimiento HILO_CLASIFICACION():
    repetir siempre:
        ev ← sacar de la cola (espera bloqueante)
        intentar:
            PROCESAR_EVENTO(ev)
        si falla: registrar el error y seguir


procedimiento PROCESAR_EVENTO(ev):
    si ev.hora_inicio es nada o duración(ev.audio) < 1 s: terminar
    r ← DECIDIR(VENTANAS(ev.audio))
    si r es "sin ventanas" o NO_DETECTADO:
        registrar en el log del motor (con el mejor candidato y su confianza); terminar

    (nombre, carpeta) ← NOMBRE_ARCHIVO(r, ev.hora_inicio)
    escribir mp3(ev.audio) en carpeta/nombre
    registrar la detección

    si no r.débil:
        intentar publicar en BirdWeather(r, ev.hora_inicio, ev.audio)
        si falla: alerta al log del sistema
    si no:
        registrar "BirdWeather salteado por evidencia débil"

    si no se pudo subir(carpeta/nombre) al servidor dentro del tiempo límite:
        alerta al log del sistema          # sin reintento; el mp3 queda en el equipo


función NOMBRE_ARCHIVO(r, hora):
    especie ← nombre común sin apóstrofes y con espacios como guiones bajos
    nombre  ← especie - round(100·r.confianza) - fecha(hora) - hh:mm:ss(hora) [ + marca débil ] .mp3
    carpeta ← por fecha / por especie
    devolver (nombre, carpeta)
```
