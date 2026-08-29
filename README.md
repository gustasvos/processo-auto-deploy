# Processo de Deploy Automatizado

> Documento de referência para automação de build, deploy e recuperação de falhas do projeto. Baseado nos princípios de **Continuous Integration** (Martin Fowler), seções *"Automate Deployment"* e *"Fix Broken Builds Immediately"*. As definições aqui são independentes de ferramenta específica (CI provider, registry, cloud), com foco no processo.

## 1. Estrutura de branches e fluxo de integração

| Branch | Papel |
|---|---|
| `feature/*` | Desenvolvimento individual/em par de uma funcionalidade, criada a partir de `develop` |
| `develop` | Branch de integração contínua **e** ambiente de homologação |
| `main` | Branch de produção — só recebe código já validado em `develop` |

**Fluxo:** `feature/*` → `develop` → `main`

### `develop` assumirá dois papéis

Por restrição do projeto, `develop` funciona simultaneamente como branch de integração e como ambiente de homologação.

- Só existe uma versão de homologação visível por vez.
- Se duas features forem mescladas em sequência, a homologação sempre reflete o último merge — não é possível homologar features isoladamente em paralelo.
- Possibilidade de existir ambientes efêmeros por feature/PR; não é viável neste projeto.

## 2. Automação por branch

### 2.1 `develop`

- **Build**: executado obrigatoriamente a cada alteração na `develop` (via PR ou push direto). Valida que o código compila e passa nos testes definidos pela equipe.
- **Deploy automático a cada alteração:** Job de deploy é disparado a cada alteração na `develop`, mas isso pode causar problemas no desenvolvimento simultâneo, não sei como ficaria na VM.
- **Deploy**: **não é automático a cada alteração**. É dado como um job separado, disparado manualmente quando o time decidir homologar. Isso evita deploys constantes de features ainda instáveis e economiza o uso do ambiente de homologação, que não pode ficar ativo o tempo todo durante as 12 semanas do projeto.
- Serão dois workflows separados (build e deploy) exatamente para permitir o desacoplamento entre "validar que compila" e "colocar no ar para homologação". Evita consumo alto dos recursos da nuvem.

### 2.2 `main`

- **Testes, build e deploy** são obrigatórios a cada alteração na `main`.
- **Cadência mínima de deploy em produção**: pelo menos uma vez ao fim de cada sprint. Deploys adicionais podem ocorrer com mais frequência conforme o andamento do projeto. **PRECISAMOS DEFINIR PARA NÃO FICAR ABSTRATO.**

## 3. Preparação de ambiente como pré-requisito, antes do desenvolvimento começar

Durante a semana de planejamento, antes do início do desenvolvimento:

1. **Versões de tecnologia serão travadas**: linguagens, frameworks e demais dependências têm suas versões definidas no arquivo de workflow, garantindo que o ambiente construído automaticamente sempre corresponda ao planejado e continue funcionando futuramente.
2. **Ambientes de produção pré-configurados**: a infraestrutura de produção (registry de imagens, máquina/VM de produção) e as credenciais necessárias já estão criadas e prontas antes do projeto começar, incluindo o necessário para a VM executar o processo de deploy (runtime de containers instalado, etc.).


## 4. Prevenção de builds quebrados na `main`

A branch `develop` será utilizada para correção de erros (com ou sem novas branches `fix`), juntamente dos recursos como criação de Issues.

- A `main` só recebe código que já foi validado (build + testes) na `develop`.
- Qualquer falha de build ou deploy é corrigida na `develop` ou nas branches `feature/*` de origem — **nunca diretamente em produção**.
- Como consequência prática, a `main` deve idealmente não apresentar uma falha de build, já que passou pelo filtro da homologação antes de chegar lá.

## 5. Rollback

Mesmo com a prevenção acima, rollback é mantido como camada complementar de segurança — cobre falhas que só se manifestam em produção (carga real, configuração específica de produção, integração com serviços externos, etc.), que nenhuma homologação reproduz com 100% de fidelidade.

Princípio adotado: **cada deploy gera um artefato versionado de forma imutável e rastreável ao commit de origem**, em vez de sempre sobrescrever a mesma versão "mais recente". Isso garante que a versão anterior continua existindo e acessível.

O job de deploy é capaz de receber qual versão deployar. Reverter um deploy problemático em produção significa **executar o mesmo job de deploy apontando para a versão anterior conhecida como estável**, em vez de criar um processo improvisado no momento do incidente.

A equipe mantém rastreabilidade de qual foi a última versão estável (via histórico de commits da `main` e/ou registro simples tipo changelog), para que o rollback possa ser executado rapidamente quando necessário.

**ROLLBACK AINDA NÃO ENTENDI COMO FUNCIONA**