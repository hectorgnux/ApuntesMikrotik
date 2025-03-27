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
