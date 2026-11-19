# EcoDestino - Assistente de Descarte de Óleo de Cozinha

## Identificação Acadêmica
**Disciplina**: Prompt and Artificial Intelligence <br>
**Curso**: Ciências da Computação - 1º Ano <br>
**Turma**: 1CCPI <br>
**Integrantes**: Dupla
**Frente B:** Descarte e sustentabilidade

| Nome | RM |
|------|----|
| Kauanne Paula de Oliveira | 574191 |
| Nayhely Estela Calle Castillo | 571416 |

## 1. O que é

Assistente que responde ao cidadão onde e quando descartar óleo de cozinha usado, com base em documentos municipais simulados. Cada resposta cita o documento de origem, usa ferramentas para status e horários, e tem a qualidade medida com RAGAS.

## 2. Pipeline

```
documentos (.txt) → chunking (500/100) → embeddings → índice ChromaDB (metadados: doc_id, data_publicacao)
pergunta → guardrails (LGPD, escopo, ponto inexistente)
         → recuperação híbrida (semântica k=4 + palavra-chave k=3, mais recente primeiro)
         → filtro de prompt injection → prompt → LLM → máscara de dados pessoais → resposta + fontes
pergunta de status/horário → agente com 2 tools (calcula, não inventa)
```

## 3. Base documental e a armadilha

Três documentos escritos pelo grupo (172 a 202 palavras cada): regulamento (10/01), cadastro de pontos (15/02) e comunicado de interdição (20/02).
- **Contradição/desatualização:** o cadastro (15/02) diz ECO-02 "ATIVO"; o comunicado (20/02) o interdita e "substitui informações anteriores". Vale o mais recente.
- **Faixa condicional:** o ECO-03 aos sábados vai só até 12h (e não até 14h do cadastro) enquanto durar a interdição.
- **Detalhe escondido:** a regra das 12h está na "INFORMAÇÃO COMPLEMENTAR" no fim do comunicado, não no título nem no primeiro parágrafo (força um chunking bem feito).
Todos os documentos têm metadados `titulo`, `data_publicacao` e `tema`.

## 4. Decisões técnicas

- **Chunking:** comparamos 500/100 e 1000/200 ![RAG com fontes](prints/03_rag.png). Escolhemos 500/100 porque isola cada bloco "PONTO ECO-xx" e o comunicado, sem misturar pontos no mesmo trecho.
- **Embeddings:** `paraphrase-multilingual-MiniLM-L12-v2` (funciona melhor em português que o MiniLM em inglês).
- **Recuperação híbrida:** a busca só semântica (v1) não distinguia ECO-01 de ECO-02 e trazia sempre o comunicado. Adicionamos busca por palavra-chave (ids citados e termos raros) e ordenamos do documento mais recente para o mais antigo. Cobertura dos trechos necessários: **v1 = 8/12 (67%) → v2 = 12/12 (100%)**.
- **LLM:** llama3.1:8b . Ollama Cloud retornou erro 402 (cota), então usamos modelo local no Colab com GPU.
- **Agente:** 2 tools (`consultar_status_ponto`, `calcular_proximo_horario`). O status e o horário são lidos dos documentos pelo código; o LLM só decide quando chamar. Limite de 6 chamadas por pergunta, erro controlado e bloqueio de chamadas repetidas.

## 5. Guardrails (4)

1. **Escopo:** pergunta fora do tema recebe recusa e o canal oficial.
2. **Dado sensível (LGPD):** pedidos de dados de moradores são recusados; CPF, e-mail e telefone na resposta são mascarados.
3. **Prompt injection na fonte:** trechos com instruções ao modelo ("ignore as instruções...") são removidos do contexto antes de chegar ao LLM.
4. **Limite de confiança:** ponto inexistente (ex.: ECO-05) e perguntas sem informação geram "não há informação na base". Há teste automatizado dos guardrails no notebook.
*Limitação:* a distância semântica sozinha não separa bem o que é do tema (0,41 a 0,75) do que não é (0,77 a 0,92); por isso o filtro de escopo usa palavras-chave e o limiar ficou conservador (0,80).

## 6. Avaliação (RAGAS 0.4.4)
8 perguntas (4 simples, 4 armadilha) com ground truth escrito pelo grupo. Juiz: llama3.1:8b (juiz pequeno: os valores são comparativos, não absolutos).

| Métrica | Geral | Nível 1 | Nível 2 |
|---|---|---|---|
| faithfulness | 0,71 | 0,60 | 0,81 |
| response_relevancy | 0,68 | 0,73 | 0,62 |
| context_precision | 0,70 | 0,54 | 0,87 |
| context_recall | 1,00 | 1,00 | 1,00 |

**Leitura dos números:** context_recall 1,00 é favorecido pela base pequena. A precision do nível 1 é baixa (0,25 no horário do ECO-01) porque ordenar do mais recente para o mais antigo, necessário para a contradição, coloca o comunicado antes do cadastro. A faithfulness oscila (0,25 a 1,0) mesmo com contexto correto, o que indica instabilidade do juiz pequeno. **Melhoria proposta:** reranking por relevância usando a data só como desempate (meta: precision do nível 1 ≥ 0,70) e juiz maior. Resultado completo em `3_avaliacao_ragas.json`.

## 7. Limitações
- Na pergunta "Posso levar óleo ao ECO-03 no sábado às 13h?" o "não pode" está certo, mas a justificativa atribui ao ECO-03 a suspensão do ECO-02; a faithfulness não detecta esse erro.
- A base é pequena e simulada; não há dados reais nem atualização automática.
- O filtro de escopo e o de injection usam regras de palavras-chave, que podem ser contornadas.
- A interface responde cada pergunta isoladamente (o histórico aparece na tela, mas não entra no modelo).

## 8. Custo aproximado
RAG: 1 chamada ao LLM por pergunta. Agente: 2 a 4 chamadas. Avaliação RAGAS: 4 métricas × 8 perguntas, cerca de 6 minutos.

## 9. Como executar
1. Abrir `1_Notebook_GS2026_2.ipynb` no Google Colab e selecionar GPU T4.
2. Em Secrets, criar `OLLAMA_API_KEY` com acesso ao notebook (a chave nunca aparece no código).
3. Executar tudo (Runtime, Run all). Versões usadas: langchain 1.5.2, langchain-core 1.8.3, langchain-ollama 1.1.0, langchain-chroma 1.1.0, chromadb 1.5.9, langchain-huggingface 1.2.3, sentence-transformers 5.7.0, ragas 0.4.4, gradio 6.31.0.

## 10. Prints
## 10. Prints
![Comparação de chunking](prints/01_chunking.png)
![Índice ChromaDB](prints/02_chroma_colecao.png)
![Busca semântica com distâncias](prints/02_chroma_busca.png)
![RAG com fontes](prints/03_rag.png)
![RAG: "não sei"](prints/03_rag_nao_sei.png)
![Agente: ECO-03 no domingo](prints/04_agente_domingo.png)
![Agente: erro controlado](prints/04_agente_erro_controlado.png)
![Interface Gradio, modo RAG](prints/05_gradio_rag.png)
![Interface Gradio, modo agente](prints/05_gradio_agente.png)
![Guardrails](prints/06_guardrails.png)
![Testes dos guardrails](prints/06_guardrails_testes.png)
![RAGAS, execução](prints/07_ragas_execucao.png)
![RAGAS, tabela](prints/07_ragas_tabela.png)
![Cobertura v1 x v2](prints/08_v1_v2.png)

## 11. Uso de IA
IA usada para rascunhar código, explicar erros e gerar parte dos documentos revisados pelo grupo. Registro em `6_Chat_Log.txt`.
