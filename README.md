# Pré check-in e Livro de Informações — Hostel Clean

Material complementar do sistema Hostel Gestão. Duas páginas HTML
estáticas, publicadas no GitHub Pages: recolhem os dados que a lei exige
para o boletim de alojamento (RGPD + Lei 23/2007) e mostram ao hóspede
o guia da estadia.

As páginas **não têm servidor próprio** nem nenhum segredo no código.
Falam com a API do Hostel Gestão (FastAPI na VM `nova-vm`), publicada
na internet pelo Tailscale Funnel. **O site nunca fala com a base de
dados.**

---

## 1. Ficheiros

| Ficheiro | Função |
|---|---|
| `index.html` | Pré check-in. Mostra os dados da reserva e o formulário. Uso único por reserva. |
| `livro.html` | Livro de informações (horários, Wi-Fi, regras, avarias, contactos). PT/EN/ES/FR. |
| `img/ponte-d-luis.jpg` | Imagem de fundo do topo. Se faltar, o topo fica navy. |

Endereço publicado:
`https://mauriciopates.github.io/site_checking_hostel_gestao/`

---

## 2. Como funciona o fluxo

1. No desktop (Hostel Gestão), o botão **Gerar link** de uma reserva
   cria um **token aleatório**. A base `hostel_prechecking` guarda só o
   **hash SHA-256** do token, mais os detalhes da reserva que o hóspede
   pode ver (sem o código da lockbox).
2. O link — `.../site_checking_hostel_gestao/?t=TOKEN` — é enviado pelo
   anfitrião na mensagem do Airbnb.
3. O hóspede abre o `index.html`:
   - o site pede os detalhes à API (`POST /api/reserva`, com o token);
   - link válido → cartão **"A sua reserva"** + formulário;
   - link inválido, já usado ou expirado → uma mensagem única (o site
     não sabe nem diz qual dos casos foi).
4. Ao carregar em **Concluir**, o site envia o pré check-in
   (`POST /api/pre-checkin`). A API grava-o como **pendente** e gasta o
   token na mesma transação — o link deixa de funcionar.
5. No desktop, o pendente é validado e importado para a ficha do
   cliente.
6. Depois do sucesso, o botão **"Ver informações da estadia"** abre o
   `livro.html?t=TOKEN`, que se pode reabrir durante a estadia.

---

## 3. Contexto legal (RGPD)

Os dados do boletim são tratados por **obrigação legal** (art. 6.º/1/c;
Lei 23/2007, arts. 15.º e 16.º) e **execução do contrato** (art.
6.º/1/b) — não por consentimento. Por isso o formulário tem três
confirmações com naturezas diferentes, **não um "aceito tudo"**:

| Confirmação | Natureza | Obrigatória? | Base legal |
|---|---|---|---|
| Declaro ter sido informado sobre o tratamento | Tomada de conhecimento | Sim | art. 13.º |
| Aceito o regulamento da casa | Aceitação contratual | Sim | art. 6.º/1/b |
| Aceito receber comunicações | Consentimento revogável | Não | art. 6.º/1/a |

O **email** só é pedido — e só é enviado — quando o hóspede marca a
terceira caixa (minimização, art. 5.º/1/c). A base recusa um email sem
consentimento e um consentimento sem email.

A versão do aviso que faz prova é a guardada com o token no momento em
que o link foi emitido (art. 5.º/2).

---

## 4. Configuração do `index.html`

No início do `<script>`:

```js
const VERSAO_AVISO = "1.0";   // sobe sempre que o texto do aviso mudar
const ENVIO = "api";          // "api" (real) ou "ficheiro" (demonstração)
const URL_API = "https://nova-vm.tailce9342.ts.net/api";
```

Com `ENVIO = "ficheiro"` o site não fala com nenhum servidor: mostra o
formulário sem o cartão da reserva e gera um ficheiro JSON no
dispositivo.

### 4.1 Pedidos à API

`POST /api/reserva` — corpo `{"token": "..."}`
→ `200` com `unidade, morada, data_entrada, data_saida, hora_checkin,
hora_checkout, regras` · `404` link inválido/usado/expirado.

`POST /api/pre-checkin` — corpo:

```json
{
  "token": "...",
  "versao_aviso": "1.0",
  "submetido_em": "2026-10-05T14:30:00.000Z",
  "cliente": {
    "nome": "", "nacionalidade": "", "data_nascimento": "AAAA-MM-DD",
    "tipo_documento": "", "numero_documento": "",
    "pais_emissor_documento": "", "pais_residencia": ""
  },
  "confirmacoes": {
    "informado_privacidade": true,
    "aceitou_regulamento": true,
    "consente_comunicacoes": false
  },
  "email": "só presente quando consente_comunicacoes = true"
}
```

### 4.2 Respostas e o que o hóspede vê

| Código | Significado | No site |
|---|---|---|
| 201 | Registado | Ecrã "Pré check-in concluído" |
| 404 | Link inválido, usado ou expirado | Ecrã "Este link não é válido ou já expirou" |
| 422 | Dados recusados pela validação | "Verifique os campos e tente outra vez" |
| 429 | Demasiados pedidos (10/min por IP) | "Aguarde um minuto" |
| 413 | Pedido demasiado grande | "Verifique os campos" |
| 503 / sem rede | Base ou ligação em baixo | "Não foi possível enviar" (os dados mantêm-se) |

---

## 5. Segurança (do lado do site)

- Nenhum segredo no código: a proteção está no token (uso único, com
  validade, guardado como hash) e na API.
- A API só aceita pedidos vindos de `https://mauriciopates.github.io`
  (CORS).
- `<meta name="referrer" content="no-referrer">`: o `?t=` do endereço
  nunca é enviado a outros sites.
- O que vem da API é mostrado com `textContent` (nunca `innerHTML`):
  um texto com `<script>` aparece como texto e não corre.
- Os campos têm `maxlength` iguais às colunas da base; a API valida tudo
  outra vez.
- O botão fica bloqueado enquanto envia (sem duplo envio).

---

## 6. Publicar uma alteração

Na pasta do site (Git Bash):

```
git add index.html README.md
git commit -m "..."
git push
```

O GitHub Pages atualiza em 1–2 minutos (Settings → Pages: *Deploy from a
branch*, `main`, `/ (root)`). O repositório tem de ser público na conta
gratuita — não tem nada de secreto.
