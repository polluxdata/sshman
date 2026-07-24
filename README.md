# sshman

Gestor de conexiones SSH interactivo con `fzf`. Diseñado para administrar de forma sencilla múltiples hosts SSH, especialmente útil cuando se trabaja con bastiones, llaves `.pem` y decenas de servidores.

---

## Características

- **Listado interactivo** con `fzf` y preview del host seleccionado
- **Conexión directa** desde el selector sin necesidad de escribir el alias
- **Agregar hosts** con validación de puertos y soporte para jump servers
- **Editar hosts** incluyendo `ProxyCommand`, `ProxyJump`, `IdentitiesOnly`, etc.
- **Eliminar hosts** con confirmación interactiva
- **Validación** con `ssh -G` para verificar que la configuración es correcta
- **Modo plano** (`--plain`) para listar hosts en formato tabla (ideal para scripts)
- **Backups automáticos** de `~/.ssh/config` antes de cualquier modificación
- **Parser completo** que preserva más de 80 opciones SSH conocidas

---

## Requisitos

- Python 3.7+
- [fzf](https://github.com/junegunn/fzf)
- OpenSSH client

Instalar `fzf` en macOS:

```bash
brew install fzf
```

---

## Instalación

Clonar el repositorio y copiar el script a un directorio en tu `PATH`:

```bash
git clone git@github.com:polluxdata/sshman.git
cd sshman
cp sshman /usr/local/bin/sshman
chmod +x /usr/local/bin/sshman
```

---

## Uso

### Listar hosts

```bash
sshman -list
```

Modo no interactivo (tabla):

```bash
sshman -list --plain
```

### Conectar a un host

Interactivo (seleccionas con `fzf`):

```bash
sshman -connect
```

Directo por alias:

```bash
sshman -connect -alias polluxdata-sites
```

### Agregar un host

Interactivo:

```bash
sshman -add
```

Con argumentos:

```bash
sshman -add \
  -alias ucuenca-prod-nuevo \
  -host 172.30.1.1 \
  -user opc \
  -key ~/.ssh/keys/UCUENCA/PROD/nuevo.pem \
  -jump opc@bastion.example.com:22
```

### Editar un host

```bash
sshman -edit
sshman -edit -alias ucuenca-prod-nuevo
```

Permite modificar:
- Hostname / IP
- Usuario
- Puerto
- Llave `.pem`
- Jump server (`ProxyCommand` / `ProxyJump`)
- `IdentitiesOnly`

### Eliminar un host

```bash
sshman -remove
sshman -remove -alias ucuenca-prod-nuevo
```

### Ver detalles de un host

```bash
sshman -show -alias polluxdata-sites
```

### Validar configuración

Ejecuta `ssh -G <alias>` para verificar que la configuración es válida sin conectar:

```bash
sshman -test -alias polluxdata-sites
```

---

## Comandos disponibles

| Comando      | Descripción                                    |
|--------------|------------------------------------------------|
| `-list`      | Listar hosts (interactivo con `fzf`)           |
| `-connect`   | Conectar a un host (interactivo o directo)     |
| `-add`       | Agregar un nuevo host                          |
| `-edit`      | Editar un host existente                       |
| `-remove`    | Eliminar un host                               |
| `-show`      | Mostrar detalles de un host                    |
| `-test`      | Validar configuración con `ssh -G`             |

### Flags comunes

| Flag         | Descripción                                    |
|--------------|------------------------------------------------|
| `-alias`     | Alias del host                                 |
| `-host`      | Hostname o IP                                  |
| `-user`      | Usuario SSH                                    |
| `-port`      | Puerto (default: 22)                           |
| `-key`       | Ruta a la llave `.pem`                         |
| `-jump`      | Jump server (`usuario@host:puerto`)            |
| `-jump-key`  | Llave del jump server                          |
| `--plain`    | Modo no interactivo para `-list`               |

---

## Backups

Cada vez que se modifica `~/.ssh/config`, se crea un backup automático en:

```
~/.ssh/backups/config.bak.YYYYMMDD_HHMMSS
```

---

## Configuración SSH

El script opera sobre el archivo estándar `~/.ssh/config`. El parser reconoce más de 80 opciones SSH conocidas y preserva las que no conoce sin modificarlas.

---

## Licencia

MIT — PolluxData / Jota
