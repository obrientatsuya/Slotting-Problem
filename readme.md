# HL Service — Sistema de Gestão de Posições de Estoque

> **Aplicação web de arquivo único para localização em tempo real de SKUs em armazém, construída para ser o mais simples possível de instalar e usar, sem framework, sem build, só abrir e rodar.**

[🇺🇸 English version](readme-en.md)

---

## Contexto

A HL Service é uma assistência técnica de eletrodomésticos. O armazém armazena milhares de peças de reposição — resistências, compressores, placas de controle, termostatos, motores — vindas de dezenas de marcas e linhas de produtos.

A lógica de guarda original era **baseada em categoria**: cada caixa física era reservada para um tipo de peça e marca específicos. Por exemplo, uma caixa guardava apenas *resistências de cafeteira da marca X*, outra guardava *resistências de cafeteira da marca Y*, e assim por diante. Parecia organizado, mas na prática criava problemas sérios:

- **Caixas presas a categorias estreitas ficavam quase vazias** enquanto outras transbordavam, desperdiçando espaço físico
- **Novos tipos de peça não tinham lugar definido** — os operadores improvisavam, quebrando silenciosamente a lógica de categorias
- **A proliferação de marcas gerou uma explosão de micro-categorias** — dezenas de caixas quase vazias que poderiam compartilhar espaço
- **Ninguém sabia qual caixa estava cheia ou disponível** — era preciso ir até a prateleira e abrir para descobrir
- **A localização dependia de memória** — se quem guardou a peça não estava presente, encontrá-la significava varrer o andar inteiro
- **Sem rastreabilidade** — não havia registro de quem moveu o quê, ou quando, tornando a reconciliação de inventário impossível

---

## A Solução

Uma aplicação web contida em um **único arquivo HTML**, conectada a um banco Supabase (PostgreSQL) e integrada opcionalmente ao Tiny ERP para sincronização automática de localização.

A mudança central: em vez de atribuir caixas a categorias, **cada SKU recebe uma coordenada**. Qualquer peça pode ir em qualquer caixa. O sistema rastreia onde ela está. A lógica de categoria sai do layout físico e entra no banco de dados.

Construída para ser o mais simples possível de adotar em um ambiente de armazém real — sem instalação, sem treinamento em uma interface complexa, sem dependência de dispositivo específico. Se tem browser e consegue escanear um código de barras, funciona.

O sistema introduz:

- **Slotting por SKU** — cada item recebe uma coordenada precisa: `Andar → Setor → Prateleira → Caixa → Lado`
- **Controle de ocupação** — cada caixa/lado possui um status de ocupação em quatro níveis (Vazia / Pouca / Média / Quase cheia), tornando o aproveitamento do espaço visível de forma imediata
- **Mapa visual do armazém** — um painel de visão geral em tempo real renderiza todos os setores, prateleiras e caixas com cores de ocupação, permitindo identificar espaço disponível sem percorrer o galpão
- **Suporte a leitor de código de barras** — o campo de busca dispara no Enter, compatível com qualquer leitor USB ou Bluetooth
- **Integração ERP** — ao salvar uma posição, o sistema envia automaticamente a localização ao Tiny ERP via servidor bridge local

---

## Funcionalidades

### Busca e Cadastro de SKU
- Busque qualquer SKU digitando ou escaneando um código de barras
- Se o SKU existir, a posição atual é exibida e o operador pode atualizá-la imediatamente
- Se não encontrado, abre um formulário de cadastro pré-preenchido com a última localização usada (reduz digitação repetitiva)
- Detecção de duplicatas avisa se um SKU já está cadastrado ou se a caixa excede a capacidade configurada

### Painel de Visão Geral
- Renderiza o armazém completo em hierarquia: setor → prateleira → grade de caixas
- Cada célula de caixa exibe seu nome (C1, C2…) colorido pelo nível de ocupação dominante:
  - 🟢 **Pouca** — verde
  - 🟡 **Média** — âmbar
  - 🔴 **Quase cheia** — vermelho
  - ⬛ **Vazia** — cinza escuro
- Clicar em uma caixa expande um drawer com todos os SKUs dentro, organizados por lado (A, B, C…)
- O badge **✏ Editar ocupação** dentro do drawer permite atualizar o nível de ocupação por lado sem sair do painel
- **Filtros de legenda** — clicar em um nível de ocupação na legenda esmaece e desfoca todas as caixas que não correspondem
- **Filtro Match SKU** — com uma busca de SKU ativa, clicar em "Match SKU" na legenda destaca apenas as caixas que contêm aquele SKU; o restante fica escurecido
- **Zoom por scroll** — posicionando o mouse sobre a grade de caixas e rolando, é possível ampliar ou reduzir a visualização

### Busca por Posição
- O operador pode consultar todos os SKUs guardados em uma coordenada específica sem precisar saber o código do SKU
- Útil para auditoria e reconciliação de inventário físico

### Histórico de Movimentações
- Toda operação de salvar é registrada: SKU, nome do operador, posição anterior, nova posição, timestamp e tipo de ação
- Acessível pelo botão "Histórico" no cabeçalho

### Gerenciamento de Estrutura
- O layout do armazém (setores, prateleiras, quantidade de caixas, capacidade por caixa) é totalmente configurável pela interface
- Sem limites fixos no código — adapta-se a qualquer planta de armazém

### Sincronização com Tiny ERP
- Um servidor local faz a ponte entre a aplicação e a API do Tiny ERP
- Quando online, salvar uma posição atualiza automaticamente o campo de localização no Tiny
- Quando offline, as posições são salvas localmente e podem ser sincronizadas em lote depois

### Resiliência Offline
- Em caso de perda de conexão, o sistema carrega o último backup local com aviso
- Os dados recentes são mantidos em cache no localStorage como fallback

---

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Frontend | HTML/CSS/JS puro — zero dependências, zero etapa de build |
| Banco de dados | Supabase (PostgreSQL) com Row-Level Security |
| Bridge ERP | Servidor Node.js local (API REST do Tiny ERP) |
| Hospedagem | Qualquer host de arquivos estáticos, ou abrindo localmente no browser |

---

## Schema do Banco de Dados

A tabela principal (`estoque`) armazena uma linha por SKU:

```sql
CREATE TABLE estoque (
  id             BIGSERIAL PRIMARY KEY,
  sku            TEXT NOT NULL UNIQUE,
  andar          TEXT,           -- andar (ex: "2A", "3A")
  setor          TEXT,           -- setor (ex: "6S")
  prateleira     TEXT,           -- prateleira (ex: "P1")
  caixa          TEXT,           -- caixa (ex: "C3")
  lado           TEXT,           -- lado (ex: "A", "B")
  nivel          TEXT DEFAULT 'pouca'
                 CHECK (nivel IN ('vazio','pouca','media','quase')),
  operador       TEXT,
  atualizado_em  TIMESTAMPTZ DEFAULT now(),
  sincronizado_erp_em TIMESTAMPTZ,
  erro_erp       TEXT
);
```

Para adicionar o controle de ocupação em uma instalação existente:

```sql
ALTER TABLE estoque
  ADD COLUMN IF NOT EXISTS nivel TEXT NOT NULL DEFAULT 'pouca'
  CHECK (nivel IN ('vazio', 'pouca', 'media', 'quase'));

CREATE INDEX IF NOT EXISTS idx_estoque_nivel ON estoque (nivel);
```

---

## Como Executar

1. Crie um projeto no Supabase e execute o schema acima
2. Abra `HL_COORDENADAS.html` em qualquer browser moderno (Chrome recomendado)
3. Preencha a URL e a chave de API do Supabase na seção `CONFIG` no topo do bloco `<script>`
4. Opcionalmente, execute `INICIAR_SERVIDOR_ERP.bat` no computador do armazém para sincronização com o Tiny ERP
5. Use "Gerenciar Estrutura" para definir andares, setores, prateleiras e quantidades de caixas

Sem servidor, sem npm install, sem pipeline de build — basta abrir o arquivo.

---

## Screenshots

> *(Insira os screenshots aqui)*

| Visão Geral | Drawer de Caixa | Busca de SKU |
|---|---|---|
| ![overview](/overview.PNG) | ![drawer](/drawer.PNG) | ![search](/search.PNG) |

---

## Impacto

| Antes | Depois |
|---|---|
| Caixas restritas a categorias estreitas (ex: "só resistências de cafeteira da marca X") | Qualquer SKU vai em qualquer caixa — o espaço é preenchido por necessidade, não por categoria |
| Proliferação de marcas gerou dezenas de caixas quase vazias dedicadas | Caixas compartilhadas livremente; ocupação visível em tempo real |
| Operadores dependiam da memória ou percorriam o andar para localizar peças | Qualquer operador localiza qualquer SKU em segundos via busca ou leitura de código |
| Sem feedback sobre a capacidade das caixas | Ocupação em cores visível em todo o mapa do armazém |
| Erro de guarda só descoberto na separação | Alertas de duplicidade e capacidade excedida no momento do cadastro |
| Sem registro de quem moveu o quê ou quando | Histórico completo com operador e timestamp |
| Campo de localização no ERP atualizado manualmente | Sincronização automática com o ERP a cada salvo |

---

## Autor

Desenvolvido para operações logísticas internas da **HL Service**.
