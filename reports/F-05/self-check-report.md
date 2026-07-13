# TON Bug Bounty Self-Check Report

## Short Assessment

DO NOT SEND!!!
Status: incorrect.
Confidence: 99.
Fit bug bounty confidence: 99.
Component: TON Multisig V1 (`multisig-code.fc`).
Class: configuration-only issue.
Self-check report: `reports/F-05/self-check-report.md`.

## Repository State

- Analysis date UTC: 2026-07-13
- Input: `reports/F-05/report.md`
- Bug bounty rules repository: `https://github.com/ton-blockchain/bug-bounty`
- Bug bounty rules commit: `fc351b9a114922ef5dd704ab53711f583df18df5`
- Repositories analyzed:
  - `https://github.com/xlabtg/multisig-contract`, branch `issue-1-189ab3244a81`, pre-report commit `cf2eefb7b3b7c03919e9b90ce6286a3e4090f53c`, submodules not present
  - `https://github.com/ton-blockchain/ton`, branch `master`, commit `6308c96bc8c4e95d4d8c598a84d648534331e92b`, submodules updated recursively
- Local modification warnings: только отчётные файлы
- Fetch/build limitations: negative deployment test отсутствует; `func`/`fift`/emulator/`acton` не установлены

## Scope Validation

- Target component: Multisig V1
- In scope: no
- Eligible category: no
- Redirect required: none
- Relevant exclusions or warnings: V1 исключён; trusted operator/local configuration issues прямо не принимаются

## Technical Finding Summary

Некорректный custom state init может задать owner index `i >= n`; дальнейшая упаковка `1 << i` в `n` бит может завершиться ошибкой. Штатный script создаёт последовательные допустимые индексы.

## Vulnerability Existence

- Exact files/functions: `multisig-code.fc`: `create_init_state`, `recv_external`, `update_pending_queries`
- Verified code path: constructor не итерирует owner dictionary; vote bit сериализуется в `n` бит
- Attacker-controlled input path: отсутствует после deployment; trigger требует заранее некорректного trusted state init
- Assumptions: нестандартный deployer нарушил invariant и затем владелец этого индекса подписал order
- Already fixed: not applicable; стандартный deployment tooling предотвращает условие

## Reproducibility

- Reproduced: partially
- Reproduction method: static
- Reproduction confidence: 90
- Missing reproduction evidence: local serialization/exit trace
- Live-target testing avoided: yes

## Bug Bounty Eligibility

- Technical validity: invalid как remote vulnerability; deployment footgun правдоподобен
- Bounty eligibility: not eligible
- Realistic attacker prerequisites: no
- Security impact: self-inflicted misconfiguration конкретного wallet
- Low-priority notes: defense-in-depth validation

## Severity and Claim Validation

- Claimed impact/severity: Low / Informational
- Validated impact: локальная ошибка deployment configuration
- Overclaiming or downgrade notes: нельзя заявлять remote brick или attacker-controlled DoS

## Report Completeness Check

- Title: present
- Summary: present
- Affected component: present
- Affected commit: present
- Affected files/functions: present
- Attack prerequisites: present
- Trigger conditions: present
- Reproduction steps: present
- Expected result: present
- Actual result: present
- Proof of concept or evidence: present (static), executable test missing
- Security impact: present, но не bounty-eligible
- Suggested remediation: present
- Environment details: present

## Common Error Scan

Основная ошибка — отсутствие attacker control. Некорректный local state и custom deployment tooling являются доверенным вводом; штатный script поддерживает invariant.

## Final Verdict

Final verdict: REJECTED

Detailed reasoning: это defense-in-depth проверка для ошибочного deployment, не vulnerability корректно развёрнутого контракта; target также исключён.

Submission guidance: do not send; при желании оформить как hardening issue для tooling.

Note: This self-check is not an official TON triage decision and does not guarantee a bounty. Invalid or low-quality reports may reduce reviewer trust and review priority.
