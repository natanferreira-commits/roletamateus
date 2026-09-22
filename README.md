# Roleta do Caumo — roleta premiada, VIP + banca pra quem girar

Página mobile-first com dois modos, escolhidos por `modo` no [`app/config.js`](app/config.js):

- **`roleta`** (ativo): roleta premiada na primeira tela → gira → para no prêmio → bilhete do prêmio (acesso ao VIP + banca) → botão fixo que abre o WhatsApp com o número do bilhete pra resgatar.
- **`bolao`**: landing (hero, oferta, como funciona, aviso) → palpites, dois por jogo da rodada → bilhete com número → registro no WhatsApp. Última rodada: Brasileirão 2026, 28ª rodada, sábado 19/09 (10 palpites). A versão da Libertadores está na tag `liberta-quartas-2026`.

## Modo roleta

Tudo fica na seção `roleta` no fim do `config.js`. Ela sobrescreve `seo`, `marquee`, `oferta`, `rodada`, `bilhete` e `aviso` do bolão, e adiciona `roleta`:

| Campo | O que é |
| --- | --- |
| `roleta.gomos` | Gomos da roda, em ordem. `{ texto, premio: true }` marca o único gomo onde ela sempre para; os outros são decorativos. |
| `roleta.voltas` | Voltas completas antes de parar. Mais voltas = giro mais demorado e mais suspense. |
| `roleta.duracaoMs` | Duração total do giro. O ponteiro "tec-teca" no mesmo ritmo da desaceleração real, calculado a partir dessa curva — não é só decoração solta. |
| `roleta.titulo`, `subtitulo`, `ctaLabel`, `ctaHint`, `comoFunciona` | Copy da tela da roleta. |
| `bilhete.premio` | Linhas do bilhete do prêmio (`item` / `valor`). `premioNome` é o que vai gravado no Supabase e aparece no chip do bilhete. |
| `rodada.id` | Vai em todo evento (GA4 e Supabase). Troque a cada ação nova pra separar no `/admin`. |
| `rodada.encerramento`, `fechaLabel`, `encerradoLabel` | Contador no rodapé fixo. |

A roleta sempre para no gomo marcado como `premio: true`: não existe sorteio de verdade, todo mundo que gira ganha. Os outros gomos só dão a sensação real de roleta — ela nunca para neles.

Eventos: `cta_start` (girou), `roleta_premio` (parou no prêmio), `bilhete_view`, `whatsapp_click`. O bilhete é salvo na mesma tabela `bilhetes`, com o prêmio no campo `palpites`, então o `/admin` funciona sem mudar o banco.

O visual evita clichê de mesa de cassino (sem vermelho/preto, sem ficha, sem carta): paleta dourada/escura igual ao resto do site, com glow e anéis suaves atrás da roda em vez de luzes piscando. A tela de prêmio também é mais sóbria que uma tela de "vitória" — um chip único com o nome do prêmio, confete discreto, sem flash.

## Modo bolão

Não tem resposta certa na hora. O afiliado recebe o bilhete no WhatsApp, confere depois do jogo e paga o prêmio conforme a faixa de acertos.

Mesmo padrão da `lp-aposta-garantida`: Next 15 estático (`output: "export"`), sem backend, deploy na Vercel.

## Onde mexer

Tudo que muda entre jogos/afiliados está em [`app/config.js`](app/config.js):

| Campo | O que é |
| --- | --- |
| `whatsappNumero` | Número que recebe o bilhete, só dígitos com DDI+DDD (`5511999999999`). **Está com placeholder.** |
| `whatsappMensagem` | Texto pré-preenchido. Aceita `{rodada}`, `{codigo}`, `{palpites}` (lista numerada com jogo, mercado e escolha). |
| `supabase` | `url` e `anonKey` do projeto. Liga a gravação de bilhetes/eventos e o `/admin`. Vazio = desligado. |
| `seo` | Título e descrição da página. |
| `oferta` | `valor` e `regra` do prêmio (bloco dourado da landing e rodapé do bilhete). |
| `rodada` | `id` (vai em todo evento do GA4), `nome`, `encerramento` (ISO com fuso, liga o contador) e `jogos` (casa, fora, quando, escudos). |
| `marquee` | Itens da faixa que roda no topo. |
| `gaId` | Google Analytics 4 (`G-XXXXXXXXXX`). Vazio = não carrega. |
| `pixelId` | Meta Pixel. Vazio = não carrega. |
| `landing` | Copy da primeira tela. `*palavra*` vira destaque. |
| `palpites` | Lista de perguntas. `jogo` é o índice em `rodada.jogos` (ou `null` pra palpite da rodada inteira), `mercado` é o rótulo curto. |
| `loading` | Eyebrow, etapas de texto e `segundos` do "fechando seu bilhete". `0` desliga. |
| `bilhete` | Copy da tela final. |
| `aviso` | Bloco "Aviso importante" (regras, desempate, +18). |

## Trocar de rodada

1. `rodada.nome`, `rodada.encerramento` e a lista `rodada.jogos`.
2. Reescrever os `palpites` apontando o `jogo` certo (mercados sugeridos: resultado, escanteios, ambas marcam, total de gols, cartão vermelho, gol no 1º tempo). Baixar os escudos novos pra `public/escudos/`.
3. Conferir `rodada.id`, `seo`, `marquee`, `landing`, `oferta`, `bilhete.slipTitulo`, as linhas do `aviso` e o `disparo.md`.

## Código do bilhete

Gerado no navegador, 6 caracteres sem 0/O/1/I. Vai na mensagem do WhatsApp (`#CODIGO`) e, com o Supabase ligado, é salvo na tabela `bilhetes` junto com os palpites. Quem atende busca o código no `/admin`. Não tem trava de duplicidade por pessoa.

## Admin e banco (Supabase)

1. Criar um projeto no Supabase e rodar o `supabase.sql` inteiro no SQL Editor, trocando a senha na primeira seção.
2. Em Project Settings → API, copiar a Project URL e a chave `anon public` pra `supabase` no `config.js`.
3. O painel fica em `/admin` (senha = a do SQL): cliques no WhatsApp, visitantes, funil com perda por etapa, números por dia, lista de bilhetes com busca por código e gabarito pra ranquear acertos.

A chave anon só consegue inserir eventos/bilhetes e chamar as funções do admin, que exigem a senha. Ninguém lê as tabelas pela API sem ela. Pra trocar a senha: `update admin_config set senha = 'nova' where id = 1;`.

Ordem segura de mudança de schema: rodar o SQL primeiro, publicar o código depois.

## Métricas (Google Analytics 4)

Preencha `gaId` no `config.js` com o ID de medição (`G-XXXXXXXXXX`). Sem isso nada é enviado.

Eventos que a página dispara (além do `page_view` automático). Todos levam o parâmetro `rodada` (`rodada.id`), pra separar uma rodada da outra no mesmo painel:

| Evento | Quando | Parâmetros |
| --- | --- | --- |
| `cta_start` | clicou em "Fazer meus palpites" | — |
| `palpite` | respondeu um palpite | `etapa` (1..N), `jogo`, `mercado`, `escolha` |
| `bilhete_view` | chegou no bilhete | `codigo` |
| `whatsapp_click` | clicou em "Registrar no WhatsApp" | `codigo` |
| `refazer` | clicou em "Refazer palpites" | `codigo` |

Onde ver no GA4:

- **Tempo real** (Relatórios → Tempo real): quem está na página agora e os eventos dos últimos 30 min.
- **Cliques no WhatsApp**: Relatórios → Engajamento → Eventos → `whatsapp_click`. Marque como evento principal em Administrador → Eventos → "Marcar como evento principal".
- **Desistência por etapa**: Explorar → Exploração de funil, com as etapas `page_view` → `cta_start` → `palpite` (etapa = 1) → … → `palpite` (etapa = 8) → `bilhete_view` → `whatsapp_click`. O GA4 mostra a taxa de abandono entre cada uma.
- Pra filtrar por etapa e por escolha nos relatórios padrão, registre `etapa`, `mercado` e `escolha` como dimensões personalizadas (Administrador → Definições personalizadas). O funil no Explorar funciona sem isso.

## Eventos do Pixel (quando `pixelId` estiver preenchido)

- `PageView` — carregou a página
- `ViewContent` — chegou no bilhete
- `Lead` — clicou em registrar no WhatsApp

Otimizar campanha por `Lead`.

## Design

Referência é a LP do Giro Premiado (produção): fundo quase preto, fonte Archivo via `next/font`, marquee no topo, seções com conteúdo real, bloco de aviso e CTA fixo no rodapé. Sem emoji, sem glow, sem animação decorativa.

## Rodar local

```bash
npm install
npm run dev
```

## Deploy (Vercel)

1. Repo: `natanferreira-commits/quizresgate`.
2. Importa na Vercel. Detecta Next e o `output: "export"` sozinho.
3. Sem env vars.

## Cuidados

- Não instalar `sharp` como devDependency (quebrou o deploy da LP Aposta Garantida).
- `robots: noindex` está ligado no `layout.js`. Tirar se quiser indexar.
