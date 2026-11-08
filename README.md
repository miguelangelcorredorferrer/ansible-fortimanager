# ansible-fortimanager

Playbook para dar de alta en FortiManager todos los FortiGate que haya en la red de gestión,
sin tener que ir equipo por equipo desde el Device Manager.

Lo he montado sobre un laboratorio en GNS3 (FortiManager 6.2.5 y tres FortiGate 6.2.2: una sede
central y dos oficinas), pero sirve para cualquier red cambiando el inventario.

La documentación completa del laboratorio (montaje en GNS3, configuración de FortiManager,
playbook y pruebas) está en [docs/FG-2026-002_FortiManager_Ansible.pdf](docs/FG-2026-002_FortiManager_Ansible.pdf).

## Qué hace

1. Escanea la red de gestión con `nmap` buscando el puerto 541 (FGFM). Si un equipo tiene el
   443 abierto pero no el 541, avisa: suele ser un FortiGate sin `fgfm` en `allowaccess`.
2. Consulta qué dispositivos hay ya en el ADOM para no registrarlos dos veces.
3. Para cada FortiGate nuevo, el FortiManager lo sondea con sus credenciales, lo añade al ADOM
   y el playbook espera a que la tarea termine.

Se puede lanzar las veces que haga falta: si no hay equipos nuevos, no cambia nada.

## Requisitos

- ansible-core y la colección de Fortinet:

  ```
  pipx install --include-deps ansible-core
  ansible-galaxy collection install fortinet.fortimanager
  ```

- `nmap` en la máquina desde la que se lanza.
- En el FortiManager, el usuario necesita permiso para la API JSON-RPC. Si no lo tiene, el
  login falla con `-11 No permission for the resource`:

  ```
  config system admin user
      edit admin
          set rpc-permit read-write
      next
  end
  ```

- En cada FortiGate, `fgfm` permitido en la interfaz de gestión.

## Uso

Ajusta `inventory.yml` (IP del FortiManager, red de gestión, ADOM y usuario de los FortiGate) y
crea el archivo de contraseñas:

```
cp secretos.example.yml secretos.yml
chmod 600 secretos.yml
```

Después:

```
ansible-playbook registrar_fortigates.yml -e @secretos.yml
```

Sin `secretos.yml`, el playbook pide las contraseñas por pantalla. Para registrar IPs concretas
sin escanear la red:

```
ansible-playbook registrar_fortigates.yml -e @secretos.yml -e fgt_ips='["192.168.122.71"]'
```

## Estructura

```
ansible.cfg
inventory.yml               FortiManager, conexión httpapi y variables de la red
registrar_fortigates.yml    descubrimiento y bucle de altas
tareas/registrar_uno.yml    sondeo, alta y espera de un FortiGate
secretos.example.yml        plantilla de contraseñas
```

## Notas

- `ansible_httpapi_validate_certs` está a `false` porque el FortiManager del laboratorio usa su
  certificado autofirmado. En producción conviene poner uno válido y activar la validación.
- Usar `admin` para la API es cómodo en un laboratorio, pero en un entorno real es mejor un
  usuario de API propio, con un perfil limitado y trusted hosts.
- El registro se hace con la IP de gestión de cada FortiGate. Si cambia, el playbook lo verá
  como un equipo nuevo.
