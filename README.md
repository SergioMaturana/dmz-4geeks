# Informe de configuración de DMZ con Cisco Packet Tracer

## 1. Objetivo del laboratorio
El objetivo de este laboratorio es implementar y configurar una Zona Desmilitarizada (DMZ) segura utilizando un router Cisco ISR 2911 en Packet Tracer. Se busca aplicar principios de ciberseguridad defensiva mediante la configuración de NAT estático y Listas de Control de Acceso (ACLs) para controlar el flujo de tráfico entre tres zonas de red: LAN interna, DMZ y red externa (Internet simulado).

Con este ejercicio se pretende:
- Aislar servicios críticos en una DMZ para proteger la red interna
- Exponer de forma controlada un servidor web público
- Implementar políticas de seguridad mediante ACLs
- Configurar traducción de direcciones (NAT) para exponer servicios internos de manera segura

## 2. Topología implementada
**Cantidad de redes:** 3 redes segmentadas

- **Red LAN Interna:** 192.168.1.0/24 (INTERNAL NETWORK)
- **Red DMZ:** 192.168.2.0/24 (DMZ)
- **Red Externa:** 192.168.3.0/24 (EXTERNAL NETWORK - simula Internet)

**Dispositivos usados:**
- 1x Router Cisco ISR 2911 (Router_FW) - Actúa como firewall central
- 3x Switches Cisco Catalyst 2960 (SW_Internal, SW_DMZ, SW_External)
- 1x PC (PC_Internal) - Usuario de la red corporativa interna
- 1x Server-PT (Web_DMZ) - Servidor web alojado en la DMZ
- 1x PC (PC_External) - Cliente externo simulando acceso desde Internet

**Función de cada zona:**
- **LAN (INTERNAL NETWORK):** Red corporativa interna donde residen los usuarios y recursos sensibles de la organización. Esta zona debe estar protegida y no debe ser accesible directamente desde Internet o desde la DMZ.
- **DMZ (Zona Desmilitarizada):** Zona intermedia que aloja servicios que deben ser accesibles desde Internet (como el servidor web). Está aislada tanto de Internet como de la LAN interna para limitar el daño en caso de compromiso.
- **Red Externa (EXTERNAL NETWORK):** Simula Internet. Desde aquí los clientes externos intentan acceder a los servicios públicos de la organización.

## 3. Plan de direccionamiento IP

| Dispositivo | IP | Máscara | Gateway |
|---|---|---|---|
| PC_Internal | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| Server_DMZ | 192.168.2.10 | 255.255.255.0 | 192.168.2.1 |
| PC_External | 192.168.3.10 | 255.255.255.0 | 192.168.3.1 |
| Router_FW Gi0/0 (LAN) | 192.168.1.1 | 255.255.255.0 | N/A |
| Router_FW Gi0/1 (DMZ) | 192.168.2.1 | 255.255.255.0 | N/A |
| Router_FW Gi0/2 (Ext) | 192.168.3.1 | 255.255.255.0 | N/A |

**Justificación del direccionamiento:**
- Se utilizaron redes privadas Clase C (/24) para cada segmento
- Cada zona tiene su propio espacio de direccionamiento aislado
- El router actúa como gateway (.1) en cada segmento
- Los dispositivos finales tienen IPs estáticas (.10) para facilitar la administración y las pruebas

## 4. Configuración aplicada (resumen)

### 4.1 Configuración de interfaces del router
Se configuraron las tres interfaces GigabitEthernet del router Router_FW para actuar como gateway de cada segmento:

```text
Router> enable
Router# configure terminal
Router(config)# hostname Router_FW

Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip address 192.168.1.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip address 192.168.2.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip address 192.168.3.1 255.255.255.0
Router_FW(config-if)# no shutdown
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```

### 4.2 Configuración de NAT estático
Se implementó NAT estático para exponer el servidor web DMZ (192.168.2.10) a través de la IP pública 192.168.3.1:

```text
Router_FW# configure terminal

Router_FW(config)# interface GigabitEthernet0/0
Router_FW(config-if)# ip nat inside
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip nat inside
Router_FW(config-if)# exit

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip nat outside
Router_FW(config-if)# exit

Router_FW(config)# ip nat inside source static 192.168.2.10 192.168.3.1

Router_FW(config)# end
Router_FW# write memory
```
**Explicación:** Este NAT crea una traducción uno a uno donde la IP privada del servidor (192.168.2.10) es accesible desde el exterior mediante la IP pública 192.168.3.1. Cuando un cliente externo accede a 192.168.3.1, el router automáticamente redirige el tráfico al servidor en 192.168.2.10.

### 4.3 Configuración de ACLs

#### ACL 100: Control de tráfico desde Internet hacia DMZ
Esta ACL controla el tráfico entrante desde la red externa, permitiendo solo HTTP y bloqueando todo lo demás (incluyendo ping):

```text
Router_FW# configure terminal

access-list 100 permit tcp any host 192.168.3.1 eq 80
access-list 100 permit tcp any host 192.168.3.1 established
access-list 100 permit icmp any any echo-reply

Router_FW(config)# interface GigabitEthernet0/2
Router_FW(config-if)# ip access-group 100 in
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```
**Función de cada línea:**
- **Línea 1:** Permite conexiones HTTP (puerto 80) hacia la IP pública del servidor
- **Línea 2:** Permite respuestas de conexiones TCP ya establecidas
- **Línea 3:** Permite respuestas ICMP echo-reply
- **Denegación implícita:** Bloquea ping entrante y otros protocolos no autorizados

#### ACL 101: Control de tráfico desde DMZ hacia LAN
Esta ACL es crítica para la seguridad, ya que impide que un servidor comprometido en la DMZ pueda atacar la red interna:

```text
Router_FW# configure terminal

access-list 101 permit tcp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255 established
access-list 101 deny tcp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 deny icmp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 deny udp 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 101 permit ip any any

Router_FW(config)# interface GigabitEthernet0/1
Router_FW(config-if)# ip access-group 101 in
Router_FW(config-if)# exit

Router_FW(config)# end
Router_FW# write memory
```
**Función de cada línea:**
- **Línea 1:** Permite respuestas TCP establecidas de DMZ hacia LAN (crucial para que el servidor pueda responder a peticiones HTTP de usuarios internos)
- **Línea 2:** Bloquea nuevas conexiones TCP iniciadas desde DMZ hacia LAN
- **Línea 3:** Bloquea ICMP (ping) desde DMZ hacia LAN
- **Línea 4:** Bloquea UDP desde DMZ hacia LAN
- **Línea 5:** Permite todo el demás tráfico (hacia Internet, etc.)

**¿Por qué es importante la línea 1?** La palabra clave `established` es fundamental. Sin ella, cuando PC_Internal accede al servidor web DMZ, el servidor no podría enviar las respuestas de vuelta, y la página no cargaría. Esta regla permite las respuestas pero bloquea conexiones nuevas iniciadas desde la DMZ.

### 4.4 Activación de servicios web en el servidor DMZ
En el Server-PT Web_DMZ se activaron los servicios web:
- Pestaña **Services** → **HTTP**
- **HTTP:** ON
- **HTTPS:** ON

Esto permite que el servidor responda a peticiones web en el puerto 80.

## 5. Verificaciones realizadas
Se ejecutaron las siguientes pruebas para validar la funcionalidad y seguridad de la DMZ:

### 5.1 Verificación de conectividad básica
**Pruebas realizadas:**
- Ping desde PC_Internal a 192.168.1.1: Exitoso
- Ping desde Server_DMZ a 192.168.2.1: Exitoso
- Ping desde PC_External a 192.168.3.1: Exitoso

**Resultado:** La conectividad básica entre todos los dispositivos y sus gateways está funcionando correctamente.

### 5.2 Acceso web desde Internet al servidor DMZ
- **Acción:** Web Browser en PC_External → URL: `http://192.168.3.1`
- **Resultado esperado:** Exitoso - Página web del servidor cargada
- **Resultado obtenido:** Exitoso

**Análisis:** Esta prueba confirma que el NAT estático está funcionando correctamente, la ACL 100 permite tráfico HTTP entrante (puerto 80) y el servidor web está operativo respondiendo peticiones.

### 5.3 Ping bloqueado desde Internet hacia servidor DMZ
- **Comando:** `ping 192.168.3.1` desde PC_External
- **Resultado esperado:** Request timed out (bloqueado)
- **Resultado obtenido:** Correcto - Paquetes perdidos (100% loss)

**Análisis:** La ACL 100 está bloqueando correctamente los intentos de ping desde Internet. Esto evita reconocimiento de red por atacantes y reduce la superficie de ataque.

### 5.4 Acceso web desde LAN interna al servidor DMZ
- **Acción:** Web Browser en PC_Internal → URL: `http://192.168.2.10`
- **Resultado esperado:** Exitoso - Página web del servidor cargada
- **Resultado obtenido:** Exitoso

**Análisis:** La ACL 101 permite respuestas TCP establecidas correctamente. Los usuarios internos pueden acceder a servicios en la DMZ.

### 5.5 Bloqueo de acceso desde DMZ hacia LAN (PRUEBA CRÍTICA)
- **Comando:** `ping 192.168.1.10` desde Server-PT Web_DMZ
- **Resultado esperado:** Request timed out (bloqueado)
- **Resultado obtenido:** Correcto - Paquetes perdidos (100% loss)

**Análisis:** Esta es la prueba más importante. Confirma que si el servidor DMZ fuera comprometido, NO podría hacer ping a dispositivos internos, escanear la red interna ni iniciar ataques laterales hacia la LAN. Esta separación es el objetivo principal de una DMZ.

### 5.6 Auto-evaluación en Packet Tracer
- **Resultado:** "Congratulations Guest! You completed the activity."

Todos los objetivos de configuración y seguridad fueron cumplidos exitosamente.

## 6. Conclusiones y recomendaciones

### 6.1 Aprendizajes clave
- **Segmentación de red:** La división en tres zonas (LAN, DMZ, Externa) es esencial para defensa en profundidad. Cada zona tiene un nivel de confianza diferente y políticas específicas.
- **NAT estático:** Permite exponer servicios internos de forma controlada sin revelar la estructura real de la red.
- **ACLs como firewall:** Las listas de control de acceso implementan políticas de seguridad granulares. La palabra clave `established` es crucial para permitir respuestas mientras se bloquean conexiones nuevas no autorizadas.
- **Principio de mínimo privilegio:** Solo se permite el tráfico estrictamente necesario. Todo lo demás se bloquea por defecto.

### 6.2 Recomendaciones
- Verificar conectividad básica antes de aplicar ACLs
- Documentar todas las reglas de firewall implementadas
- Realizar pruebas exhaustivas de seguridad tras cada cambio
- Considerar implementar IDS/IPS en la DMZ para detección de intrusiones
- Mantener logs de acceso para auditoría

### 6.3 Mejoras futuras
- Implementar HTTPS (puerto 443) además de HTTP
- Configurar NAT dinámico para usuarios internos que necesiten salir a Internet
- Añadir más servicios a la DMZ (DNS, correo) con sus respectivas reglas ACL
- Implementar VLANs para mayor segmentación
- Configurar logging en el router para auditoría de seguridad
- Establecer reglas de rate-limiting para prevenir ataques DDoS

### 6.4 Importancia en entornos reales
Esta configuración es fundamental en organizaciones que necesitan exponer servicios públicos mientras protegen su infraestructura interna. La DMZ actúa como una zona de sacrificio: si un atacante compromete el servidor web, no tendrá acceso directo a la red corporativa.

## 7. Capturas de evidencia
Las capturas de pantalla que documentan todas las configuraciones y pruebas realizadas se encuentran organizadas en la carpeta `evidencias/` del repositorio:

Cada captura demuestra la correcta implementación de los componentes de seguridad y la funcionalidad de la DMZ.
