+++
id = "P-004"
title = "Patient timeline UI: full consultation history view"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["v2"]
+++

## Ideia

Visão cronológica completa do paciente: todas as consultas como
linha do tempo visual (data, queixa, conduta, documentos), não só
o resumo das últimas 3 visitas.

## Escopo provável

- Lista/página por paciente com todas as consultas ordenadas
- Cada entrada: data, anamnese resumida, link para a consulta
- Filtros por período/tipo se fizer sentido

## Depende de

Modelo de paciente estável (patient_id + agregação de histórico).

## Cuidados

- Volume: pacientes com muitas consultas precisam paginação/agregação
- Posicionamento: é visualização de registro, não decisão —
  sem badges de "risco" ou "atrasado"
