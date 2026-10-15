# Cisco Secure Client no Ubuntu 26.04

Correção do erro `libxml2.so.2` no serviço `vpnagentd`.

> **Aviso**
>
> Este procedimento utiliza bibliotecas do Ubuntu 24.04 (Noble) para restaurar compatibilidade com versões do Cisco Secure Client que não funcionam corretamente no Ubuntu 26.04.

---

## Sumário

- [Problema](#problema)
- [Pré-requisitos](#pré-requisitos)
- [1. Baixar as bibliotecas](#1-baixar-as-bibliotecas)
- [2. Extrair os pacotes](#2-extrair-os-pacotes)
- [3. Instalar as bibliotecas](#3-instalar-as-bibliotecas)
- [4. Verificar dependências](#4-verificar-dependências)
- [5. Reiniciar o serviço VPN](#5-reiniciar-o-serviço-vpn)
- [Abrir a interface gráfica](#abrir-a-interface-gráfica)
- [Validação final](#validação-final)
- [Observações importantes](#observações-importantes)

---

## Problema

O Cisco Secure Client não inicia e o serviço `vpnagentd` falha com erros semelhantes a:

```text
libxml2.so.2: cannot open shared object file: No such file or directory
```

Isso ocorre porque algumas versões do cliente dependem de bibliotecas que não estão disponíveis no Ubuntu 26.04.

---

## Pré-requisitos

- Ubuntu 26.04 (64 bits)
- Cisco Secure Client instalado
- Permissões administrativas (`sudo`)
- Conexão com a Internet

---

## 1. Baixar as bibliotecas

Crie um diretório de trabalho:

```bash
mkdir -p ~/Downloads/cisco-compat
cd ~/Downloads/cisco-compat
```

Execute o script abaixo:

```bash
python3 - <<'PY'
import gzip
import hashlib
import urllib.request
from pathlib import Path

base = "https://archive.ubuntu.com/ubuntu/"
index = base + "dists/noble/main/binary-amd64/Packages.gz"
wanted = {"libicu74", "libxml2"}

with urllib.request.urlopen(index) as r:
    packages = gzip.decompress(r.read()).decode()

found = {}

for stanza in packages.split("\n\n"):
    fields = {}

    for line in stanza.splitlines():
        if line and not line[0].isspace() and ":" in line:
            key, value = line.split(":", 1)
            fields[key] = value.strip()

    name = fields.get("Package")

    if name in wanted and fields.get("Architecture") == "amd64":
        found[name] = fields

for name in sorted(wanted):
    pkg = found[name]
    filename = pkg["Filename"]

    data_url = base + filename
    data = urllib.request.urlopen(data_url).read()

    if hashlib.sha256(data).hexdigest() != pkg["SHA256"]:
        raise SystemExit(f"Checksum inválido: {name}")

    Path(filename.split("/")[-1]).write_bytes(data)

    print(f"OK: {name} {pkg['Version']}")
PY
```

---

## 2. Extrair os pacotes

```bash
mkdir -p extract

dpkg-deb -x libicu74_74.2-1ubuntu3_amd64.deb extract

dpkg-deb -x libxml2_2.9.14+dfsg-1.3ubuntu3_amd64.deb extract
```

---

## 3. Instalar as bibliotecas

```bash
sudo install -o root -g root -m 755 \
  extract/usr/lib/x86_64-linux-gnu/libicuuc.so.74.2 \
  /usr/local/lib/libicuuc.so.74.2

sudo install -o root -g root -m 755 \
  extract/usr/lib/x86_64-linux-gnu/libicudata.so.74.2 \
  /usr/local/lib/libicudata.so.74.2

sudo install -o root -g root -m 755 \
  extract/usr/lib/x86_64-linux-gnu/libxml2.so.2.9.14 \
  /usr/local/lib/libxml2.so.2.9.14

sudo ln -sfn libicuuc.so.74.2 /usr/local/lib/libicuuc.so.74
sudo ln -sfn libicudata.so.74.2 /usr/local/lib/libicudata.so.74
sudo ln -sfn libxml2.so.2.9.14 /usr/local/lib/libxml2.so.2

sudo ldconfig
```

---

## 4. Verificar dependências

```bash
ldd /opt/cisco/secureclient/bin/vpnagentd | grep "not found"

ldd /opt/cisco/secureclient/bin/manifesttool_vpn | grep "not found"
```

Nenhum dos comandos deve retornar saída.

---

## 5. Reiniciar o serviço VPN

```bash
sudo systemctl reset-failed vpnagentd

sudo systemctl restart vpnagentd

systemctl status vpnagentd --no-pager -l
```

Resultado esperado:

```text
Active: active (running)
```

---

## Abrir a interface gráfica

```bash
/opt/cisco/secureclient/bin/vpnui
```

---

## Validação final

```bash
ldd /opt/cisco/secureclient/bin/vpnagentd | grep "not found"

sudo systemctl restart vpnagentd

systemctl is-active vpnagentd
```

Resultado esperado:

```text
active
```

---

## Observações importantes

### Não crie links simbólicos incompatíveis

❌ Errado:

```bash
ln -s libxml2.so.16 libxml2.so.2
```

### Não substitua bibliotecas do Ubuntu

❌ Errado:

```bash
sudo cp libxml2.so.* /usr/lib/x86_64-linux-gnu/
```

### Não copie bibliotecas para dentro do Cisco Secure Client

❌ Errado:

```text
/opt/cisco/secureclient/
```

### Ambiente considerado

- Ubuntu 26.04 (64 bits)
- Cisco Secure Client
- Bibliotecas oficiais do Ubuntu 24.04
- Permissões administrativas (`sudo`)
