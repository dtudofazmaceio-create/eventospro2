# Enlace — Verificação dos painéis e melhorias

Marketplace de casamentos com 4 painéis que compartilham o **mesmo** Firebase
Realtime Database (`imobpro-667da`). A comunicação entre eles é feita inteiramente
por nós (paths) do banco — não há backend próprio.

## Mapa de comunicação (quem escreve / quem lê)

| Path no banco        | Escreve                                  | Lê                                            |
|----------------------|------------------------------------------|-----------------------------------------------|
| `users/{uid}`        | todos (próprio perfil) · Gestor          | cada painel (auth) · Gestor (todos)           |
| `professionals/{id}` | Fornecedor (próprio perfil público)      | Painel Noivos (lista de fornecedores)         |
| `profAds/{id}`       | Fornecedor (próprios anúncios)           | Marketplace · Fornecedor · Gestor             |
| `ads/{id}`           | Gestor                                   | Marketplace · Fornecedor                      |
| `requests/{id}`      | Noivos (cria) · Fornecedor (status)      | Noivos (filtra clientId) · Fornecedor (profId)|
| `siteSettings`       | Gestor                                   | **Marketplace** (corrigido)                   |

## Ciclo de vida da solicitação (funcionava — verificado)

Noivos cria `pending` → Fornecedor **Aceita** (`active`) / **Recusa** (`cancelled`)
→ Fornecedor **Conclui** (`completed`). O painel Noivos vê a atualização em tempo
real porque filtra `requests` por `clientId`; o Fornecedor filtra por `professionalId`.

## Problemas encontrados e correções aplicadas

1. **Navegação entre painéis quebrada.** Os links internos apontam para
   `index.html`, `painel-cliente.html`, `painel-profissional.html` e
   `painel-gestor.html`, mas os arquivos tinham nomes `deepseek_html_*`.
   → Arquivos renomeados para os nomes corretos. Mantenha exatamente esses nomes.

2. **`siteSettings` era um recurso "morto".** O Gestor salvava imagem/título/
   subtítulo do hero e a galeria, mas o Marketplace nunca lia esses dados.
   → O Marketplace agora ouve `siteSettings` e atualiza hero (imagem de fundo,
   título, subtítulo) e a galeria em tempo real.

3. **Sem moderação no Gestor.** Ele listava `profAds` e `users` mas não podia agir.
   → Adicionados: remover anúncio de profissional e remover usuário (não permite
   remover a si mesmo nem outros admins). A remoção de profissional também limpa
   o nó `professionals/{id}`.

4. **Sem regras de segurança no banco (risco grave).** Sem regras, qualquer pessoa
   — até sem login — podia ler/gravar qualquer path, inclusive se autopromover a
   admin gravando `users/{uid}/role: "admin"`, ou editar anúncios/solicitações de
   terceiros. → Criado `database.rules.json` (ver deploy abaixo).

5. **Falhas silenciosas.** Listeners `.on(...)` sem callback de erro.
   → Adicionado tratamento de erro nos listeners (erros de permissão agora
   aparecem no console em vez de falhar em silêncio).

## Deploy das regras de segurança

No console do Firebase → Realtime Database → aba **Regras**, cole o conteúdo de
`database.rules.json` e publique. Ou via CLI:

```bash
firebase deploy --only database
```

### Importante — primeiro administrador (bootstrap)

As regras impedem que um usuário se promova a `admin` (essa é a proteção
principal). Por isso, o **primeiro** admin precisa ser criado manualmente:

1. Crie a conta normalmente pela aba de cadastro do painel do Gestor
   (código de acesso atual: `EVENTLINK-ADMIN`). Ela nascerá bloqueada pelas regras
   ao tentar gravar `role: "admin"` — então:
2. No console do Firebase → Realtime Database, edite `users/{uid}/role` para
   `admin` no usuário desejado.

A partir daí, admins existentes podem criar/gerenciar outros usuários normalmente.

## Pontos de atenção (recomendações, não bloqueantes)

- **Privacidade das solicitações:** com a arquitetura atual, cada painel lê a
  coleção `requests` inteira e filtra no cliente. As regras exigem login para ler,
  mas qualquer usuário autenticado ainda consegue, tecnicamente, ler todas as
  solicitações. Para privacidade estrita, reestruture para
  `requests/{clientId}/{reqId}` e `requestsByPro/{professionalId}/{reqId}`.
- **Código de admin no cliente:** `EVENTLINK-ADMIN` está no código-fonte do
  painel. Ele é apenas uma barreira de conveniência; a segurança real vem das
  regras do banco.
- **Chave de API exposta:** normal em apps web client-side. A proteção vem das
  regras do banco e do Firebase Authentication, não de esconder a chave.
