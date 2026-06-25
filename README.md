# Tá na Massa — Pizzaria Artesanal

Aplicação web de pedidos online para a pizzaria **Tá na Massa**, desenvolvida com **Vue 3** e **Vue Router**. O sistema permite ao cliente navegar pelo cardápio, montar seu pedido com personalizações e acompanhar o status dos pedidos em tempo real.

---

## Visão Geral

### Segmento de Negócio

O segmento escolhido é o de **restaurante/delivery de pizzaria artesanal**. A identidade visual foi inspirada nas cores da bandeira da Itália, sendo construída em torno de dois tons principais — verde escuro (`#2e7d32`) para transmitir artesanalidade e frescor, e vermelho escuro (`#c62828`) como cor de destaque e chamada para ação — reforçando a proposta de pizza feita com ingredientes frescos e massa de fermentação lenta.

### Estrutura de Páginas e Rotas

| Rota | View | Descrição |
|---|---|---|
| `/` | `HomeView` | Landing page com hero, diferenciais, pizzas em destaque e fluxo de uso |
| `/menu` | `MenuView` | Cardápio completo separado em abas (Salgadas / Doces) com scroll horizontal |
| `/config` | `ConfiguracaoPedidoView` | Formulário de montagem do pedido com validação |
| `/pedidos` | `PedidosView` | Painel de pedidos com tabela, troca de status e exclusão |

### Alterações Visuais e Estruturais

**NavBar** — barra verde com borda inferior vermelha e logo clicável com efeito de escala no hover:

```html
<!-- NavBarComponent.vue -->
<nav id="nav">
  <router-link to="/" id="logo-url">
    <img src="/img/Logotipo-sembg.png" id="logo" alt="Tá na Massa" />
  </router-link>
  <router-link to="/">Home</router-link>
  <router-link to="/menu">Cardápio</router-link>
  <router-link to="/pedidos">Pedidos</router-link>
</nav>
```

```css
#nav {
  background-color: #2e7d32;
  border-bottom: #c62828 4px solid;
}
#nav a.router-link-exact-active {
  color: #fff8e7;
  border-bottom: 2px solid #fff8e7;
}
```

**HomeView — Hero com gradiente e seção de destaque dinâmica.** As três primeiras pizzas salgadas são carregadas da API e exibidas como cards com foto, preço e botão de ação direta:

```javascript
// HomeView.vue — carrega os destaques ao montar o componente
async carregarDestaques() {
  const response = await fetch(`${this.$apiUrl}/menu`);
  const dados = await response.json();
  this.pizzasDestaque = (dados.salgadas || []).slice(0, 3);
},
pedirPizza(pizza) {
  const pizzaJson = encodeURIComponent(JSON.stringify(pizza));
  this.$router.push({ path: "/config", query: { pizza: pizzaJson } });
},
```

**MenuView — Cardápio com abas e scroll horizontal com snap.** A aba ativa controla qual lista é exibida via `computed`:

```javascript
// MenuView.vue
computed: {
  listaPizzasAtivas() {
    return this.categoriaAtiva === "salgadas"
      ? this.listaSalgadas
      : this.listaDoces;
  },
},
```

```css
#scroll-horizontal {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
}
#card-content { scroll-snap-align: start; }
```

**PedidoComponent — Formulário com validação inline e modal de confirmação** antes de enviar o pedido para a API:

```javascript
// PedidoComponent.vue
validar() {
  this.erros.nomeCliente = !this.nomeCliente.trim();
  this.erros.tamanho = !this.tamanhoSelecionado;
  if (this.erros.nomeCliente || this.erros.tamanho) {
    this.$refs.alerta.mostrar("erro", "Preencha os campos obrigatórios antes de confirmar o pedido.");
    return false;
  }
  return true;
},
```

Campos inválidos recebem a classe `campo-invalido` que destaca a borda em vermelho:

```css
.campo-invalido input,
.campo-invalido select {
  border-color: #c62828;
  background: #fff5f5;
}
```

---

## Solução Técnica dos Alertas

Os alertas semânticos são implementados no componente `AlertComponent.vue` como um **modal flutuante de sobreposição total** (`position: fixed; inset: 0; z-index: 9999`), desacoplado do fluxo de layout das páginas.

### Tipos suportados

| Tipo | Cor da borda | Ícone | Cor do ícone |
|---|---|---|---|
| `erro` | `#c62828` (vermelho) | ✗ | `#c62828` |
| `aviso` | `#e65100` (laranja) | ⚠ | `#e65100` |
| `info` | `#1565c0` (azul) | ℹ | `#1565c0` |
| `sucesso` | `#2e7d32` (verde) | ✓ | `#2e7d32` |

### Como funciona

O componente expõe um único método público — `mostrar(tipo, mensagem, duracao)` — que é acionado pelos componentes pai via `$refs`:

```javascript
// AlertComponent.vue
data() {
  return {
    visivel: false,
    tipo: "info",
    mensagem: "",
    titulos: { erro: "Erro", aviso: "Atenção", info: "Informação", sucesso: "Sucesso" },
    timer: null,
  };
},
methods: {
  mostrar(tipo, mensagem, duracao = 2500) {
    clearTimeout(this.timer);   // cancela timer anterior se houver
    this.tipo = tipo;
    this.mensagem = mensagem;
    this.visivel = true;
    if (duracao > 0) {
      this.timer = setTimeout(() => { this.visivel = false; }, duracao);
    }
  },
  fechar() {
    clearTimeout(this.timer);
    this.visivel = false;
  },
},
```

A exibição é controlada por `v-if="visivel"` dentro de `<transition name="modal-fade">`, que aplica uma animação de `opacity` + `scale` na entrada e saída. O tipo ativo é injetado como classe CSS no elemento raiz do modal (`:class="tipo"`), alterando dinamicamente a cor da borda superior e do ícone sem lógica adicional no template.

O ícone correto é selecionado com `v-if`/`v-else-if` aninhados:

```html
<!-- AlertComponent.vue -->
<span v-if="tipo === 'erro'">&#10007;</span>
<span v-else-if="tipo === 'aviso'">&#9888;</span>
<span v-else-if="tipo === 'info'">&#9432;</span>
<span v-else-if="tipo === 'sucesso'">&#10003;</span>
```

### Exemplos de uso nos componentes

```javascript
// PedidoComponent.vue — erro de validação
this.$refs.alerta.mostrar("erro", "Preencha os campos obrigatórios antes de confirmar o pedido.");

// PedidoComponent.vue — pedido enviado com sucesso (fecha em 2s e redireciona)
this.$refs.alerta.mostrar("sucesso", `Pedido de ${this.nomeCliente} confirmado! Redirecionando...`, 2000);

// ListaPedidoComponent.vue — status atualizado
this.$refs.alerta.mostrar("sucesso", `Status atualizado para "${descricao}".`);

// ListaPedidoComponent.vue — pedido marcado como "Entregue"
this.$refs.alerta.mostrar("info", `Pedido marcado como "${descricao}". Pedido finalizado!`);

// ListaPedidoComponent.vue — exclusão bem-sucedida
this.$refs.alerta.mostrar("sucesso", `Pedido de ${nome} removido com sucesso.`);
```

---

## Tecnologias

- **Vue 3** — Options API
- **Vue Router 4** — navegação SPA com `createWebHistory`
- **JSON Server** — API REST via `db/db.json`
- **Vue CLI 5** — build e servidor de desenvolvimento

---

## Scripts

```bash
# Instalar dependências
npm install

# Iniciar a aplicação (front-end)
npm run serve

# Iniciar a API local (json-server)
npm run bancojson

# Gerar build de produção
npm run build
```

---

## Links

| | |
|---|---|
| **API** | https://ta-na-massa-pizza-place.onrender.com/ |
| **Produção** | https://ta-na-massa-pizza-place.onrender.com/ |
| **Repositório** | https://github.com/rezelay/ta-na-massa-pizza-place |
