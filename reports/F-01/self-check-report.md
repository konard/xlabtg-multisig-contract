# TON Bug Bounty Self-Check Report

## Short Assessment

DO NOT SEND!!!
Status: out-of-scope but technically valid.
Confidence: 98.
Fit bug bounty confidence: 99.
Component: TON Multisig V1 (`multisig-code.fc`).
Class: out-of-scope known peculiarity.
Self-check report: `reports/F-01/self-check-report.md`.

## Repository State

- Analysis date UTC: 2026-07-13
- Input: `reports/F-01/report.md`
- Bug bounty rules repository: `https://github.com/ton-blockchain/bug-bounty`
- Bug bounty rules commit: `fc351b9a114922ef5dd704ab53711f583df18df5`
- Repositories analyzed:
  - `https://github.com/xlabtg/multisig-contract`, branch `issue-1-189ab3244a81`, pre-report commit `cf2eefb7b3b7c03919e9b90ce6286a3e4090f53c`, submodules not present
  - `https://github.com/ton-blockchain/ton`, branch `master`, commit `6308c96bc8c4e95d4d8c598a84d648534331e92b`, submodules updated recursively
- Local modification warnings: рабочая ветка изменена только создаваемыми отчётами; это не upstream evidence
- Fetch/build limitations: `func`, `fift`, emulator и `acton` отсутствуют; build и исполняемый PoC не запускались

## Scope Validation

- Target component: Multisig V1
- In scope: no
- Eligible category: no
- Redirect required: none
- Relevant exclusions or warnings: официальный skill прямо исключает `crypto/smartcont/multisig-code.fc`; актуальные правила принимают Multisig V2, а остальные `crypto/smartcont` обычно считают examples. Поведение давно известно (`ton-blockchain/ton#168`).

## Technical Finding Summary

Один валидный владелец может создавать under-quorum orders, оплачиваемые балансом V1, и циклически тратить средства на gas/storage fees. Это griefing, не перевод средств атакующему.

## Vulnerability Existence

- Exact files/functions: `multisig-code.fc`: `recv_external`, `update_pending_queries`
- Verified code path: подпись одного owner проверяется до `set_gas_limit`; under-quorum order сохраняется и committed; README описывает оплату fee самим multisig
- Attacker-controlled input path: валидно подписанный внешний запрос владельца с уникальным `query_id`
- Assumptions: `k >= 2`, один owner key скомпрометирован/вредоносен, есть баланс; flood cap 10 и minimum lifetime один час ограничивают скорость
- Already fixed: no для этой ветки; risk смягчён, но не устранён. Multisig V2 использует другой design

## Reproducibility

- Reproduced: partially
- Reproduction method: static
- Reproduction confidence: 90
- Missing reproduction evidence: локальная balance/gas trace и executable PoC
- Live-target testing avoided: yes; правила запрещают state-changing checks на mainnet/testnet

## Bug Bounty Eligibility

- Technical validity: valid
- Bounty eligibility: not eligible
- Realistic attacker prerequisites: yes для security model с одним leaked owner key; однако текущие правила отдельно исключают issues, требующие leaked private information
- Security impact: постепенное расходование баланса на комиссии, без кражи и обхода кворума
- Low-priority notes: известный design risk с существующими rate mitigations

## Severity and Claim Validation

- Claimed impact/severity: Medium–High, fee drain/griefing
- Validated impact: ограниченный по скорости экономический griefing одним owner key
- Overclaiming or downgrade notes: не называть money stealing, unlimited instantaneous drain или quorum bypass

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
- Proof of concept or evidence: present (source evidence), executable PoC missing
- Security impact: present
- Suggested remediation: present
- Environment details: present

## Common Error Scan

Риск известен публично; первоначальная оценка могла смешивать fee griefing с кражей. Per-owner cap и one-hour rule обязательны в threat model. Component и leaked-key prerequisite исключены текущими bounty rules.

## Final Verdict

Final verdict: REJECTED

Detailed reasoning: поведение подтверждается исходниками и публичным issue, но Multisig V1 прямо исключён, leaked-key reports не принимаются, а finding является известной особенностью дизайна.

Submission guidance: do not send в TON bug bounty bot; сохранить как техническое предупреждение для пользователей V1.

Note: This self-check is not an official TON triage decision and does not guarantee a bounty. Invalid or low-quality reports may reduce reviewer trust and review priority.
