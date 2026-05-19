# thechatroom — instalação com whitelist

Instalação do [fr33n0w/thechatroom](https://github.com/fr33n0w/thechatroom) no nó NomadNet/Reticulum com acesso restrito por hash LXMF.

**Nó:** rawmesh · Beira Baixa · storage@192.168.0.170
**Data:** 2026-05-19

---

## o que é

IRC-style chatroom para NomadNet/Reticulum, otimizado para MeshChat v2.1+.
Corre como script Python dentro do NomadNet — páginas `.mu` executáveis.

Por defeito qualquer pessoa na rede pode aceder.
Esta instalação adiciona uma **whitelist por hash LXMF** — só entra quem estiver na lista.

---

## passo 1 — clonar o repositório

```bash
cd ~/.nomadnetwork/storage/pages/
git clone https://github.com/fr33n0w/thechatroom.git chatroom
cd chatroom
chmod +x nomadnet.mu meshchat.mu fullchat.mu last100.mu
```

---

## passo 2 — criar o ficheiro whitelist

Adicionar os hashes LXMF autorizados.
O hash LXMF está em **Settings → Identity** no Sideband/Columba.

```bash
cat > ~/.nomadnetwork/storage/pages/chatroom/whitelist.json << 'EOF'
[
  "35de307ad9c6dcbe76c4285a97d65adc",
  "c5e2dfe9778fa2c1ca9fd67de19b3f4d"
]
EOF
```

Hashes iniciais:
- `35de307ad9c6dcbe76c4285a97d65adc` — Field Signal (MeshChatX)
- `c5e2dfe9778fa2c1ca9fd67de19b3f4d` — Roaming Signal (Sideband)

---

## passo 3 — aplicar o patch de whitelist

O patch insere um bloco de verificação nos scripts `nomadnet.mu` e `meshchat.mu`
logo após a linha onde o `dest` é recuperado do ambiente.

### criar o script de patch

```bash
cat > /tmp/patch_whitelist.py << 'EOF'
import os

WHITELIST_BLOCK = '''
# whitelist
import json as _json
try:
    with open(os.path.join(os.path.dirname(__file__), "whitelist.json")) as _f:
        _allowed = _json.load(_f)
except:
    _allowed = []

_caller = remote_identity or dest
if _caller not in _allowed:
    print(">acesso restrito / restricted access")
    print("")
    print(">esta sala e privada.")
    print(">contacta o operador do no para acesso.")
    print("")
    print(">this room is private.")
    print(">contact the node operator to request access.")
    sys.exit()
'''

FILES = [
    "~/.nomadnetwork/storage/pages/chatroom/nomadnet.mu",
    "~/.nomadnetwork/storage/pages/chatroom/meshchat.mu"
]

for path in FILES:
    path = os.path.expanduser(path)
    with open(path, "r") as f:
        content = f.read()
    if "# whitelist" in content:
        print(f"{path} — ja tem whitelist, ignorado")
        continue
    anchor = 'dest             = recover_input("dest")'
    if anchor not in content:
        print(f"{path} — anchor nao encontrado")
        continue
    content = content.replace(anchor, anchor + WHITELIST_BLOCK)
    with open(path, "w") as f:
        f.write(content)
    print(f"{path} — OK")
EOF
```

### correr o patch

```bash
python3 /tmp/patch_whitelist.py
```

Output esperado:
```
/home/storage/.nomadnetwork/storage/pages/chatroom/nomadnet.mu — OK
/home/storage/.nomadnetwork/storage/pages/chatroom/meshchat.mu — OK
```

### verificar que foi inserido

```bash
grep -n "whitelist" ~/.nomadnetwork/storage/pages/chatroom/nomadnet.mu
grep -n "whitelist" ~/.nomadnetwork/storage/pages/chatroom/meshchat.mu
```

---

## passo 4 — reiniciar o nomadnet

```bash
sudo systemctl restart nomadnet.service
```

---

## aceder ao chatroom

No NomadNet ou MeshChat, navegar para:

```
/page/chatroom/nomadnet
```

ou via hash do nó:
```
2f756995d6febcde6b850c1c005774c7:/page/chatroom/nomadnet
```

Quem não estiver no whitelist vê:
```
acesso restrito / restricted access
esta sala e privada.
contacta o operador do no para acesso.
```

---

## convidar alguém

1. Pedir o hash LXMF à pessoa (visível em Settings → Identity no Sideband/Columba)
2. Adicionar ao whitelist:

```bash
nano ~/.nomadnetwork/storage/pages/chatroom/whitelist.json
```

```json
[
  "35de307ad9c6dcbe76c4285a97d65adc",
  "c5e2dfe9778fa2c1ca9fd67de19b3f4d",
  "hash_do_novo_convidado"
]
```

3. Reiniciar o nomadnet:

```bash
sudo systemctl restart nomadnet.service
```

---

## notas técnicas

- O chatroom usa variáveis de ambiente para identificar quem faz o pedido
- `remote_identity` — hash da identidade Reticulum do utilizador
- `dest` — hash de destino (fallback)
- O patch usa `dest             = recover_input("dest")` como anchor (com espaços — é o formato real do código)
- O whitelist é verificado a cada pedido de página — não precisa reiniciar para adicionar convidados
- O ficheiro `whitelist.json` fica em `~/.nomadnetwork/storage/pages/chatroom/`

---

## ficheiros relevantes no nó

```
~/.nomadnetwork/storage/pages/chatroom/
├── nomadnet.mu        # interface NomadNet (patched)
├── meshchat.mu        # interface MeshChat (patched)
├── fullchat.mu        # histórico completo
├── last100.mu         # últimas 100 mensagens
├── index.mu           # página de entrada
├── whitelist.json     # hashes autorizados
├── chatusers.db       # base de dados de utilizadores
└── chat_log.json      # log de mensagens
```
