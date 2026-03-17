# Configuración ExpressRoute con un solo interface en FortiGate

## Objetivo

Documentar la configuración de **Azure ExpressRoute private peering** en **FortiGate** usando **un único interface físico**, manteniendo dos sesiones BGP hacia Microsoft sobre la misma interfaz lógica.

---

## Consideración importante

Esta configuración es válida **solo si el proveedor entrega ambos peers por el mismo handoff**.

Con este diseño se mantiene:

- una única interfaz física en FortiGate,
- una única subinterface/VLAN,
- dos direcciones IP en esa misma interfaz,
- dos sesiones BGP contra Microsoft,
- y redundancia lógica de peering.

### Limitación

No hay redundancia física en el lado on-prem.

Si falla:

- el puerto físico,
- el cable,
- la óptica,
- o el handoff único,

caerán las dos sesiones BGP.

---

## Supuestos del ejemplo

- ASN cliente: `65010`
- ASN Microsoft: `12076`
- VLAN private peering: `200`
- Primary subnet: `192.168.100.128/30`
  - FortiGate: `192.168.100.129`
  - Microsoft: `192.168.100.130`
- Secondary subnet: `192.168.100.132/30`
  - FortiGate: `192.168.100.133`
  - Microsoft: `192.168.100.134`
- Prefijo del CPD a anunciar: `10.60.0.0/16`

---

## 1. Configuración de la interfaz

Se utiliza un único puerto físico, por ejemplo `port1`, con una subinterface VLAN.

### Ejemplo de configuración

```plaintext
config system interface
    edit "ER-MAD"
        set interface "port1"
        set vlanid 200
        set ip 192.168.100.129 255.255.255.252
        set allowaccess ping
        set secondary-IP enable

        config secondaryip
            edit 1
                set ip 192.168.100.133 255.255.255.252
            next
        end
    next
end
```

### Explicación

- La IP principal de la interfaz corresponde al `/30` del **primary peer**.
- La IP secundaria corresponde al `/30` del **secondary peer**.
- Ambas IP quedan asociadas a la misma interfaz lógica `ER-MAD`.

---

## 2. Configuración de prefijos a anunciar

Se recomienda anunciar únicamente los prefijos on-prem necesarios hacia Azure.

### Prefix-list de ejemplo

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

---

## 3. Configuración del route-map de salida

El `route-map` permite filtrar qué prefijos se anunciarán hacia Microsoft.

### Ejemplo

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

---

## 4. Configuración BGP

Se crean **dos vecinos BGP** hacia Microsoft, ambos usando la misma interfaz lógica.

### Ejemplo

```plaintext
config router bgp
    set as 65010
    set router-id 10.255.255.1

    config neighbor
        edit "192.168.100.130"
            set remote-as 12076
            set interface "ER-MAD"
            set soft-reconfiguration enable
            set route-map-out "RM-AZURE-OUT"
            set bfd enable
        next
        edit "192.168.100.134"
            set remote-as 12076
            set interface "ER-MAD"
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

---

## 5. Significado del `router-id`

En la línea:

```plaintext
set router-id 10.255.255.1
```

la IP `10.255.255.1` es únicamente el **identificador BGP** del FortiGate.

### No es:

- la IP del peer de Microsoft,
- la IP del enlace primary,
- la IP del enlace secondary,
- ni una IP usada necesariamente para tráfico.

### Recomendación

Usar una IP:

- única,
- estable,
- y preferiblemente asociada a una loopback o a una IP fija interna.

---

## 6. Diferencia respecto al diseño con dos interfaces

### Diseño con dos interfaces

- `ER-MAD-PRI` en `port1`
- `ER-MAD-SEC` en `port2`

### Diseño con un único interface

- una sola interfaz lógica: `ER-MAD`
- una única VLAN
- una IP primaria
- una IP secundaria
- dos vecinos BGP sobre la misma interfaz

---

## 7. Validación

Una vez aplicada la configuración, verificar:

```plaintext
get router info bgp summary
get router info bgp neighbors
get router info routing-table bgp
```

### Resultado esperado

- los dos vecinos BGP aparecen en estado `Established`
- se reciben rutas Azure desde ASN `12076`
- se anuncian los prefijos on-prem definidos

---

## 8. Recomendación técnica

Esta configuración es válida si no hay más remedio y el proveedor entrega ambos peers por el mismo enlace.

Sin embargo, si se requiere **alta disponibilidad real en on-prem**, es preferible usar:

- dos interfaces físicas,
- un `aggregate` o LAG,
- o dos handoffs físicos del proveedor.

---

## 9. Resumen

Con un solo interface en FortiGate para ExpressRoute private peering:

- se puede mantener la configuración BGP con los dos peers de Microsoft,
- se usan dos direcciones IP sobre la misma subinterface,
- se mantiene la lógica de primary/secondary,
- pero se pierde redundancia física en el lado del cliente.
