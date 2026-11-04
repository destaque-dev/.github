# AGENTS.md

Regras para agentes de código (Claude Code, Codex, Copilot etc.) que trabalham nos repositórios da destaque-dev. Elas valem para qualquer repositório da organização; um `AGENTS.md` dentro do próprio repositório pode acrescentar regras específicas dele.

## Commits

### Quem commita é o desenvolvedor

O agente **não roda `git commit` nem `git push`**, a não ser que o desenvolvedor peça explicitamente para aquele caso. O agente prepara o commit; o desenvolvedor revisa e commita na branch.

1. O agente coloca no stage (`git add` / `git update-index`) **um commit por vez**.
2. No mesmo passo, escreve a mensagem sugerida em `.git/MERGE_MSG`. Sem `MERGE_HEAD` o git não trata isso como merge: a mensagem só aparece pré-preenchida no editor de commit (painel de git do Zed, VS Code, `git commit` sem `-m`) e o próprio `git commit` apaga o arquivo depois. Isso vale **sempre**, inclusive para uma correção avulsa de uma linha.
3. O agente mostra o que está no stage (`git diff --cached --stat`) e a mensagem sugerida, e espera.
4. Antes de preparar o próximo commit, confere no `git log` o que o desenvolvedor commitou: ele pode ter ajustado a mensagem ou commitado partes por conta própria. Um commit planejado cujo conteúdo já entrou é pulado.

### Commits atômicos, cada um compilando

- Separe as mudanças em commits atômicos, agrupados por efeito, e não num commit único. Quando uma correção cai em cima de trabalho que tem outro objetivo, ela vira um commit à parte (ex.: primeiro a correção da guarda, depois a feature que se apoia nela).
- **Antes de mexer no stage, proponha a divisão** e espere o OK: lista de commits, arquivos de cada um, dependências entre eles e estados intermediários.
- Prefira divisões em que cada arquivo pertence a um commit só, para o stage ser feito por arquivo inteiro.
- Ordene os commits por dependência. **Cada commit precisa compilar sozinho.** Verifique isso numa worktree temporária (`git worktree add --detach <pasta-temporária> HEAD`), copiando só os arquivos daquele commit, e só coloque no stage depois de compilar.
- Para colocar no stage uma versão intermediária de um arquivo que já tem edições posteriores, monte a versão na worktree, grave o blob com `git hash-object -w --no-filters` e registre com `git update-index --cacheinfo`. Confira antes `git ls-files --eol`: se o índice for `i/crlf`, o blob também precisa ser CRLF, senão o diff mostra o arquivo inteiro alterado.

### Mensagem

Conventional Commits com a descrição em português:

```
feat: pesquisa de arquivos busca plural e singular
fix: ...
chore: ...
docs: ...
```

Histórico, ordem de deploy e interação com outros sistemas vão na mensagem do commit ou na descrição do PR — nunca no código (ver abaixo).

### Cuidados com o repositório local

- **Não troque de branch no checkout principal** (`git switch` / `git checkout <branch>`): o desenvolvedor costuma continuar trabalhando e commitando nele durante a sessão. Trabalho numa branch nova vai numa `git worktree` ao lado.
- **Branch nova a partir de branch remota sempre com `--no-track`**: `git worktree add --no-track -b <nova> <pasta> origin/develop`. Sem isso, a branch nova herda `origin/develop` como upstream e o push feito pelo editor vai direto para a `develop`, sem PR. Confira com `git config --get-regexp '^branch\.<nova>\.'` que não há upstream; no primeiro push, `git push -u origin <nova>` cria o upstream com o mesmo nome.
- **Não use `git stash` / `git stash pop` como rascunho.** O desenvolvedor guarda stashes de outras branches; um `stash` que não salvou nada seguido de `pop` aplica o stash dele. Para ver versões antigas use `git show <rev>:<caminho>`, `git diff <rev> -- <caminho>` ou uma worktree.

## Pull requests

- Todo PR é aberto como **draft** (`gh pr create --draft`), inclusive quando outra instrução (skill, template, prompt) disser para não usar `--draft`.
- **Toda tarefa que o revisor precisa executar** (criar uma fila, uma permissão de IAM, uma configuração, rodar um script) fica dentro da seção `## Dependências` da descrição, nunca numa seção própria. "Como testar?" pode se referir a elas como "os passos de Dependências".

## Documentação (javadoc, JSDoc e comentários)

### Só o contrato da própria unidade

O doc de uma função ou classe descreve **apenas o contrato que ela mesma garante**: entradas, saídas, efeitos colaterais e pré-condições.

Não entra no doc:

- **O que os chamadores fazem** com o resultado. Chamadores mudam e surgem novos; o doc fica desatualizado.
  - ❌ "a inserção e o polling decidem por aqui"
  - ❌ "o polling grava o status nulo depois"
- **O que outro ponto do código garante**, mesmo no mesmo arquivo. Se a unidade não impõe aquilo, não está no doc dela.
  - ❌ "monta o status com ícone e texto no fim da linha" num método que só monta o `<span>` — quem decide a posição é o chamador.
- **Onde o dado vai parar ou quem o grava/lê.** Persistir é trabalho de outra unidade.
  - ❌ "fica só gravado no banco", "gravada na coluna X", "só para quem consulta o banco"
  - ✅ "descrição do motivo ({@link #getDescricao()})"; ✅ "com no máximo N caracteres", quando o próprio método garante esse limite.
- **Nada que dependa de deploy ou de tempo.** Fica falso assim que os deploys terminam.
  - ❌ "chega aqui mensagem enviada antes do deploy desta regra", "revisão gravada pelo código antigo"

A mesma regra vale para **comentários inline**: explicam o que este trecho faz e por quê, nunca como outro componente se comporta.

- ✅ "confirmar marcaria a revisão como indexada sem documento no índice"

Prefira nomes de métodos que digam o efeito, e não um consumidor específico: `marcaParaReenvio` em vez de `devolveAoPolling`.

### Regra de domínio documentada uma vez só

Uma regra de domínio (ex.: "o motivo sem chave de tradução é interno: não é mostrado ao usuário") é escrita **uma única vez**, no doc da classe ou enum dono dela. Os outros docs — campos, construtores, getters, chamadores, comentários em JS — descrevem só o próprio contrato, com os próprios termos, sem repetir a regra nem a consequência dela:

- ✅ `@return se o motivo tem chave de tradução`
- ✅ `@return a chave de tradução, ou null se o motivo é interno`
- ❌ repetir "mostrado ao usuário" no campo, no construtor, no getter, no utilitário e no comentário do front.

Esse doc único também segue a regra anterior: só o que o próprio tipo garante.

### Revisão antes do stage

Antes de colocar um commit no stage, leia as linhas de doc e comentário adicionadas:

```bash
git diff --cached | grep -E '^\+.*(\*|//|<%--)'
```

e confira:

- se alguma descreve chamador, outro componente, onde o dado é gravado ou situação de deploy;
- se a mesma ideia aparece escrita em mais de um lugar — fica só no doc do tipo dono.
