# Pré check-in e Livro de Informações — Hostel Clean

Material complementar para o sistema de reservas do Hostel Clean.
Duas páginas HTML estáticas que recolhem os dados legalmente exigidos
para o boletim de alojamento (RGPD + Lei 23/2007) e apresentam ao
hóspede o guia da estadia.

Estas páginas **não têm backend próprio**. São ficheiros estáticos
que comunicam com a API do sistema de reservas através de um `POST`
(ver secção "Integração com a API").

---

## 1. Ficheiros

| Ficheiro | Função |
|---|---|
| `index.html` | Pré check-in. Formulário que o hóspede preenche **3 horas antes** da chegada. Uso único por reserva. |
| `livro.html` | Livro de informações. Guia consultável durante a estadia (horários, Wi-Fi, regras, avarias, contactos). Multilíngue (PT/EN/ES/FR). |
| `ponte-d-luis.jpg` | Imagem de fundo (hero) usada nas duas páginas. Opcional — se faltar, o hero fica com a cor navy. |

Os três ficheiros devem ficar **na mesma pasta** no servidor.

---

## 2. Como funciona o fluxo

1. O sistema de reservas confirma uma reserva e gera um **token único**
   (ex.: UUID v4) que guarda na tabela `reservas.token`.
2. **3 horas antes do check-in**, o sistema envia ao hóspede (email/SMS)
   um link do tipo:

   https://«DOMINIO»/pre-checkin/index.html?t=TOKEN


3. O hóspede abre o `index.html`, preenche os dados do boletim e submete.
- O formulário faz `POST` para a API (ver secção 4) com o `token` no corpo.
- A API grava os dados e marca a reserva como `pre_checkin_concluido = 1`.
4. Depois do sucesso, aparece um botão **"Ver informações da estadia"**
que abre o `livro.html?t=TOKEN` na mesma reserva.
5. Durante a estadia, o hóspede pode reabrir o link do livro sempre que
precisar (ao contrário do pré check-in, que é de uso único).

---

## 3. Contexto legal (RGPD)

Os dados recolhidos no `index.html` são tratados ao abrigo do
**artigo 6.º/1/c do RGPD** (obrigação legal) e da **Lei 23/2007,
arts. 15.º e 16.º** (boletim de alojamento de hóspedes estrangeiros).

Por isso, o formulário tem três confirmações com naturezas distintas —
**não é um "aceito tudo"**:

| Confirmação | Natureza | Obrigatória? | Base legal |
|---|---|---|---|
| Declaro ter sido informado sobre o tratamento | Tomada de conhecimento | Sim | art. 13.º |
| Aceito o regulamento da casa | Aceitação contratual | Sim | art. 6.º/1/b |
| Aceito receber comunicações de marketing | Consentimento revogável | Não | art. 6.º/1/a |

O sistema de reservas **deve guardar separadamente** o estado de cada
uma destas três confirmações, com data/hora e a versão do aviso de
privacidade apresentada (constante `VERSAO_AVISO` no `index.html`).
Isto é o que permite, em caso de fiscalização da CNPD, provar o que
foi informação, o que foi contrato e o que foi consentimento.

---

## 4. Integração com a API

### 4.1 Configuração no `index.html`

No início do `<script>`:

```js
const VERSAO_AVISO = "1.0";        // sobe sempre que o texto do aviso mudar
const ENVIO = "ficheiro";          // "ficheiro" (demonstração) ou "api"
const URL_API = "https://«DOMINIO»/api/pre-checkin";