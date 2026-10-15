Corrigir Cisco Secure Client no Ubuntu 26.04
Problema

O Cisco Secure Client não abre e o serviço vpnagentd falha ao iniciar, exibindo um erro semelhante a:

Plain Text
libxml2.so.2: cannot open shared object file: No such file or directory

A solução consiste em baixar versões compatíveis das bibliotecas do Ubuntu 24.04 e instalá-las em um diretório separado, sem modificar os pacotes do sistema.

Passo 1: Baixar as bibliotecas necessárias

Crie um diretório de trabalho:

Shell
mkdir -p ~/Downloads/cisco-compat
cd ~/Downloads/cisco-compat

Execute o script abaixo para localizar e baixar os pacotes oficiais:

Shell
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

Resultado esperado:

Plain Text
OK: libicu74 <versão>
OK: libxml2 <versão>
Passo 2: Extrair os pacotes

Crie um diretório para extração:

Shell
mkdir -p extract

Extraia os arquivos .deb baixados:

Shell
dpkg-deb -x libicu74_74.2-1ubuntu3_amd64.deb extract
dpkg-deb -x libxml2_2.9.14+dfsg-1.3ubuntu3_amd64.deb extract
Passo 3: Instalar as bibliotecas em um diretório isolado

Copie apenas as bibliotecas necessárias para /usr/local/lib:

Shell
sudo install -o root -g root -m 755 \
extract/usr/lib/x86_64-linux-gnu/libicuuc.so.74.2 \
/usr/local/lib/libicuuc.so.74.2
 
sudo install -o root -g root -m 755 \
extract/usr/lib/x86_64-linux-gnu/libicudata.so.74.2 \
/usr/local/lib/libicudata.so.74.2
 
sudo install -o root -g root -m 755 \
extract/usr/lib/x86_64-linux-gnu/libxml2.so.2.9.14 \
/usr/local/lib/libxml2.so.2.9.14

Crie os links simbólicos:

Shell
sudo ln -sfn libicuuc.so.74.2 /usr/local/lib/libicuuc.so.74
sudo ln -sfn libicudata.so.74.2 /usr/local/lib/libicudata.so.74
sudo ln -sfn libxml2.so.2.9.14 /usr/local/lib/libxml2.so.2

Atualize o cache de bibliotecas:

Shell
sudo ldconfig
Passo 4: Verificar dependências

Valide se ainda existem bibliotecas ausentes:

Shell
ldd /opt/cisco/secureclient/bin/vpnagentd | grep "not found"
Shell
ldd /opt/cisco/secureclient/bin/manifesttool_vpn | grep "not found"
Resultado esperado

Nenhum dos comandos deve retornar saída.

Se houver alguma linha contendo:

Plain Text
=> not found

a dependência correspondente ainda não foi resolvida.

Passo 5: Reiniciar o serviço VPN

Limpe o estado de falha do serviço:

Shell
sudo systemctl reset-failed vpnagentd

Reinicie o serviço:

Shell
sudo systemctl restart vpnagentd

Verifique o status:

Shell
systemctl status vpnagentd --no-pager -l
``
Resultado esperado
Plain Text
Active: active (running)
Abrir a interface do Cisco Secure Client

Se o serviço estiver em execução, inicie a interface gráfica:

Shell
/opt/cisco/secureclient/bin/vpnui
Observações Importantes
✅ Recomendado
Instalar bibliotecas compatíveis em /usr/local/lib.
Utilizar apenas pacotes oficiais do Ubuntu 24.04.
Executar ldconfig após a instalação.
❌ Não faça

Não crie links simbólicos entre versões incompatíveis:

Shell
# ERRADO
ln -s libxml2.so.16 libxml2.so.2

Não substitua bibliotecas fornecidas pelo Ubuntu:

Shell
# ERRADO
sudo cp libxml2.so.* /usr/lib/x86_64-linux-gnu/

Não copie bibliotecas para dentro do diretório do Cisco:

Shell
# ERRADO
/opt/cisco/secureclient/

Isso pode causar problemas com verificações de integridade e futuras atualizações do cliente.

Ambiente Considerado

Este procedimento pressupõe:

Ubuntu 26.04 (64 bits)
Cisco Secure Client instalado em /opt/cisco/secureclient
Bibliotecas libxml2 e libicu74 obtidas do repositório oficial do Ubuntu 24.04 (Noble)
Permissões de administrador (sudo) disponíveis no sistema
Verificação Final

Execute a sequência abaixo:

Shell
ldd /opt/cisco/secureclient/bin/vpnagentd | grep "not found"
 
sudo systemctl restart vpnagentd
 
systemctl is-active vpnagentd
 

Se o último comando retornar:

Plain Text
active

o problema foi corrigido com sucesso. ✅
