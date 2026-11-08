+++
id = "P-003"
title = "Patient context: anamnesis references previous consultations"
status = "done"
created = "2026-11-08"
priority = "medium"
+++

## Problema

Cada consulta é tratada como ilhazca: a anamnese não enxerga o
histórico do paciente. O médico redigita contexto que já existe,
e o produto não acumula o ativo que o diferencia — memória clínica
contínua por paciente.

## Proposta

Anamnese passa a referenciar consultas anteriores do mesmo paciente:

- **Link paciente ↔ consultas**: a consulta ganha `patient_id`
  estável; pacientes viram entidade própria (já existe plugin
  `patients`, verificar o que falta).
- **Resumo de contexto**: antes/durante a estruturação, o pipeline
  recebe um resumo das últimas N consultas (queixa, conduta,
  medicações citadas) — não o prontuário inteiro.
- **Seções "desde a última consulta"**: a anamnese destaca o que
  mudou (novos sintomas, retorno, adesão) em vez de reescrever
  tudo — o médico revisa delta, não volume.
- **Prompt discipline**: o LLM recebe histórico como *contexto
  factual*, nunca como sugestão clínica. Posicionamento
  documentation-only se mantém: o sistema registra e organiza,
  não decide.

## Escopo (MVP)

1. `patient_id` na consulta + vínculo paciente↔consultas
2. Endpoint/service que agrega resumo das últimas 3 consultas
3. Template de anamnese com bloco "histórico" preenchido
4. UI: badge "N consultas anteriores" + seção de contexto
   colapsável na revisão

## Fora de escopo (por ora)

- Timeline visual completa do paciente
- Alertas de retorno/vencimento (beira de decisão clínica)
- Compartilhamento de histórico entre médicos/clínicas

## Critérios de aceite

- Nova consulta de paciente recorrente mostra resumo das consultas
  anteriores na tela de revisão
- Anamnese gerada referencia fatos do histórico sem inventar
- Sem paciente vinculado, comportamento atual inalterado
- Testes: agregação de histórico, renderização do bloco, fallback
  sem histórico
