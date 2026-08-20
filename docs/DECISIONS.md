# Decisões

Registro append-only. Cada decisão ganha um número que nunca é reaproveitado e
uma data que nunca muda. Decisão errada não é apagada, é substituída por uma nova
com status `substitui ADR NNNN`.

Formato: contexto, decisão, consequência.

---

## ADR 0001, guardar os dados no Supabase em vez de falar com a API do Google Sheets

**Data:** 2026-08-14
**Status:** aceita

### Contexto

O controle já existia numa planilha do Google Sheets, um bloco por mês. A opção
mais barata em esforço seria manter a planilha como banco e escrever um front que
lê e escreve pela API do Sheets. A planilha continuaria aberta para edição manual,
o que é confortável.

Só que a planilha não tem tipo, não tem restrição e não tem relação. O que quebrou
na planilha (contador de parcela escrito à mão, sem categoria, sem visão de futuro)
não quebrou por falta de interface. Quebrou por falta de modelo. Escrever um front
bonito por cima da mesma estrutura levaria os mesmos erros para um lugar mais caro
de manter.

Além disso a API do Sheets tem cota por minuto, latência alta para leitura de
intervalo e nenhuma noção de transação. Materializar N parcelas de uma compra
exigiria escrever N linhas sem garantia de atomicidade.

### Decisão

Postgres no Supabase. A planilha vira insumo de migração inicial e depois é
aposentada.

### Consequência

- Ganha tipo, restrição, transação, view e Row Level Security.
- Ganha `SUM` agrupado por mês, que é a operação que o produto inteiro depende.
- Perde a edição manual em massa, que era confortável. Compensa com o bot do
  Telegram, que cobre o caso "lançar rápido sem abrir o app".
- Passa a existir um esquema para manter e migrations para versionar.
- O plano gratuito do Supabase pausa projeto ocioso. Para um app de uso diário
  isso não deve acontecer, mas é um risco conhecido.

---

## ADR 0002, separar `entries` de `installments`

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Esta é a decisão que existe para consertar o problema número um da planilha. Lá,
uma compra parcelada era uma linha só, com um contador de parcelas restantes
escrito à mão e decrementado todo mês. Já foi errado pelo menos uma vez, e o pior
dessa classe de erro é que ela não avisa: o número fica errado e continua parecendo
certo.

O desenho alternativo seria guardar a compra numa linha só com `installments_total`
e `installments_paid`, e calcular o resto na leitura. Isso resolve o erro humano,
mas continua não resolvendo a pergunta mais importante do produto, que é quanto de
um mês futuro já está comprometido. Para responder isso seria preciso projetar em
tempo de consulta, todo mês, para toda compra ativa.

### Decisão

A compra fica em `entries`. Cada mês de vencimento vira uma linha em `installments`.
Uma compra de R$ 500,00 em 5x é uma `entry` e cinco `installments` de R$ 100,00.

Materializar é responsabilidade de `create_entry_with_installments`, que grava a
entry e as parcelas na mesma transação. Nenhum cliente escreve em `installments`
diretamente.

### Consequência

- "Quanto de janeiro já está comprometido" vira `sum(amount) group by month`.
  Sem projeção, sem contador, sem loop.
- Nenhum número precisa ser decrementado por uma pessoa, então nenhum número pode
  ficar errado em silêncio.
- Editar uma compra parcelada depois de criada é mais caro: significa recriar
  parcelas futuras, preservando as já confirmadas.
- Recorrentes não têm fim, então precisam de um horizonte rolante materializado.
  É trabalho extra, e um horizonte parado vira tela vazia no sexto mês. Como
  manter esse horizonte é a [ADR 0011](#adr-0011-o-horizonte-dos-recorrentes-é-estendido-no-boot-do-app-e-não-por-job-agendado).
- Arredondamento vira regra explícita: as primeiras parcelas ficam com o centavo
  para baixo e a última absorve a diferença.

---

## ADR 0003, `previsto` e `confirmado` como status na mesma tabela

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Na planilha, gasto estimado era marcado com fundo amarelo. É a mesma linha, com a
mesma descrição e a mesma categoria, só que ainda não aconteceu. A alternativa
seria uma tabela `planned_expenses` separada, promovida para `installments` quando
o gasto acontece.

Duas tabelas com os mesmos campos significam duas escritas, dois lugares para
editar, e toda consulta virando `union all`. E significaria que a home precisaria
somar de dois lugares para dar o número de "ainda vai sair".

### Decisão

Um enum `installment_status` com exatamente dois valores, `previsto` e
`confirmado`, na própria linha da parcela. Todo mês futuro nasce `previsto`.
Confirmar é um `update` que troca o status e, quando o valor real difere da
estimativa, corrige o `amount`.

### Consequência

- Os três números da home saem de uma consulta só, com `filter (where status = ...)`.
- Confirmar preserva o histórico da linha, não cria linha nova.
- Só existem dois estados. Não cabe "parcialmente pago" nem "atrasado" sem uma
  nova decisão. Atraso, se virar necessidade, é `due_date < today and status =
  'previsto'`, que é derivado, não um terceiro estado.
- O valor de um previsto muda quando confirma. Isso quer dizer que o total de um
  mês passado pode ser diferente do que era antes, o que é correto, mas exige que
  a interface deixe claro o que é estimativa.

---

## ADR 0004, login com Google em vez de magic link

**Data:** 2026-08-14
**Status:** aceita

### Contexto

O app tem uma usuária. Magic link por e-mail é mais simples de configurar, não
exige registrar aplicação em provedor nenhum e não depende de OAuth consent screen.

O problema é o uso real. O app vive instalado como PWA na tela de início do
celular, aberto várias vezes por dia, às vezes em segundos, para lançar uma compra
no caixa do mercado. Magic link significa: abrir, esperar o e-mail, sair do app,
abrir o e-mail, voltar. Toda vez que a sessão expira. Isso é atrito o bastante
para o app não ser usado.

### Decisão

OAuth com Google, via Supabase Auth, com refresh token de longa duração e sessão
persistida.

### Consequência

- Login uma vez. O objetivo é nunca ver tela de login no dia a dia.
- Dependência do Google como provedor de identidade, e de manter uma aplicação
  OAuth registrada.
- O e-mail do Google vira a identidade. Trocar de conta significa migrar dados.
- Precisa configurar a URL de redirect no Supabase e no Google Cloud, o que é
  chato uma vez e nunca mais.

---

## ADR 0005, reembolso derivado da soma dos lançamentos

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Outra pessoa usa o cartão roxo dela. As compras dela passam no cartão dela,
aparecem na fatura dela, e ela paga de volta. O jeito óbvio seria um campo "fulano me deve
R$ X", atualizado à mão.

Mas isso é exatamente o mesmo erro do contador de parcelas do ADR 0002: um número
mantido por uma pessoa, que fica errado sem avisar. E ainda por cima um número
que muda toda vez que ele usa o cartão.

### Decisão

O lançamento recebe `reimburser_id`. Quanto uma pessoa deve é sempre
`sum(amount)` dos lançamentos com aquele `reimburser_id`, exposto pela view
`v_reimbursements`. Não existe campo de saldo de reembolso em lugar nenhum.

Consequência do desenho: lançamento com `reimburser_id` **não entra** nos totais
de gasto dela, nem nos três números da home, nem no gráfico de categorias. O
dinheiro sai do cartão dela, mas o gasto é de outra pessoa. Se a fatura do cartão roxo
for maior que o total dele, a diferença é gasto dela, e aparece sozinha, porque o
que não tem `reimburser_id` já entrou nos totais normais.

### Consequência

- O número nunca fica errado, porque não existe para ficar errado.
- A fatura do cartão bate por construção: total da fatura menos total dele é igual
  ao gasto dela naquele cartão.
- Quando ele paga de volta, o pagamento não é lançado como receita. O que muda é
  o mês de referência dos lançamentos dele, que saem da conta de "a receber".
  Isso ainda precisa de uma decisão própria sobre o que fazer com um reembolso
  que atrasa e cruza o mês.
- Reembolso parcial não é representável hoje. Se acontecer, vira nova ADR.

---

## ADR 0006, dívidas como módulo separado, não como um `type`

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Dívida com pessoa (dinheiro emprestado por alguém próximo) parece um gasto
parcelado: tem um valor original, uma série de pagamentos, e um saldo que cai. Daria
para representar como `type = 'parcelado'` com N parcelas.

Só que as duas coisas respondem perguntas diferentes. Um parcelado responde
"quanto sai por mês". Uma dívida responde "quanto ainda falta, e quando isso
acaba". E o pagamento de dívida raramente é uma série fixa: às vezes paga mais,
às vezes pula um mês, às vezes quita de uma vez. Encaixar isso em parcelas fixas
significaria recriar a série a cada pagamento fora do combinado.

### Decisão

Duas tabelas próprias, `debts` e `debt_payments`, com tela própria. O saldo é
sempre derivado, `principal - sum(payments)`, na view `v_debt_balances`.

Cada pagamento entra também como lançamento `avulso` na categoria dívidas, ligado
pelo `debt_payments.entry_id`. Assim o dinheiro aparece na home no dia em que sai,
e o saldo aparece na tela de dívidas.

### Consequência

- Pagamento irregular é o caso normal, não a exceção.
- O saldo é derivado, seguindo o mesmo princípio do ADR 0005.
- Um pagamento vive em dois lugares, o que exige que quem cria escreva os dois de
  uma vez. Vale a pena, porque a alternativa seria a home não mostrar dinheiro
  que de fato saiu.
- A tabela suporta as duas direções (`eu_devo` e `me_devem`), mas o v1 só tem tela
  para o que ela deve. Dinheiro que devem a ela já é coberto por reembolso.

---

## ADR 0007, importar fatura em PDF ou foto de cupom fica fora do v1

**Data:** 2026-08-14
**Status:** aceita

### Contexto

O cartão principal demora muito para atualizar a fatura, e por isso cada
compra é lançada na mão. A ideia de fotografar o cupom ou jogar o PDF da fatura no
bot resolve o lançamento manual de uma vez.

Mas exige um modelo com visão. Cada chamada passa a carregar uma imagem ou um PDF
inteiro, o que custa uma ordem de grandeza mais do que interpretar uma frase curta.
E o resultado precisaria de revisão item a item de qualquer jeito, porque fatura
tem estorno, taxa, parcela de compra antiga e descrição truncada.

O problema real, que é não conseguir ver o futuro, não é resolvido por importação.
É resolvido pelo modelo de dados do ADR 0002.

### Decisão

Fora do v1. O lançamento é manual, pelo app ou pelo bot em linguagem solta.

### Consequência

- Custo por chamada fica baixo e previsível, com Haiku sobre texto curto.
- Lançar continua sendo trabalho manual, o que já é o hábito de hoje.
- Se voltar à mesa, o caminho mais provável é importar CSV ou OFX da fatura, que
  não precisa de visão nenhuma.

---

## ADR 0008, a home é sempre por data de saída, sem toggle de competência e caixa

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Todo gasto tem dois momentos: quando aconteceu e quando o dinheiro sai. Uma compra
parcelada em maio ainda sai em janeiro. Ferramentas de finanças costumam resolver
isso com um botão de trocar entre as duas visões.

Um botão desses transforma toda leitura de todo número numa pergunta anterior:
"em qual modo eu estou?". Num app cuja razão de existir é responder "quanto sobra"
com confiança, isso é caro.

E as duas visões não têm o mesmo peso. A pergunta do dia a dia é sobre o dinheiro
saindo da conta. A pergunta sobre onde o dinheiro foi gasto aparece uma vez por
mês, olhando a distribuição.

### Decisão

Nenhum toggle na interface. A home, os próximos meses e as dívidas são sempre por
`due_date`, quando o dinheiro sai. A visão por `reference_month`, data da compra,
aparece só no gráfico de categorias, com o rótulo escrito por extenso na tela.

O schema guarda os dois campos, então a decisão é de interface e pode mudar sem
migração.

### Consequência

- Nenhum número da tela precisa de contexto de modo para ser lido.
- O gráfico de categorias não soma o mesmo total da home do mesmo mês, e isso é
  esperado. O rótulo escrito por extenso existe justamente para isso.
- Quem quiser a distribuição por data de saída não tem, no v1.

---

## ADR 0009, a cor da marca não é a cor do dado

**Data:** 2026-08-14
**Status:** aceita

### Contexto

A identidade é malva `#B0779E`. O caminho natural seria pintar os números e as
barras de malva, o que deixa a tela coesa com a marca.

O problema é que quando a cor da marca também é a cor do dado, ela perde a
capacidade de significar qualquer coisa. Uma barra malva ao lado de um número
malva ao lado de um logo malva não diz se aquilo é bom, ruim, real ou estimado.

E este app precisa dessa distinção com força: confirmado contra previsto, sobra
positiva contra negativa.

### Decisão

Malva fica reservado para marca e para o elemento principal de identidade.

Os estados usam neutro: confirmado em preto, previsto em cinza médio ou hachurado.
Sobra positiva em verde bem dessaturado, negativa em vinho. As categorias usam uma
paleta análoga e dessaturada (malva, sálvia, ocre, azul-acinzentado, terracota
suave), nunca arco-íris saturado.

Para texto, link e rótulo, o malva puro é substituído por `#7A4A68`, porque
`#B0779E` não tem contraste suficiente em corpo pequeno.

### Consequência

- Hachura vira o marcador visual de "estimativa", consistente entre a barra
  empilhada da home e as barras dos próximos meses.
- A tela fica menos colorida do que a marca sugere, e isso é intencional.
- Treze categorias dentro de uma paleta dessaturada e análoga significa que as
  cores ficam parecidas entre si. Por isso a cor nunca é o único identificador:
  toda categoria tem também um ícone e um nome escrito.

---

## ADR 0010, gráfico de barras em vez de círculos concêntricos

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Uma das referências visuais usa círculos concêntricos para comparar valores.
É bonito e chama atenção.

Área de círculo é péssima para comparar valor. As pessoas leem raio, não área, e
o raio cresce com a raiz do valor, então a diferença percebida é sistematicamente
menor do que a real. Em círculos concêntricos ainda se soma o problema de comparar
formas que se sobrepõem, sem linha de base comum.

### Decisão

Barras, com origem comum, para categoria e para próximos meses. Comprimento
alinhado a partir de uma base, que é a comparação visual mais precisa que existe.

### Consequência

- A tela fica menos chamativa do que a referência, e mais legível.
- Barra horizontal deixa o nome da categoria e o valor no mesmo eixo de leitura,
  e funciona igual no desktop e no celular sem rotacionar rótulo.
- Nada de rosquinha, medidor circular ou área proporcional em lugar nenhum do app.

---

## ADR 0011, o horizonte dos recorrentes é estendido no boot do app, e não por job agendado

**Data:** 2026-08-14
**Status:** aceita

### Contexto

Parcelado tem fim conhecido, então as N parcelas são gravadas de uma vez e nunca
mais precisam de manutenção. Recorrente não tem fim, e não dá para gravar infinitas
linhas. A [ADR 0002](#adr-0002-separar-entries-de-installments) resolveu isso com
um horizonte rolante de 12 meses materializados, o que transferiu o problema para
outro lugar: alguém precisa empurrar esse horizonte conforme o tempo passa.

A opção óbvia era um job diário no banco, com `pg_cron`. Funciona, mas tem um
defeito específico: ele mantém dado atualizado para um app que ninguém está
olhando, e quando para de rodar não existe nenhum momento em que isso apareça.
Um agendamento quebrado é silencioso pela própria natureza, e o custo dele aparece
meses depois, como número errado para menos numa tela cuja razão de existir é
dizer quanto sobra.

A tela mostra seis meses à frente e o horizonte é de doze, então existe margem de
dois para um antes de qualquer coisa ficar visível.

### Decisão

Nenhum job agendado. A função `sobra_extend_recurring` é chamada pelo front no
boot do app e pelo Worker do Telegram antes de gravar.

A função preenche a partir do último mês já gravado, nunca a partir de hoje. Isso
é o que torna a decisão segura: rodar atrasada preenche o que faltou, em vez de
criar um buraco no meio. Ela também é idempotente, então rodar duas vezes seguidas
não cria nada na segunda.

### Consequência

- Some uma peça de infraestrutura, e some junto a classe de erro de agendamento
  parado sem ninguém saber.
- A manutenção fica amarrada ao uso. Se o dado importa é porque alguém abriu, e
  se alguém abriu ele acabou de ser estendido.
- Ficar meses sem abrir o app encurta o horizonte. Isso é aceitável porque o uso
  real olha três meses à frente, e a volta recompõe tudo antes da primeira leitura.
- **O horizonte pode ficar curto, mas nunca esburacado.** Essa é a garantia que
  faz a decisão valer.
- Custo real: a chamada precisa ser aguardada antes da primeira leitura de dados
  no boot. Sem isso, uma volta depois de muito tempo renderiza a tela com o
  horizonte ainda curto. É uma linha fácil de esquecer de escrever.
- Se um dia o app deixar de ser de uma pessoa só, ou passar a mandar notificação
  sem ninguém abrir nada, esta decisão precisa ser revista.

---

## ADR 0012, os rótulos dos três números, e o estado de estouro da barra

**Data:** 2026-08-17
**Status:** aceita

### Contexto

Revisão do bloco do topo da home. Quatro problemas apareceram juntos.

O nome **já saiu** promete extrato bancário e entrega checklist. `confirmado` é um
ato manual dela, não um débito observado. Uma parcela que venceu dia 5 e não foi
confirmada aparece como se ainda fosse sair. E **ainda vai sair** é comprido demais
para um rótulo em caixa alta.

A **legenda da barra** repetia informação que já estava logo acima. Os
quadradinhos de cor estão ao lado dos rótulos dos números, então cada valor
aparecia três vezes: como número, como largura e como porcentagem com o nome
escrito de novo.

As **notas embaixo dos valores** diziam coisas que a tela já diz. Contagem de
lançamento e "sobre renda prevista de tanto" ocupavam a linha mais nobre do card
para repetir contexto.

A **renda prevista** era o dado de onde todos os outros saem, e era o único sem
lugar fixo. Aparecia duas vezes de carona, como nota embaixo da sobra e como
último item da legenda.

Foi testada uma versão com quatro números, promovendo a renda ao mesmo peso dos
outros três. Ela foi recusada na revisão: os quatro valores dividindo a mesma
largura derrubaram a tipografia do número de 40px para 32px, e o bloco perdeu o
impacto que é a razão de ele existir.

### Decisão

Continuam três números: confirmado, previsto, sobra. Os rótulos passam a ser
`confirmado` e `previsto`, as mesmas duas palavras do enum `installment_status` e
do vocabulário do projeto. Nenhuma palavra nova para aprender.

Nenhum dos três leva nota embaixo do valor.

A renda prevista ganha uma linha própria no alto do card, acima dos três, com o
valor e quanto dele já foi recebido. É o único lugar da tela onde ela aparece.

A legenda da barra fica, mas só com a porcentagem. Sem palavra, sem renda.

A barra passa a ter dois estados. A trilha vale o maior dos dois, renda ou gasto.
Num mês que cabe na renda, a trilha é a renda e o último segmento é a sobra em
verde. Num mês que estoura, a trilha é o gasto, o segmento de sobra deixa de
existir, a renda vira uma marca vertical com o rótulo `renda` escrito, e o trecho
depois da marca recebe hachura vinho por cima. A porcentagem da legenda é sempre
fatia da trilha, e o valor do estouro aparece em real, não em porcentagem.

Os dois estados estão desenhados em `mockup/estados-do-topo.html`.

As colunas de `v_month_cashflow` passam de `ja_saiu` e `ainda_vai_sair` para
`confirmado` e `previsto`, para casar com o rótulo que ela lê na tela.

Não mexe em `installment_status`, que continua com dois valores.

### Consequência

- O nome do número passa a dizer a verdade sobre o que ele mede, que é o que ela
  confirmou, não o que o banco debitou.
- Cada valor aparece uma vez como número, uma vez como largura e uma vez como
  porcentagem, e a porcentagem não repete o nome.
- A renda prevista sai de nota de rodapé e vira o cabeçalho do card, sem custar
  tamanho de tipografia aos três números.
- **O mês que estoura tem desenho.** Era o furo mais grave da versão anterior, e
  era justo o mês que mais importa.
- **Custo:** a barra tem duas escalas. Num mês normal a porcentagem é fatia da
  renda, num mês estourado é fatia do gasto. A marca com rótulo escrito é o que
  avisa da troca, e ela é obrigatória por isso. Sem o rótulo, a barra mente.
- **Custo:** a legenda sem palavra só funciona porque está a poucos pixels dos
  rótulos que carregam as mesmas cores. Se a barra for separada dos números, ou
  reusada em outra tela, ela volta a precisar de legenda escrita.
- **Custo:** três porcentagens soltas na legenda dependem inteiramente da cor para
  serem lidas, o que contraria a regra de que cor nunca é o único identificador.
  Aqui passa porque o quadradinho de previsto é hachurado e não só cinza, e porque
  a ordem da legenda é a mesma ordem dos números acima. É uma exceção estreita e
  não vale para categoria.
- **Custo:** some da tela a contagem de lançamentos confirmados e previstos. Quem
  quiser esse número tem que contar na lista abaixo.
- A ADR 0003 fala em três números na consequência dela, e continua valendo.

---

## ADR 0013, fronteira de idioma entre estrutura e domínio

**Data:** 2026-08-17
**Status:** aceita

### Contexto

O repositório é público e serve de portfólio. Isso levantou a pergunta do idioma,
e a auditoria mostrou que o problema não era o idioma escolhido, era não existir
escolha nenhuma.

O schema estava metade e metade, sem regra. Dois enums em português
(`entry_type` com `recorrente`, `parcelado`, `avulso`, e `installment_status` com
`previsto`, `confirmado`) e dois em inglês (`card_mode` com `detailed` e
`aggregate`, `debt_direction` com `i_owe` e `they_owe`). Colunas de view
misturando `month` com `qtd_confirmados` e `renda_prevista`. Funções
`create_entry_with_installments` e `confirm_installment` convivendo com
`sobra_estender_recorrentes` e `sobra_seed_categorias`. Índices e constraints
todos em português.

Meia língua por acaso lê como falta de decisão. Meia língua por regra escrita lê
como critério.

A alternativa considerada foi levar tudo para o inglês, inclusive os valores de
enum. Ela foi recusada porque exigiria substituir a ADR 0003, que fixou `previsto`
e `confirmado` como as duas palavras exatas do domínio, e porque `sobra`, que dá
nome ao produto, não tem tradução que preserve o sentido.

### Decisão

**Identificador em inglês, valor em português.**

Nome de tabela, de coluna, de função, de índice, de constraint e de tipo é em
inglês, porque é a língua da plataforma e do Postgres.

Valor de enum é em português, sempre, porque valor de enum é vocabulário de
domínio e o domínio é o dinheiro de uma pessoa que pensa em português. Por isso
`card_mode` passa a ser `detalhado` e `agregado`, e `debt_direction` passa a ser
`eu_devo` e `me_devem`.

Coluna de view nomeada a partir de um valor de domínio mantém a palavra do
domínio: `confirmado`, `previsto`, `recorrentes`, `parcelas`, `avulsos`. Coluna
estrutural é em inglês: `total`, `confirmado_count`, `entry_count`, `paid`,
`balance`, `last_payment`.

A interface continua inteira em português, e a documentação também. O README é a
única peça em inglês, porque é a única que precisa ser lida por quem não fala
português, e ele carrega um glossário dos termos que aparecem nas telas.

### Consequência

- O schema passa a ter uma regra que se explica em uma frase, e qualquer nome novo
  é decidido sem discussão.
- O vocabulário fixo da ADR 0003 continua intacto, e o produto continua falando a
  língua de quem usa.
- **Custo:** quem lê o SQL vê duas línguas na mesma linha, como em
  `sum(amount) filter (where status = 'confirmado') as confirmado_count`. Sem a
  regra escrita ao lado, isso parece descuido. A regra precisa estar visível, e é
  por isso que ela está em CONVENTIONS.md e no README.
- **Custo:** o README vira uma peça em idioma diferente do resto, então toda
  mudança de escopo precisa ser refletida em duas línguas. É uma página só, mas é
  trabalho recorrente.
- **Custo:** valor de enum em português trava a internacionalização futura. Se um
  dia o app precisar de outro idioma, os valores gravados no banco continuam
  `previsto` e `confirmado`, e a tradução vira responsabilidade da camada de
  interface. Para um app de uma pessoa só, é um custo que não chega.

---

## ADR 0014, o fechamento do cartão desloca a data de saída, nunca o mês de competência

**Data:** 2026-08-20
**Status:** aceita

### Contexto

`closing_day` existia em `cards` desde o primeiro desenho e nenhuma linha de
código a lia. `sobra_due_date` montava o vencimento com `due_day` só, sempre
dentro do mês da compra.

Isso não é um erro de arredondamento, é um mês inteiro de diferença. Num cartão
que fecha dia 2 e vence dia 7, quase toda compra do mês pertence à fatura que
fecha no dia 2 do mês seguinte. Uma compra de 15 de setembro recebia vencimento
em 7 de setembro, uma data anterior à própria compra.

São dois deslocamentos distintos, e a versão anterior tratava os dois como
nenhum:

1. **Compra a partir do dia do fechamento entra na fatura seguinte.** Fecha dia
   2, compra no dia 2 de setembro, fatura que fecha em 2 de outubro.
2. **Vencimento anterior ao fechamento vence no mês seguinte ao que a fatura
   fecha.** Fecha dia 25 e vence dia 5: a fatura que fecha em 25 de agosto vence
   em 5 de setembro. Quando o vencimento é posterior ao fechamento, como fecha 2
   e vence 7, os dois caem no mesmo mês.

A alternativa considerada foi pedir a data de saída digitada a cada lançamento.
Ela foi recusada porque obriga ela a fazer a conta de cabeça toda vez, e o ponto
do app é justamente ela não precisar.

### Decisão

**O fechamento desloca `due_date`, e nunca `reference_month`.** A compra
aconteceu no mês em que aconteceu, só o dinheiro sai depois. O gráfico de
categorias, que agrupa por `reference_month`, não muda. A home, que agrupa por
`due_date`, passa a acertar o mês.

O deslocamento vira uma função, `sobra_due_shift`, que vale 0, 1 ou 2 meses, e
uma função de conveniência `sobra_due_month`. `sobra_due_date` continua fazendo
uma coisa só: colocar `due_day` dentro de um mês já resolvido, encurtando para o
último dia quando o mês é mais curto que o dia.

A comparação com o fechamento é `>=`, então **a compra feita no próprio dia do
fechamento já é da fatura seguinte**.

`closing_day` nulo quer dizer sem fechamento, deslocamento zero, e `due_day` nulo
quer dizer que o dinheiro sai no dia da compra. Juntos, os dois nulos descrevem
débito e dinheiro, e um cartão cadastrado pela metade cai no comportamento mais
conservador em vez de inventar o último dia do mês.

Em parcelado, as N parcelas andam juntas a partir do mês da primeira fatura, e
as N carregam o mesmo `reference_month`, o mês da compra. Em recorrente,
`reference_month` e mês de fatura andam em trilhos paralelos, com a distância
fixa que `sobra_due_shift` devolveu.

### Consequência

- A home passa a mostrar a compra no mês em que o dinheiro sai de verdade, que é
  a única promessa que ela faz (ADR 0008).
- O gráfico de categorias continua contando a compra no mês em que ela
  aconteceu, sem toggle e sem exceção.
- **Custo:** o cadastro de cartão passa a ter dois números que precisam estar
  certos, e um deles ela vai ter que procurar na fatura. `closing_day` errado
  desloca todas as compras do fim do mês, em silêncio e sem erro na tela.
- **Custo:** mudar `closing_day` depois não recalcula o que já foi gravado. As
  parcelas existentes ficam com a data antiga, porque nada escreve em
  `installments` fora de `create_entry_with_installments` (ADR 0002). Corrigir um
  cartão cadastrado errado é apagar e relançar.
- **Custo:** num cartão com vencimento antes do fechamento, o deslocamento vale
  2, e uma compra de agosto aparece vencendo em outubro. Está certo, e parece
  defeito para quem olha sem a regra ao lado.
- **Custo:** a regra `>=` não é universal entre emissores. Alguns colocam a
  compra do próprio dia do fechamento ainda na fatura que fecha. Se aparecer um
  cartão assim, ele vai precisar de uma coluna, e esta ADR de uma substituta.

---

## ADR 0015, a data vem da mensagem, e previsto sem dia vive no mês

**Data:** 2026-08-20
**Status:** aceita

### Contexto

`installments.due_date` é `not null`, então toda parcela é obrigada a ter um dia,
inclusive as que ninguém sabe qual é. Dizer "em setembro vou no dentista, uns
R$ 400" dá o mês e não dá o dia, e mesmo assim o banco exige uma data. O sistema
inventa uma, e uma vez gravada ela fica idêntica a uma data verdadeira. Não
existe como perguntar depois se aquele dia é real ou é enchimento.

Três coisas quebram por causa disso:

1. `confirm_installment` trocava `status`, `amount` e `confirmed_at`, e não tocava
   em `due_date`. O previsto com dia inventado, ao ser confirmado, guardava o dia
   inventado, e confirmado é agrupado por dia na home. O lançamento aparecia no
   cabeçalho de dia errado, sem nada na tela denunciando.
2. Atraso, que a ADR 0003 deixou como derivado de `due_date < today`, dispara
   sozinho num dia que nunca foi prometido.
3. A ADR 0014 decide em que fatura a compra cai olhando o dia da compra. Sem dia,
   não há fatura, e a diferença entre uma fatura e a seguinte é um mês inteiro de
   sobra.

A pendência aberta chamava isso de `due_day_exato`. O nome está em português e é
coluna estrutural, o que contraria a ADR 0013.

### Decisão

**A data de um gasto que já aconteceu é a data da mensagem.** O bot não pergunta.
Quando ela escreve a data junto, como em "gastei 40 no cartão dia 12", a data
escrita manda. Só isso: ou o que ela disse, ou hoje.

**O hoje é o dela, não o do servidor.** O Worker roda na Cloudflare e o Postgres
responde em UTC, e o cartão fecha num dia baixo do mês. Uma mensagem enviada às
23h30 do dia 1 é dia 2 em UTC, cai do outro lado do fechamento e desloca a fatura
em um mês inteiro. A data passa a ser resolvida em `America/Sao_Paulo`, por
`sobra_today()`, e `current_date` some do schema.

**Previsto sem dia é só o mês.** Ele grava `due_day_exact = false`, recebe
`due_date` no último dia do mês nomeado, e **o corte do cartão não se aplica**,
porque não existe dia de compra para comparar com o fechamento. O último dia do
mês mantém a parcela no mês certo, ordena para o fim e não lê como atrasada antes
do mês acabar.

**Confirmar acerta a data.** `confirm_installment` passa a aceitar a data real.
Sem data, um previsto inexato vira a data de hoje, e um previsto que já tinha dia
exato, como parcela de cartão, mantém o dia dele. Confirmar sempre deixa
`due_day_exact = true`, porque confirmar é o momento em que o dia deixa de ser
chute.

A coluna se chama `due_day_exact`, em `installments`, `not null default true`. É
sobre o dia, e não sobre a data, porque o mês é sempre conhecido: previsto sem mês
não existe.

### Consequência

- O caminho normal fica sem pergunta nenhuma. Ela manda "40 no cartão", e data,
  fatura e mês saem sozinhos.
- A aba de previsto continua sem data na tela, como já era, e agora isso é
  verdade no banco também, em vez de ser uma data escondida.
- **Custo:** previsto sem dia no cartão fica no mês que ela nomeou, e o dinheiro
  pode sair no mês seguinte. É um erro de até um mês, aceito de propósito, porque
  um previsto sem dia é linha de orçamento e não compra. Quando ela disser o dia,
  o corte volta a valer e a fatura sai certa.
- **Custo:** lançar hoje uma compra de ontem exige que ela escreva a data. Se
  esquecer, e a compra for do dia 1 com a mensagem no dia 2, a fatura inteira
  muda de mês. É a borda mais afiada do sistema, e ela existe por causa do
  fechamento cair num dia baixo.
- **Custo:** `due_day_exact` é quase sempre `true`, e campo que quase nunca varia
  é campo que se esquece de preencher. Isso só se segura porque nada escreve em
  `installments` fora de `create_entry_with_installments` (ADR 0002).
- **Custo:** o fuso fica escrito no schema. Mudar de país passa a exigir
  migration, e não é configuração.

---

## ADR 0016, o dia do fechamento é o único dia que ela decide

**Data:** 2026-08-20
**Status:** aceita, complementa a ADR 0014

### Contexto

A ADR 0014 fixou que compra a partir do dia do fechamento entra na fatura
seguinte, com a comparação em `>=`. Isso é verdade no dia 3 em diante e é
verdade até o dia 1. **No próprio dia do fechamento não é verdade nem falso**, é
uma questão de hora: a fatura fecha num instante do dia, e a compra da manhã pode
estar dentro dela enquanto a da noite já não está.

O erro de errar esse dia não é pequeno. Ele desloca a compra em um mês inteiro na
home, que é a tela que responde quanto sobra. E é justamente o dia em que o
sistema tem menos informação, porque a hora do fechamento não está em lugar
nenhum e varia por emissor.

O prompt do bot já diz "faltando informação, pergunta, nunca chuta". O dia do
fechamento é o único dia do mês em que a informação falta de verdade.

A alternativa considerada foi guardar a hora do fechamento no cartão e decidir
pela hora da mensagem. Ela foi recusada porque essa hora não é publicada, ela
mudaria de emissor para emissor, e o resultado seria um chute com aparência de
regra.

### Decisão

**A pergunta não é uma pergunta, é um botão na confirmação que já existe.** O bot
já responde com o que entendeu e espera confirmação antes de gravar. No dia do
fechamento, essa mesma resposta carrega o mês do pagamento e um botão
`ainda não fechou`. Nenhuma volta a mais na conversa, e nenhuma pergunta no
caminho normal.

O padrão continua sendo o da ADR 0014, fatura seguinte. O botão é a saída, não a
entrada. Tocar nele tira o deslocamento daquele lançamento e a compra fica na
fatura que está fechando.

**Depois de ela dizer uma vez que já fechou, o mês inteiro para de perguntar.** A
confirmação normal, sem tocar no botão, é essa resposta. `cards` ganha
`closing_confirmed_month`, o mês do ciclo em que ela confirmou o fechamento, e
enquanto ele for o ciclo corrente o botão não aparece mais naquele cartão. Dizer
`ainda não fechou` não grava nada, então o botão volta no próximo gasto do mesmo
dia, que é exatamente o que se quer: pode ter fechado no meio.

O botão só existe quando o dia da compra é igual ao `closing_day` do cartão. Fora
disso não há ambiguidade e não há escolha a oferecer. Isso vale também para
lançamento retroativo: uma compra do dia 2 informada no dia 5 recebe o mesmo
botão, se o mês ainda não foi confirmado.

O estado é por cartão, porque dois cartões que fecham no mesmo dia não fecham na
mesma hora.

### Consequência

- O único dia ambíguo do mês passa a ser decidido por quem tem como saber, e nos
  outros vinte e nove nada muda e nada é perguntado.
- A ADR 0014 continua valendo inteira. Esta ADR só acrescenta uma saída no dia do
  fechamento, e por isso não substitui aquela.
- **Custo:** ela pode não saber a resposta. Saber se a fatura fechou às duas da
  tarde do dia 2 pode exigir abrir o app do banco, e nesse caso o chute só mudou
  de dono, com um toque a mais. O botão ser opcional é o que segura isso: ignorar
  o botão é responder o padrão.
- **Custo:** resposta errada não se conserta. Mover um lançamento de uma fatura
  para outra não existe no app, e a correção é apagar e relançar.
- **Custo:** com dois cartões fechando no mesmo dia, o botão pode aparecer duas
  vezes no dia 2, uma por cartão.
- **Custo:** depois de confirmado o mês, um lançamento retroativo daquele mesmo
  dia 2 entra no padrão sem oferecer o botão, mesmo que aquela compra específica
  tenha sido antes do fechamento.
- **Custo:** `closing_confirmed_month` é estado de conversa morando numa tabela
  de cadastro. Ele não significa nada para o front, que nunca lê essa coluna.

---

## ADR 0017, gasto fixo confirma sozinho, e a fatura do cartão não é confirmada

**Data:** 2026-08-20
**Status:** aceita

### Contexto

Recorrente nasce `previsto` e é materializado meses à frente (ADR 0003 e ADR
0011). Confirmar um por um significa, todo mês, uma dezena de toques que não
mudam número nenhum na tela, porque o valor já estava certo antes de ela tocar.

A ADR 0003 embrulhou três coisas na mesma palavra: `previsto` é o que vai
acontecer, o que ainda não aconteceu, e o que tem valor estimado. Para o aluguel
do mês que vem as duas primeiras valem e a terceira não. O aluguel é R$ 2.000 e
vai ser R$ 2.000. O embrulho não se sustenta, e o que gera trabalho é justamente
a terceira.

Do outro lado apareceu a pergunta oposta, a do cartão, que não está em débito
automático. Ela paga a fatura no dia 7 e queria avisar o bot, ou ser lembrada. Só
que os lançamentos daquela fatura já são `confirmado` desde a compra, porque o
valor deles é real desde a compra. Marcar a fatura como paga não muda nenhum dos
três números do topo, nem o gráfico, nem a sobra. É registro que não vira tela.

### Decisão

**O critério de `confirmado` é o valor ser real, e não o dinheiro ter saído.** É
o que a ADR 0003 já dizia, e as duas perguntas se resolvem por ele em direções
opostas.

`entries` ganha `amount_exact`. Quando é `true`, o valor não é estimativa, é o
mesmo todo mês. **Recorrente com `amount_exact` confirma sozinho ao vencer**, por
`sobra_confirm_fixed()`, que roda no mesmo boot do `sobra_extend_recurring` e
nunca confirma para frente. Aluguel, DAS e assinatura param de pedir toque. Luz e
mercado continuam pedindo, porque ali o valor futuro é chute de verdade e
confirmar é o momento em que o chute vira número.

**A fatura do cartão não ganha estado de paga.** Nenhum `card_payments`, nenhum
booleano. O que ela quer não é um registro, é não esquecer de pagar no dia 7, e
isso é lembrete, que mora no Worker e não no schema.

Ela não estar em débito automático não muda nada disso. O dia em que ela paga um
gasto de valor fixo não altera a sobra do mês, altera só debaixo de qual
cabeçalho de dia a linha aparece. Quando ela avisar a data, `confirm_installment`
já aceita a data real (ADR 0015).

### Consequência

- A rotina mensal cai para as contas que variam de verdade, que são poucas.
- Não entra tabela nova, não entra terceiro estado, e a ADR 0003 continua com
  dois valores.
- **Custo:** um recorrente fixo que ela parou de pagar continua se confirmando
  sozinho para sempre. Quem corrige é `ends_on`, e lembrar de preencher `ends_on`
  é dela. É o único jeito de o app afirmar um gasto que não aconteceu.
- **Custo:** passa a existir `previsto` que não é estimativa, e a tela ainda
  hachura todo `previsto` como se fosse (ADR 0012, e CONVENTIONS diz que a
  hachura é o marcador de estimativa). Ou a hachura muda de significado, ou
  aparece um segundo tratamento para o previsto certo. Fica em aberto de
  propósito, porque é decisão de tela e não de modelo.
- **Custo:** ficar sem registro de pagamento de fatura quer dizer que o app nunca
  vai saber responder se ela pagou. Se um dia ela pagar atrasada e quiser ver
  isso na tela, esta ADR precisa de uma substituta e de uma tabela.
- **Custo:** `amount_exact` é dela para marcar, e ninguém verifica. Marcar como
  fixo uma conta que varia faz o app confirmar sozinho um valor errado.

---

## ADR 0018, previsto não confirmado é resolvido no fechamento

**Data:** 2026-08-20
**Status:** aceita

### Contexto

Um `previsto` que ninguém confirma não tem fim. Ele fica parado no mês em que
foi lançado, contando no número de previsto daquele mês para sempre, e meses
passados vão acumulando gastos que nunca se resolveram. A tela mostra um mês
fechado com um "ainda vai sair" que nunca saiu.

Nada no modelo cuida disso. A ADR 0003 fixou dois status e nenhum dos dois quer
dizer "não aconteceu".

O momento de perguntar podia ser o fim do mês ou o fechamento da fatura. **O
fechamento vem antes**, e é a diferença entre ela ainda conseguir fazer alguma
coisa e só receber a notícia. Perguntar depois do mês fechado é contar um fato,
perguntar no fechamento é dar uma escolha.

### Decisão

**No dia do fechamento, o bot manda uma mensagem só**, listando todo previsto
cujo `due_date` já passou e que continua previsto. A regra não olha ciclo nem
cartão, olha data vencida, então ela cobre também o previsto sem cartão, que não
tem fatura nenhuma.

Cada item da lista tem três saídas, e nenhuma delas inventa um status novo:

- **confirmar**, que é `confirm_installment` e já existe (ADR 0015). Pode vir com
  o valor real e com a data real.
- **não aconteceu**, que apaga. `sobra_drop_installment` remove a parcela e, se
  a entry ficar sem nenhuma, remove a entry junto. Um gasto que não aconteceu
  não vira registro, vira ausência.
- **adia**, que empurra `due_date` e `reference_month` um mês para frente, por
  `sobra_postpone_installment`. Os dois andam juntos, porque adiar quer dizer
  que o gasto vai acontecer no outro mês, e não que ele aconteceu neste e paga
  no seguinte.

Sem resposta, nada acontece e a lista volta no fechamento seguinte. O silêncio
não pode significar nem que aconteceu nem que não aconteceu.

### Consequência

- Mês passado para de acumular previsto pendurado, e o número de previsto de um
  mês fechado passa a querer dizer alguma coisa.
- As três saídas cabem nos dois status da ADR 0003, então ela continua de pé.
- **Custo:** é a primeira mensagem que o bot manda sem ela ter falado primeiro.
  Isso exige agendamento no Worker, que a ADR 0011 tinha evitado de propósito ao
  escolher estender recorrentes no boot do app em vez de por job. Esta ADR
  reintroduz a necessidade de um agendador, e essa é a peça mais cara aqui.
- **Custo:** mensagem não pedida é mensagem que cansa. Se num mês houver quinze
  previstos vencidos, a lista fica longa e a chance de ela responder tudo cai.
- **Custo:** `sobra_drop_installment` é a primeira coisa que apaga dado do
  histórico, e apagar não tem desfazer. Um toque errado na lista perde o
  lançamento.
- **Custo:** adiar mexe em `reference_month`, que até aqui era imutável depois de
  gravado. O gráfico de categorias do mês passado muda quando ela adia, o que é
  correto e ainda assim significa que um mês fechado não é mais imutável.

---

## ADR 0019, o login continua Google, e o Telegram é pareado por deep link

**Data:** 2026-08-20
**Status:** aceita, complementa a ADR 0004

### Contexto

O bot é a porta de entrada do dia a dia, então apareceu a pergunta de trocar o
login do Google pelo próprio Telegram, para a pessoa ter uma conexão só em vez de
duas.

O Telegram não é provider do Supabase. Não está na lista. Virar login significa
receber o payload assinado do Login Widget, validar o HMAC com o token do bot, e
o Worker mesmo criar ou buscar o usuário e emitir a sessão. É autenticação
escrita à mão, na parte de um sistema em que errar é mais caro que em qualquer
outra.

Do outro lado, o ganho é menor do que parece. O vínculo do Telegram não é um
segundo login, é um pareamento que acontece uma vez na vida. A sessão do Google é
longa, então na prática a pessoa loga uma vez e pareia uma vez, no mesmo
onboarding em que já vai cadastrar cartão e renda.

### Decisão

**O login continua sendo Google, e a ADR 0004 fica de pé.**

O Telegram é vinculado por deep link, que é o mecanismo documentado do Telegram
para isso. No app, logada, ela pede para conectar. O app gera um token de uso
único e abre `t.me/<bot>?start=<token>`. O Telegram entrega `/start <token>` ao
Worker, que acha o token, grava o par `chat_id` e `user_id`, e queima o token.

O token é de uso único, tem validade curta e é guardado com hash, nunca em texto.
`chat_id` desconhecido é ignorado em silêncio, depois da validação do
`secret_token` do header, que continua sendo a primeira coisa que o Worker faz.

**O pareamento é um passo do onboarding**, logo depois dos cartões, e não um item
escondido nas configurações. Assim não são duas conexões, é um fluxo só.

### Consequência

- Login continua sendo configuração no Supabase, sem uma linha de código de
  autenticação, e o RLS continua saindo de `auth.uid()` sem intermediário.
- Existe app sem Telegram. Perder o Telegram não é perder o acesso.
- **Custo:** são duas coisas para conectar, ainda que cada uma aconteça uma vez.
  Para quem só quer usar o bot, é um passo que não pediu.
- **Custo:** o token de pareamento é um segredo novo, com uma janela em que ele
  vale. Uso único e validade curta são o que segura isso, e os dois precisam
  existir de fato.
- **Custo:** ignorar `chat_id` desconhecido em silêncio é certo contra abuso e
  ruim para diagnóstico. Pareamento que falhou parece bot morto, sem mensagem
  nenhuma.
- **Custo:** escolher Google **não** tira a service role key do Worker. Ele
  resolve `chat_id` para usuário, o que cruza a fronteira do RLS, então a chave
  elevada mora lá de qualquer jeito. O que esta decisão evita é emitir sessão,
  não é guardar a chave.
- **Custo:** a bridge que não foi construída também não foi aprendida. Se um dia
  o Telegram virar login, o trabalho está inteiro esperando.

---

## ADR 0020, categoria é proposta pelo bot, não é lista fixa

**Data:** 2026-08-20
**Status:** aceita, substitui a ADR 0003 em nada, altera só o seed

### Contexto

O seed criava treze categorias fixas para todo mundo. Isso presume que o gasto de
uma pessoa se parece com o de outra, e a primeira coisa que aparece quando alguém
começa a usar é uma lista com metade das linhas vazias e a categoria que ela
realmente usa faltando.

O risco de deixar o bot criar não é criar demais. É **deriva de nome**: escrever
`mercado` num mês e `supermercado` no outro. O gráfico de categorias existe para
comparar mês com mês, e conjunto instável quebra exatamente isso, em silêncio e
sem parecer defeito.

### Decisão

**O seed encolhe para seis**, as que valem para praticamente qualquer pessoa:
moradia, mercado, transporte, saúde, restaurantes e dívidas. Dívidas entra porque
o módulo de dívida depende dela.

**O bot propõe o resto, e nunca cria em silêncio.** Não encaixando em nenhuma
existente, ele diz qual categoria pretende criar e espera confirmação, que é a
regra que o prompt dele já tem. Toda proposta leva junto a lista do que já
existe, para ele encaixar em vez de inventar sinônimo.

**Cor sai da paleta por rodízio**, sempre a primeira ainda não usada, então
categoria nova nunca nasce sem cor nem repetindo uma que está em uso.

Editar, renomear e fundir categoria ficam fora do v1.

### Consequência

- A primeira tela de quem começa mostra o gasto dela, e não um formulário de
  categorias de outra pessoa.
- O custo em API é desprezível: a lista de categorias no prompt são uns duzentos
  tokens de entrada por lançamento, e a proposta sai na mesma resposta que o bot
  já ia mandar, sem uma volta a mais.
- **Custo:** a deriva continua possível. Se ela aceitar `supermercado` sem
  reparar que `mercado` existe, o gráfico racha e nada avisa. O que segura é a
  lista ir junto no prompt, e isso depende do modelo prestar atenção nela.
- **Custo:** sem edição no v1, categoria criada errada fica errada. A saída é
  apagar o lançamento e refazer, o que é pior que renomear.
- **Custo:** a paleta tem treze cores. Passando de treze categorias, a cor
  repete. Salva o fato de a cor nunca ser o único identificador, já que ícone e
  nome estão sempre lá, mas o gráfico fica pior de ler.
- **Custo:** categoria nova não tem ícone, e a direção visual pede ícone em toda
  categoria e proíbe cifrão e cofre genéricos. Fica em aberto de propósito, e é
  decisão de tela.
