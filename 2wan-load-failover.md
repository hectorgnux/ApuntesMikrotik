Configuración de Balanceo y Failover en Mikrotik RouterOS v7 con 2 WANs
Este documento describe cómo configurar un balanceo de carga con failover en Mikrotik RouterOS v7 utilizando dos conexiones WAN (ether1 y ether2) y una LAN (ether3). También incluye pasos de depuración para el problema "tráfico pasa por mangle pero no sale por las rutas".
Escenario
WAN1: Conexión en ether1 (ejemplo: DHCP o PPPoE).

WAN2: Conexión en ether2 (ejemplo: DHCP o PPPoE).

LAN: Red 192.168.1.0/24 en ether3.

Objetivo: Balancear tráfico entre WAN1 y WAN2 con failover automático.

Configuración Inicial
Paso 1: Configurar Interfaces WAN
Para DHCP:
bash

/ip dhcp-client
add interface=ether1 add-default-route=no
add interface=ether2 add-default-route=no

Para PPPoE:
bash

/interface pppoe-client
add interface=ether1 name=pppoe-out1 user="usuario1" password="contraseña1" add-default-route=no
add interface=ether2 name=pppoe-out2 user="usuario2" password="contraseña2" add-default-route=no

Paso 2: Configurar LAN
bash

/ip address
add address=192.168.1.1/24 interface=ether3

/ip pool
add name=dhcp_pool ranges=192.168.1.2-192.168.1.254

/ip dhcp-server
add name=dhcp1 interface=ether3 address-pool=dhcp_pool lease-time=10m

/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8

Paso 3: Crear Tablas de Enrutamiento
bash

/routing table
add name=to_wan1 fib
add name=to_wan2 fib

Paso 4: Configurar Rutas Recursivas para Failover
Suponiendo:
Gateway WAN1: 10.0.0.1

Gateway WAN2: 20.0.0.1

bash

/ip route
add dst-address=8.8.8.8 gateway=10.0.0.1 scope=10
add dst-address=8.8.4.4 gateway=20.0.0.1 scope=10
add gateway=8.8.8.8 routing-table=to_wan1 distance=1 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.4.4 routing-table=to_wan1 distance=2 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.4.4 routing-table=to_wan2 distance=1 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.8.8 routing-table=to_wan2 distance=2 scope=30 target-scope=10 check-gateway=ping

Paso 5: Configurar Balanceo con PCC
bash

/ip firewall mangle
add chain=prerouting in-interface=ether1 action=mark-connection new-connection-mark=wan1_conn passthrough=yes
add chain=prerouting in-interface=ether2 action=mark-connection new-connection-mark=wan2_conn passthrough=yes
add chain=prerouting dst-address-type=!local in-interface=ether3 per-connection-classifier=both-addresses:2/0 action=mark-connection new-connection-mark=wan1_conn passthrough=yes
add chain=prerouting dst-address-type=!local in-interface=ether3 per-connection-classifier=both-addresses:2/1 action=mark-connection new-connection-mark=wan2_conn passthrough=yes
add chain=prerouting connection-mark=wan1_conn in-interface=ether3 action=mark-routing new-routing-mark=to_wan1 passthrough=yes
add chain=prerouting connection-mark=wan2_conn in-interface=ether3 action=mark-routing new-routing-mark=to_wan2 passthrough=yes

Paso 6: Configurar NAT
bash

/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
add chain=srcnat out-interface=ether2 action=masquerade

Depuración: "Tráfico pasa por mangle pero no sale por las rutas"
Si las reglas de mangle cuentan paquetes pero el tráfico no sale por las WANs, sigue estos pasos:
1. Verificar Tablas de Enrutamiento
bash

/routing table print

Asegúrate de que to_wan1 y to_wan2 tengan fib=yes.

2. Revisar Rutas
bash

/ip route print

Confirma que las rutas en to_wan1 y to_wan2 estén activas (A) y no "unreachable".

Si están inactivas, prueba ping:
bash

ping 8.8.8.8 src-address=<IP_WAN1>
ping 8.8.4.4 src-address=<IP_WAN2>

3. Comprobar Marcas en Mangle
bash

/ip firewall mangle print stats

Verifica que las reglas de mark-connection y mark-routing cuenten paquetes.

Si mark-routing no cuenta, revisa in-interface y connection-mark.

4. Verificar NAT
bash

/ip firewall nat print stats

Asegúrate de que las reglas de masquerade cuenten paquetes.

Corrige out-interface si es necesario (ejemplo: pppoe-out1 para PPPoE).

5. Probar Rutas Manualmente
bash

/tool traceroute 8.8.8.8 routing-table=to_wan1
/tool traceroute 8.8.8.8 routing-table=to_wan2

Si no hay respuesta, revisa los gateways.

6. Monitorear Tráfico
bash

/tool torch interface=ether1
/tool torch interface=ether2

Si no hay tráfico, el problema está en el enrutamiento o NAT.

Solución Rápida
Si las rutas recursivas fallan, usa gateways directos:
bash

/ip route
add gateway=<gateway_WAN1> routing-table=to_wan1 distance=1
add gateway=<gateway_WAN2> routing-table=to_wan2 distance=1

Notas
Ajusta <gateway_WAN1> y <gateway_WAN2> según tu configuración (verifica con /ip route print).

Para PPPoE, reemplaza ether1 y ether2 por pppoe-out1 y pppoe-out2.

Usa src-address en PCC si both-addresses no distribuye bien el tráfico.

