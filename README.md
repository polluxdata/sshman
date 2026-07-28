# sshman

Gestor de conexiones SSH **portable** e interactivo con `fzf`. Diseñado para administrar decenas de hosts SSH (bastiones, llaves `.pem`, múltiples nubes) y **llevártelos entre máquinas** (macOS ↔ Linux) de forma transparente, incluyendo las llaves.

---

## Concepto: el vault

Todo vive en un directorio autocontenido y portable:

```
~/.sshman/
├── config        # SSH config gestionado por sshman
├── keys/         # llaves privadas (chmod 600 automático)
├── meta.json     # metadata por host (entorno / color)
└── backups/      # backups con timestamp antes de cada cambio
```

sshman garantiza que tu `~/.ssh/config` incluya, **al inicio** (prioridad a sshman):

```
Include ~/.sshman/config
```

Esto significa que:
- No se mezclan tus hosts manuales con los gestionados por sshman.
- Puedes empaquetar todo el vault (config + llaves + metadata) y restaurarlo en otra máquina.
- Al importar en otra máquina, las rutas de las llaves se reescriben automáticamente al `$HOME` de destino.

---

## Características

- **Vault portable** con `export`/`import` cifrado (openssl AES-256).
- **Gestión de llaves**: al agregar un host, la llave descargada se **copia** al vault con permisos `600`.
- **Asistente guiado** para nuevas conexiones (host, usuario, bastión, entorno).
- **Migración** de tu `~/.ssh/config` actual al vault, relocalizando llaves.
- **Selección interactiva** con `fzf` que **conecta directamente**.
- **Atajo**: `sshman <alias>` conecta sin escribir `connect`.
- **Color de terminal** por entorno (configurable en `meta.json`, no por adivinar del nombre).
- **`doctor`** que valida permisos, llaves rotas y el `Include`, con `--fix`.
- **Sugerencias** de aliases parecidos cuando escribes uno mal.
- **Cero dependencias pip** — solo Python stdlib, `fzf`, `ssh` y `openssl`.

---

## Requisitos

- Python 3.7+
- [fzf](https://github.com/junegunn/fzf)
- OpenSSH client
- `openssl` (para export/import cifrado; presente por defecto en macOS y Linux)

```bash
brew install fzf     # macOS
```

---

## Instalación

```bash
cp sshman /usr/local/bin/sshman
chmod +x /usr/local/bin/sshman
```

La primera ejecución crea `~/.sshman/` y añade el `Include` a `~/.ssh/config`.

---

## Uso

### Listar / conectar

```bash
sshman                      # lista (selector interactivo, Enter conecta)
sshman list                 # igual que arriba
sshman list --plain         # tabla no interactiva (scripts)
sshman connect              # selector, conecta al elegir
sshman connect acme-prod    # conexión directa
sshman acme-prod            # atajo: conecta directo
```

Al conectar, el fondo del terminal cambia según el entorno del host (definido en `meta.json`) y se restaura al salir.

### Agregar un host (con llave descargada)

Asistente guiado — el caso típico tras descargar una llave de OCI/AWS:

```bash
sshman add --key ~/Downloads/oci-key.pem
```

Pregunta alias, host/IP, usuario, puerto, si tiene bastión y el entorno. La llave se copia a `~/.sshman/keys/<alias>.pem` con `chmod 600` (te avisa de borrar el original) y valida con `ssh -G`.

No interactivo:

```bash
sshman add --alias acme-prod-api --host 172.16.0.10 --user opc \
  --key ~/Downloads/oci-key.pem \
  --jump opc@bastion.example.com:22 --env prod
```

### Editar / renombrar / eliminar

```bash
sshman edit acme-prod-api
sshman rename acme-prod-api acme-prod-web
sshman remove acme-prod-api      # ofrece borrar también la llave del vault
```

### Ver / validar

```bash
sshman show acme-prod-api
sshman test acme-prod-api        # ssh -G
```

### Migrar tu config actual al vault

```bash
sshman migrate
```

Importa todos los hosts de `~/.ssh/config` al vault y copia sus llaves:
- **IdentityFile** → `~/.sshman/keys/<alias>.pem` (renombrada por host)
- **ProxyCommand `-i`** → `~/.sshman/keys/<nombre-original>` (compartidas entre hosts)

Las llaves de bastión compartidas en `ProxyCommand` (p.ej. una misma llave usada por 30 hosts) se copian **una sola vez** con su nombre original. Las rutas se reescriben automáticamente al vault.

Después de migrar, limpia `~/.ssh/config` dejando solo el `Include` (línea 1), otros `Include` (colima, etc.), el `Host *` global y comentarios. Así el vault tiene **prioridad** y no hay duplicados que causen colisiones.

```bash
# ~/.ssh/config tras migrar:
Include ~/.sshman/config
Include ~/.colima/ssh_config

Host *
    ServerAliveInterval 30
    AddKeysToAgent yes
    UseKeychain yes
```

### Portabilidad: export / import

En la máquina origen:

```bash
sshman export                    # → sshman-vault-YYYYMMDD.tar.gz.enc (cifrado)
sshman export -o /ruta/mi.enc    # nombre personalizado
sshman export --no-encrypt       # sin cifrar (con aviso; NO recomendado)
```

El export cifrado usa `openssl enc -aes-256-cbc -pbkdf2` y pide una passphrase.

En la máquina destino (por ejemplo un Linux nuevo):

```bash
sshman import sshman-vault-YYYYMMDD.tar.gz.enc
sshman doctor                    # verifica permisos y rutas
```

El import descifra, reescribe las rutas de llaves al `$HOME` de destino, aplica `chmod 600` y asegura el `Include`.

> **Seguridad**: el export contiene tus llaves privadas. Cífralo siempre (por defecto lo hace). No subas un export `--no-encrypt` a la nube ni lo envíes por canales inseguros.

### Diagnóstico

```bash
sshman doctor          # reporta problemas
sshman doctor --fix    # corrige permisos y Include
```

---

## Comandos

Cada subcomando tiene `--help` detallado (descripción, flags y ejemplos):

```bash
sshman add --help
sshman export --help
```

| Comando    | Descripción                                        |
|------------|----------------------------------------------------|
| `list`     | Listar hosts (interactivo; `--plain` para tabla)   |
| `connect`  | Conectar (interactivo o `connect <alias>`)         |
| `add`      | Agregar host (asistente o flags; `--key` gestiona la llave) |
| `edit`     | Editar host                                        |
| `rename`   | Renombrar alias (y su llave)                       |
| `remove`   | Eliminar host (y opcionalmente su llave)           |
| `show`     | Ver detalles                                       |
| `test`     | Validar con `ssh -G`                               |
| `migrate`  | Importar `~/.ssh/config` al vault                  |
| `export`   | Exportar el vault (cifrado por defecto)            |
| `import`   | Importar un vault exportado                        |
| `doctor`   | Diagnosticar/reparar (`--fix`)                     |

### Flags de `add`

| Flag         | Descripción                          |
|--------------|--------------------------------------|
| `--alias`    | Alias del host                       |
| `--host`     | Hostname o IP                        |
| `--user`     | Usuario SSH                          |
| `--port`     | Puerto (default: 22)                 |
| `--key`      | Ruta a la llave descargada (se copia al vault) |
| `--jump`     | Jump server (`usuario@host:puerto`)  |
| `--jump-key` | Llave del jump server                |
| `--env`      | Entorno (prod/test/dev/bastion/otro) |

---

## Colores por entorno

Al conectar, sshman cambia el **fondo del terminal** según el entorno del host (OSC 11) y lo **restaura** al salir. La paleta por defecto usa **acentos discretos** (tonos oscuros) para no romper la legibilidad:

| Env       | Color     | Tinte          |
|-----------|----------|----------------|
| `prod`    | `#2E1414` | rojo oscuro    |
| `test`    | `#2E2414` | ámbar oscuro   |
| `staging` | `#2E2414` | ámbar oscuro   |
| `dr`      | `#2E2414` | ámbar oscuro   |
| `dev`     | `#142414` | verde oscuro   |
| `lab`     | `#142414` | verde oscuro   |
| `bastion` | `#241A2E` | violeta oscuro |
| *(default)* | `#181A20` | slate neutro |

Configurables en `~/.sshman/meta.json`:

```json
{
  "color_rules": {
    "prod":    "#2E1414",
    "dev":     "#142414",
    "bastion": "#241A2E"
  },
  "default_color": "#181A20",
  "hosts": {
    "acme-prod-api": { "env": "prod", "color": "#3A0000" }
  }
}
```

Prioridad: `color` explícito del host → regla del `env` del host → `default_color`. La metadata viaja en el export/import.

> **Nota**: el color del fondo se captura antes de cambiarlo (OSC 11 `?`) y se restaura exacto al salir. Si tu terminal no responde a OSC 11, se restaura a `"default"`. Un `color` explícito por host siempre tiene prioridad.

---

## Autocompletado (zsh)

El repo incluye `_sshman` (función de autocompletado zsh). Completa subcomandos, flags y **aliases del vault**.

Instalación:

```bash
# Opción A: copiar a un directorio de $fpath
cp _sshman /opt/homebrew/share/zsh/site-functions/_sshman   # macOS (Homebrew)
cp _sshman /usr/local/share/zsh/site-functions/_sshman       # Linux

# Opción B: añadir el dir del repo a fpath (útil en dev)
# en ~/.zshrc:  fpath+=(/ruta/al/repo/sshman)

rm -f ~/.zcompdump* && exec zsh
```

Después:

```bash
sshman <Tab>            # subcomandos + aliases del vault
sshman connect <Tab>    # solo aliases
sshman add --<Tab>      # flags de add
```

---

## Backups

Antes de cada modificación se crea un backup en:

```
~/.sshman/backups/config.bak.YYYYMMDD_HHMMSS
```

---

## Licencia

MIT — PolluxData / Jota
