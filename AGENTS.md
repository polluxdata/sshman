# AGENTS.md — sshman

## Resumen

`sshman` es un gestor de conexiones SSH portable (Python 3, un solo archivo) que vive en
`/usr/local/bin/sshman` y mantiene un vault autocontenido en `~/.sshman/`.

## Estructura del repo

- `sshman` — script principal (Python 3.7+, solo stdlib + fzf + openssl)
- `_sshman` — función de autocompletado zsh (subcomandos, flags y aliases del vault)
- `README.md` — documentación de usuario
- `AGENTS.md` — este archivo

## Comandos para verificar cambios

```bash
python3 -m py_compile sshman        # check de sintaxis
./sshman --help                     # CLI carga correctamente
```

No hay suite de tests. Para validar funcionalmente, usar un HOME temporal:

```bash
TMPHOME=$(mktemp -d) && export HOME="$TMPHOME" && mkdir -p "$TMPHOME/.ssh"
cp sshman "$TMPHOME/sshman"
cd "$TMPHOME"
./sshman add --alias test-prod --host 1.2.3.4 --user opc --env prod
./sshman list --plain
./sshman doctor
rm -rf "$TMPHOME"
```

## Convenciones

- **Sin dependencias pip**: solo Python stdlib, `fzf`, `ssh`, `openssl`.
- **Sin comentarios** salvo docstrings de funciones.
- Mensajes de estado con prefijos `[OK]`, `[ERR]`, `[WARN]` (helpers `ok`, `err`, `warn`, `info`).
- Backups automáticos en `~/.sshman/backups/` antes de cada modificación del config.
- Las llaves del vault siempre con permisos `600`.

## Identidad git

```
user.name  = Fernando Jose Andrade
user.email = fjandrade@polluxdata.com
```

## Notas de diseño

- El vault (`~/.sshman/config`) se incluye al **inicio** de `~/.ssh/config` vía `Include` para
  tener prioridad sobre hosts manuales.
- `migrate` copia tanto `IdentityFile` (renombrada a `<alias>.pem`) como llaves referenciadas
  en `ProxyCommand -i` (con su nombre original, deduplicadas).
- `export`/`import` cifra con `openssl aes-256-cbc -pbkdf2`; la passphrase se pide via
  `getpass` y se pasa a openssl como `env:SSHMAN_PASS`.
- `doctor` valida `IdentityFile` + llaves de `ProxyCommand`, permisos `600`, y el `Include`.
- Los valores del SSH config se parsean eliminando comillas envolventes (`_unquote`).
- **Color de terminal (OSC 11)**: al conectar, sshman captura el fondo actual vía query OSC 11
  `?` leída de `/dev/tty` (no de `stdin`, para no filtrar la respuesta al prompt), cambia el
  fondo al color del entorno del host, lo re-afirma con `LocalCommand` tras `ssh -t`, y restaura
  el color original al desconectar. Si la query OSC 11 falla, restaura a `"default"`.
- **Paleta de colores**: acentos discretos (tonos oscuros) en `meta.json` para no romper
  legibilidad. La paleta por defecto (espesa negra con tinte) está en `default_meta()`.
- **`--help` por subcomando**: cada subparser tiene `description` + `epilog` con ejemplos.
- **Autocompletado zsh** (`_sshman`): completa subcomandos, flags y aliases del vault leyendo
  `~/.sshman/config`. Usa solo `compadd -a` (sin `_describe`/`_alternative` que rompen en
  algunos zsh).