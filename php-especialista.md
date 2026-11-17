# 🐘 Desafio Técnico: Desenvolvedor(a) PHP + Laravel Especialista

Este desafio é sobre **PHP** e sobre **Laravel**, nessa ordem.

Procuramos alguém que acompanha a linguagem de perto: lê RFC no `wiki.php.net`, sabe o que entrou na última release antes do post do blog sair, entende por que um array de 1 milhão de inteiros ocupa o que ocupa e tem opinião sobre a votação do último operador novo. E que, no dia a dia, conhece o Laravel por dentro: container, filas, Eloquent, e sabe quando o framework ajuda e quando atrapalha.

Se você é essa pessoa, vai se divertir. Se não é, tudo bem: provavelmente temos outro desafio para você no [README](README.md).

> **Regras do jogo**
> - Prazo: **72 horas** a partir do recebimento.
> - Um único repositório com duas camadas: um **núcleo em PHP puro** (Parte 1, sem nenhuma dependência de framework) e uma **aplicação Laravel** que usa esse núcleo (Parte 2).
> - Pode usar IA como ferramenta, mas **você vai defender cada linha ao vivo**, sem consulta, numa sessão de pair programming sobre o seu próprio código. Código que você não sabe explicar conta contra.
> - Escreva em `declare(strict_types=1);` em todos os arquivos que você criar.

---

## Parte 1: HeroForce Ledger (núcleo em PHP puro)

A HeroForce precisa consolidar o saldo das contas dos seus heróis a partir de um arquivo gigante de transações. O arquivo é grande, a máquina é pequena e dinheiro não admite arredondamento.

O núcleo fica em um diretório próprio (ex.: `packages/ledger` ou `src/Ledger`) e **não pode importar nada de `Illuminate\`** nem de outro framework. Composer só para ferramentas de desenvolvimento.

### Entrada

**`accounts.csv`** (até 50.000 contas):

```csv
account,currency,overdraft_limit
ACC-00001,BRL,0.00
ACC-00002,USD,1500.00
```

**`transactions.ndjson`** (um JSON por linha, **5.000.000 de linhas** no teste de avaliação):

```json
{"id":"0193a1c2-...","account":"ACC-00001","type":"credit","amount":"150.75","at":"2026-11-17T10:00:00-03:00"}
{"id":"0193a1c3-...","account":"ACC-00001","type":"debit","amount":"20.10","at":"2026-11-17T10:01:00-03:00"}
{"id":"0193a1c4-...","account":"ACC-00001","type":"transfer","amount":"10.00","to":"ACC-00007","at":"2026-11-17T10:02:00-03:00"}
{"id":"0193a1c5-...","account":"ACC-00001","type":"reversal","ref":"0193a1c3-...","at":"2026-11-17T10:03:00-03:00"}
```

### Regras de negócio

1. `credit` soma, `debit` subtrai, `transfer` move de `account` para `to`.
2. `reversal` desfaz a transação referenciada em `ref` (inclusive transferência, nas duas pontas). Uma transação só pode ser estornada uma vez. Estorno de estorno é rejeitado.
3. Um débito ou transferência que deixaria o saldo abaixo de `-overdraft_limit` é **rejeitado** (não aplicado).
4. Transferência entre contas de moedas diferentes é rejeitada.
5. **Idempotência:** o mesmo `id` pode aparecer mais de uma vez. Só a primeira ocorrência vale; as demais são ignoradas (não contam como rejeição).
6. **Garantia do arquivo:** duplicatas e `ref` de estorno sempre apontam para uma transação que está no máximo **10.000 linhas antes**. Fora dessa janela, trate como `ref` inexistente.
7. Linhas malformadas (JSON inválido, campo faltando, valor com mais de 2 casas, conta inexistente) são rejeitadas com o motivo correspondente e o processamento continua.
8. Valores monetários **nunca** passam por `float`, em nenhum ponto do fluxo (incluindo a Parte 2 e o banco).

### Saída

Um JSON em `stdout`, **determinístico** (mesma entrada, mesmos bytes de saída), com:

```json
{
  "accounts": { "ACC-00001": { "currency": "BRL", "balance": "120.65" } },
  "rejected": { "insufficient_funds": 12, "currency_mismatch": 3, "unknown_ref": 1 },
  "stats": { "lines": 5000000, "applied": 4987001, "duplicates": 900 }
}
```

`accounts` em ordem de chave, `balance` como string decimal com 2 casas. Os nomes dos motivos de rejeição são seus, mas devem vir de um `enum`.

### Restrições técnicas (aqui mora o desafio)

- **PHP 8.5**, rodando em Docker.
- O núcleo expõe um CLI próprio, e o comando abaixo precisa terminar com sucesso no arquivo de 5 milhões de linhas:

  ```bash
  php -d memory_limit=64M bin/ledger accounts.csv transactions.ndjson > report.json
  ```

  Sim, 64 MB. Explique no README **por que** a sua estrutura cabe, com números (custo por elemento, não "porque usei generator").
- **PHPStan nível `max`** (ou Psalm nível 1) no núcleo, **sem baseline e sem `@phpstan-ignore`**.
- Testes automatizados cobrindo as regras de negócio. Cada regra acima deve ter ao menos um teste com nome que a identifique.
- Use a linguagem moderna **onde fizer sentido** e justifique as escolhas. Esperamos ver, no mínimo, uso consciente de: `enum`, classes `readonly`, *property hooks* e/ou visibilidade assimétrica (8.4), e pelo menos dois recursos que entraram no **PHP 8.5**. Usar recurso novo só para marcar presença conta contra: queremos ver critério.
- Um script `bin/generate` que gera um `transactions.ndjson` de N linhas com **seed fixa** (mesma seed, mesmo arquivo, em qualquer máquina), respeitando a garantia da regra 6.

---

## Parte 2: HeroForce Ledger API (Laravel)

Agora o financeiro quer usar o ledger pelo sistema. Crie uma aplicação na **última versão estável do Laravel**, rodando em PHP 8.5 com **PostgreSQL**, que usa o núcleo da Parte 1 como dependência (via `path` repository do Composer ou equivalente). A regra de negócio continua no núcleo: o Laravel orquestra.

### Requisitos

1. **Autenticação** com Sanctum (token). Cada usuário só enxerga as próprias execuções (use **Policy**).
2. **`POST /api/ledger-runs`**: recebe `accounts.csv` e `transactions.ndjson` (upload que pode passar de 500 MB; não carregue o arquivo em memória). Responde `202` com o id da execução e despacha o processamento para uma **fila**.
3. O **job** de processamento:
   - roda o núcleo e grava o resultado no banco;
   - é **idempotente**: se falhar no meio e for reexecutado (`tries`, `backoff`), não duplica nem corrompe saldos;
   - não permite duas execuções simultâneas do mesmo arquivo (mesmo hash de conteúdo para o mesmo usuário);
   - grava os até 50.000 saldos de forma eficiente (não 50.000 `save()` individuais);
   - ao terminar, dispara um **evento** que notifica o usuário (e-mail ou database notification).
4. **`GET /api/ledger-runs/{id}`**: status (`queued`, `processing`, `done`, `failed`), estatísticas e rejeições por motivo, via **API Resource**.
5. **`GET /api/ledger-runs/{id}/balances`**: saldos com filtros por moeda e faixa de saldo (`min`, `max`), ordenação e **paginação por cursor**. Dinheiro no banco como inteiro (centavos) e, no model, como value object via **cast customizado**. Nada de `float`.
6. Um comando **Artisan** `ledger:process {accounts} {transactions}` que roda o mesmo fluxo pelo terminal, com barra de progresso.
7. `Model::shouldBeStrict()` (ou equivalente) ligado fora de produção. Nenhum N+1 nos endpoints.
8. **Testes de feature** (PHPUnit ou Pest) cobrindo: upload e despacho do job, policy (usuário A não vê execução de B), reexecução do job sem duplicar saldos, filtros e paginação. Use os *fakes* do framework onde fizer sentido.
9. **Larastan** no maior nível que você conseguir na aplicação, sem baseline. Diga no README qual nível e por quê.
10. `docker compose up` sobe aplicação, banco e worker da fila. Sail é permitido.

---

## Parte 3: Arqueologia (migração de legado)

O trecho abaixo roda em produção num cliente desde a época do PHP 7.4. Precisamos subir para o 8.5.

```php
<?php
class Hero
{
    function __construct($name, $power = null)
    {
        $this->name = $name;
        $this->power = $power;
    }
}

class HeroRegistry
{
    private $heroes = [];

    public function add(Hero $hero, Logger $logger = null)
    {
        $this->heroes[] = $hero;
        $logger && $logger->log("Hero ${hero->name} joined at " . strftime('%d/%m/%Y'));
    }

    public function findByCode($code)
    {
        foreach ($this->heroes as $hero) {
            if ($hero->power == $code) {
                return $hero;
            }
        }
        return null;
    }

    public function exportLegacy()
    {
        $out = '';
        foreach ($this->heroes as $hero) {
            $out .= utf8_encode($hero->name) . ';' . (integer) $hero->power . ";\n";
        }
        $count = `wc -l /tmp/heroes.txt`;
        return [$out, trim(strtolower($count))];
    }

    public function sortByName()
    {
        usort($this->heroes, function ($a, $b) {
            return $a->name > $b->name;
        });
    }
}

$registry = new HeroRegistry();
$registry->add(new Hero('Tempestade', 0));
$registry->add(new Hero('Wolverine', 'abc'));
var_dump($registry->findByCode('xyz'));
```

Entregue um arquivo `MIGRATION.md` no mesmo repositório com:

1. **Cada problema** encontrado, com a **versão do PHP** em que o comportamento mudou (deprecation, remoção ou mudança silenciosa de semântica) e o link da RFC ou da seção do guia de migração oficial.
2. A versão corrigida do código, em PHP 8.5 idiomático.
3. Pelo menos um problema aqui **não gera warning nenhum**: o código só passa a devolver outro resultado. Identifique, explique o que mudou e escreva um teste que prove que a sua versão preserva o comportamento que o negócio realmente espera (e diga qual você assumiu que é).

---

## Parte 4: Changelog pessoal (texto curto, sem IA)

No máximo uma página, em `CHANGELOG-PESSOAL.md`. Escreva como você fala. Vamos conversar sobre isso ao vivo.

1. Uma RFC do PHP dos últimos dois anos que você acompanhou durante a discussão (aprovada ou recusada). Qual era a sua posição, o que te convenceu ou não, e como foi a votação.
2. O que entra no PHP 8.6 que você vai usar na semana seguinte, e o que você teria votado contra.
3. O que mudou na última versão major do Laravel que afetou (ou afetaria) um projeto seu, e uma convenção do Laravel que você costuma quebrar de propósito, e por quê.

---

## ⭐ Diferenciais

- Benchmark do CLI do núcleo (tempo e `memory_get_peak_usage(true)`) com OPcache/JIT desligado e ligado, com a sua interpretação. Se o JIT não ajudou, diga por quê.
- GitHub Actions com matriz **PHP 8.5 e 8.6** (o 8.6 sai em 19/11/2026), contando no README o que quebrou ou gerou deprecation.
- Laravel Octane rodando a API, com atenção a estado vazando entre requisições.
- Testes de mutação (Infection) no núcleo.

---

## Como avaliamos

| Peso | Critério |
| --- | --- |
| Alto | Corretude das regras e determinismo da saída (rodamos com o **nosso** arquivo de avaliação) |
| Alto | Caber em 64 MB e saber explicar o porquê |
| Alto | Domínio da linguagem: escolhas modernas com critério, sem recurso "de enfeite" |
| Alto | Laravel: fila idempotente, Eloquent sem N+1, policy, cast de dinheiro, separação núcleo x framework |
| Alto | Parte 3: precisão das versões e achar a mudança silenciosa |
| Médio | Análise estática limpa e testes nas duas camadas |
| Médio | Parte 4 e a conversa ao vivo |
| Baixo | Histórico de commits, README e diferenciais |

Depois da entrega, marcamos uma conversa de ~90 minutos: você apresenta, a gente faz perguntas sobre a linguagem, o runtime e o Laravel, e fazemos uma alteração ao vivo no seu código.

## 📦 Entrega

1. Repositório público no GitHub com histórico de commits real (não um commit único).
2. `README.md` (como rodar, decisões de design, conta de memória), `MIGRATION.md` e `CHANGELOG-PESSOAL.md` na raiz.
3. Envie o link para nós.

Boa sorte. E se você souber de cabeça o que sai no dia 19/11, já começou bem. 🐘
