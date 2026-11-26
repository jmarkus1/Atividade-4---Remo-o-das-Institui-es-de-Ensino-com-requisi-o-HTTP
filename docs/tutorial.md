# Tutorial: removendo uma Instituição de Ensino da tabela

Este tutorial explica, passo a passo, como foi implementada a remoção de
itens na tela de Instituições de Ensino (`src/views/InstituicaoEnsino.jsx`).

O foco principal é entender **como o botão de lixeira de cada linha "sabe"
qual instituição deve ser removida**, ou seja, a relação entre o item da
tabela e a chamada da função `removerHandleClick`.

---

## Visão geral

O fluxo completo da remoção é:

```
Usuário clica na lixeira da linha X
        │
        ▼
onClick da linha X chama removerHandleClick(id da IE X)
        │
        ▼
window.confirm pergunta se deseja remover
        │ (OK)
        ▼
fetch DELETE http://localhost:5000/instituicoesensino/{id}
        │ (resposta ok)
        ▼
setInstituicoesEnsino(lista sem a IE X)
        │
        ▼
React redesenha a tabela sem a linha X
```

Vamos ver cada parte.

---

## Passo 1: entender o endpoint de remoção

A API é um **json-server** (projeto `minhaappapi`), rodando na porta 5000.
No json-server, a remoção de um registro segue o padrão REST:

```
DELETE /instituicoesensino/:id
```

Ou seja, **o `id` vai no caminho da URL**. Para remover a IE de `id` 2:

```
DELETE http://localhost:5000/instituicoesensino/2
```

Chamar `DELETE /instituicoesensino` sem o `id` não funciona: a API
precisa saber *qual* registro remover.

> ⚠️ O json-server grava as alterações direto no arquivo `db.json`.
> Uma remoção é **permanente**. Faça um backup antes de testar.

Cada IE que vem da API tem um campo `id`:

```json
{
  "id": 2,
  "no_entidade": "ECI EEM AGENOR MENDES PEDROSA",
  ...
}
```

Esse `id` é a peça-chave de toda a funcionalidade.

---

## Passo 2: criar a função `removerHandleClick(id)`

A função recebe o `id` da instituição que deve ser removida:

```jsx
const removerHandleClick = (id) => {
  if (!window.confirm('Deseja realmente remover esta Instituição de Ensino?')) {
    return;
  }

  // Itens criados só no estado local (botão "+") não têm id na API
  if (id === undefined) {
    setInstituicoesEnsino(
      instituicoesEnsino.filter((ie) => ie.id !== undefined)
    );
    return;
  }

  fetch(`http://localhost:5000/instituicoesensino/${id}`, {
    method: 'DELETE',
  })
    .then((response) => {
      if (!response.ok) {
        throw new Error(`Status ${response.status}`);
      }
      // Remove a IE da lista local sem recarregar da API
      setInstituicoesEnsino((anteriores) =>
        anteriores.filter((ie) => ie.id !== id)
      );
    })
    .catch((error) => {
      console.log('Erro ao remover a IE.', error);
    });
};
```

Vamos por partes.

### 2.1 Confirmação com `window.confirm`

```jsx
if (!window.confirm('Deseja realmente remover esta Instituição de Ensino?')) {
  return;
}
```

`window.confirm` abre uma caixa com os botões **OK** e **Cancelar** e
retorna `true` (OK) ou `false` (Cancelar). Se o usuário cancelar, o
`return` encerra a função e nada é removido.

### 2.2 Itens sem `id`

```jsx
if (id === undefined) { ... }
```

O botão **"+"** (`handleClickAdicionarIE`) cria itens apenas no estado do
React, sem enviar para a API. Esses itens **não têm `id`**. Se
chamássemos a API com eles, a URL ficaria
`/instituicoesensino/undefined`. Por isso, nesse caso, apenas removemos
o item da lista local.

### 2.3 A requisição `DELETE`

```jsx
fetch(`http://localhost:5000/instituicoesensino/${id}`, {
  method: 'DELETE',
})
```

- A URL usa **template string** (crases `` ` ``) para inserir o `id`.
- O segundo parâmetro do `fetch` define o método HTTP. Sem ele, o
  `fetch` faria um `GET`.

### 2.4 Atualizar o estado só depois do sucesso

```jsx
.then((response) => {
  if (!response.ok) {
    throw new Error(`Status ${response.status}`);
  }
  setInstituicoesEnsino((anteriores) =>
    anteriores.filter((ie) => ie.id !== id)
  );
})
```

- `response.ok` é `true` quando o status está entre 200 e 299. Se a API
  responder com erro (ex.: 404, id inexistente), lançamos um erro e
  caímos no `.catch`. **A tabela só muda se a API confirmou a remoção.**
- `filter` cria um **novo array** com todos os itens cujo `id` é
  diferente do removido. No React, o estado deve ser substituído por um
  novo array, e não alterado diretamente (nada de `splice`).
- Usamos a forma `setInstituicoesEnsino((anteriores) => ...)`, que recebe
  o valor **mais atual** do estado. Como o `fetch` é assíncrono, isso
  evita problemas se o usuário remover vários itens rapidamente.

### 2.5 Tratamento de erro

```jsx
.catch((error) => {
  console.log('Erro ao remover a IE.', error);
});
```

Captura falhas de rede (API desligada) e os erros lançados no passo
anterior.

---

## Passo 3: ligar o botão de cada linha à função ⭐

Esta é a parte mais importante do tutorial.

A tabela é montada com `map`, que percorre o array de instituições e
gera uma linha (`<tr>`) para cada uma:

```jsx
{instituicoesEnsino.map((instituicaoEnsino, i) => {
  return (
    <tr key={instituicaoEnsino.id ?? `local-${i}`}>
      <td>{i + 1}</td>
      <td>{instituicaoEnsino.no_entidade}</td>
      <td>
        <OverlayTrigger placement="top" overlay={<Tooltip>Remover</Tooltip>}>
          <Button
            variant="danger"
            size="sm"
            onClick={() => removerHandleClick(instituicaoEnsino.id)}
          >
            <FaTrash />
          </Button>
        </OverlayTrigger>
      </td>
    </tr>
  );
})}
```

### 3.1 Cada linha tem sua própria `instituicaoEnsino`

A cada volta do `map`, a variável `instituicaoEnsino` representa **um
item diferente** do array. Imagine que a lista tem três IEs:

| Volta do `map` | `instituicaoEnsino.id` | Linha gerada |
|---|---|---|
| 1ª | 2 | ECI EEM AGENOR MENDES PEDROSA |
| 2ª | 3 | EMEF AMELIA ARANTES LEITE |
| 3ª | 4 | ... |

### 3.2 A arrow function "guarda" o item da linha

Observe o `onClick`:

```jsx
onClick={() => removerHandleClick(instituicaoEnsino.id)}
```

Aqui **não estamos chamando** `removerHandleClick` na hora de desenhar a
tabela. Estamos criando uma **nova função** (a arrow function
`() => ...`) para cada linha. Essa função só será executada quando o
usuário clicar.

Cada uma dessas funções "lembra" do `instituicaoEnsino` da volta do `map`
em que foi criada. Em JavaScript, isso se chama **closure**: a função
guarda acesso às variáveis do lugar onde foi criada.

Na prática, é como se o React tivesse montado:

```jsx
// linha 1
<Button onClick={() => removerHandleClick(2)}>
// linha 2
<Button onClick={() => removerHandleClick(3)}>
// linha 3
<Button onClick={() => removerHandleClick(4)}>
```

Por isso, ao clicar na lixeira da segunda linha, a função recebe
`id = 3`, e a requisição vai para `/instituicoesensino/3`. **O botão
carrega consigo o `id` do item da sua própria linha.**

### 3.3 Erros comuns (e por que não funcionam)

**❌ Chamar a função diretamente:**

```jsx
onClick={removerHandleClick(instituicaoEnsino.id)}
```

Sem o `() =>`, a função é **executada durante a renderização**, para
todas as linhas, sem ninguém clicar. O `onClick` recebe o *resultado* da
função (`undefined`), não a função em si. Resultado: confirmações
aparecendo ao carregar a página e nenhuma reação ao clique.

**❌ Passar a função sem argumento:**

```jsx
onClick={removerHandleClick}
```

Aqui a função só roda no clique, mas o React a chama passando o **objeto
do evento** de clique como primeiro parâmetro. Então `id` seria um
evento, e não o identificador da IE. A URL ficaria algo como
`/instituicoesensino/[object Object]`.

**❌ `id` fixo no código:**

```jsx
fetch('http://localhost:5000/instituicoesensino/1', { method: 'DELETE' });
```

Essa era a versão inicial de teste: qualquer clique removia sempre a IE
de `id` 1, independente da linha.

**✅ Forma correta:**

```jsx
onClick={() => removerHandleClick(instituicaoEnsino.id)}
```

A arrow function adia a execução até o clique **e** passa o `id` do item
daquela linha.

### 3.4 A `key` da linha

```jsx
<tr key={instituicaoEnsino.id ?? `local-${i}`}>
```

O React usa a `key` para identificar cada linha entre uma renderização e
outra. Usar o índice `i` como `key` é arriscado quando itens são
removidos do meio da lista: os índices "andam" e o React pode reaproveitar
a linha errada. Usando o `id`, cada linha mantém uma identidade estável.

O operador `??` usa `local-${i}` apenas quando o `id` não existe (itens
criados pelo botão "+").

Note que a coluna **#** continua mostrando `i + 1`: ela é só a posição
na tabela, não o `id`. Por isso a numeração se reorganiza após remover.

---

## Passo 4: testar

1. Suba a API (na pasta `minhaappapi`):
   ```bash
   npm start
   ```
2. Suba o front-end (nesta pasta):
   ```bash
   npm start
   ```
3. Acesse a tela de Instituições de Ensino.
4. Clique na lixeira de uma linha:
   - **Cancelar** → nada acontece.
   - **OK** → a linha some da tabela.
5. Confira na API que o registro foi removido. Por exemplo, para o id 3:
   ```bash
   curl http://localhost:5000/instituicoesensino/3
   ```
   A resposta deve ser `404 Not Found`.
6. Com a API desligada, tente remover: a linha permanece e o console
   mostra `Erro ao remover a IE.`

---

## Resumo

| Conceito | Onde aparece | Para quê |
|---|---|---|
| `DELETE /instituicoesensino/:id` | `fetch` | Remover o registro na API |
| `window.confirm` | início da função | Evitar remoções acidentais |
| `response.ok` | `.then` | Só alterar a tela se a API confirmou |
| `filter` + `set...(anteriores => ...)` | `.then` | Gerar uma nova lista sem o item |
| `() => removerHandleClick(instituicaoEnsino.id)` | `onClick` do botão | **Ligar cada botão ao item da sua linha** |
| `key={instituicaoEnsino.id}` | `<tr>` | Identidade estável das linhas |
