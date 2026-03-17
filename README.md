# Guía paso a paso para ampliar ExpressRoute y enrutar tráfico hacia el CPD

## 1. Objetivo

Esta guía describe cómo:

- añadir un nuevo circuito ExpressRoute en Madrid con otro proveedor,
- conectarlo al entorno de Spain Central,
- configurar el private peering,
- preparar la parte on-prem con FortiGate,
- y enrutar de forma preferente el tráfico entre recursos de Spain Central y el CPD.

---

## 2. Escenario de partida

### Circuitos existentes

- **Circuito 1**
  - Región Azure: **Spain Central**
  - Peering location: **Madrid**
- **Circuito 2**
  - Región Azure: **West Europe**
  - Peering location: **Amsterdam**
- **Circuito 3 nuevo**
  - Peering location: **Madrid**
  - Nuevo proveedor

### Requisito funcional

- Los recursos de **West Europe** deben usar preferentemente la ExpressRoute de **Amsterdam**.
- Los recursos de **Spain Central** deben usar preferentemente la **nueva ExpressRoute de Madrid** para llegar al **CPD**.
- Las redes del CPD se anunciarán por **todos los circuitos**, manteniendo preferencia y failover.

---

## 3. Decisión de diseño

### 3.1 Tipo de conexión recomendado

Para la nueva ExpressRoute de Madrid, la opción recomendada es:

- **Standard resiliency**

### 3.2 Motivo

No se recomienda basar este diseño en **Maximum resiliency** porque el objetivo principal no es agrupar Madrid y Amsterdam detrás del mismo gateway, sino mantener:

- afinidad regional,
- control del tráfico por prefijo,
- y preferencia diferenciada entre West Europe y Spain Central.

### 3.3 Arquitectura recomendada

- **Gateway West Europe**
  - conectado al circuito de **Amsterdam**
- **Gateway Spain Central**
  - conectado al circuito **Madrid actual**
  - conectado al circuito **Madrid nuevo**

Con esto:

- **West Europe → CPD** usará preferentemente **Amsterdam**
- **Spain Central → CPD** usará preferentemente **Madrid nuevo**

---

## 4. Paso 1 - Verificar el tipo de entrega del proveedor

Antes de configurar el private peering, confirmar si el proveedor entrega:

### Opción A - Servicio gestionado Layer 3

El proveedor levanta el peering con Microsoft.

En este caso, en el FortiGate normalmente **no se configura BGP directamente contra Microsoft**.

### Opción B - Handoff Layer 2

El cliente configura:

- la VLAN del private peering,
- las IP de primary y secondary,
- y las dos sesiones BGP con Microsoft.

> Esta guía asume **handoff Layer 2**.

---

## 5. Paso 2 - Preparar los datos del private peering

Para crear el peering privado en Azure se necesita:

- **Peer ASN** del cliente: `65010`
- **VLAN ID** del private peering
- **Primary subnet** IPv4 (`/30`)
- **Secondary subnet** IPv4 (`/30`)
- **Shared key / MD5** si se desea usar

### Reglas de Azure para private peering

- Microsoft usa ASN **12076**
- Tu router usa la **primera IP utilizable** de cada `/30`
- Microsoft usa la **segunda IP utilizable**
- Deben levantarse **las dos sesiones BGP** para cumplir la SLA
- Las subredes de peering no deben solaparse con VNets ni con otros rangos usados en Azure
- El ASN del cliente puede ser privado, pero **no** debe estar entre **65515 y 65520**

### Ejemplo de direccionamiento

- Primary subnet: `192.168.100.128/30`
  - FortiGate: `192.168.100.129`
  - Microsoft: `192.168.100.130`
- Secondary subnet: `192.168.100.132/30`
  - FortiGate: `192.168.100.133`
  - Microsoft: `192.168.100.134`

---

## 6. Paso 3 - Crear el Azure Private Peering en el circuito

En el circuito ExpressRoute nuevo:

1. Ir a **Peerings**.
2. Seleccionar **Azure private**.
3. Configurar:
   - **Peer ASN**: `65010`
   - **IPv4 Primary subnet**: por ejemplo `192.168.100.128/30`
   - **IPv4 Secondary subnet**: por ejemplo `192.168.100.132/30`
   - **VLAN ID**: la acordada con el proveedor
   - **Shared key**: opcional
4. Guardar la configuración.

---

## 7. Paso 4 - Crear la conexión al gateway de Spain Central

En el `ExpressRoute VNet Gateway` de **Spain Central**:

1. Crear una nueva **Connection**.
2. Elegir:
   - **Connection type**: `ExpressRoute`
   - **Resiliency**: `Standard resiliency`
3. Asociar la conexión al **nuevo circuito de Madrid**.
4. Configurar un `routing weight` superior al del circuito Madrid actual.

### Recomendación de pesos

- **Madrid nuevo**: `200`
- **Madrid actual**: `100`

Con esto, si el mismo prefijo llega por ambos circuitos al gateway de Spain Central, Azure preferirá el **Madrid nuevo**.

---

## 8. Paso 5 - Configurar FortiGate para el private peering

> Ejemplo orientativo. Ajustar nombres de interfaz, VLAN e IPs al entorno real.

### 8.1 Configurar interfaces

```plaintext
config system interface
    edit "ER-MAD-PRI"
        set interface "port1"
        set vlanid 200
        set ip 192.168.100.129 255.255.255.252
        set allowaccess ping
    next
    edit "ER-MAD-SEC"
        set interface "port2"
        set vlanid 200
        set ip 192.168.100.133 255.255.255.252
        set allowaccess ping
    next
end
```

### 8.2 Configurar prefijos a anunciar

Ejemplo para el agregado del CPD:

```plaintext
config router prefix-list
    edit "PL-AZURE-OUT"
        config rule
            edit 1
                set prefix 10.60.0.0 255.255.0.0
            next
        end
    next
end
```

### 8.3 Configurar el route-map de salida

```plaintext
config router route-map
    edit "RM-AZURE-OUT"
        config rule
            edit 1
                set match-ip-address "PL-AZURE-OUT"
            next
        end
    next
end
```

### 8.4 Configurar BGP

```plaintext
config router bgp
    set as 65010
    set router-id 10.255.255.1

    config neighbor
        edit "192.168.100.130"
            set remote-as 12076
            set interface "ER-MAD-PRI"
            set soft-reconfiguration enable
            set route-map-out "RM-AZURE-OUT"
            set bfd enable
        next
        edit "192.168.100.134"
            set remote-as 12076
            set interface "ER-MAD-SEC"
            set soft-reconfiguration enable
            set route-map-out "RM-AZURE-OUT"
            set bfd enable
        next
    end

    config network
        edit 1
            set prefix 10.60.0.0 255.255.0.0
        next
    end
end
```

### 8.5 Qué es `router-id`

La línea:

```plaintext
set router-id 10.255.255.1
```

solo define el **identificador BGP** del FortiGate.

- No es la IP del peer de Microsoft
- No es la IP del `/30`
- No tiene que ser una IP de tráfico

Se recomienda usar una IP:

- única,
- estable,
- preferiblemente una loopback o una IP interna fija.

---

## 9. Paso 6 - Diseñar el routing entre Azure y el CPD

## 9.1 Objetivo del routing

- **Spain Central → CPD**: preferir **Madrid nuevo**
- **West Europe → CPD**: preferir **Amsterdam**
- mantener backup por los otros circuitos

## 9.2 Prefijos de ejemplo

Supongamos que el CPD tiene:

- agregado: `10.60.0.0/16`
- subredes críticas:
  - `10.60.10.0/24`
  - `10.60.11.0/24`
  - `10.60.12.0/24`

## 9.3 Anuncios recomendados por circuito

### Circuito Madrid nuevo

Anunciar:

- `10.60.0.0/16`
- `10.60.10.0/24`
- `10.60.11.0/24`
- `10.60.12.0/24`

Sin prepend.

### Circuito Madrid actual

Anunciar:

- `10.60.0.0/16`
- `/24` críticos con prepend medio

### Circuito Amsterdam

Anunciar:

- `10.60.0.0/16`
- `/24` críticos con prepend alto

---

## 10. Paso 7 - Aplicar preferencia con AS-path prepend

## 10.1 Qué es AS-path prepend

`AS-path prepend` consiste en repetir tu ASN varias veces al anunciar una ruta por un circuito menos preferido.

Ejemplo con ASN `65010`:

- ruta primaria: `65010`
- ruta secundaria: `65010 65010 65010`
- ruta terciaria: `65010 65010 65010 65010 65010`

## 10.2 Valores recomendados

- **Madrid nuevo**: prepend `0`
- **Madrid actual**: prepend `2`
- **Amsterdam**: prepend `4`

## 10.3 Ejemplo conceptual

Para `10.60.10.0/24`:

- Madrid nuevo → `65010`
- Madrid actual → `65010 65010 65010`
- Amsterdam → `65010 65010 65010 65010 65010`

Resultado esperado:

1. Azure preferirá **Madrid nuevo**
2. Si falla, usará **Madrid actual**
3. Si también falla, quedará **Amsterdam**

> Nota: si un prefijo es más específico, el prefijo más específico tendrá prioridad antes que el AS-path.

---

## 11. Paso 8 - Controlar el retorno con local-preference

El prepend por sí solo no garantiza simetría. Para controlar el tráfico **on-prem → Azure**, se debe ajustar `local-preference` en la red interna.

### Recomendación

#### Prefijos de Azure Spain Central

- aprendidos por **Madrid nuevo** → `local-preference 250`
- aprendidos por **Madrid actual** → `local-preference 200`
- aprendidos por **Amsterdam** → `local-preference 100`

#### Prefijos de Azure West Europe

- aprendidos por **Amsterdam** → `local-preference 250`
- aprendidos por **Madrid nuevo** → `local-preference 100`
- aprendidos por **Madrid actual** → `local-preference 90`

### Resultado

- tráfico **Spain Central ↔ CPD** por **Madrid nuevo**
- tráfico **West Europe ↔ CPD** por **Amsterdam**

---

## 12. Paso 9 - Validar la configuración

### En FortiGate

```plaintext
get router info bgp summary
get router info bgp neighbors
get router info routing-table bgp
get router info bgp network
get system arp
```

Comprobar que:

- ambos vecinos BGP están en `Established`
- se reciben rutas Azure desde ASN `12076`
- se anuncian los prefijos del CPD
- la resolución ARP es correcta

### En Azure

Comprobar:

- el estado del **Azure private peering**
- que el circuito está correctamente enlazado al gateway
- que la conexión del gateway tiene el `routing weight` esperado
- que las rutas se ven en las tablas del gateway/circuito

---

## 13. Recomendaciones operativas

- No usar **UDR** para forzar el camino hacia ExpressRoute.
- No anunciar exactamente los mismos prefijos por todos los circuitos sin política.
- No usar NAT para tráfico privado Azure ↔ CPD salvo necesidad específica.
- Mantener agregación de prefijos cuando sea posible.
- Probar el failover de forma periódica.
- Si el proveedor ofrece **Layer 3 managed**, confirmar si el peering lo gestiona él antes de configurar BGP en FortiGate.

---

## 14. Resumen final

### West Europe

- gateway propio
- circuito Amsterdam
- preferido para recursos de West Europe

### Spain Central

- gateway propio
- circuito Madrid actual
- circuito Madrid nuevo
- el Madrid nuevo será el preferido para el tráfico hacia el CPD

### Política recomendada

- anunciar redes del CPD por los tres circuitos
- usar prefijos críticos más específicos cuando convenga
- usar `routing weight` en Azure para preferir Madrid nuevo en Spain Central
- usar `AS-path prepend` en circuitos secundarios
- usar `local-preference` on-prem para garantizar simetría y afinidad regional

---

## 15. Datos que conviene cerrar antes de implantar

- VLAN ID real del private peering
- subred `/30` primaria
- subred `/30` secundaria
- interfaces físicas o lógicas del FortiGate
- versión de FortiOS
- lista real de prefijos del CPD
- prefijos Azure de Spain Central
- prefijos Azure de West Europe
- si el proveedor entrega Layer 2 o Layer 3 managed
