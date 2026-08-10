# 🎨 Fase 7: Personalizando o Design (Tailwind CSS v4 e daisyUI)

Até aqui, construímos uma aplicação funcional e interativa. Agora vamos entender **como o visual é definido** e **como personalizá-lo**.

O Phoenix 1.8 vem configurado com o **Tailwind CSS v4**, um framework CSS baseado em **classes utilitárias**, e com o **daisyUI**, um plugin que fornece componentes prontos (botões, alertas, inputs) e **temas** pré-configurados.

!!! warning
    **Atenção à versão!** Se você pesquisar por "configurar tema daisyUI Phoenix", encontrará muitos guias mandando editar o arquivo `assets/tailwind.config.js`. **Esse arquivo não existe mais**: o Tailwind **v4** (usado pelo Phoenix 1.8) abandonou a configuração em JavaScript — agora tudo é configurado **dentro do próprio CSS**, no arquivo `assets/css/app.css`. Guias baseados no Phoenix ≤ 1.7 não se aplicam aqui.


### 🔧 Passo 7.1: Como o Tailwind funciona

O Tailwind é diferente de frameworks tradicionais como o Bootstrap. Em vez de classes de componentes prontos, ele oferece pequenas classes **de propósito único**:

| Propósito           | Classe Tailwind | Efeito             |
| ------------------- | --------------- | ------------------ |
| Cor de fundo        | `bg-blue-500`   | Fundo azul médio   |
| Cor do texto        | `text-gray-800` | Texto cinza escuro |
| Tamanho do texto    | `text-3xl`      | Texto grande       |
| Espaçamento interno | `p-4`           | Padding de 1rem    |
| Bordas arredondadas | `rounded-lg`    | Cantos suavizados  |
| Sombra              | `shadow-md`     | Sombra média       |

Essas classes se combinam livremente — é o padrão usado em todo o nosso `TodoLive`:

```elixir
<div class="w-full max-w-lg mx-auto mt-12 p-6 bg-white rounded-lg shadow-md">
  <h1 class="text-3xl font-bold mb-6 text-center text-gray-800">
    Minha Lista de Tarefas (com DB!)
  </h1>
```

- `w-full max-w-lg mx-auto` centraliza o bloco com largura máxima;
- `bg-white rounded-lg shadow-md` cria um "cartão" branco com sombra;
- `text-3xl font-bold text-gray-800` formata o título.

### 🌈 Passo 7.2: daisyUI — o "tema visual" do Tailwind

O **daisyUI** adiciona classes mais semânticas para componentes comuns. Observe o componente `<.button>` no arquivo `lib/elixir_todo_list_web/components/core_components.ex`:

```elixir
variants = %{"primary" => "btn-primary", nil => "btn-primary btn-soft"}
```

As classes `btn`, `btn-primary` e `btn-soft` vêm do **daisyUI** — é por isso que escrevemos `<.button variant="primary">` na Fase 4. Outros exemplos no mesmo arquivo: `alert-info`/`alert-error` (mensagens flash), `checkbox` e `input` (formulários).

A grande vantagem: esses componentes **obedecem ao tema ativo** (claro, escuro, etc.) sem que você redefina cores manualmente.

### 🌗 Passo 7.3: Os Temas (e um mistério resolvido)

Faça um teste: se o seu sistema operacional estiver no **modo escuro**, abra a aplicação. Notou que partes da página (o fundo, o cabeçalho do `Layouts.app`, os inputs) ficam **escuras**, enquanto o nosso "cartão" continua branco (`bg-white`)? O contraste fica estranho.

**O que está acontecendo?** O Phoenix 1.8 já vem com **dois temas daisyUI** (um claro e um escuro) definidos no `assets/css/app.css`:

```css
@plugin "../vendor/daisyui" {
  themes: false;
}

@plugin "../vendor/daisyui-theme" {
  name: "dark";
  default: false;
  prefersdark: true;   /* 👈 segue a preferência do sistema! */
  ...
}

@plugin "../vendor/daisyui-theme" {
  name: "light";
  default: true;
  ...
}
```

E o `root.html.heex` inclui um pequeno script que aplica o tema conforme o atributo `data-theme` (ou a preferência do sistema, por causa do `prefersdark: true`).

**Como forçar o tema claro** (a opção mais simples para o nosso visual): abra

```
lib/elixir_todo_list_web/components/layouts/root.html.heex
```

e adicione `data-theme="light"` ao `<body>`:

_Mude:_

```html
<body>
  {@inner_content}
</body>
```

_Para:_

```html
<body data-theme="light">
  {@inner_content}
</body>
```

Pronto: a aplicação fica no tema claro para todos, independentemente da preferência do sistema.

!!! tip
    **Alternativa avançada:** em vez de forçar o claro, você pode abraçar o modo escuro trocando as classes "fixas" do nosso card (`bg-white`, `text-gray-800`) por classes **semânticas** do daisyUI, que mudam com o tema: `bg-base-100`, `text-base-content`, etc. Fica como exercício!


### 💡 Passo 7.4: Exemplos práticos de personalização

**Quer destacar mais as tarefas concluídas?** Ajuste a classe condicional no `TodoLive`:

```elixir
<label class={
  if task.completed,
    do: "line-through text-gray-400 italic",
    else: "text-gray-900 font-medium"
}>
  {task.title}
</label>
```

**Quer um botão de exclusão mais discreto?** É só trocar as classes:

```elixir
<.button
  type="button"
  phx-click="delete"
  phx-value-id={task.id}
  class="!p-1 !bg-red-500 hover:!bg-red-700"
>
  &times;
</.button>
```

(O `!` prefixando a classe força a prioridade sobre o estilo padrão do componente `<.button>`.)

Sem tocar em arquivos CSS, você molda toda a interface.

### 💾 Passo 7.5: Commit

```bash
git add .
git commit -m "Fase 7: Ajusta o tema e personaliza o visual (Tailwind/daisyUI)"
```

---

**Fim da Fase 7!** 🏁

O Tailwind (com daisyUI) é o que dá ao Phoenix sua **agilidade visual**: elimina o CSS manual, mantém o template declarativo e ainda permite personalização total. A partir daqui, você pode experimentar temas, criar componentes próprios e dar identidade à sua aplicação.

Falta um último passo para o projeto ficar apresentável ao mundo: a **📄 Fase 8: README e Entrega**.

---

