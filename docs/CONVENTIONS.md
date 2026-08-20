# Convenções do Sobra

Instruções para manter consistência entre sessões. Leia isto antes de escrever
código ou texto neste repositório.

Documentos irmãos: [README](../README.md), [arquitetura](ARCHITECTURE.md),
[schema](SCHEMA.md), [decisões](DECISIONS.md).

---

## Regras de escrita

**Nunca use travessão como separador no meio de frase.** Vale para todo texto do
projeto: documentação, comentário de código, mensagem de commit, texto de
interface, prompt do bot. Use vírgula, dois pontos, parênteses ou duas frases.

## Idioma

**Identificador em inglês, valor em português.** Ver [ADR 0013](DECISIONS.md).

| O quê | Idioma | Exemplo |
| --- | --- | --- |
| tabela, coluna, função, índice, constraint, tipo | inglês | `installments`, `due_date`, `create_entry_with_installments` |
| valor de enum | português, sempre | `previsto`, `parcelado`, `detalhado`, `eu_devo` |
| coluna de view nomeada a partir de um valor | português | `confirmado`, `recorrentes`, `avulsos` |
| coluna de view estrutural | inglês | `total`, `confirmado_count`, `balance` |
| interface | português | `sobra`, `lançamentos` |
| documentação e comentário de código | português | |
| README | inglês, com glossário dos termos da tela | |

O README é a única peça em inglês, porque é a única que precisa ser lida por quem
não fala português.

Escreva valores em real no formato brasileiro: `R$ 1.234,56`. **O `R$` acompanha
todo valor, sem exceção**, inclusive em valor de linha de lista e de tabela. Ele vai
num pouco menor que o número e com opacidade menor, para herdar a cor do valor.

Não invente número. Se um dado não existe, escreva que não existe.

## Vocabulário fixo

Estas palavras têm significado exato e não podem ser trocadas por sinônimo.

| Palavra | Significa |
| --- | --- |
| **entry** | a compra ou a conta, uma linha por decisão de gastar |
| **installment** | uma parcela, uma linha por mês de vencimento |
| **recorrente** | sem fim definido. Cobre aluguel, contas de casa, DAS e assinaturas. **Nunca chame de assinatura**, porque nem toda recorrente é assinatura |
| **parcelado** | número fechado de parcelas |
| **avulso** | compra única |
| **previsto** | vai acontecer, ainda não aconteceu |
| **confirmado** | valor é real. Não quer dizer que o dinheiro já saiu da conta: a compra no cartão é confirmada no dia da compra, e sai na fatura. Ver [ADR 0017](DECISIONS.md) |
| **fixo** | recorrente cujo valor é o mesmo todo mês, marcado com `amount_exact`. Confirma sozinho ao vencer |
| **reference_month** | quando o gasto aconteceu, sempre dia 1 |
| **due_date** | quando o dinheiro sai |
| **sobra** | renda prevista menos tudo que sai no mês |
| **reembolso** | gasto de outra pessoa que passou no cartão dela |
| **dívida** | dinheiro que ela pegou emprestado de uma pessoa. Módulo separado, não é um `type` |

`type` tem exatamente três valores. `status` tem exatamente dois. Não adicione um
quarto ou um terceiro sem uma ADR.

**pet** é a categoria dos gastos com animal.

## Regras de modelo que não se negociam

1. **Compra parcelada nunca é uma linha só.** Entry mais N installments, sempre.
   Ver [ADR 0002](DECISIONS.md).
2. **Nenhum número que uma pessoa precisa decrementar.** Saldo de reembolso e
   saldo de dívida são sempre derivados por soma. Ver ADR 0005 e ADR 0006.
3. **Nada escreve em `installments` direto.** Só `create_entry_with_installments`.
4. **Lançamento com `reimburser_id` não entra nos totais de gasto dela.** Nem nos
   números da home, nem no gráfico de categorias.
5. **A soma das parcelas bate com o total ao centavo.** As primeiras ficam com o
   centavo para baixo, a última absorve a diferença.
6. **RLS ligado em toda tabela**, sempre `user_id = auth.uid()`.

## Regras de interface

- **O título da home é o seletor de mês.** As setas ficam coladas nele e não
  existe pílula separada, então o mês aparece escrito uma vez só na tela. O ano vai
  junto, em tom mais claro e peso menor, sem separador: `agosto 2026`.
- **A home é sempre por `due_date`.** Sem toggle de competência e caixa em lugar
  nenhum. A única visão por data da compra é o gráfico de categorias.
  Ver [ADR 0008](DECISIONS.md), e a pendência 3 em Estado do projeto.
- **Três números no topo da home, não um:** confirmado, previsto, sobra.
- **Nenhum dos três leva nota embaixo do valor.** A tela inteira já é o mês, então
  contagem de lançamento e porcentagem ali são ruído.
- **A renda prevista mora no alto do card dos três números, e em nenhum outro
  lugar da tela.** Ela é o dado de entrada, não é um dos três.
- **A legenda da barra só carrega porcentagem, sem palavra.** A cor de cada
  segmento já está ao lado do rótulo do número, logo acima.
- **A barra tem dois estados, e o de estouro não é opcional.** A trilha vale o
  maior dos dois, renda ou gasto. Passando da renda, o segmento de sobra deixa de
  existir, a renda vira marca vertical com rótulo escrito, e o trecho depois dela
  ganha hachura vinho. Ver [ADR 0012](DECISIONS.md) .
- **Lançamento é linha, não card.** Ícone circular colorido por categoria à
  esquerda, descrição e valor na mesma linha.
- **Lançamento é o que já aconteceu.** O card tem duas abas exclusivas, e elas são
  o próprio título dele: `Lançamentos` e `Previsto`, com a contagem ao lado. Não
  existe estado "tudo", e as abas nunca mostram valor em real, porque confirmado e
  previsto já são dois dos três números do topo.
- **Confirmado vai agrupado por dia, previsto não leva data nenhuma.** O cabeçalho
  de dia carrega a data e a linha não repete. Previsto é do mês, e o mês já está no
  seletor do topo da tela, então na aba de previsto não existe cabeçalho de dia nem
  rótulo dizendo que falta data. A ordem é: confirmado do dia mais recente para o
  mais antigo, previsto do maior valor para o menor.
- **Barras, nunca área de círculo.** Sem rosquinha, sem medidor circular, sem
  círculos concêntricos. Ver [ADR 0010](DECISIONS.md).
- **No bloco por categoria, cada categoria é uma linha só:** ícone e nome, barra,
  percentual, valor, nessa ordem. Todas as categorias aparecem, nenhuma é somada
  numa linha de "outras". A barra é proporcional à maior categoria do mês e o
  percentual é sobre o total, então os dois não medem a mesma coisa de propósito:
  a barra compara categorias entre si, o percentual diz o quanto do mês foi ali.
  A barra tem largura mínima de 3px, senão categoria de 1% some.
- **Escala de barra é sempre linear.** Nada de raiz quadrada para dar corpo a
  categoria pequena. Quando o valor é miúdo demais para a forma, quem responde é
  o número escrito ao lado. Ver [ADR 0010](DECISIONS.md).
- **As duas colunas da home terminam na mesma linha.** Quem manda na altura é a
  coluna da direita. A lista de lançamentos não rola: ela mostra um punhado e o
  resto fica atrás do "ver todos", que desce para o pé do card. Embaixo das duas,
  o card de parcelamentos ocupa a largura inteira.
- **Parcelamento se mede em parcela, não em real.** A tela responde quantas faltam.
  Cada compra é uma cápsula deitada num eixo de mês, com um corte por parcela, o
  trecho pago apagado e o que falta escuro. Sem legenda, sem soma em dinheiro, sem
  data da última de todas. As linhas vão ordenadas por quem acaba primeiro.
- **Texto de interface é todo em minúscula.** Título de tela, título de card,
  aba, descrição de lançamento, categoria, nome de cartão e nome próprio, tudo
  minúsculo: `agosto de 2026`, `lançamentos`, `notebook dell`, `cartão azul`, `pet`,
  `pix`. Sigla continua em caixa alta (`DAS`, `IPVA`, `IPTU`), e `R$` também.
- **A única exceção é o micro-rótulo de número, que vai em caixa alta.** É o
  rótulo pequeno e espaçado que fica junto de um valor: `RENDA PREVISTA`,
  `CONFIRMADO`, `PREVISTO`, `SOBRA`, `SOBRA PREVISTA`, `SALDO`, `FALTAM`.
  Ele é escrito em minúscula no HTML e vira caixa alta pelo CSS.
- **Lista de lançamento não tem linha entre os itens.** Quem agrupa é o cabeçalho
  de dia mais o espaço em branco. Com dia de um item só, o traço aparecia em uns e
  não em outros, e o padrão furado lia como defeito.
- **O onboarding tem três passos, nessa ordem: cartões, renda prevista,
  Telegram.** Cartão primeiro porque é o único que muda a conta, já que sem
  `closing_day` a ADR 0014 fica sem entrada e sem cartão padrão o bot pergunta
  qual é a cada lançamento. Renda em seguida porque é ela que faz a sobra
  existir. **Telegram por último porque é o único passo que tira a pessoa do
  app**, e quem sai no meio de um fluxo pode não voltar.
- **Pessoa e categoria não entram no onboarding.** Ninguém sabe responder quem
  vai lhe dever dinheiro antes de usar o app. As duas nascem propostas pelo bot
  no primeiro lançamento que precisar delas. Ver [ADR 0020](DECISIONS.md).
- **Nenhum passo do onboarding é obrigatório, e todos moram também nas
  configurações.** Pulando, a tela diz o que deixa de funcionar, em vez de
  travar.
- **Todo valor monetário usa `font-variant-numeric: tabular-nums`.**
- **A cor nunca é o único identificador.** Categoria sempre tem cor, ícone e nome.

## Direção visual

**Estética Apple.** Fundo creme muito claro, cards em branco quase puro, sem
borda, raio generoso (24px nos cards, pílula completa em botões e dropdowns),
muito espaço em branco, ícone sempre dentro de círculo com traço fino.
Hierarquia por peso e tamanho de tipografia, não por caixa e linha.

**Tipografia.** Geométrica sans com forte contraste de peso. Número grande e
preto, rótulo pequeno e cinza acima.

**Cor.** Malva `#B0779E` é a marca. Para texto, link e rótulo, use `#7A4A68`,
porque o malva puro não tem contraste em corpo pequeno.

A cor da marca não é a cor do dado. Ver [ADR 0009](DECISIONS.md).

```
superfícies   creme  #F7F3EE    branco  #FFFDFC    linha  #EDE6DD
tinta         #1A1613   #5C534B   #8D847A   #B4ABA1
marca         #B0779E   texto #7A4A68   suave #F0E4EC
dados         positivo #4F7A5E   negativo #8C3A46   previsto #9A9189
categorias    #7E94A8 moradia        #8FA68E mercado      #C9A46A restaurantes
  (quinze,    #6E9490 transporte     #B0779E saúde        #C08573 pet
   fechadas)  #B7A88F casa e objetos #9A8FB0 roupas e bel #8A93A0 assinaturas
              #83A8AF lazer          #7D7A9C educação     #8FA9C0 viagem
              #C48B8B presentes      #9C9186 impostos     #9E6B72 dívidas
              #ADA49B outros, neutra de propósito, ver ADR 0023
```

Confirmado em preto. Previsto em cinza médio ou hachurado (a hachura é o marcador
de estimativa, e é a mesma em toda tela). Sobra positiva em verde dessaturado,
negativa em vinho.

**Proibido:** roxo escuro, fundo escuro de qualquer tipo, gradiente pesado, sombra
dramática, ícone genérico de cifrão ou cofre. Sem dark mode no v1.

**Formatos.** Desktop é o formato de projeto, mas precisa funcionar de verdade no
celular, porque é lá que o app é aberto no dia a dia, instalado como PWA. No
mobile, navegação inferior com três ou quatro ícones.

## Stack

React com Vite e TypeScript, publicado na Cloudflare Pages. Supabase para dados e
auth, login com Google, sessão longa. Um Cloudflare Worker recebe o webhook do
Telegram. O bot interpreta linguagem solta com a API da Anthropic, modelo Haiku.

O front não recalcula regra de negócio. Ele lê view e desenha.

## O bot do Telegram

Sempre nessa ordem, antes de qualquer outra coisa: valida o `secret_token` do
header, resolve o `chat_id` para uma pessoa em `telegram_links`, e só então lê a
mensagem. `chat_id` sem vínculo é ignorado em silêncio. O vínculo é criado por
deep link no onboarding, com token de uso único. Ver [ADR 0019](DECISIONS.md).

Regras fixas do prompt:

- **Cartão padrão é o que estiver marcado como padrão nas configurações**, e não
  um nome fixo no prompt. No elenco do mockup esse cartão é o `cartão azul`.
- Nome de pessoa cadastrada em reembolso significa reembolso, nunca gasto dela.
- "previsto", "programado" e "vou gastar" criam status `previsto`.
- **A data de um gasto que já aconteceu é a data da mensagem**, no fuso de
  Brasília. Data escrita na mensagem manda: "gastei 40 no cartão dia 12" grava
  dia 12. Ver [ADR 0015](DECISIONS.md).
- **Previsto sem dia é só o mês**, e o bot não inventa um dia para ele.
- **No dia em que o cartão fecha, a confirmação carrega um botão
  `ainda não fechou`.** O padrão continua sendo a fatura seguinte, o botão é a
  saída. Ele só aparece quando o dia da compra é o `closing_day` daquele cartão,
  e some pelo resto do mês assim que ela confirmar um lançamento sem tocar nele.
  Ver [ADR 0016](DECISIONS.md).
- "pet", "gato" e "gatu" são todos a categoria pet.
- Nome de remédio e de consulta é saúde.
- **Categoria só sai da lista fechada de quinze, e o bot nunca cria uma nova.**
  Ver [ADR 0021](DECISIONS.md).
- **`outros` existe e nunca é a escolha automática.** O bot tenta as quinze,
  pergunta, e só cai em `outros` se ela disser que nenhuma serve. Ver
  [ADR 0023](DECISIONS.md).
- **Categoria repetida não passa pelo modelo, e descrição nova leva o histórico
  dela junto no prompt.** `outros` fica fora desse aprendizado, porque precedente
  de "não sei" ensina o bot a desistir cedo. Ver [ADR 0022](DECISIONS.md).
- **Pessoa de reembolso e de dívida, essa sim, é proposta pelo bot** no primeiro
  lançamento que precisar dela, e nunca criada em silêncio.
- Faltando informação, pergunta. Nunca chuta. A data é a única exceção, porque
  ela tem um padrão que acerta quase sempre.

Sempre responde com o que entendeu e pede confirmação antes de gravar. Havendo
cartão, essa confirmação diz **em que mês o pagamento cai**, sempre com o nome do
mês escrito: `pagamento em outubro`. Nunca relativo. `próximo mês` precisa de uma
âncora, e lançamento retroativo lê ao contrário: compra de 28 de setembro
informada em 3 de outubro sai em outubro, que é este mês e não o próximo. Dia
nenhum aparece ali, porque o dia é sempre o mesmo e o mês é o que muda. Para
confirmar um previsto, procura entre os previstos abertos do mês e, havendo
ambiguidade, lista as opções com botão.

**Nenhum ID aparece na conversa.** Ela nunca precisa saber o ID de nada.

## Repositório público

O repositório é público. Vale para todo commit:

- **Segredo nunca entra em arquivo versionado.** `.env.example` carrega os nomes,
  nunca os valores. Valor vai para `.env.local`, para as variáveis do Cloudflare
  Pages e para `wrangler secret put`.
- **Prefixo `VITE_` quer dizer publicado.** O Vite embute no bundle. Só `VITE_` na
  URL do Supabase, na anon key e na URL do app. Nunca na service role, no token do
  bot ou na chave da Anthropic.
- **A anon key ser pública é esperado**, não é vazamento. Quem protege é o RLS.
- **Dado real nunca é commitado.** Nem dump, nem backup, nem planilha, nem captura
  de tela com número verdadeiro. O que existe em `mockup/` é ficção escrita à mão.
- **O mockup usa um elenco fictício, e ele não muda.** Cartões `cartão azul` e
  `cartão roxo`, pessoas `joão` (reembolso), `ana` e `bia` (dívida), categoria
  `pet`. Nada de nome de banco de verdade, de pessoa de verdade, de remédio ou de
  qualquer outro fato real. Valor falso não basta: a descrição do lançamento também
  conta uma história sobre quem usa o app.
- **Se um segredo for commitado, trocar o segredo, não só apagar o arquivo.** Ele
  fica no histórico do git para sempre. Rotacionar a chave é o conserto, o resto é
  faxina.

## Estrutura de pastas

```
sobra/
├── docs/            ARCHITECTURE.md, SCHEMA.md, DECISIONS.md
├── logo/            identidade, não edite
├── mockup/          fase 1, HTML estático com dados falsos
├── refs/            referências visuais, NÃO versionada, é trabalho de terceiros
├── src/             front (ainda não existe)
├── supabase/        migrations (ainda não existem)
└── worker/          bot do Telegram (ainda não existe)
```

## Como commitar

Um commit é uma mudança coerente, que cabe numa frase e que dá para desfazer
sozinha. **Se a mensagem precisa de "e", são dois commits.**

Commite quando chegar num estado ao qual você voltaria. Experimento pela metade
fica na máquina, não no histórico. Na prática isso dá várias vezes por dia, e o
push acontece ao terminar cada bloco.

Mensagem em inglês, no padrão `tipo: ação no imperativo`, porque a lista de commits
é a peça mais visível do repositório para quem não lê português. Tipos em uso:
`feat`, `fix`, `docs`, `chore`, `refactor`, `assets`.

O corpo explica **por que**, não o que. O diff já diz o que mudou.

```
feat(mockup): make the month title double as the month selector

The month was written twice on the same line, once in the heading and once
inside the pill on the right. Collapsing them removes a control and frees the
whole right side of the header.
```

## Como registrar decisão

`docs/DECISIONS.md` é append-only. Número novo, data, status, e as três seções:
contexto, decisão, consequência.

Nunca edite uma ADR antiga para mudar o que ela decidiu. Escreva uma nova com
status `substitui ADR NNNN` e marque a antiga como `substituída pela ADR NNNN`.

A seção de consequência precisa listar o que se perde, não só o que se ganha.
Uma ADR sem custo declarado não foi pensada.

## Estado do projeto

**Fase 1, em revisão.** Existem documentação e mockup estático. Não existem
backend, autenticação nem banco.

### Em aberto

Nada aqui trava mais a migration. O que faltava do modelo foi decidido nas ADR
0014 a 0017. O que sobrou é tela e portfólio.

1. **A tela de configurações não existe, nem o onboarding.** A ordem dos passos
   está decidida em Regras de interface, o desenho não. O que ela precisa
   guardar, levantado pelas tabelas que hoje não têm porta de entrada nenhuma:

   | O quê | Por que precisa |
   | --- | --- |
   | cartões | nome, fechamento, vencimento, padrão, detalhado ou agregado, cor, arquivar. Sem isso as ADR 0014 e 0016 ficam sem entrada de dado |
   | renda prevista | está escrita no topo da home e não se cadastra em lugar nenhum. É o furo maior |
   | telegram | conectar e desconectar, por deep link. Ver ADR 0019 |
   | pessoas | joão, ana e bia. Criadas pelo bot no uso, editáveis aqui |
   | categorias | as quinze são fixas, ver ADR 0021. Nada a configurar no v1 |
   | conta | e-mail do Google, sair, apagar conta e dados |

2. **`previsto` fixo ainda é hachurado como se fosse estimativa.** A ADR 0017
   criou o previsto de valor certo, e a ADR 0012 e a direção visual dizem que a
   hachura marca estimativa. Ou a hachura muda de significado, ou o previsto
   certo ganha um segundo tratamento. É decisão de tela.
3. **A tela do mês não existe ainda.** O "ver mais" do card de lançamentos leva
   para uma tela própria em largura cheia, com os lançamentos do mês inteiro,
   busca e filtro por categoria e por cartão. Decidido, ainda não desenhado. A
   home não estende a lista no lugar, justamente para as duas colunas continuarem
   terminando na mesma linha.
4. **Falta imagem no topo do README.** Fica para o fim da fase de telas, quando
   existirem home, login e as outras páginas prontas. É o item de maior impacto
   isolado num repositório de portfólio, então não pode ser esquecido.
5. **Falta publicar a demo.** O mockup é estático e cabe na Cloudflare Pages, com
   o link no topo do README. Também fica para o fim, junto com a imagem.
6. **A ADR 0008 exige que a visão por data da compra leve rótulo escrito por
   extenso, e esse rótulo saiu da tela.** O texto era um rodapé no card de
   categorias e foi recusado na revisão. Ou o rótulo volta em outra forma, ou a
   ADR 0008 precisa de uma nova que a substitua nesse ponto.

Não construa Supabase, Telegram ou auth até a tela estar aprovada.
