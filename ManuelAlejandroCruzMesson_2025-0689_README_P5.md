# VPN Site-to-Site IKEv2 Basada en Enrutamiento (VTI)

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Cruz
**Docente:** Jonathan Rondón
**Fecha:** 2 de julio de 2026

---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)](#3-especificación-de-políticas-y-parámetros-de-seguridad-ikev2)
- [4. Configuración de la Interfaz de Túnel Virtual (VTI)](#4-configuración-de-la-interfaz-de-túnel-virtual-vti)
- [5. Puntos Críticos Analizados](#5-puntos-críticos-analizados)
- [6. Protocolo de Verificación y Diagnóstico Técnico](#6-protocolo-de-verificación-y-diagnóstico-técnico)

---

## 1. Resumen y Objetivos

Este documento describe la evolución tecnológica y la reestructuración del laboratorio original de VPN Site-to-Site. Se migra la arquitectura basada en políticas (Crypto Maps) hacia una arquitectura basada en rutas (**Route-Based VPN**), mediante el uso de **Interfaces de Túnel Virtual (VTI)** sobre el protocolo **IKEv2**, manteniendo intacto el direccionamiento IP de la infraestructura física del proyecto original.

### Objetivos del Proyecto

- **Modernización de la Arquitectura de Reenvío:** reemplazar la lógica de intercepción de tráfico por Crypto Maps con una interfaz `Tunnel0` lógica, permitiendo un enrutamiento dinámico y estático nativo hacia la VPN.
- **Confidencialidad e Integridad Avanzada:** garantizar la protección estricta del flujo de datos entre la LAN de PEER A (`172.16.1.0/24`) y la LAN de PEER B (`172.16.2.0/24`) mediante algoritmos de cifrado simétrico robustos de extremo a extremo (AES-256 / SHA-256).
- **Coexistencia de Servicios:** asegurar el correcto funcionamiento del enrutamiento estático hacia la interfaz `Tunnel0` y el aislamiento criptográfico frente a la traducción de direcciones de red (NAT/PAT) mediante listas de acceso ajustadas.

---

## 2. Topología de Red y Direccionamiento

La topología lógica mantiene el diseño de dos sedes principales interconectadas a través de un gateway WAN que emula un proveedor de servicios de Internet (R-ISP). Se añade la interfaz lógica `Tunnel0` en cada extremo como punto de encapsulamiento del túnel IPsec.

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER A | Tunnel0 | 192.168.100.1 | 255.255.255.252 | Interfaz Túnel VPN Ruta-Basada |
| PEER B | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| PEER B | Tunnel0 | 192.168.100.2 | 255.255.255.252 | Interfaz Túnel VPN Ruta-Basada |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada WAN |

---

## 3. Especificación de Políticas y Parámetros de Seguridad (IKEv2)

A diferencia de la suite basada en Crypto Maps, la nueva implementación agrupa las directivas criptográficas de forma modular y las asocia a una interfaz de túnel virtual mediante un perfil IPsec.

### Fase 1: IKEv2 Proposal, Policy, Keyring y Profile — PEER A

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES-CBC-256 (Advanced Encryption Standard con encadenamiento de bloques de cifrado de 256 bits) |
| Integridad y Hash | SHA-256 (Secure Hash Algorithm de 256 bits para procesos de autenticación e intercambio seguro) |
| Grupo Diffie-Hellman | Grupo 14 (Exponenciación modular de 2048 bits para la generación segura de claves efímeras) |
| Autenticación | Llaves precompartidas (pre-share) mediante contenedores locales (keyring), con identidad local declarada por dirección IP |
| Pre-Shared Key | `cisco123!` |

### Fase 2: IPsec Transform Set y Perfil IPsec

| Parámetro | Valor |
|---|---|
| Nombre del Set | `TS_VPN` |
| Encapsulación Criptográfica | ESP con AES de 256 bits (esp-aes 256) |
| Autenticación/Hashing de Datos | ESP bajo código de autenticación de mensajes cifrados (esp-sha256-hmac) |
| Modo de Operación | Modo Túnel nativo (mode tunnel), encapsulando la cabecera IP original completa |
| Perfil IPsec | `IPSEC_PROFILE`, el cual vincula el transform-set con el perfil IKEv2 y se aplica directamente sobre la interfaz `Tunnel0` |

---

## 4. Configuración de la Interfaz de Túnel Virtual (VTI)

El elemento central de la migración es la interfaz `Tunnel0`, la cual actúa como una interfaz lógica de reenvío IP. El tráfico enrutado hacia esta interfaz es automáticamente cifrado y encapsulado conforme al perfil IPsec asociado.

- **PEER A** — Interfaz Tunnel0 y Enrutamiento
- **PEER B** — Interfaz Tunnel0 y Enrutamiento

---

## 5. Puntos Críticos Analizados

### A. Enrutamiento hacia una Interfaz Lógica (Route-Based VPN)

A diferencia de una VPN basada en políticas (Crypto Maps), donde el motor criptográfico intercepta el tráfico saliente por la interfaz física WAN, en este modelo basado en rutas el router simplemente enruta los paquetes destinados a la LAN remota hacia la interfaz `Tunnel0` mediante una ruta estática. Es la propia interfaz `Tunnel0`, protegida por el perfil IPsec, la que se encarga de cifrar y encapsular el tráfico de forma transparente antes de reenviarlo por la interfaz física de salida (Gi0/0).

Esta arquitectura resulta más escalable y compatible con protocolos de enrutamiento dinámico, ya que la interfaz `Tunnel0` se comporta como cualquier otra interfaz IP del router.

---

## 6. Protocolo de Verificación y Diagnóstico Técnico

### Paso 1: Generación de Tráfico Interesante para Levantamiento de Túnel

Por definición, los mecanismos IPsec levantan los túneles de comunicación bajo demanda en presencia del primer paquete de datos legítimo. Para activar la infraestructura, se emite una solicitud de eco ICMP desde un host terminal interno, forzando el direccionamiento de la LAN remota.

### Paso 2: Validación del Canal de Control en Fase 1 (IKEv2 SA)

Para examinar el estado del intercambio de llaves y la correcta concordancia criptográfica de los peers remotos, se emplea el comando de diagnóstico avanzado para IKEv2:

```
Router# show crypto ikev2 sa
```

**Criterio de Aceptación:** el resultado en consola debe declarar de forma explícita el parámetro **READY** en la columna de estado. Esto certifica que la autenticación mutua mediante las claves precompartidas y los perfiles asignados ha culminado con éxito.

### Paso 3: Validación del Canal de Datos en Fase 2 (IPsec SA)

Una vez establecido el enlace de control, es indispensable auditar que los flujos de datos de usuario sean procesados por los algoritmos de encriptación simétrica ESP:

```
Router# show crypto ipsec sa | include pkts
```

Los registros y contadores del sistema para `#pkts encaps` (paquetes cifrados salientes) y `#pkts decaps` (paquetes descifrados entrantes) deben mostrar valores enteros mayores a cero e incrementar en tiempo real conforme persista el envío de ráfagas de datos.
