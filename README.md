# bifrost

Binarios de la CLI `bifrost`, el cliente de línea de comandos de Bifröst (portal de infraestructura de gravity). Este repo solo contiene los
**Releases**; el código no se publica aquí.

## Instalar

Linux y macOS (verifica el checksum SHA-256 antes de instalar):

```sh
curl -fsSL https://github.com/acalvoa/bifrost/releases/latest/download/install.sh | sh
```

Windows: descarga `bifrost_windows_amd64.exe` (o `arm64`/`386`) desde la [última versión](https://github.com/acalvoa/bifrost/releases/latest)
y ponlo en tu `PATH`.

| Sistema | Arquitecturas |
|---|---|
| macOS | Apple Silicon (arm64), Intel (amd64) |
| Linux | amd64, arm64, 386 |
| Windows | amd64, arm64, 386 |

Cada Release incluye `SHA256SUMS`. Para verificar a mano: `sha256sum -c SHA256SUMS --ignore-missing`.

## Primeros pasos

```sh
bifrost auth login https://tu-bifrost.ejemplo.com --account casa
bifrost account list
bifrost apps list -a casa
```

Con `bifrost -h` ves todos los comandos. Puedes tener varias cuentas (cada una con su Bifröst e identidad) y elegir con `-a/--account`
o `BIFROST_ACCOUNT`.
