# Avaliação do Framework Chaperone para Design de Peptídeos em Complexo ABC e Uso no Google Colab

Este documento apresenta a análise do framework Chaperone, considerando sua viabilidade para o design de peptídeos ligantes ao complexo ABC de procariontes, bem como a avaliação sobre o seu uso em ambiente Google Colab, finalizando com um plano de design estruturado.

## 1. Avaliação do Framework Chaperone

O Chaperone é um framework unificado, de alto nível, projetado para facilitar a interoperabilidade entre diversas ferramentas de biologia estrutural, organizando o pipeline como blocos independentes (Dataframe in -> Dataframe out).

**Para o Design de Peptídeos em Complexo ABC:**
- **Adequação:** O Chaperone é **extremamente adequado** e é uma das ferramentas no estado-da-arte para esta tarefa. Ele suporta blocos nativos como `bindcraft` e `bindcraft2`, além de ferramentas de difusão de backbone (como `rfd3`) ou co-design de complexos, perfeitos para a geração de "binders" (ligantes).
- **Recursos Nativos:** O framework já traz suporte para *ProteinMPNN* (design de sequência) e uma série de oráculos para validação da estrutura, incluindo o *AlphaFold3 (AF3)*, *RoseTTAFold3 (RF3)*, *Protenix*, e *ESMFold2*, que são essenciais para validar se o peptídeo predito realmente se liga à superfície do complexo ABC procarionte alvo.
- **Suporte Multímero:** Como o alvo é um complexo proteico (Complexo ABC), as ferramentas integradas como `bindcraft2` permitem passar a estrutura alvo e criar binders baseados em regiões de hotspot, sendo ideal para interações PPI (Protein-Protein Interaction).

## 2. Viabilidade no Google Colab

**A avaliação aponta que o Chaperone NÃO é a ferramenta ideal para ser rodada nativamente no Google Colab de forma direta.**

**Motivos:**
1. **Orquestração e Execução:** O Chaperone foi construído em volta de orquestração via *Slurm* (`ExecutionEngine`), gerenciamento de chunks (pedaços de execução) em diretórios de *sandbox* locais, e requer submissões destacadas e persistentes. O Google Colab não oferece um sistema de escalonamento Slurm.
2. **Dependência de Containers:** O framework confia massivamente em *Apptainer* e *Docker* para encapsular os ambientes de execução das redes complexas (ex: `bindcraft2.sif`, `colabfold.sif`, `rfd3.sif`). Executar Apptainer/Docker dentro do ambiente do Google Colab (que já é um container) é um processo bastante complexo, instável e frequentemente bloqueado pelas políticas da plataforma.
3. **Requisitos de Hardware e Tempo de Vida:** Pipelines de design requerem grande poder computacional e horas ininterruptas (às vezes 24h+ de alocação `time=24:00:00` nos containers do BindCraft). O Colab tem tempos de sessão voláteis (máximo 12h, mas frequentemente desconecta em inatividade).
4. **Armazenamento:** O peso das *weights* (pesos das redes neurais) de diversos modelos e os próprios containers facilmente esgotariam a cota gratuita de disco do Colab.

**Alternativas para o Colab:**
Se o uso do Colab for um requisito estrito para você, o ideal seria não usar a orquestração do *Chaperone*, mas sim utilizar os notebooks originais providos pelos criadores de algumas destas redes como o **ColabDesign** (que encapsula AF2/RF para geração de binders de forma nativa e amigável ao Colab) ou os notebooks providos pelo **BindCraft**.

Se você tiver acesso a um cluster HPC, servidor local com GPUs e Slurm, o **Chaperone é a escolha ideal**.

---

## 3. Plano Estruturado de Design (via Chaperone HPC/Slurm)

Caso prossiga com o Chaperone no ambiente adequado, este é o plano estruturado de 4 fases para resolver o seu problema.

### Fase 1: Implementação e Preparação do Alvo (Code & Inputs)
1. **Limpeza do Alvo:** Obter o PDB do complexo ABC procarionte. Escolher a cadeia ou cadeias específicas que interagirão com o peptídeo e remover moléculas de água, ligantes irrelevantes, e normalizar o PDB.
2. **Definição dos Hotspots:** Identificar no PDB os resíduos da interface alvo onde o peptídeo deve se acoplar.
3. **Escrita do Script Híbrido (`pipeline.py`):**
   - Configurar o `ExecutionEngine` apontando para o diretório de run (`rundir`).
   - Usar o bloco `bindcraft2` (focado em mini-proteínas/peptídeos ligantes).
   - Inserir a cadeia alvo, hotspots mapeados, e definir os limites de tamanho do peptídeo (ex: `lengths=(15, 40)`).

### Fase 2: Integration Test
1. Rodar um teste simples e rápido localmente numa GPU de teste ou submissão curta usando `/slurm-interactive`.
2. Usar o bloco `esmfold2` ou `colabfold` para uma triagem inicial muito barata, apenas para verificar se o pipeline compila, se o `bindcraft2` roda a primeira etapa com sucesso sem erro de OOM (Out-of-Memory) e se o output está fluindo no Polars DataFrame esperado.

### Fase 3: Pilot Experiments (Sweep de Parâmetros)
1. Lançar o job em *background* via Slurm.
2. Rodar uma amostragem de 100 a 500 backbones com variações de *hotspots* (testar diferentes cavidades do complexo ABC).
3. Após o backbone (BindCraft / RFD3), refinar/re-desenhar a sequência usando o bloco `mpnn` (LigandMPNN ou ProteinMPNN).
4. Usar um oráculo de predição rápido para esse pilot, como `rf3` ou `af3`, apenas com número de sementes (seeds) pequeno. Analisar no final métricas como `RMSD`, `pLDDT` e métricas de contato.

### Fase 4: Campaign Scale (Produção de Designs para Compra/Testes *in vitro*)
1. Filtrar as melhores topologias derivadas da fase piloto.
2. Submeter ao `bindcraft2` / MPNN para gerar entre 10.000 e 50.000 designs distintos, aproveitando as funções nativas de `catcif` para o armazenamento (evitando estourar os limites de inóde do filesystem).
3. Usar o `af3` (AlphaFold 3) no final do funil de filtragem apenas para a fração de peptídeos que passaram pelas portas rígidas (hard gates) de geometria e métricas de H-bond/Contato. O AF3 atuará como oráculo rigoroso final.
4. Extrair os melhores candidatos da tabela final em formato Fasta/CSV prontos para síntese.
