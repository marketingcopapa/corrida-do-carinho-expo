# Corrida do Carinho — Modo Evento (Expo Pádua 2026)

## O que mudou no jogo

**Cadastro antes de jogar.** No lugar das 3 iniciais depois da partida, o jogador agora se
cadastra antes: nome e sobrenome, @ do Instagram, WhatsApp com DDD e o aceite LGPD.
Quem clica em "JOGAR DE NOVO" continua com o mesmo cadastro; voltar à tela inicial
(ou 45 segundos de inatividade) limpa o cadastro para o próximo jogador.

**Ranking central e diário.** As pontuações de todos os aparelhos (tablets e celulares)
vão para um banco na nuvem (Firebase/Firestore). O ranking é separado por dia no fuso
de Brasília: virou meia-noite, o placar do novo dia nasce zerado sozinho — ninguém
precisa apertar nada.

**Fila offline.** Se a internet cair, as pontuações ficam guardadas no aparelho e sobem
automaticamente quando a rede volta (tentativa a cada 30 segundos). Nada se perde.

**Contatos separados do ranking.** O WhatsApp NÃO fica visível publicamente. Ele vai para
uma coleção separada (`contatos`) que só a equipe acessa pelo painel do Firebase. No
ranking público aparecem apenas nome abreviado ("Maria S."), @ e pontos.

**Dificuldade recalibrada para torneio.** Queda mais rápida e com rampa maior, itens mais
frequentes no fim, papel comum cada vez mais presente, carrinho 10% menor e — o mais
importante — o multiplicador de combo agora tem teto (x4). Sem o teto, quem emendava um
combo longo disparava na pontuação e o ranking perdia a graça. Todos os valores estão
agrupados na constante `DIFICULDADE` no topo do script, com o valor original comentado
ao lado — ajuste fino é trocar um número.

**Identificação da origem.** Abra o jogo nos tablets do estande com `?estande` no fim do
endereço (ex.: `https://.../index.html?estande`). Essas partidas são marcadas como
"tablet" — é o ranking oficial da premiação, conforme o regulamento. Partidas pelo QR
code no celular são marcadas como "celular".

**Desligar tudo.** `EVENTO.ativo = false` no topo do script devolve o jogo 100% original
(iniciais + ranking local), sem tocar em mais nada.

---

## Passo 1 — Criar o projeto no Firebase (uma vez, ~15 minutos)

1. Acesse https://console.firebase.google.com com uma conta Google da equipe
   (sugestão: a conta do marketing, não uma pessoal).
2. **Adicionar projeto** → nome `copapa-corrida` → pode desativar o Google Analytics.
3. No menu lateral: **Build → Firestore Database → Criar banco de dados** →
   modo de produção → local `southamerica-east1 (São Paulo)`.
4. Na aba **Regras** do Firestore, apague tudo e cole as regras abaixo → **Publicar**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    // ranking público: qualquer um lê, só cria (nunca edita/apaga)
    match /{colecao}/{doc} {
      allow read: if colecao.matches('ranking_.*');
      allow create: if colecao.matches('ranking_.*')
        && request.resource.data.keys().hasOnly(['nome','instagram','pontos','origem','criadoEm'])
        && request.resource.data.pontos is int
        && request.resource.data.pontos > 0
        && request.resource.data.pontos <= 60000
        && request.resource.data.nome is string
        && request.resource.data.nome.size() >= 3
        && request.resource.data.nome.size() <= 60
        && request.resource.data.instagram is string
        && request.resource.data.instagram.size() <= 40
        && request.resource.data.origem in ['tablet','celular'];
      allow update, delete: if false;
    }

    // contatos (WhatsApp): NINGUÉM lê pela internet — só a equipe, pelo console
    match /contatos/{doc} {
      allow read: if false;
      allow create: if request.resource.data.keys().hasOnly(['nome','instagram','whatsapp','dia','criadoEm'])
        && request.resource.data.whatsapp is string
        && request.resource.data.whatsapp.size() >= 10
        && request.resource.data.whatsapp.size() <= 13;
      allow update, delete: if false;
    }
  }
}
```

O teto de 60.000 pontos nas regras é o anti-fraude básico: é mais do que o máximo
humanamente possível em 60 segundos com combo x4 — qualquer envio acima disso é
recusado pelo próprio banco.

5. Engrenagem ⚙ → **Configurações do projeto** → aba **Geral**:
   - copie o **ID do projeto** (ex.: `copapa-corrida`);
   - em "Seus apps", clique em **</>** (app da Web), registre com qualquer apelido e
     copie a **apiKey** que aparece no código de exemplo.

## Passo 2 — Preencher a configuração nos dois arquivos

No `index.html` e no `ranking.html`, procure o bloco `EVENTO` no início do script e
preencha os dois campos com os valores copiados:

```js
projetoFirebase: 'copapa-corrida',
apiKey: 'AIza...',
```

Publique os dois arquivos no GitHub Pages como sempre. Pronto.

> Atenção ao `sw.js` (cache do PWA): se o repositório tem service worker, ele pode
> continuar servindo a versão ANTIGA do jogo depois da atualização. Troque o nome do
> cache dentro do `sw.js` (ex.: `v2`) a cada publicação, ou apague o `sw.js` do
> repositório durante o evento — o jogo precisa de internet para o ranking de qualquer
> forma.

## Passo 3 — Testar (antes do evento, no escritório)

1. Abra o jogo, cadastre-se e jogue uma partida.
2. A tela final deve mostrar "MELHORES DE HOJE" com sua pontuação.
3. Abra o `ranking.html` em outro aparelho: sua partida deve aparecer em até 10 s.
4. No console do Firebase → Firestore, confira as coleções `ranking_AAAA-MM-DD`
   (pontuações) e `contatos` (cadastros com WhatsApp).
5. Teste a fila offline: modo avião, jogue, tela final avisa "sem internet";
   religue a rede e veja a pontuação subir sozinha em até 30 s.

## Operação durante a Expo

**Tablets do estande** — abra `index.html?estande` no Chrome → menu ⋮ → "Adicionar à
tela inicial" (instala como app em tela cheia). Para travar o tablet no jogo, use o
recurso nativo de fixar app (Configurações → Segurança → Fixação de tela) ou o
aplicativo Fully Kiosk Browser. Leve um roteador 4G próprio: a rede da festa satura à
noite, justamente no pico do estande.

**TV do estande** — abra `ranking.html?origem=tablet` (só o ranking oficial da
premiação) ou `ranking.html` (todas as partidas). O placar se atualiza sozinho a cada
10 segundos e mostra o total de partidas do dia no rodapé. Cada pessoa aparece uma
única vez (a melhor pontuação dela).

**Premiação às 22h** — o 1º lugar mostra o próprio Instagram logado no celular (o @
tem que bater com o do ranking) e o voluntário confere se segue o perfil da COPAPA/
Carinho. Não apareceu até 22h15? O WhatsApp de todos os cadastrados está na coleção
`contatos` no console do Firebase — chame o próximo da fila.

**Remover pontuação abusiva** (nome ofensivo, suspeita de fraude) — console do
Firebase → Firestore → coleção `ranking_` do dia → clicar no documento → excluir.
Some do placar da TV em até 10 segundos. Pela internet ninguém consegue apagar ou
editar nada (as regras bloqueiam).

## Custo

Zero. O plano gratuito do Firestore dá 20.000 gravações e 50.000 leituras por dia.
Estimativa de pico da Expo (500 partidas/dia + TV consultando a cada 10 s) usa menos
de 10% disso.

## Ajuste de dificuldade

Tudo em `DIFICULDADE`, no topo do script do `index.html` (valor original comentado ao
lado). Sugestão: rodem 10–15 partidas com a equipe antes do evento; o ideal para
torneio é que um bom jogador fique na faixa de 6–10 mil pontos e um iniciante em
1,5–3 mil. Se todo mundo passar de 12 mil, subam `pesoComumExtra` ou reduzam
`comboMax` para 3. As medalhas (`MEDALHAS`, logo abaixo) podem precisar de ajuste
depois disso — hoje o OURO exige 2.500 pontos.

## LGPD no jogo

A tela de cadastro tem: checkbox obrigatório "Tenho 16 anos ou mais" (menores jogam e
ganham brinde, mas não entram no ranking com coleta de contato — LGPD art. 14); e o
link "Aviso de Privacidade", que abre um modal dentro do próprio jogo (não quebra o
modo quiosque) com: finalidades específicas da ação, contato do DPO (Claudemir Morais
Rodrigues · privacidade@copapa.com.br · (22) 3854-9900) e QR code + endereço da
política completa (copapa.com.br/politica-protecao-dados).

O texto do modal está em HTML simples dentro do index.html (procure por "modal-lgpd")
— passe pelo jurídico e ajuste as palavras à vontade. Recomendação: publiquem também
um "Regulamento do Torneio" de 1 página cobrindo premiação (só tablets do estande
valem prêmio, resgate até 22h15, 16+) e retenção dos dados.
