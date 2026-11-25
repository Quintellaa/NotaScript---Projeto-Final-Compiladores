# 🎓 NotaScript — cálculo de notas e aprovação

**Integrantes:**

- João Victor Brandão_2359197
- Kathleen Aquino Lima_2364196
- Lucas Costa_2361186
- Victor Quintella_2544963


Projeto Final de Compiladores: um **compilador completo em Python** para a **NotaScript**, uma DSL para lançar notas,
calcular médias e decidir aprovação, reprovação ou exame final. O compilador passa por todas as fases:

```
texto → Léxico → Sintático (Lark LALR) + AST → Semântico → TAC → Otimização → VM (execução)
                                                                   └→ Assembly (bônus)
```

Tudo está no notebook **[`NotaScript.ipynb`](NotaScript.ipynb)**, que roda do início ao fim com *Reiniciar e executar tudo*.

---

## 1. Tema e ficha da linguagem

| Item | NotaScript |
|---|---|
| **Nome** | NotaScript |
| **Domínio** | cálculo de notas e aprovação de alunos |
| **Tipos** | `inteiro`, `decimal`, `texto`, `logico` |
| **Comandos do domínio** | `lancar <disciplina>, <nota>;` · `aprovar <disciplina>;` · `reprovar <disciplina>;` · `agendar_exame <disciplina>, <dias>;` |
| **Controle de fluxo** | `se` / `senao` / `senao se`, `enquanto` |
| **Saída** | `mostrar <expr>;` |
| **Funções (bônus)** | `funcao <tipo> nome(<tipo> p, ...) { ... retornar <expr>; }` · chamada `nome(args)` |

### Regras de negócio

| Regra | Onde é verificada | Tipo |
|---|---|---|
| Nota lançada deve estar entre 0 e 10 | Semântico (valor constante) **e** VM (valor real calculado) | erro |
| Pesos de uma média ponderada (`a*0.4 + b*0.6`) devem somar 1.0 | Semântico | **aviso** |
| Aprovar com média < 6.0 | VM (valor real) | erro de execução |
| Aprovar/reprovar/agendar disciplina sem nota lançada | Semântico (+ "Você quis dizer...?") e VM | erro |
| Prazo do exame final entre 1 e 30 dias | Semântico e VM | erro |

### Exemplo

```
decimal a1 = 7.5;
decimal a2 = 5.0;
decimal media = a1 * 0.4 + a2 * 0.6;
lancar "Compiladores", media;
se media >= 6.0 { aprovar "Compiladores"; } senao { agendar_exame "Compiladores", 3; }
```

O notebook traz **5 programas-exemplo** (média ponderada, recuperação com `enquanto`, boletim com `senao se`,
turma com pesos fixos, funções com recursão), além de um de escopo de bloco e um do semestre inteiro.

### Exemplo com funções (bônus)

```
funcao decimal media_ponderada(decimal n1, decimal n2, decimal peso1) {
    retornar n1 * peso1 + n2 * (1.0 - peso1);
}

funcao inteiro semanas_ate(decimal nota, decimal meta) {
    se nota >= meta { retornar 0; }
    retornar 1 + semanas_ate(nota + 0.75, meta);    // recursão
}

decimal m_redes = media_ponderada(4, 5.0, 0.5);
lancar "Redes", m_redes;
mostrar "Redes: faltam " + semanas_ate(m_redes, 6.0) + " semana(s) de estudo";
```

---

## 2. Tabela de tokens (41 tipos, 19 palavras reservadas)

| Categoria | Token | Lexema |
|---|---|---|
| Tipos (reservadas) | `INTEIRO` `DECIMAL` `TEXTO` `LOGICO` | `inteiro` `decimal` `texto` `logico` |
| Controle (reservadas) | `SE` `SENAO` `ENQUANTO` | `se` `senao` `enquanto` |
| Domínio (reservadas) | `LANCAR` `APROVAR` `REPROVAR` `AGENDAR_EXAME` `MOSTRAR` | `lancar` `aprovar` `reprovar` `agendar_exame` `mostrar` |
| Funções (reservadas, bônus) | `FUNCAO` `RETORNAR` | `funcao` `retornar` |
| Lógicos (reservadas) | `VERDADEIRO` `FALSO` `E` `OU` `NAO` | `verdadeiro` `falso` `e` `ou` `nao` |
| Identificador | `ID` | `[a-zA-Z_][a-zA-Z0-9_]*` |
| Literais | `NUM_DECIMAL` `NUM_INTEIRO` `TEXTO_LITERAL` | `7.5` · `3` · `"Compiladores"` |
| Relacionais | `IGUAL_IGUAL` `DIFERENTE` `MENOR_IGUAL` `MAIOR_IGUAL` `MENOR` `MAIOR` | `==` `!=` `<=` `>=` `<` `>` |
| Atribuição | `IGUAL` | `=` |
| Aritméticos | `MAIS` `MENOS` `VEZES` `DIVIDIDO` `RESTO` | `+` `-` `*` `/` `%` |
| Delimitadores | `ABRE_PAR` `FECHA_PAR` `ABRE_CHAVE` `FECHA_CHAVE` `VIRGULA` `PV` | `(` `)` `{` `}` `,` `;` |
| Ignorados | `COMENTARIO`, espaços | `// ...` |

**Erros léxicos** têm linha, coluna, o trecho marcado com `^` e dicas específicas: acento em nome (`média`), texto sem
fechar aspas, `&`/`|` no lugar de `e`/`ou`, `!` sozinho, `#` como comentário, aspas simples, `[ ]`, `:`.

---

## 3. Gramática (Lark, LALR)

```
start: (funcao_decl | comando)*
?comando: declaracao | atribuicao | se_cmd | enquanto_cmd
        | lancar_cmd | aprovar_cmd | reprovar_cmd | agendar_cmd | mostrar_cmd
        | retornar_cmd | chamada_cmd

// funções (bônus) — só no nível principal
funcao_decl:  FUNCAO tipo ID ABRE_PAR parametros? FECHA_PAR bloco
parametros:   parametro (VIRGULA parametro)*
parametro:    tipo ID
retornar_cmd: RETORNAR expr PV
chamada_cmd:  chamada PV
chamada:      ID ABRE_PAR argumentos? FECHA_PAR
argumentos:   expr (VIRGULA expr)*

declaracao: tipo ID IGUAL expr PV | tipo ID PV
tipo: INTEIRO | DECIMAL | TEXTO | LOGICO
atribuicao: ID IGUAL expr PV
se_cmd: SE expr bloco | SE expr bloco SENAO bloco | SE expr bloco SENAO se_cmd
enquanto_cmd: ENQUANTO expr bloco
bloco: ABRE_CHAVE comando* FECHA_CHAVE

lancar_cmd:   LANCAR expr VIRGULA expr PV
aprovar_cmd:  APROVAR expr PV
reprovar_cmd: REPROVAR expr PV
agendar_cmd:  AGENDAR_EXAME expr VIRGULA expr PV
mostrar_cmd:  MOSTRAR expr PV

// 8 níveis de precedência (do menor para o maior)
?expr: ou_expr
?ou_expr:    ou_expr OU e_expr | e_expr
?e_expr:     e_expr E nao_expr | nao_expr
?nao_expr:   NAO nao_expr | igualdade
?igualdade:  igualdade (IGUAL_IGUAL | DIFERENTE) relacional | relacional
?relacional: soma (MENOR | MAIOR | MENOR_IGUAL | MAIOR_IGUAL) soma | soma
?soma:       soma (MAIS | MENOS) termo | termo
?termo:      termo (VEZES | DIVIDIDO | RESTO) unario | unario
?unario:     MENOS unario | primario
?primario:   NUM_DECIMAL | NUM_INTEIRO | TEXTO_LITERAL | VERDADEIRO | FALSO | ID | chamada | ABRE_PAR expr FECHA_PAR
```

A AST é feita de `dataclass`es (uma por comando do domínio) e desenhada com `mostrar_ast`. Erros sintáticos dizem
linha, coluna, **o que era esperado** e dão dica (faltou `;`, `7,5` em vez de `7.5`, `=` em vez de `==`, `if` em vez de `se`,
bloco sem `}`).

---

## 4. O que cada fase faz

| Fase | Destaques |
|---|---|
| **Semântico** | Tabela de símbolos com **escopo de bloco** (sombreamento vira nome único no TAC, ex. `nota_1`); 4 tipos com coerção `inteiro→decimal` e `qualquer + texto → texto`; 5 regras de negócio; avisos (pesos, variável nunca usada, sombreamento, laço provavelmente infinito); **galeria com 12 erros semânticos diferentes** (3 deles de funções). |
| **TAC** | Desvios com **condição invertida** (`se media >= 6.0` → `if media < 6.0 goto L2`); uma instrução por comando do domínio: `LANCAR`, `APROVAR`, `REPROVAR`, `AGENDAR_EXAME`, `MOSTRAR`; coerção explícita `x = (decimal) t1`. |
| **Otimização** | 6 técnicas até o ponto fixo: dobramento de constantes, propagação de constantes/cópias, simplificação algébrica, coalescência de temporários, eliminação de código morto, simplificação de desvios. **Provado pela VM** em 5 programas: mesma saída e menos instruções executadas. |
| **VM** | Executa o TAC. Cada comando do domínio gera saída e alimenta o resumo (notas lançadas, média geral, aprovações, reprovações, exames, taxa de aprovação, instruções executadas). Erros tratados: divisão por zero, laço infinito, aprovar com média < 6, nota real fora de 0..10, prazo de exame inválido, disciplina sem nota. |
| **Testes** | 37 casos automáticos: 9 válidos, 3 léxicos, 5 sintáticos, 9 semânticos, 2 avisos, 6 de execução, 3 de otimização (11 deles cobrem funções). **Todos passando.** |
| **Studio** | Painel `ipywidgets` com os exemplos e 8 visões: Execução, Tokens, AST, Semântico, TAC, Otimização, Assembly e Gráfico. |

### Bônus implementados

- **Funções**: `funcao`/`retornar` com parâmetros tipados, chamadas como expressão ou comando, **recursão** e **pilha de chamadas** na VM.
  - *Semântico*: escopo próprio (só parâmetros e locais), aridade e tipos dos argumentos (com coerção), tipo do retorno, `retornar` fora de função, função sem `retornar`, função não declarada (com sugestão), aviso de função nunca chamada.
  - *TAC*: `param x` · `t = call f, n` · `funcao f(p1, p2):` · `return x` · `halt`.
  - *VM*: cada `call` empilha (memória, endereço de retorno, destino) e cria um quadro novo; `return` desempilha. O resumo mostra chamadas e profundidade máxima da pilha, e há um **rastro** das chamadas. Erros: estouro da pilha (recursão sem caso base) e função que termina sem `retornar`.
  - *Otimização/Assembly*: `call` nunca é eliminado; Assembly com `PUSH`, `CALL`, `RET` e registrador de retorno `RV`.
- **Código objeto**: Assembly com `k` registradores, alocação LRU e **spilling** (`STORE ... ; spill de t1`).
- **"Você quis dizer...?"**: `difflib.get_close_matches` para variáveis e para nomes de disciplinas.
- **Integração contínua**: [`.github/workflows/testes.yml`](.github/workflows/testes.yml) executa o notebook inteiro (inclusive a bateria de testes) a cada push.
- **Saída visual**: gráfico de barras com a média de cada disciplina, a linha de corte 6.0 e a situação final.

---

## 5. Como rodar

**Jupyter / VS Code / Colab:** abra `NotaScript.ipynb` e use *Reiniciar e executar tudo*. A primeira célula instala
`lark`, `ipywidgets` e `matplotlib` se estiverem faltando. A última célula abre o **NotaScript Studio**.

**Linha de comando (o mesmo que o CI faz):**

```bash
pip install -r requirements.txt
```

```bash
jupyter nbconvert --to notebook --execute NotaScript.ipynb --output executado.ipynb
```

Se algum teste falhar, a célula do Passo 8 dispara um `AssertionError` e o comando termina com erro.

---

## 6. Exemplos de saída

**Execução do Exemplo 2 (recuperação com `enquanto`):**

```
📤 SAÍDA
   Semana 1: nota 4.25
   Semana 2: nota 5.00
   Semana 3: nota 5.75
   Semana 4: nota 6.50
   📝 Nota lançada — Calculo: 6.50
   Precisou de muito esforço!
   ✅ Calculo: APROVADO (média 6.50)

📊 RESUMO
   Notas lançadas ........ 1
   Média geral ........... 6.50
   Aprovações ............ 1
   ...
```

**TAC do Exemplo 1 (condição invertida):**

```
    a1 = 7.5
    a2 = 5.0
    t1 = a1 * 0.4
    t2 = a2 * 0.6
    t3 = t1 + t2
    media = t3
    LANCAR "Compiladores", media
    if media < 6.0 goto L2
    APROVAR "Compiladores"
    goto L1
L2:
    AGENDAR_EXAME "Compiladores", 3
L1:
```

**Otimização provada pela VM:**

| Programa | Instruções executadas (antes → depois) | Mesma saída? |
|---|---|---|
| Exemplo 1 — média ponderada | 10 → 2 | ✅ |
| Exemplo 2 — recuperação (laço) | 48 → 40 | ✅ |
| Exemplo 3 — boletim | 18 → 5 | ✅ |
| Exemplo 4 — turma com pesos fixos | 51 → 26 | ✅ |
| Exemplo 5 — funções e recursão | 64 → 61 | ✅ |

**Rastro da pilha de chamadas (Exemplo 5):**

```
→ semanas_ate(4.50, 6.00)
   → semanas_ate(5.25, 6.00)
      → semanas_ate(6.00, 6.00)
      ← semanas_ate devolveu 0
   ← semanas_ate devolveu 1
← semanas_ate devolveu 2
```

**Erro semântico com sugestão:**

```
❌ Erro semântico (linha 2, coluna 9): 'aprovar' na disciplina "Compiladore", que ainda não tem nota lançada
     2 | aprovar "Compiladore";
                 ^
   💡 Dica: Você quis dizer "Compiladores"?
```

**Erro de execução (regra de negócio com valor real):**

```
❌ Erro de execução (linha 3): não é possível aprovar "Compiladores" com média 5.50 (mínimo 6.0)
   💡 Dica: Proteja o comando: se media >= 6.0 { aprovar ...; } senao { agendar_exame ...; }
```

---

## 7. 🖥️ NotaScript Studio — passo a passo

Telas reais do painel (última célula do notebook), capturadas depois de *Reiniciar e executar tudo*.

### 7.1 Painel aberto

O painel abre parado. Escolha um exemplo (ou edite o código) e clique em **▶ Compilar e executar**.

![Studio inicial](docs/studio/01_inicial.png)

### 7.2 Exemplo 1 — média ponderada, aba por aba

Ao compilar, o pipeline lateral marca cada fase (✓ passou · ✕ falhou) e as abas mostram cada visão.

**▶ Execução** — indicadores, tabela de disciplinas e saída do programa:

![Exemplo 1 — Execução](docs/studio/02_ex1_execucao.png)

**Tokens:**

![Exemplo 1 — Tokens](docs/studio/03_ex1_aba_tokens.png)

**AST:**

![Exemplo 1 — AST](docs/studio/04_ex1_aba_ast.png)

**Semântico** — tabela de símbolos e avisos:

![Exemplo 1 — Semântico](docs/studio/05_ex1_aba_semantico.png)

**TAC:**

![Exemplo 1 — TAC](docs/studio/06_ex1_aba_tac.png)

**Otimização** — antes × depois, técnicas usadas e prova pela VM:

![Exemplo 1 — Otimização](docs/studio/07_ex1_aba_otimizacao.png)

**Assembly** — registradores e spilling:

![Exemplo 1 — Assembly](docs/studio/08_ex1_aba_assembly.png)

**Gráfico:**

![Exemplo 1 — Gráfico](docs/studio/09_ex1_aba_grafico.png)

**Com "Otimizar o TAC" desligado** — a fase de otimização aparece como desligada e a VM executa mais instruções:

![Exemplo 1 — sem otimização](docs/studio/10_ex1_sem_otimizacao.png)

### 7.3 Exemplo 2 — recuperação (`enquanto`)

![Exemplo 2 — Execução](docs/studio/11_2_recuperacao_enquanto_execucao.png)

![Exemplo 2 — TAC](docs/studio/12_2_recuperacao_enquanto_aba_tac.png)

### 7.4 Exemplo 3 — boletim (`senao se`)

![Exemplo 3 — Execução](docs/studio/13_3_boletim_senao_se_execucao.png)

![Exemplo 3 — Gráfico](docs/studio/14_3_boletim_senao_se_aba_grafico.png)

### 7.5 Exemplo 4 — turma com pesos fixos (otimização)

![Exemplo 4 — Execução](docs/studio/15_4_turma_com_pesos_fixos_otimizacao_execucao.png)

![Exemplo 4 — Otimização](docs/studio/16_4_turma_com_pesos_fixos_otimizacao_aba_otimizacao.png)

### 7.6 Exemplo 5 — escopo de bloco e avisos

![Escopo — Execução](docs/studio/17_5_escopo_de_bloco_e_avisos_execucao.png)

![Escopo — Semântico](docs/studio/18_5_escopo_de_bloco_e_avisos_aba_semantico.png)

### 7.7 Exemplo 6 — semestre inteiro (gráfico)

![Semestre — Execução](docs/studio/19_6_semestre_inteiro_grafico_execucao.png)

![Semestre — Gráfico](docs/studio/20_6_semestre_inteiro_grafico_aba_grafico.png)

### 7.8 Exemplo 7 — funções e recursão (bônus)

![Funções — Execução](docs/studio/21_7_funcoes_e_recursao_bonus_execucao.png)

![Funções — Semântico](docs/studio/22_7_funcoes_e_recursao_bonus_aba_semantico.png)

![Funções — TAC](docs/studio/23_7_funcoes_e_recursao_bonus_aba_tac.png)

![Funções — Assembly](docs/studio/24_7_funcoes_e_recursao_bonus_aba_assembly.png)

### 7.9 Erros de cada fase

**Erro léxico** — acento no nome da variável:

![Erro léxico](docs/studio/25_erro_lexico_acento_no_nome_execucao.png)

**Erro sintático** — faltou `;`:

![Erro sintático](docs/studio/26_erro_sintatico_faltou_execucao.png)

**Erro semântico** — variável não declarada, com "Você quis dizer...?":

![Erro semântico — variável não declarada](docs/studio/27_erro_semantico_variavel_nao_declarada.png)

**Erro semântico** — regra de negócio: nota fora de 0 a 10:

![Erro semântico — nota fora de 0 a 10](docs/studio/28_erro_semantico_nota_fora_de_0_a_10.png)

**Erro de execução** — divisão por zero:

![Erro de execução — divisão por zero](docs/studio/29_erro_de_execucao_divisao_por_zero_execucao.png)

**Erro de execução** — laço infinito:

![Erro de execução — laço infinito](docs/studio/30_erro_de_execucao_laco_infinito_execucao.png)

---

## 8. 👥 Quem fez o quê

| Integrante | Responsável por | Passos |
|---|---|---|
| João Victor Brandão | Análise léxica e sintática: tokens, gramática Lark LALR, AST e mensagens de erro | 1 e 2 |
| Kathleen Aquino Lima | Ficha da linguagem, programas-exemplo, README, testes automáticos e interface (Studio) | 0, 8 e 9 |
| Lucas Costa | Análise semântica: tabela de símbolos, tipos, regras de negócio e avisos | 3 |
| Victor Quintella | TAC, otimização e VM (comandos do domínio), além dos bônus de funções e Assembly | 4, 5 e 6 |

---

## 9. ✅ Checklist

### Antes de entregar

- [x] A ficha está completa e os 3 programas-exemplo compilam e executam.
- [x] Testamos pelo menos um erro de cada fase e as mensagens têm linha e dica.
- [x] As regras de negócio aparecem como erro/aviso no semântico.
- [x] O programa otimizado dá a mesma saída que o não otimizado.
- [x] Todos os testes estão ✅ (37/37).
- [x] *Reiniciar e executar tudo* roda sem erros.
- [x] O repositório é público e o README começa com os nomes de todos.

### Requisitos obrigatórios

- [x] **R1 · Ficha e programas-exemplo** — `MINHA_FICHA` com todos os itens ✅ em `conferir_ficha`; 5 programas-exemplo com `se`/`senao` e `enquanto`.
- [x] **R2 · Léxico** — 41 tipos de token (mín. 20), 19 palavras reservadas (mín. 8), comentários `//` ignorados, erro léxico com linha, coluna e 9 dicas específicas (mín. 3), tabela de tokens exibida.
- [x] **R3 · Sintático + AST** — gramática Lark LALR com declaração, atribuição, 8 níveis de precedência (mín. 4), `se`/`senao`, `enquanto` e 4 comandos do domínio (mín. 2); AST com `dataclass` desenhada por `mostrar_ast`; erro sintático com linha, coluna e o que era esperado.
- [x] **R4 · Semântico** — tabela de símbolos com escopo de bloco; 4 tipos (mín. 3) com coerção e erro de tipo; 5 regras de negócio (mín. 2, várias como erro); 4 tipos de aviso (mín. 1); galeria com 12 erros semânticos diferentes (mín. 6), todos passando pelo léxico e pelo sintático.
- [x] **R5 · TAC** — rótulos e desvios com condição invertida; instruções próprias `LANCAR`, `APROVAR`, `REPROVAR`, `AGENDAR_EXAME`; TAC mostrado para programa com laço e com `se`/`senao`.
- [x] **R6 · Otimização** — 6 técnicas (mín. 3); antes × depois e prova pela VM em 5 programas (mín. 2): mesma saída e menos instruções executadas.
- [x] **R7 · Execução** — VM executa o TAC; cada comando do domínio gera saída e alimenta o resumo; 8 erros de execução tratados (mín. 2): divisão por zero, laço infinito, aprovar com média < 6, nota real fora de 0..10, prazo inválido, disciplina sem nota, estouro da pilha de chamadas, função sem `retornar`.
- [x] **R8 · Testes automáticos** — 37 casos (mín. 12): 9 válidos (mín. 4) e erros de todas as fases (léxico, sintático, semântico, execução). Todos passando.
- [x] **R9 · Interface + README** — painel `ipywidgets` com os exemplos e 8 visões (mín. 3); README com integrantes, tema, ficha, tabela de tokens, gramática, como rodar, exemplos de saída e quem fez o quê.

### Bônus

- [x] **Código objeto (+0,5)** — Assembly com registradores e spilling.
- [x] **Funções (+0,5)** — `funcao`/`retornar` com parâmetros, recursão e pilha de chamadas na VM.
- [x] **"Você quis dizer...?" (+0,25)** — `difflib.get_close_matches` para variáveis e disciplinas.
- [x] **Integração contínua (+0,25)** — GitHub Actions roda o notebook e a bateria de testes a cada push.
- [x] **Saída visual (+0,25)** — gráfico das médias por disciplina gerado pela execução.

### Formato de entrega

- [x] Compilador em notebook `.ipynb` que roda do início ao fim.
- [x] Nomes de todos os integrantes no começo do README.
- [x] Repositório público no GitHub, entregue por um único membro do grupo.
