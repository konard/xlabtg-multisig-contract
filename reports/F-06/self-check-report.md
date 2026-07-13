# TON Bug Bounty Self-Check Report

## Short Assessment

DO NOT SEND!!!
Status: incorrect.
Confidence: 99.
Fit bug bounty confidence: 99.
Component: TON Multisig V1 (`multisig-code.fc`).
Class: out-of-scope known peculiarity.
Self-check report: `reports/F-06/self-check-report.md`.

## Repository State

- Analysis date UTC: 2026-07-13
- Input: `reports/F-06/report.md`
- Bug bounty rules repository: `https://github.com/ton-blockchain/bug-bounty`
- Bug bounty rules commit: `fc351b9a114922ef5dd704ab53711f583df18df5`
- Repositories analyzed:
  - `https://github.com/xlabtg/multisig-contract`, branch `issue-1-189ab3244a81`, pre-report commit `cf2eefb7b3b7c03919e9b90ce6286a3e4090f53c`, submodules not present
  - `https://github.com/ton-blockchain/ton`, branch `master`, commit `6308c96bc8c4e95d4d8c598a84d648534331e92b`, submodules updated recursively
- Local modification warnings: только отчётные файлы
- Fetch/build limitations: emulator boundary test не запускался; он не заменит отсутствующий security impact

## Scope Validation

- Target component: Multisig V1
- In scope: no
- Eligible category: no
- Redirect required: none
- Relevant exclusions or warnings: V1 прямо исключён; общие жалобы на особенности TON и поведение без реалистичного exploit path не принимаются

## Technical Finding Summary

Expiry и cleanup используют TVM `now()`. Исходный report не демонстрирует, что допустимая вариативность block timestamp нарушает какое-либо security property.

## Vulnerability Existence

- Exact files/functions: `multisig-code.fc`: `recv_external`
- Verified code path: `now() << 32` участвует в проверках query lifetime и cleanup
- Attacker-controlled input path: конкретный вредоносный путь отсутствует; block time является штатным consensus input
- Assumptions: order находится у точной временной границы; impact не задан
- Already fixed: not applicable; это ожидаемая семантика

## Reproducibility

- Reproduced: no
- Reproduction method: static
- Reproduction confidence: 99 для наличия `now()`, 0 для vulnerability impact
- Missing reproduction evidence: exploit scenario, violated invariant, measurable impact
- Live-target testing avoided: yes

## Bug Bounty Eligibility

- Technical validity: invalid
- Bounty eligibility: not eligible
- Realistic attacker prerequisites: no
- Security impact: отсутствует
- Low-priority notes: обычная expiry boundary semantics

## Severity and Claim Validation

- Claimed impact/severity: Informational
- Validated impact: none
- Overclaiming or downgrade notes: использование `now()` само по себе не CWE/security issue

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
- Proof of concept or evidence: missing как vulnerability evidence
- Security impact: missing
- Suggested remediation: present (не менять без exploit)
- Environment details: present

## Common Error Scan

Наличие времени блока ошибочно представлено как finding без unexpected behavior или impact. CWE-классификация для этого утверждения не обоснована.

## Final Verdict

Final verdict: REJECTED

Detailed reasoning: код использует стандартный clock именно по назначению; ни exploitability, ни ущерб не показаны, а target исключён.

Submission guidance: do not send.

Note: This self-check is not an official TON triage decision and does not guarantee a bounty. Invalid or low-quality reports may reduce reviewer trust and review priority.
