# Devoluções

App de controle de devoluções de marketplaces, feito em Power Apps + SharePoint. Mobile-first, com leitura de código de barras e QR code direto da câmera do celular.

Usado em produção numa empresa pequena de Piraquara-PR que vende em Site próprio, Mercado Livre, Shopee e Shein. O fluxo é simples: quando chega um produto devolvido, o operador abre o app no celular, escaneia o código de barras do produto e a etiqueta interna (TAG), seleciona a plataforma de origem e o motivo, anexa foto se for o caso e salva. Tudo cai numa lista do SharePoint que serve tanto de banco quanto de painel administrativo.

Este repositório é a comprovação de código da Atividade Extensionista II (CST GTI, UNINTER) do aluno Gabriel Ruthes de Oliveira (RU 3586559).

## Stack

- **Power Apps Canvas** (cliente mobile do Power Apps no Android/iOS)
- **SharePoint Online** como persistência
- **Power Fx** para a lógica do app
- Componente nativo **BarcodeReader** para leitura de códigos
- Autenticação por **Microsoft 365** (já existente na empresa)

Sem servidor, sem deploy, sem container. Toda a infra é a própria Power Platform.

## A lista do SharePoint

A lista se chama `Devoluções` e tem essas colunas:

- `Title` — texto. É o título do item, preenchido automaticamente com a plataforma escolhida pra dar identidade visual na galeria.
- `Plataformas` — Choice: Site, Mercado Livre, Shopee, Shein.
- `Motivodedevolu_x00e7__x00e3_o` — texto. Lista de motivos das devoluções apontadas pelas plataformas (Não usado no aplicativo final pois somente visualizando o motivo da reclamação na plataforma é possivel ).
- `C_x00f3_digodebarras` — texto. Código de barras EAN do produto. No app aparece como "Código de referência".
- `Produto_x0028_SKU_x0029_` — texto. SKU/TAG interna do produto.
- `Quantidade` — texto.
- `Status` — Choice. Estado pós-análise (ex: Apto para Venda).
- `Anexos` — anexos da lista, usado pra fotos. (Usado apenas em caso de necessidade de registro do processo: Caso seja produto de outro vendedor ou o produto tenha retornado com defeito)
- `Created` — data automática, usada na ordenação.

Os nomes internos codificados em hex (`_x00e7_`, `_x0028_` etc) são gerados pelo próprio SharePoint quando o nome de exibição tem acento ou caractere especial, e é assim que precisam ser referenciados nas fórmulas.

## Como o app está organizado

Duas telas principais:

**BrowseScreen1** — galeria da lista. Tem busca por título, botão de alternância de ordenação por data (mais recentes ↔ mais antigos) e botão de adicionar.

**EditScreen1** — formulário de edição/criação. Cada campo da lista vira um DataCard. Plataforma, Motivo e Status usam ComboBox moderno. Código de referência e Produto/SKU são campos de texto com botão de scanner ao lado. Anexos e Quantidade são campos simples.

A galeria foi criada pelo caminho padrão do SharePoint (Integrar → Power Apps → Criar um aplicativo), e o formulário foi gerado pelo "Personalizar formulários". A partir daí veio um trabalho de refatoração pra deixar o app utilizável no celular real, que é onde ele de fato roda.

## Pedaços de código relevantes

### Ordenação da galeria (Items do BrowseGallery1)

```powerfx
SortByColumns(
    Filter([@Devoluções]; StartsWith(Título; TextSearchBox1.Text));
    "Created";
    If(SortDescending1; SortOrder.Descending; SortOrder.Ascending)
)
```

A coluna originalmente era `"Title"`, então a ordenação alternava entre A–Z e Z–A. Trocando pra `"Created"`, o botão passa a alternar entre mais recente e mais antigo, que é o que o operador realmente queria.

### Botão de alternância (OnSelect do IconSortUpDown1)

```powerfx
UpdateContext({SortDescending1: !SortDescending1})
```

### ComboBox dos campos Choice

Os três campos Choice (Plataformas, Motivo, Status) usam ComboBox moderno com essa configuração:

```powerfx
// Items
Choices([@Devoluções].Plataformas)

// DefaultSelectedItems
Parent.Default

// InputTextPlaceholder
"Selecione..."

// SelectMultiple
false

// IsSearchable
false
```

O `IsSearchable: false` é o detalhe que muda tudo no mobile: sem barra de busca embutida, o teclado virtual não abre, e a lista de opções aparece comportada em vez de ocupar a tela inteira.

### Update do card (na propriedade Update do DataCard)

```powerfx
ComboBox1.Selected
```

Simples assim. O ComboBox moderno entrega um registro Choice já no formato que o SharePoint espera, sem precisar daquele `{ Value: ..., '@odata.type': "..." }` que controles clássicos pedem.

### Title automático

O campo Title da lista é preenchido automaticamente com a plataforma escolhida. No DataCardValue do card de Title (que fica oculto no form):

```powerfx
// Default
ComboBox1.Selected.Value
```

### Leitura do scanner

O fluxo do scanner é: o BarcodeReader captura, joga numa coleção, e o campo de texto lê dessa coleção.

```powerfx
// OnScan do BarcodeReader1 ("Ler código")
ClearCollect(Col_Codigos; BarcodeReader1.Barcodes)

// Default do DataCardValue10 (Código de referência)
First(Col_Codigos).Value
```

Pro outro scanner ("Ler TAG"), a mesma coisa com coleção própria:

```powerfx
// OnScan do BarcodeReader3
ClearCollect(Col_tag; BarcodeReader3.Barcodes)

// Default do DataCardValue11 (Produto SKU)
First(Col_tag).Value
```

### Largura responsiva dos campos

Depois de migrar pra layout responsivo, várias larguras fixas em pixel precisaram virar relativas. O padrão usado:

- Campos sem botão lateral: `Width: Parent.Width - 60` (ex: Quantidade, Anexos)
- Campos com botão lateral: `Width: Parent.Width - 180`, com o botão ancorado em `X: Parent.Width - 150` e `Width: 120`

Isso resolve a exibição em qualquer largura de tela sem deixar nada cortado nem nada sobrando.

## Notas sobre o ambiente

Algumas coisas que custaram tempo até entender, e que ficam aqui pra quem for replicar:

**Separador de argumentos.** O ambiente está configurado em pt-BR, então o separador de argumentos das funções Power Fx é `;` e não `,`. Se você copiar exemplo da documentação oficial (que vem com vírgula), vai dar erro de fórmula e às vezes a mensagem não deixa óbvio que o problema é só o separador.

**Controles modernos.** Vêm desligados por padrão em alguns ambientes. Pra ligar: Configurações → Atualizações → "Controles e temas modernos" → Ativado. Sem isso, o ComboBox moderno nem aparece na lista do Inserir, e você fica preso ao clássico (que tem a barra de busca embutida e estoura a tela no celular).

**Layout responsivo.** Apps gerados pela personalização da lista do SharePoint nascem em modo Fixo com proporção travada. Pra rodar bonito no celular, é Configurações → Tela → desativar "Fixar proporção" e mudar pra layout responsivo. Depois disso, qualquer controle que estiver com largura/posição fixa em pixel vai precisar de revisão pra usar `Parent.Width`.

**Tipo Choice no SharePoint.** O ComboBox moderno lida com isso nativamente (`Update: ComboBox1.Selected` é suficiente). O Dropdown clássico, não — ele trabalha com texto puro e exige um `Update` que monte o registro Choice na mão, incluindo o `@odata.type`. Isso foi tentado durante o desenvolvimento e dá muito problema; ComboBox moderno é o caminho.

## Problemas que ficaram

**Scanner às vezes trava em "Carregando..."** Se o usuário abre o scanner e cancela antes de ler qualquer código, o botão pode ficar travado mostrando "Carregando..." e parar de responder. O evento `OnCancel` do BarcodeReader existe, aceita fórmula, mas não dispara consistentemente nessa versão do controle. Tentativas via `Clear()` da coleção ou via reset do controle com variável de contexto não resolveram, todas dependem do mesmo evento que não roda. Na prática isso é raro (a maioria das vezes o usuário escaneia mesmo) e a saída quando acontece é sair do formulário e reabrir.

**Edição não pré-seleciona a plataforma no ComboBox.** Pra evitar conflito de tipo entre `Parent.Default` (registro Choice) e o placeholder "Selecione..." (texto), o `Default` ficou fixo em "Selecione..." em vez de tentar exibir o valor salvo. Quando uma devolução existente é aberta pra edição, o campo aparece como "Selecione..." e precisa ser reescolhido se for salvar. Na operação real isso pouco importa, porque o app é usado quase 100% pra criar registros novos, não pra editar.

**Sem modo offline.** O app precisa de conexão pra gravar. Quem opera o app trabalha em rede da empresa, então não chegou a ser um problema na prática, mas fica como melhoria futura.

## Reproduzindo em outro ambiente

1. Tenha uma conta Microsoft 365 com Power Apps habilitado.
2. Crie uma lista no SharePoint chamada `Devoluções` com as colunas da seção acima.
3. Na lista, vá em Integrar → Power Apps → Criar um aplicativo. Isso gera a galeria.
4. Depois Integrar → Power Apps → Personalizar formulários. Isso gera o EditForm.
5. Em Configurações → Atualizações, ative os controles modernos.
6. Em Configurações → Tela, mude pra responsivo e desative o fixar proporção.
7. Aplique nos respectivos controles as fórmulas dos pedaços acima.
8. Publique e compartilhe com os usuários.

## Estado atual

Em uso na empresa desde o mês anterior à entrega. Passou pela fase de testes com os ajustes documentados neste README até atingir o estado atual de uso diário sem reclamações funcionais.

Eventuais melhorias futuras: modo offline com sincronização, dashboard em Power BI sobre os dados acumulados, e notificação automatizada via Power Automate pros motivos críticos de devolução.
