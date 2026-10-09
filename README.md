# LSD-Tector 2: software

Documentación técnica del software de **Tector 2**, una estación autónoma de monitoreo acústico pasivo de aves
desarrollada en el Laboratorio de Sistemas Dinámicos (LSD), Departamento de Física, FCEyN, Universidad de Buenos Aires.

**Autores:** Diego Arroyo y Tomás Gold. **Director:** Gabriel Mindlin.

Tector 2 graba el paisaje sonoro en ventanas de amanecer y atardecer, detecta vocalizaciones de aves con una red
neuronal (BirdNET) adaptada a la avifauna local, y envía cada detección a una plataforma web, **Tector Hub**
([tectorhub.com](https://tectorhub.com)), desde donde los usuarios escuchan los cantos, consultan estadísticas y
administran los equipos a distancia. Todo funciona sin intervención humana, alimentado por un panel solar.

Este repositorio acompaña al informe de Laboratorio 6/7 de Tector 2. Describe **cómo está organizado todo el
software**: qué sistemas existen, qué hace cada uno, cómo se comunican entre sí y cómo funcionan sus algoritmos
principales, con diagramas y pseudocódigo. No contiene el código fuente de los equipos ni del servidor.

## Mapa del repositorio

| Carpeta | Contenido |
|---|---|
| [`1_arquitectura`](1_arquitectura/) | Visión general: la cadena de decisiones, los tres sistemas, principios de diseño y un día del equipo. **Empezar acá.** |
| [`2_dispositivo`](2_dispositivo/) | La capa que corre en la Raspberry Pi y gobierna el ciclo de vida del equipo: puesta en marcha, ventanas, energía, red, configuración remota y tolerancia a fallas. |
| [`3_motor`](3_motor/) | El motor de detección (TectorNET-Pi): grabación continua, disparador, eventos, decisión por racha, envío y autoactualización. |
| [`4_red`](4_red/) | La red neuronal por dentro y sus dos mejoras: corrección puntual de confusiones (regresión logística binaria) y filtro regional. |
| [`5_hub`](5_hub/) | Tector Hub: servidor, aplicación web, usuarios y equipos, reporte de audios mal etiquetados, distribución de actualizaciones. |
| [`6_interfaces`](6_interfaces/) | Los contratos entre sistemas: carpetas, nombres de archivo, estado del equipo y archivos de configuración. |

## Lectura sugerida

- Para entender el sistema completo en diez minutos: [`1_arquitectura`](1_arquitectura/).
- Para entender cómo se detecta un ave: [`3_motor`](3_motor/) y luego [`4_red`](4_red/).
- Para integrar algo con Tector 2 o entender qué datos produce: [`6_interfaces`](6_interfaces/).

## Cómo citar

D. Arroyo, T. Gold, *LSD-Tector 2: software* [documentación técnica], Laboratorio de Sistemas Dinámicos,
FCEyN-UBA (2026). https://github.com/LSDArroyoGold/LSD-Tector-2-Software

## Licencia

CC BY-NC-ND 4.0. Ver [`LICENSE.md`](LICENSE.md).
