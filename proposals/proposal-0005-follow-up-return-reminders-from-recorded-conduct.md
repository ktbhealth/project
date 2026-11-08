+++
id = "P-005"
title = "Follow-up return reminders from recorded conduct"
status = "open"
created = "2026-11-08"
priority = "medium"
+++

## Ideia

Alertas de retorno: "paciente X deveria voltar" baseado em
conduta registrada (ex.: "retornar em 30 dias", "rever exame").

## Tensão de posicionamento

Beira de decisão clínica: o sistema estaria sugerindo conduta de
retorno. Versão segura: **o médico registra a data de retorno na
conduta** (já é documentação), e o app apenas *lembra* — lista de
retornos do dia, sem inferir nem sugerir datas que o médico não
escreveu.

## Escopo provável

- Campo estruturado "retorno em" na conduta da anamnese
- Dashboard/lista de retornos pendentes
- Sem NLP inferindo prazo a partir do texto livre (fase 2, se houver)

## Depende de

T-009 (contexto de paciente estável).
