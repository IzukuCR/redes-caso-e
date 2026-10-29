# Bitácora de incidencias

Una entrada por problema real encontrado durante el desarrollo. El informe pide al menos dos, con diagnóstico y solución.

## Plantilla

```
### INC-NN - Título corto
- Fecha:
- Quién:
- Equipo(s):
- Síntoma: qué se observó (comando y salida)
- Diagnóstico: qué se revisó y en qué orden
- Causa:
- Solución: qué se cambió (commit o PR)
- Prueba que lo confirma: ID de la matriz
```

## Incidencias

### INC-01 - El FortiGate exige cambiar la clave en el primer ingreso
- Fecha: 2026-10-28
- Quién: Roy Arias RoyG247
- Equipo(s): FGT-01
- Síntoma: `uv run gns3ctl.py todo` se detuvo en el paso `licenciar` con "El FortiGate pide cambiar la clave en el primer ingreso: hacelo a mano una vez y reintenta."
- Diagnóstico: en la consola de FGT-01, al entrar como `admin` con la clave vacía (valor de fábrica), FortiOS respondió "New Password:". En `automatizacion/maqueta/bordes/fortigate_cli.py`, el script entra con la clave de `topologia/proyecto.yml` (`Fortinet.2026`) y se detiene a propósito si ve ese aviso.
- Causa: FortiOS obliga a cambiar la clave de `admin` en el primer ingreso de una instancia nueva, y el script no elige claves por su cuenta.
- Solución: cambiar la clave a mano por consola a `Fortinet.2026` (la de `proyecto.yml`) y correr solo `uv run gns3ctl.py licenciar`. Sin cambios en el repo: es un paso manual de la primera vez.
- Prueba que lo confirma: `licenciar` terminó con "licencia despues del reinicio: Valid"; habilita P01–P07.

### INC-02 - R-LAN rechaza SSH desde la VLAN 20
- Fecha: 2026-10-28
- Quién: Roy Arias RoyG247
- Equipo(s): R-LAN, PC-ADMIN
- Síntoma: en `pruebas/verificar_maqueta.py`, E12b falló: desde PC-ADMIN, `timeout 3 bash -c '</dev/tcp/10.50.1.1/22'` devolvió "Connection refused".
- Diagnóstico: "Connection refused" (y no un timeout) indica que R-LAN recibe la conexión pero no tiene SSH escuchando. Se descartó la ACL-VTY-ADMIN, porque PC-ADMIN (10.50.1.10) está permitido por su línea 10. Al correr `construir`, el script había avisado: "R-LAN: ejecutar por consola ya encendido: crypto key generate rsa modulus 2048".
- Causa: las claves RSA no forman parte de la configuración de texto. `construir` carga `configs/R-LAN.txt` como startup-config, y `crypto key generate rsa` no se ejecuta desde ahí, así que el router arrancó sin claves y con SSH deshabilitado.
- Solución: por consola de R-LAN: `crypto key generate rsa modulus 2048`, `ip ssh version 2` y `write memory`. Sin cambios en el repo: es un paso manual cada vez que se construye la maqueta desde cero.
- Prueba que lo confirma: E12 (E12b responde "ssh_rlan_abierto"); evidencia `maqueta-20261028-1814.txt`.

### INC-03 - R01 y F01 fallan justo después de encender la maqueta
- Fecha: 2026-10-28
- Quién: Roy Arias RoyG247
- Equipo(s): FGT-01
- Síntoma: `pruebas/verificar_maqueta.py` dio 25/27: R01 falló con la salida vacía y F01 dio "reactivada: http4=000", mientras todas las pruebas de tráfico pasaban.
- Diagnóstico: la salida vacía indica que el FortiGate no respondió a los comandos, no que OSPF estuviera caído. En la misma corrida, F02 mostró a R-LAN (5.5.5.2) en estado Full y la publicación externa en http4=200. La prueba había empezado unos 2 minutos después de `gns3ctl.py iniciar`.
- Causa: el FortiGate ya reenviaba tráfico, pero su CLI por consola todavía no respondía a tiempo después del arranque.
- Solución: esperar unos 3 minutos después de `iniciar`, cerrar las consolas de FGT-01 abiertas en GNS3 y volver a correr. Sin cambios en el repo.
- Prueba que lo confirma: R01 y F01 en OK; evidencia `maqueta-20261028-1814.txt` (27/27).