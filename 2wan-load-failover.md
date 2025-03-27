# Configuración de Balanceo y Failover en Mikrotik RouterOS v7 con 2 WANs

Este documento detalla cómo configurar un balanceo de carga con failover en Mikrotik RouterOS v7 usando dos conexiones WAN (`ether1` y `ether2`) y una LAN (`ether3`). También incluye pasos de depuración para el problema "el tráfico pasa por mangle pero no sale por las rutas".

## Escenario
- **WAN1**: Conexión en `ether1` (ejemplo: DHCP o PPPoE).
- **WAN2**: Conexión en `ether2` (ejemplo: DHCP o PPPoE).
- **LAN**: Red `192.168.1.0/24` en `ether3`.
- **Objetivo**: Balancear tráfico entre WAN1 y WAN2 con failover automático.

---

## Configuración Inicial

### Paso 1: Configurar Interfaces WAN

#### Para DHCP:
```
/ip dhcp-client
add interface=ether1 add-default-route=no
add interface=ether2 add-default-route=no
```

#### Para PPPoE:
```
/ip address
add address=192.168.1.1/24 interface=ether3
/ip pool
add name=dhcp_pool ranges=192.168.1.2-192.168.1.254
/ip dhcp-server
add name=dhcp1 interface=ether3 address-pool=dhcp_pool lease-time=10m
/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8
```


### Paso 3: Crear Tablas de Enrutamiento
```
/routing table
add name=to_wan1 fib
add name=to_wan2 fib
```

### Paso 4: Configurar Rutas Recursivas para Failover
Suponiendo:
- Gateway WAN1: `10.0.0.1`
- Gateway WAN2: `20.0.0.1`
```
/ip route
add dst-address=8.8.8.8 gateway=10.0.0.1 scope=10
add dst-address=8.8.4.4 gateway=20.0.0.1 scope=10
add gateway=8.8.8.8 routing-table=to_wan1 distance=1 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.4.4 routing-table=to_wan1 distance=2 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.4.4 routing-table=to_wan2 distance=1 scope=30 target-scope=10 check-gateway=ping
add gateway=8.8.8.8 routing-table=to_wan2 distance=2 scope=30 target-scope=10 check-gateway=ping
```

### Paso 5: Configurar Balanceo con PCC
```
/ip firewall mangle
add chain=prerouting in-interface=ether1 action=mark-connection new-connection-mark=wan1_conn passthrough=yes
add chain=prerouting in-interface=ether2 action=mark-connection new-connection-mark=wan2_conn passthrough=yes
add chain=prerouting dst-address-type=!local in-interface=ether3 per-connection-classifier=both-addresses:2/0 action=mark-connection new-connection-mark=wan1_conn passthrough=yes
add chain=prerouting dst-address-type=!local in-interface=ether3 per-connection-classifier=both-addresses:2/1 action=mark-connection new-connection-mark=wan2_conn passthrough=yes
add chain=prerouting connection-mark=wan1_conn in-interface=ether3 action=mark-routing new-routing-mark=to_wan1 passthrough=yes
add chain=prerouting connection-mark=wan2_conn in-interface=ether3 action=mark-routing new-routing-mark=to_wan2 passthrough=yes
```

### Paso 6: Configurar NAT
```
/ip firewall nat
add chain=srcnat out-interface=ether1 action=masquerade
add chain=srcnat out-interface=ether2 action=masquerade
```

---


