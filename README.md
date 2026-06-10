# Devoluções

Aplicativo para registrar e acompanhar as devoluções de marketplace de uma pequena empresa de Piraquara-PR, desenvolvido em Power Apps com SharePoint. Roda no celular e usa a câmera para ler o código de barras do produto.

## Stack

Não há servidor, deploy ou container. Tudo roda na própria Power Platform:

- **Power Apps Canvas**, usado no celular (Android/iOS)
- **SharePoint Online** como armazenamento dos dados
- **Power Fx** para a lógica do app
- O componente nativo **BarcodeReader** para a leitura dos códigos
- Autenticação pelo **Microsoft 365** que a empresa já utilizava

## A lista do SharePoint

A lista se chama `Devoluções` e tem as seguintes colunas:

- `Title`: texto. É o título do item. Preenchi automaticamente com a plataforma escolhida, para dar uma identidade visual na galeria.
- `Plataformas`: Choice (Site, Mercado Livre, Shopee, Shein).
- `Motivodedevolu_x00e7__x00e3_o`: texto. O motivo apontado pela plataforma na devolução. Acabei não usando no app final, porque esse campo só é possível preencher consultando o motivo da reclamação diretamente na plataforma.
- `C_x00f3_digodebarras`: texto. Código de barras EAN do produto. No app aparece como "Código de referência".
- `Produto_x0028_SKU_x0029_`: texto. O SKU/TAG interna do produto.
- `Quantidade`: texto.
- `Status`: Choice. O estado do produto após a análise (por exemplo, "Apto para Venda").
- `Anexos`: anexos da lista, usados para foto. A foto só entra quando é preciso registrar o processo, como produto de outro vendedor ou que retornou com defeito.
- `Created`: data automática, usada na ordenação.

Os nomes internos com código hexadecimal (`_x00e7_`, `_x0028_` e similares) são gerados pelo próprio SharePoint quando o nome de exibição da coluna tem acento ou caractere especial. É dessa forma que eles precisam ser referenciados nas fórmulas.

## Como o app está organizado

São duas telas principais.

A **BrowseScreen1** é a galeria da lista: tem busca por título, um botão que alterna a ordenação por data (mais recente / mais antigo) e o botão de adicionar.

A **EditScreen1** é o formulário de criação e edição. Cada campo da lista virou um DataCard. Plataforma, Motivo e Status usam ComboBox moderno; Código de referência e Produto/SKU são campos de texto com um botão de scanner ao lado; Anexos e Quantidade ficaram como campos simples.

A galeria foi gerada pelo caminho padrão do SharePoint (Integrar > Power Apps > Criar um aplicativo) e o formulário pelo "Personalizar formulários". Do jeito que vêm prontos, porém, não dava para usar no celular. Boa parte do trabalho foi refatorar isso para deixar o app utilizável na tela real, que é onde ele roda.

## Pedaços de código relevantes

### Ordenação da galeria (Items do BrowseGallery1)

```powerfx
SortByColumns(
    Filter([@Devoluções]; StartsWith(Título; TextSearchBox1.Text));
    "Created";
    If(SortDescending1; SortOrder.Descending; SortOrder.Ascending)
)
```

Originalmente a ordenação era por `"Title"`, então o botão alternava entre A–Z e Z–A, o que não tinha utilidade na prática. Troquei para `"Created"` e assim ele passou a alternar entre mais recente e mais antigo, que era o que o operador precisava.

### Botão de alternância (OnSelect do IconSortUpDown1)

```powerfx
UpdateContext({SortDescending1: !SortDescending1})
```

### ComboBox dos campos Choice

Os três campos Choice (Plataformas, Motivo, Status) usam ComboBox moderno, com esta configuração:

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

O `IsSearchable: false` parece um detalhe pequeno, mas faz diferença no celular. Sem a barra de busca embutida, o teclado virtual não abre sozinho e a lista de opções aparece no tamanho certo, em vez de ocupar a tela inteira. Levei um tempo até identificar que era essa propriedade.

### Update do card (na propriedade Update do DataCard)

```powerfx
ComboBox1.Selected
```

Apenas isso. O ComboBox moderno já entrega o registro Choice no formato que o SharePoint espera, sem precisar montar o `{ Value: ..., '@odata.type': "..." }` que o controle clássico exige.

### Title automático

O Title da lista é preenchido automaticamente com a plataforma escolhida. No DataCardValue do card de Title (que deixei oculto no formulário):

```powerfx
// Default
ComboBox1.Selected.Value
```

### Leitura do scanner

O fluxo do scanner é: o BarcodeReader captura, joga o resultado numa coleção, e o campo de texto lê dessa coleção.

```powerfx
// OnScan do BarcodeReader1 ("Ler código")
ClearCollect(Col_Codigos; BarcodeReader1.Barcodes)

// Default do DataCardValue10 (Código de referência)
First(Col_Codigos).Value
```

Para o outro scanner ("Ler TAG"), a mesma ideia, com uma coleção própria:

```powerfx
// OnScan do BarcodeReader3
ClearCollect(Col_tag; BarcodeReader3.Barcodes)

// Default do DataCardValue11 (Produto SKU)
First(Col_tag).Value
```

### Largura responsiva dos campos

Depois de migrar para o layout responsivo, várias larguras que estavam fixas em pixel precisaram virar relativas. O padrão que adotei foi:

- Campo sem botão ao lado: `Width: Parent.Width - 60` (Quantidade, Anexos)
- Campo com botão ao lado: `Width: Parent.Width - 180`, com o botão ancorado em `X: Parent.Width - 150` e `Width: 120`

Com isso o app se ajusta a qualquer largura de tela, sem cortar campo nem deixar espaço sobrando.

## Notas sobre o ambiente

Algumas coisas que tomaram tempo até eu entender. Deixo registrado aqui para quem for replicar.

**Separador de argumentos** - O ambiente está configurado em pt-BR, então o separador das funções Power Fx é `;`, e não `,`. Ao copiar um exemplo da documentação oficial (que vem com vírgula), o resultado é erro de fórmula, e a mensagem nem sempre deixa claro que o problema é só o separador. Se a fórmula está idêntica ao exemplo e mesmo assim não funciona, vale verificar o separador antes de qualquer outra coisa.

**Controles modernos** - Em alguns ambientes eles vêm desativados. Para ativar: Configurações > Atualizações > "Controles e temas modernos" > Ativado. Enquanto isso estiver desligado, o ComboBox moderno não aparece no menu Inserir e você fica restrito ao controle clássico, que tem a barra de busca embutida e ocupa a tela inteira no celular.

**Layout responsivo** - Um app gerado pela personalização da lista do SharePoint nasce em modo Fixo, com proporção travada. Para rodar bem no celular, vá em Configurações > Tela, desative o "Fixar proporção" e mude para responsivo. Depois disso, qualquer controle com largura ou posição fixa em pixel vai precisar de ajuste para usar `Parent.Width`.

**Tipo Choice no SharePoint** - O ComboBox moderno lida com isso nativamente (`Update: ComboBox1.Selected` já é suficiente). O Dropdown clássico não: ele trabalha com texto puro e exige um `Update` que monte o registro Choice manualmente, incluindo o `@odata.type`. Tentei por esse caminho no início e gera muito problema. O ComboBox moderno é a melhor opção.

## Problemas que ficaram

São limitações que conheço, mas que não chegaram a atrapalhar a operação real.

**O scanner às vezes trava em "Carregando..."** - Se o usuário abre o scanner e cancela antes de ler qualquer código, o botão pode ficar preso exibindo "Carregando..." e parar de responder. O evento `OnCancel` do BarcodeReader existe e aceita fórmula, mas nessa versão do controle ele não dispara de forma consistente. Tentei resolver com `Clear()` na coleção e com reset do controle por variável de contexto, mas as duas alternativas dependem justamente desse evento que não roda. Na prática isso é raro (quase sempre o usuário conclui a leitura), e quando trava, basta sair do formulário e abrir novamente.

**A edição não pré-seleciona a plataforma no ComboBox** - Para evitar conflito de tipo entre o `Parent.Default` (registro Choice) e o placeholder "Selecione..." (texto), mantive o `Default` fixo em "Selecione..." em vez de tentar exibir o valor já salvo. Por isso, ao abrir uma devolução existente para edição, o campo volta como "Selecione..." e precisa ser escolhido novamente antes de salvar. Como o app é usado quase totalmente para criar registros novos, e quase nada para editar, essa limitação tem pouco impacto.

**Não há modo offline** - O app precisa de conexão para gravar. Como quem opera trabalha dentro da rede da empresa, isso nunca foi um problema real, mas fica registrado como melhoria.

## Reproduzindo em outro ambiente

1. Tenha uma conta Microsoft 365 com Power Apps habilitado.
2. Crie uma lista no SharePoint chamada `Devoluções` com as colunas descritas acima.
3. Na lista, vá em Integrar > Power Apps > Criar um aplicativo. Isso gera a galeria.
4. Em seguida, Integrar > Power Apps > Personalizar formulários, para gerar o EditForm.
5. Em Configurações > Atualizações, ative os controles modernos.
6. Em Configurações > Tela, mude para responsivo e desative o "Fixar proporção".
7. Aplique nos controles as fórmulas dos blocos acima.
8. Publique e compartilhe com os usuários.

## Estado atual

Em uso na empresa desde 27/05/2026. O app passou pela fase de testes e ajustes documentada acima até chegar ao uso diário, sem falhas funcionais relatadas.

Melhorias previstas: modo offline com sincronização posterior, um dashboard no Power BI sobre os dados acumulados, e notificação automática via Power Automate quando entrar uma devolução com motivo crítico.
