# Ataque MAC Flooding

> **Estudiante:** Emmanuel Báez Ramírez
> **Matrícula:** 2022-0375
> **Video ilustrativo:** https://youtu.be/_l7GSrS0wds
> **Playlist (Práctica 1):** https://www.youtube.com/playlist?list=PLp7pfUFf22-zekAmQ7hncCvmJHZe7lLHk

---

# Objetivo del laboratorio.
Demostrar de forma controlada un ataque MAC Flooding contra un switch Cisco, saturando su tabla de direcciones MAC (tabla CAM) para forzar un comportamiento inseguro, e implementar y verificar la contramedida que lo mitiga.

# Objetivo del script.
Inundar la tabla CAM del switch con miles de tramas, cada una con una dirección MAC de origen aleatoria y distinta. El switch aprende cada MAC y la almacena, hasta agotar el espacio de la tabla. Cuando la tabla se llena, algunos switches entran en modo "fail-open" y comienzan a difundir el tráfico por todos los puertos como un hub, lo que permitiría interceptar comunicaciones de otros equipos.

# Parámetros usados.
Comando de ejecución:

    sudo python3 ./mac_flooding.py -i eth1 -n 1000

- `-i eth1` : interfaz de red del atacante.
- `-n 1000` : número de tramas por ráfaga.
- `-d`      : retardo entre tramas en segundos (opcional, por defecto 0).
- `-c`      : número de ráfagas (opcional, 0 = infinito).

Elementos internos del script:
- `RandMAC()` : dirección MAC de origen aleatoria por cada trama.
- `RandIP()` : direcciones IP de origen/destino aleatorias.
- Encapsulado `Ether / IP / UDP` con puertos 1234 -> 80.
- Envío masivo por ráfagas mediante `sendp()`.

# Requisitos para utilizar la herramienta.
- Python3
- Scapy
- Permisos root
- Estar conectado al switch víctima por LAN (puerto de acceso).

# Documentación del funcionamiento del script.
El script genera tramas Ethernet en ráfagas. Por cada trama crea una dirección MAC de origen aleatoria (`RandMAC()`) y direcciones IP aleatorias, encapsuladas en UDP. Cada trama se envía por la interfaz del atacante. El switch, al recibir tramas con MACs de origen siempre distintas, las va aprendiendo y almacenando en su tabla CAM, asociadas al puerto del atacante. Como las MACs nunca se repiten, la tabla crece sin límite hasta saturarse. El script imprime el progreso por ráfagas y corre hasta detenerse con Ctrl+C.

# Documentación de la Red.
Topología en VLAN 130 / 10.3.75.128/25.

| Dispositivo | Interfaz | VLAN | Dirección IP | Rol |
|-------------|----------|------|--------------|-----|
| R1 (c3725) | Fa0/0 | 130 | 10.3.75.129 | Gateway / Servidor DHCP |
| SW1 (vIOS-L2) | — | 130 | — | Switch de acceso (víctima) |
| Kali (atacante) | eth1 -> Gi0/2 | 130 | 10.3.75.137 | Atacante |
| PC1 (VPCS) | -> Gi0/1 | 130 | 10.3.75.130 | Cliente / víctima |

Imágenes usadas:
- `vios_l2-adventerprisek9-m.vmdk.SSA.152-4.0.55.E` (switch SW1)
- `c3725-adventerprisek9-mz.124-15.T14` (router R1)

# Capturas de pantalla.
- Topología (nombre y matrícula)



<img width="426" height="278" alt="mac_topologia" src="https://github.com/user-attachments/assets/d84ed783-89a2-44c8-9eb8-943531bd2c44" />

- Ejecución del script en Kali

<img width="388" height="157" alt="mac_ejecucion" src="https://github.com/user-attachments/assets/6495294b-2322-479a-80cb-532076ef78b6" />

- Tabla MAC del switch antes del ataque (pocas entradas)

<img width="396" height="228" alt="mac_tabla_977" src="https://github.com/user-attachments/assets/481875ec-7ff8-40e4-8963-e6e82c08e94a" />

- Tabla MAC saturándose durante el ataque (progresión 977 -> 2445 -> 2943 entradas)

<img width="395" height="192" alt="mac_tabla_2445" src="https://github.com/user-attachments/assets/40578c4c-8b7c-43e1-9586-e985cea53be5" />

<img width="282" height="190" alt="mac_tabla_2943" src="https://github.com/user-attachments/assets/6c7d40f3-f5b9-4a7b-a131-0078d443b181" />

- Avalancha de tramas con MACs aleatorias capturada en Wireshark

<img width="633" height="481" alt="mac_wireshark" src="https://github.com/user-attachments/assets/77107f27-a402-4a35-8e9b-4ff497c0c760" />

# Documentación de contra-medidas.
La mitigación es Port Security, que limita cuántas direcciones MAC puede aprender el puerto del atacante. Al superar el límite, el switch detecta la violación y bloquea el puerto.

    SW1# configure terminal
    SW1(config)# interface GigabitEthernet0/2
    SW1(config-if)# switchport port-security
    SW1(config-if)# switchport port-security maximum 2
    SW1(config-if)# switchport port-security mac-address sticky
    SW1(config-if)# switchport port-security violation shutdown
    SW1(config-if)# exit

Verificación:

    SW1# show port-security interface GigabitEthernet0/2
    SW1# show mac address-table count
    SW1# show interfaces status

Con Port Security activo, al intentar inyectar más de 2 MACs el puerto Gi0/2 entra en estado err-disabled (secure-shutdown): el atacante queda aislado y la tabla CAM ya no se satura.

Otras buenas prácticas:
- Configurar Port Security en todos los puertos de acceso.
- Usar direcciones MAC sticky para registrar las MAC legítimas.
- Monitorear el tamaño de la tabla de direcciones MAC.
