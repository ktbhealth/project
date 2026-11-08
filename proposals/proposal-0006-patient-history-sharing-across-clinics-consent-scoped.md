+++
id = "P-006"
title = "Patient history sharing across clinics (consent-scoped)"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["v2"]
+++

## Ideia

Compartilhamento de histórico do paciente: paciente autoriza
(mesmo consentimento LGPD da captura) que outras clínicas/médicos
acessos suas consultas — o "prontuário portátil" da KTB.

## Superfície

- Escopo de consentimento por paciente (ler vs. escrever)
- Link/token de acesso com expiração
- Auditoria de quem acessou o quê
- Multi-tenant: dados de clínicas diferentes no mesmo paciente

## Cuidados

- LGPD: consentimento granular e revogável; minimização
- Nunca compartilhar dados entre médicos sem consent explícito
- Começar read-only; escrita compartilhada é fase posterior

## Depende de

Modelo paciente↔consultas maduro.
