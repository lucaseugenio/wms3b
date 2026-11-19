# WMS 3B

Sistema de gestão de armazém (WMS) completo em **um único arquivo HTML**, com backend **Supabase** (PostgreSQL + Auth + RPCs). Foi construído para operar o fluxo real de um CD integrado ao ERP **CIGAM**: produtos, endereçamento, estoque, pedidos, picking, conferência por código de barras, notas fiscais e conciliação de movimentos.

> **Versão atual: v14.8** — Transportadora da NF vinculada automaticamente pelo CSV NF × Pedido + importação de transportadoras via CSV.

---

## Como funciona

- **Zero build, zero servidor:** todo o sistema (interface + regras + integrações) vive em um único arquivo `wms_3b_v14_8_transportadora_nf.html`. Basta abrir no navegador.
- **Backend Supabase:** dados, autenticação (e-mail/senha) e regras de negócio pesadas (RPCs PostgreSQL) ficam no Supabase. O navegador acessa com a chave *publishable*, protegida por **RLS** (só usuários autenticados leem/gravam).
- **Integração CIGAM:** relatórios do ERP (movimentos, valores de pedidos, notas fiscais) são importados como CSV/RTF e conciliados com o estoque e os pedidos do WMS via RPCs.

## Estrutura do repositório

| Arquivo | Descrição |
|---|---|
| `wms_3b_v14_8_transportadora_nf.html` | **Aplicação atual (v14.8).** Inclui todas as versões anteriores. |
| `wms_3b_v14_7_importa_produtos.html` | v14.7 — importação de produtos ignora SKUs já cadastrados. |
| `wms_3b_v14_6_coleta_picking.html` | v14.6 — coleta de picking pré-faturamento. |
| `wms_3b_v14_5_base_trabalho.html` | v14.5 — base com sincronização segura de valores. |
| `WMS_3B_v14_6_migracao.sql` | Migração SQL obrigatória para a v14.6+ (tabela `picking_coletas`). |

Arquivos de versões antigas são mantidos como histórico — **use sempre o mais recente**.

## Configuração

1. **Supabase:** crie o projeto e aplique o schema (tabelas `produtos`, `produto_eans`, `enderecos`, `estoque`, `movimentacoes`, `pedidos`, `pedido_itens`, `conferencias`, `picking_coletas`, `nf_controle`, `transportadoras`, `configuracoes`, `movimentos_cigam`, `conciliacoes_cigam_wms`, view `vw_produtos_ean` e as RPCs usadas pela aplicação).
2. **Migração v14.6:** execute `WMS_3B_v14_6_migracao.sql` no SQL Editor do Supabase (cria a tabela `picking_coletas` com RLS). É idempotente — pode rodar novamente sem danos.
3. **Credenciais:** no topo do `<script>` principal do HTML, ajuste `SUPABASE_URL` e `SUPABASE_KEY` (chave *publishable* / anon — não é segredo, a proteção é feita por RLS).
4. **Usuários:** crie os usuários no Supabase Auth. O app abre na tela de login (e-mail + senha).
5. Bibliotecas externas carregadas por CDN: `@supabase/supabase-js@2`, `JsBarcode`, `pdf.js`.

## Módulos

| Módulo | O que faz |
|---|---|
| **Dashboard** | KPIs gerais da operação. |
| **Produtos** | Cadastro com SKU, TAG, descrição, unidade, peso e múltiplos EANs. Importação/exportação CSV com deduplicação por SKU. |
| **Endereços** | Cadastro de endereços de armazenagem e picking. |
| **Estoque** | Saldos por produto/endereço, consulta e ajuste. |
| **Mapa** | Visão espacial dos endereços. |
| **Movimentações** | Entradas, saídas e transferências com histórico. |
| **Transferência** | Transferência de estoque entre endereços (RPC `transferir_estoque`). |
| **Pedidos** | Importação CSV, valores (CIGAM), status, pendências de picking, finalização total manual. |
| **Picking** | Coleta item a item pré-faturamento (v14.6) + pendências por pedido/produto. |
| **Conferência** | Leitura por leitor físico ou câmera, com fila de leituras, sons e bloqueio por divergência. |
| **Etiquetas** | Geração de etiquetas com código de barras (JsBarcode). |
| **Inventário** | Contagem por endereço com matriz de divergências. |
| **Transportadoras** | Cadastro manual e **importação em massa via CSV** (v14.8). |

## Fluxos principais

### Importação de produtos (v14.7)

- CSV colunas: `ACAO, SKU, TAG, DESCRICAO, UNIDADE_MEDIDA, PESO_KG` + `EAN, EAN2, …` (quantos quiser).
- **SKU é a única chave de unicidade.** TAG, EAN e descrição podem repetir entre produtos.
- Linhas com SKU **já cadastrado** são marcadas como *ignoradas* na prévia (não geram erro nem duplicam) — só novas são gravadas. A "carga completa" reutiliza o `produto_id` das TAGs existentes, preservando vínculos com estoque e pedidos.
- EANs novos são vinculados ao produto **pelo SKU** (não pela TAG), o que elimina vínculo cruzado quando TAGs se repetem.
- O cadastro pode ser exportado em CSV e reimportado sem perda (formato idêntico).

### Pedidos

- CSV (um pedido por arquivo): `PEDIDO, CLIENTE, CIDADE, ESTADO, TAG, CAIXAS`. A TAG é casada com o cadastro para obter o `produto_id`.
- Reimportar o mesmo pedido substitui os itens e preserva os valores já digitados dos SKUs que permanecerem.
- Contagem de inventário (CSV): `TAG, ENDERECO, QUANTIDADE`.

### Coleta de picking pré-faturamento (v14.6)

- Registro item a item (`picking_coletas`), com botões +/− e digitação.
- **Sobrevive à reimportação do pedido aberto** (diferente de `conferencias`, que é limpa a cada reimportação) — base para o relatório de pendências de picking.

### Conferência

- Aceita leitor físico (com buffer/fila que garante ordem) e câmera.
- O EAN lido resolve o **produto** e casa o item do pedido por `produto_id`; TAG e EAN são portas de entrada, o vínculo real é o produto.
- Caixas já coletadas no picking não precisam ser relidas. Divergência bloqueia e exige confirmação.

### Notas fiscais

- **Relatório de itens (RTF do CIGAM):** importa os movimentos fiscais (RPC `importar_nf_itens_cigam`, idempotente — reimportar não duplica).
- **Relação NF × Pedido (CSV):** colunas `NF, Pedido, Nome Cliente, Status NF-e, Espécie` + `Transportadora` (v14.8). O WMS avalia cada NF (pronta, já conciliada, cancelada, sem pedido, conflito…) e concilia em lote com a mesma rotina do "Registrar NF".
- **Transportadora automática (v14.8):** o nome da coluna Transportadora é casado com o cadastro (ignora maiúsculas/minúsculas e acentos). Casou → NF já nasce vinculada. Não casou → destaque âmbar na prévia para escolha manual. O botão de conciliação só fica travado para NFs que ficaram sem transportadora.
- **Transportadoras via CSV (v14.8):** coluna `NOME` (aceita `TRANSPORTADORA`). Nomes já cadastrados são ignorados automaticamente; use o mesmo texto do relatório para o casamento funcionar.

### Integração CIGAM

- Movimentos, conciliação de saídas por NF, atualização de valores de pedidos e sincronização de reservas — tudo por RPC (`sincronizar_movimentos_cigam`, `conciliar_saidas_cigam_por_nf`, `atualizar_valores_pedidos_cigam_lote`, `sincronizar_reservas_liberadas`, entre outras).
- No cadastro atual, o código "Material" do CIGAM corresponde ao **SKU** do WMS.

## Regras de negócio importantes

- **SKU único; TAG/EAN repetíveis** (v14.7). Todo o sistema casa itens por `produto_id`; fluxos que entram por TAG pegam o primeiro cadastro com aquela TAG — evite TAGs duplicadas ou cadastre pelo CSV.
- **Fluxo NF × Pedido:** NFs canceladas/nulas, sem pedido, com conflito de pedido ou já vinculadas nunca são auto-conciliadas.
- **RLS em todas as tabelas:** acesso apenas para usuários autenticados.
- Unidade `MI` aceita quantidade fracionada; as demais exigem inteiro.

## Histórico de versões

| Versão | Destaques |
|---|---|
| **v14.8** | Transportadora da NF vinculada automaticamente pelo CSV NF × Pedido; cadastro de transportadoras via CSV; seleção manual só quando faltar. |
| v14.7 | Importação de produtos ignora SKUs já cadastrados; SKU como única chave de unicidade; EAN vinculado por SKU. |
| v14.6 | Coleta de picking pré-faturamento (`picking_coletas`) + migração SQL. |
| v14.5 | Base de trabalho com sincronização segura de valores. |

## Boas práticas de atualização

1. Aplique primeiro qualquer migração SQL nova no Supabase.
2. Substitua o arquivo HTML pela nova versão (o histórico de versões fica no repositório).
3. Reimporte os relatórios acumulados quando indicado — as importações de NF e produtos são idempotentes.
