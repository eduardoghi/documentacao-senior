# Senior Gestão Empresarial (ERP) no Linux com Wine

[Voltar para o README](../README.md)

Configuração usada para executar o Senior Gestão Empresarial (ERP / Sapiens) no Linux através do Wine, validada no seguinte ambiente:

| Componente | Ambiente |
|---|---|
| ERP | Senior Gestão Empresarial 5.10.4 |
| Distribuição | Arch Linux / EndeavourOS |
| Desktop | KDE Plasma |
| Sessão gráfica | Wayland |
| Driver Wine | `winex11.drv` através do XWayland |
| Wine | Wine upstream (`wine-git` durante os testes) |
| Cliente de banco | Oracle Client 19c 32 bits, dentro do prefixo |
| Autenticação de rede | Active Directory + Kerberos + SSSD |
| Compartilhamentos | CIFS/SMB com Kerberos |

> [!IMPORTANT]
> Esta configuração não é oficial nem suportada pela Senior Sistemas.
>
> O ERP é uma aplicação Windows e o funcionamento através do Wine pode variar conforme a versão do ERP, Wine, bibliotecas utilizadas e ambiente gráfico.

---

## Arquitetura utilizada

Embora a sessão gráfica do Linux seja Wayland, o ERP roda com o driver X11 do Wine:

```text
Senior Gestão Empresarial
        ↓
Wine
        ↓
winex11.drv
        ↓
XWayland
        ↓
KWin / Plasma Wayland
```

O `winewayland.drv` também foi testado. O ERP inicia, mas a página de login, baseada em CEF4/Chromium, não renderiza. Os testes estão em [`winewayland.drv`](#winewayland). Por isso o ambiente validado usa `winex11.drv`.

## Instalação do Wine

Em distribuições baseadas em Arch Linux:

```bash
sudo pacman -S --needed \
    wine \
    winetricks \
    lib32-gnutls \
    lib32-libldap \
    lib32-libpulse \
    lib32-alsa-plugins
```

Os comandos usados nas seções seguintes vêm destes pacotes:

```bash
sudo pacman -S --needed \
    cifs-utils \
    krb5 \
    wmctrl \
    xorg-xprop \
    desktop-file-utils
```

O `cifs-utils` traz o `mount.cifs`, o `krb5` traz o `klist` e o `kvno`, o `wmctrl` e o `xorg-xprop` são usados no diagnóstico de janelas, e o `desktop-file-utils` traz o `desktop-file-validate`. A integração da máquina ao Active Directory com SSSD é pré-requisito e não está coberta aqui.

### Wine Git

O Wine 11.17 introduziu uma regressão no tratamento dos botões do mouse no `winex11.drv`, descrita em [Mouse deixa de clicar após abrir relatórios](#mouse-deixa-de-clicar-após-abrir-relatórios).

A correção foi incorporada depois ao Wine upstream. Enquanto o pacote estável da distribuição não contiver esse fix, é possível usar o `wine-git` do AUR:

```bash
yay -S wine-git
```

O pacote substitui o `wine` oficial do sistema. Para conferir qual está em uso:

```bash
wine --version
pacman -Qo /usr/bin/wine
```

> [!NOTE]
> Assim que o pacote estável do Wine da distribuição tiver a correção, é preferível voltar para ele.

## Prefixo dedicado para o ERP

Use um prefixo Wine exclusivo para o Senior, nunca o padrão `~/.wine`:

```bash
export WINEPREFIX="$HOME/.local/share/wineprefixes/senior"

wineboot
```

`echo "$WINEPREFIX"` deve devolver:

```text
/home/<usuario>/.local/share/wineprefixes/senior
```

## Instalação do Client/Estação Senior

O ERP precisa ser instalado pelo `SeniorInstaller.exe`, que prepara as bibliotecas, os executáveis e a estrutura da estação. Copiar a pasta de uma instalação Windows existente para dentro do prefixo não é recomendado.

Com o prefixo criado e exportado, execute o instalador:

```bash
wine <caminho>/SeniorInstaller.exe
```

Mantenha o diretório padrão da estação, `C:\SeniorEstacao`. No fim a estrutura fica parecida com:

```text
$WINEPREFIX/drive_c/SeniorEstacao/
└── Sapiens/
    ├── sapiens.exe
    ├── *.bpl
    ├── *.dll
    └── ...
```

> [!IMPORTANT]
> Instale o Client no mesmo `WINEPREFIX` que será usado para executar o ERP. Se o Client for instalado no prefixo padrão `~/.wine` e o ERP for executado em outro prefixo, os componentes instalados não estarão disponíveis.

## Cliente Oracle

O `sapiens.exe` é um binário 32 bits (`PE32`, Intel i386) e conecta no banco por OCI, carregando o `oci.dll` no próprio processo. Por isso o cliente Oracle de 32 bits precisa estar instalado dentro do mesmo prefixo, e um cliente 64 bits não serve.

No ambiente validado é o Oracle Client 19c:

| Item | Valor |
|---|---|
| Home | `OraClient19Home1_32bit` |
| `ORACLE_HOME` | `C:\app\client\<usuario>\product\19.0.0\client_1` |
| Chave de registro | `Software\Wow6432Node\ORACLE\KEY_OraClient19Home1_32bit` |
| `TNS_ADMIN` | `%ORACLE_HOME%\network\admin` |
| `NLS_LANG` | `AMERICAN_AMERICA.WE8MSWIN1252` |

Instale com o prefixo já exportado:

```bash
export WINEPREFIX="$HOME/.local/share/wineprefixes/senior"

wine <caminho-do-instalador-oracle>/setup.exe
```

O instalador grava `ORACLE_HOME`, `TNS_ADMIN` e `NLS_LANG` no registro do prefixo, então não é preciso exportar essas variáveis no lado Linux.

O `tnsnames.ora` e o `sqlnet.ora` ficam em `$WINEPREFIX/drive_c/app/client/<usuario>/product/19.0.0/client_1/network/admin`. Um alias por base, no formato normal:

```text
<ALIAS> =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = <servidor>)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = <servico>)
    )
  )
```

O `sqlnet.ora` do ambiente validado tem:

```text
SQLNET.AUTHENTICATION_SERVICES= (NTS)
NAMES.DIRECTORY_PATH= (TNSNAMES, EZCONNECT)
DIAG_SIGHANDLER_ENABLED=FALSE
DIAG_ADR_ENABLED=OFF
DIAG_DDE_ENABLED=OFF
```

> [!IMPORTANT]
> O cliente Oracle deve ser instalado no mesmo `WINEPREFIX` do ERP. O `sapiens.exe` resolve o `ORACLE_HOME` pelo registro do prefixo, portanto um cliente instalado em outro prefixo ou no `~/.wine` não é encontrado.

## Execução do ERP

O executável fica em `$WINEPREFIX/drive_c/SeniorEstacao/Sapiens`:

```bash
export WINEPREFIX="$HOME/.local/share/wineprefixes/senior"

cd "$WINEPREFIX/drive_c/SeniorEstacao/Sapiens"

wine sapiens.exe
```

Para encerrar todos os processos Wine do prefixo:

```bash
WINEPREFIX="$HOME/.local/share/wineprefixes/senior" wineserver -k
```

## Locale regional (pt-BR)

Configurar `Control Panel\International` no registro do prefixo (data, moeda, separador decimal etc., via `wine reg add` ou `winecfg`) não é suficiente para o `LocaleName` (o valor moderno, usado por `GetUserDefaultLocaleName`). O Wine recalcula esse valor a partir do `LANG`/`LC_ALL` do processo Unix que inicia o `wine`, ignorando o que está gravado no registro para esse campo específico.

Confirme com:

```bash
WINEPREFIX="$HOME/.local/share/wineprefixes/senior" \
    wine reg query "HKCU\Control Panel\International" /v LocaleName
```

Se a sessão gráfica não estiver com `LANG=pt_BR.UTF-8`, esse comando devolve `en-US` mesmo com o restante do registro já em pt-BR. O locale `pt_BR.utf8` precisa estar instalado no sistema (`locale -a`).

A correção é passar `LANG`/`LC_ALL` no ambiente de quem lança o `wine`, por exemplo no `Exec` do `.desktop`:

```ini
Exec=env WINEPREFIX=/home/<usuario>/.local/share/wineprefixes/senior LANG=pt_BR.UTF-8 LC_ALL=pt_BR.UTF-8 wine /home/<usuario>/.local/share/wineprefixes/senior/drive_c/SeniorEstacao/Sapiens/sapiens.exe
```

Processos Wine já abertos antes dessa mudança mantêm o locale antigo em memória. Só reflete numa próxima abertura.

## Fontes e renderização

Sem as fontes que normalmente existem no Windows, alguns componentes do ERP ficam visualmente diferentes. O `fontsmooth=rgb` habilita suavização semelhante ao ClearType.

```bash
export WINEPREFIX="$HOME/.local/share/wineprefixes/senior"

winetricks tahoma
winetricks micross
winetricks fontsmooth=rgb
```

Os verbos acima instalam Tahoma e Microsoft Sans Serif. Quando disponíveis, também vale instalar:

```text
Arial
Verdana
Segoe UI
```

Depois encerre o Wine com `wineserver -k`. O que já está no prefixo aparece em:

```bash
ls "$WINEPREFIX/drive_c/windows/Fonts"
```

## Compartilhamentos SMB utilizados pelo ERP

O ERP pode guardar caminhos UNC no banco ou em configurações, como `\\SERVIDOR\Dados\Documentos`. No Linux o compartilhamento é montado localmente, por exemplo em `$HOME/mnt/dados`, e exposto para o Wine pelos aliases UNC que ficam em `$WINEPREFIX/dosdevices/unc/`.

```bash
mkdir -p \
    "$WINEPREFIX/dosdevices/unc/SERVIDOR"

ln -sfn \
    "$HOME/mnt/dados" \
    "$WINEPREFIX/dosdevices/unc/SERVIDOR/Dados"
```

Com isso `\\SERVIDOR\Dados` dentro do Wine aponta para `$HOME/mnt/dados`.

Também é possível criar uma unidade, fazendo `S:\` passar a existir nas aplicações Wine:

```bash
ln -sfn \
    "$HOME/mnt/dados" \
    "$WINEPREFIX/dosdevices/s:"
```

## SMB com Active Directory e Kerberos

Em máquinas integradas ao Active Directory os compartilhamentos podem ser acessados sem armazenar usuário e senha do SMB. O ticket é obtido com `kvno cifs/<servidor-fqdn>` e verificado com `klist`, e a montagem usa `sec=krb5`.

O comando abaixo serve para entender as opções e testar manualmente. A montagem do dia a dia é feita pelo pam_mount, como descrito na seção seguinte.

```bash
sudo mount -t cifs \
    //<servidor-fqdn>/<share> \
    "$HOME/mnt/dados" \
    -o sec=krb5,cruid=$(id -u),uid=$(id -u),gid=$(id -g)
```

### Problema com `/etc/fstab`

Montar os compartilhamentos por `systemd.automount` no `/etc/fstab` resultou em:

```text
mount error(126): Required key not available
```

A montagem executada pelo systemd de sistema não encontrava o cache Kerberos criado durante a sessão gráfica do usuário. A solução foi montar os compartilhamentos através do PAM, depois da autenticação.

## Montagem automática com pam_mount

```bash
sudo pacman -S --needed pam_mount
```

Em `/etc/security/pam_mount.conf.xml`:

```xml
<volume
    user="<usuario>"
    fstype="cifs"
    server="<servidor-fqdn>"
    path="<share>"
    mountpoint="/home/<usuario>/mnt/dados"
    options="sec=krb5,cruid=<uid>,uid=<uid>,gid=<gid>"
/>
```

Os valores de uid e gid saem do comando `id`.

O PAM da sessão também precisa invocar o `pam_mount.so`. No ambiente testado as entradas ficaram em `/etc/pam.d/system-login`:

```pam
auth       optional   pam_mount.so
password   optional   pam_mount.so
session    optional   pam_mount.so
```

> [!NOTE]
> A cadeia PAM onde essas linhas entram depende da distribuição e do gerenciador de login. Aqui o Plasma Login Manager usa `/usr/lib/pam.d/plasmalogin`, que faz `include system-login` em auth, account, password e session, então alterar o `system-login` cobre o login gráfico. Em outro ambiente, confira qual cadeia é usada antes de copiar as linhas.

> [!WARNING]
> Faça backup dos arquivos PAM antes de alterá-los. Uma configuração incorreta pode impedir o login no sistema.

Depois de um novo login, `klist` e `findmnt` devem mostrar o ticket Kerberos e os compartilhamentos montados.

## Atalho no KDE Plasma

Crie `~/.local/share/applications/senior-sapiens.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Senior Sapiens
Comment=Senior Gestão Empresarial
Exec=env WINEPREFIX=/home/<usuario>/.local/share/wineprefixes/senior LANG=pt_BR.UTF-8 LC_ALL=pt_BR.UTF-8 wine /home/<usuario>/.local/share/wineprefixes/senior/drive_c/SeniorEstacao/Sapiens/sapiens.exe
Path=/home/<usuario>/.local/share/wineprefixes/senior/drive_c/SeniorEstacao/Sapiens
Icon=/home/<usuario>/.local/share/icons/senior-sapiens.png
StartupWMClass=sapiens.exe
Terminal=false
Categories=Office;
StartupNotify=true
```

Os caminhos ficam absolutos porque o `Exec` do `.desktop` não expande variáveis como `$HOME`.

`LANG`/`LC_ALL` no `Exec` são o que faz o Wine reportar `pt-BR` como `LocaleName`, ver [Locale regional (pt-BR)](#locale-regional-pt-br).

O `StartupWMClass` associa a janela criada pelo Wine ao atalho. Sem ele, o Plasma pode tratar o launcher fixado e a janela do ERP como aplicações diferentes, resultando em dois ícones na barra ou em uma janela que não agrupa com o atalho. O valor vem do `WM_CLASS` da janela, que o `wmctrl -lx` mostra como `sapiens.exe.sapiens.exe` e o `xprop` confirma:

```text
WM_CLASS(STRING) = "sapiens.exe", "sapiens.exe"
```

Valide o arquivo e reconstrua o cache de aplicações do KDE:

```bash
chmod +x \
    "$HOME/.local/share/applications/senior-sapiens.desktop"

desktop-file-validate \
    "$HOME/.local/share/applications/senior-sapiens.desktop"

kbuildsycoca6
```

O Plasma costuma detectar o `.desktop` novo sozinho, e o `kbuildsycoca6` força a reconstrução quando isso não acontece. Ele vem do `kservice`, que já acompanha o Plasma. O `update-desktop-database` não resolve aqui porque reconstrói o cache de associações MIME, e este `.desktop` não declara `MimeType`.

## Extração do ícone do ERP

Instale o `icoutils`:

```bash
sudo pacman -S --needed icoutils
```

O `wrestool` lista os ícones do executável:

```bash
SAPIENS="$WINEPREFIX/drive_c/SeniorEstacao/Sapiens/sapiens.exe"

wrestool -l -t 14 "$SAPIENS"
```

Extraia o `MAINICON`:

```bash
mkdir -p "$HOME/.local/share/icons"

wrestool \
    -x \
    -t 14 \
    -n MAINICON \
    "$SAPIENS" \
    > "$HOME/.local/share/icons/senior-sapiens.ico"
```

Converta para PNG e copie a maior imagem extraída:

```bash
TMP_ICONS="$(mktemp -d)"

icotool \
    -x \
    -o "$TMP_ICONS" \
    "$HOME/.local/share/icons/senior-sapiens.ico"

cp \
    "$TMP_ICONS/<arquivo>.png" \
    "$HOME/.local/share/icons/senior-sapiens.png"

rm -rf "$TMP_ICONS"
```

---

## Validação da instalação

```bash
export WINEPREFIX="$HOME/.local/share/wineprefixes/senior"

wine --version

WINEDEBUG=+loaddll \
    wine "$WINEPREFIX/drive_c/SeniorEstacao/Sapiens/sapiens.exe" 2>&1 |
    grep -Ei 'winex11|winewayland'

klist
findmnt
```

Confirme que:

- o ERP abre e apresenta a tela de login
- o login conclui, ou seja o cliente Oracle resolve o alias do `tnsnames.ora` e alcança o banco
- o driver carregado é o `winex11.drv`
- os compartilhamentos usados pelo ERP estão montados
- os caminhos UNC do ERP são acessíveis
- os relatórios abrem corretamente
- o mouse continua funcional depois de abrir, fechar e reabrir um relatório

---

## Problemas conhecidos

### Relatórios distorcidos com dois monitores

Determinados relatórios saem distorcidos quando o ERP roda com dois monitores. Com um único monitor:

```text
HORZRES        = 2560
DESKTOPHORZRES = 2560
```

Com dois monitores:

```text
HORZRES        = 2560
DESKTOPHORZRES = 4480
```

O ERP calcula uma transformação utilizando:

```text
M11 = DESKTOPHORZRES / HORZRES
M22 = DESKTOPVERTRES / VERTRES
```

No exemplo acima `M11 = 4480 / 2560`, ou seja `1.75`, uma escala horizontal de 175%.

A análise da `svcl02.bpl`, responsável por essa etapa da geração dos relatórios, mostrou chamadas equivalentes a:

```text
GetDeviceCaps(DESKTOPHORZRES)   ; index 0x76
GetDeviceCaps(HORZRES)          ; index 0x08

GetDeviceCaps(DESKTOPVERTRES)   ; index 0x75
GetDeviceCaps(VERTRES)          ; index 0x0A

SetWorldTransform(...)
```

Os índices acima são o que se procura no disassembly para localizar a sequência.

O comportamento ocorre porque o Wine retorna a largura total da área virtual dos monitores em `DESKTOPHORZRES`.

#### Workaround utilizado

Foi aplicado um patch local na `svcl02.bpl` para detectar quando o ERP está executando no Wine. Na build analisada a sequência fica próxima do VA `0x440403da`. A detecção usa uma função que só o `ntdll.dll` do Wine exporta:

```c
HMODULE ntdll;

ntdll = GetModuleHandleA("ntdll.dll");

if (ntdll &&
    GetProcAddress(ntdll, "wine_get_version")) {
    /* Wine */
}
```

No Wine, os valores usados no cálculo são alterados para que `M11 = 1` e `M22 = 1`. No Windows o comportamento original é preservado.

> [!WARNING]
> O patch é específico da versão da `svcl02.bpl` analisada. Offsets, endereços e bytes não devem ser reutilizados em outra versão sem nova análise. Sempre mantenha backup da biblioteca original.
>
> Nenhum binário proprietário da Senior é distribuído neste repositório. O procedimento documenta apenas o comportamento identificado e a técnica usada para contorná-lo.

### Mouse deixa de clicar após abrir relatórios

#### Sintoma

Em determinadas versões do Wine:

```text
abre relatório
→ fecha relatório
→ abre o relatório novamente
→ cliques do mouse deixam de funcionar
```

Teclado e movimento do mouse continuam funcionando normalmente.

#### Diagnóstico

Durante a falha foi verificado:

```text
KWin / foco da janela                OK
X11 input focus                      OK
Capture Win32                        NULL
XI_RawButtonPress                    OK
XI_RawButtonRelease                  OK
X11DRV_RawButtonEvent                OK
WH_MOUSE_LL                          não recebe eventos
SendInput() artificial               funciona
```

Portanto o problema não estava no ERP, no mouse físico nem no foco da janela, e sim no processamento de eventos de botão dentro do `winex11.drv`.

#### Regressão do Wine

A regressão foi introduzida por:

```text
44a822d2374c71ee74fa8dc930b63f5a8e9ae256

winex11: Listen to raw mouse button events on the root window.
```

que passou a utilizar `XI_RawButtonPress` e `XI_RawButtonRelease` para os botões do mouse. Durante a investigação upstream constatou-se que múltiplas threads podiam estar escutando eventos raw simultaneamente.

Esse commit entrou como correção do [Bug 60256](https://bugs.winehq.org/show_bug.cgi?id=60256), em que uma janela Wine com o foco impedia a root window do X11 de receber eventos de botão. Resolveu o 60256 e introduziu duas regressões:

- [Bug 60290](https://bugs.winehq.org/show_bug.cgi?id=60290), "Mouse button handling regressions in several games when launched through Steam", que registra o `44a822d...` como Regression SHA1 e o `24309da4...` como Fixed by SHA1
- [Bug 60291](https://bugs.winehq.org/show_bug.cgi?id=60291), janelas Wine reagindo a cliques feitos fora delas

As duas estão corrigidas no upstream.

A correção:

```text
24309da4c30a57283f4779ab218f5f2ecdada1e2

winex11: Restore XI_ButtonPress event instead of raw button events.
```

deixa de utilizar raw events para os botões e restaura a máscara `XI_ButtonPress` apenas quando o cursor está confinado. Segundo a mensagem do commit, rastrear grabs de mouse com eventos raw exigiria acompanhar o estado entre processos, já que um botão pode ser pressionado em uma janela e liberado em outra, inclusive em uma janela que não é do Wine.

#### Workarounds

As seguintes opções solucionaram o problema:

1. utilizar uma versão do Wine contendo o fix `24309da4...`
2. utilizar o Wine 11.17 revertendo o commit `44a822d...`

O segundo método foi usado no diagnóstico para confirmar a regressão. Para uso normal, prefira uma versão atual do Wine com a correção oficial.

<a name="winewayland"></a>
### `winewayland.drv`

O driver Wayland nativo do Wine também foi testado:

```bash
env -u DISPLAY \
    wine sapiens.exe
```

O Wine carrega o `winewayland.drv` e o ERP inicia. O que não é renderizado corretamente é a página de login, baseada em CEF4/Chromium.

Desabilitando o CEF4 temporariamente, o ERP abre normalmente, o que indica que a incompatibilidade está no caminho usado pelo componente Chromium/CEF do login, e não no funcionamento básico da aplicação VCL.

Os parâmetros `--disable-gpu`, `--disable-gpu-sandbox`, `--use-angle=gl` e `--use-angle=swiftshader` foram testados na linha de comando do ERP, como em `wine sapiens.exe --disable-gpu`, sem sucesso. Com `--in-process-gpu` a tela de login passa a ser exibida, porém o ERP apresenta vários outros problemas de funcionamento, então o parâmetro serviu apenas como teste diagnóstico e não é usado como workaround.

O `--in-process-gpu` faz o Chromium executar o GPU process como uma thread dentro do browser process, em vez de usar um processo de GPU separado, o que aponta para uma incompatibilidade no caminho multiprocessos de GPU do CEF/Chromium sob `winewayland.drv`. O subprocesso de GPU não foi instrumentado, e também não foi investigado como o `sapiens.exe` repassa esses argumentos ao CEF, então a causa exata continua em aberto.

Por isso o ambiente validado para uso normal continua com `winex11.drv` através do XWayland:

```text
Plasma Wayland
    ↓
XWayland
    ↓
winex11.drv
    ↓
Senior ERP
```

---

## Comandos de diagnóstico

### Descobrir qual driver gráfico o Wine está utilizando

```bash
WINEDEBUG=+loaddll \
    wine sapiens.exe 2>&1 |
    grep -Ei 'winex11|winewayland'
```

No ambiente validado deve aparecer o `winex11.drv`.

### Verificar sessão gráfica

```bash
echo "$XDG_SESSION_TYPE"
echo "$DISPLAY"
echo "$WAYLAND_DISPLAY"
```

Exemplo:

```text
wayland
:0
wayland-0
```

### Inspecionar janelas Wine

O `wmctrl -lx` lista as janelas e o `xprop -id <window-id>` mostra as propriedades de uma delas.

### Encerrar Wine

```bash
WINEPREFIX="$HOME/.local/share/wineprefixes/senior" wineserver -k
```

---

## Estado atual

Neste ambiente o ERP está funcional com Wine contendo o fix do Bug 60290, `winex11.drv` sobre XWayland no Plasma Wayland, prefixo Wine dedicado, Client/Estação Senior e cliente Oracle 19c de 32 bits instalados no próprio prefixo, fontes do Windows instaladas, SMB com Kerberos montado por pam_mount e aliases UNC no prefixo.

As principais incompatibilidades encontradas estão na interação entre o ERP e as camadas gráficas e de entrada do Wine. No `winex11.drv` foram observados problemas no tratamento dos botões do mouse via XInput e na geometria exposta em ambientes com vários monitores. No `winewayland.drv`, o comportamento observado aponta para o caminho multiprocessos de GPU do CEF/Chromium usado pela tela de login.
