# TON Bug Bounty Self-Check Report

## Short Assessment

DO NOT SEND!!!
Status: partially correct.
Confidence: 95.
Fit bug bounty confidence: 99.
Component: TON Multisig V1 (`multisig-code.fc`).
Class: configuration-only issue.
Self-check report: `reports/F-03/self-check-report.md`.

## Repository State

- Analysis date UTC: 2026-07-13
- Input: `reports/F-03/report.md`
- Bug bounty rules repository: `https://github.com/ton-blockchain/bug-bounty`
- Bug bounty rules commit: `fc351b9a114922ef5dd704ab53711f583df18df5`
- Repositories analyzed:
  - `https://github.com/xlabtg/multisig-contract`, branch `issue-1-189ab3244a81`, pre-report commit `cf2eefb7b3b7c03919e9b90ce6286a3e4090f53c`, submodules not present
  - `https://github.com/ton-blockchain/ton`, branch `master`, commit `6308c96bc8c4e95d4d8c598a84d648534331e92b`, submodules updated recursively
- Local modification warnings: только отчётные файлы
- Fetch/build limitations: executable two-contract PoC не запускался; `func`/`fift`/emulator/`acton` отсутствуют

## Scope Validation

- Target component: Multisig V1
- In scope: no
- Eligible category: no
- Redirect required: none
- Relevant exclusions or warnings: V1 прямо исключён; configuration/operator-controlled issues и поведение без реалистичного impact path не принимаются

## Technical Finding Summary

Signed preimage не включает адрес контракта. При одинаковом `wallet_id`, общем owner key и byte-identical order co-signature криптографически переносима между двумя V1 instances.

## Vulnerability Existence

- Exact files/functions: `multisig-code.fc`: `check_signatures`, `recv_external`
- Verified code path: `hash = slice_hash(in_msg)` вычисляется над order data; contract address/owner-set hash отсутствует; `wallet_id` проверяется
- Attacker-controlled input path: зависит от получения реальной подписи и специально совпадающих deployment parameters/order bytes
- Assumptions: два оператора переиспользовали `wallet_id`, owner key и query/order; root envelope второго wallet также валиден; вред от одинакового order во втором wallet не показан
- Already fixed: no для V1; уникальный `wallet_id` является design domain separator

## Reproducibility

- Reproduced: partially
- Reproduction method: static
- Reproduction confidence: 80
- Missing reproduction evidence: две локальные instances, signature/preimage trace, transaction impact
- Live-target testing avoided: yes

## Bug Bounty Eligibility

- Technical validity: partially valid
- Bounty eligibility: not eligible
- Realistic attacker prerequisites: uncertain
- Security impact: подпись переносима в искусственно совпадающей конфигурации; quorum bypass или конкретная потеря средств не доказаны
- Low-priority notes: корректная уникализация `wallet_id` устраняет scenario

## Severity and Claim Validation

- Claimed impact/severity: Low, cross-wallet reuse
- Validated impact: cryptographic replay property без доказанного security consequence
- Overclaiming or downgrade notes: нельзя классифицировать как signature/auth bypass; порог `k` не обходится

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
- Proof of concept or evidence: present (static), executable PoC missing
- Security impact: present but uncertain
- Suggested remediation: present
- Environment details: present

## Common Error Scan

Impact зависит от operator-controlled duplicate `wallet_id` и shared keys. Нет демонстрации, что повтор одинакового order на втором wallet нарушает намерение с практическим ущербом. Нельзя улучшать exploit story сверх этих данных.

## Final Verdict

Final verdict: REJECTED

Detailed reasoning: cryptographic observation правдоподобно подтверждается кодом, но attacker-controlled practical impact не доказан, конфигурация искусственная, а V1 исключён.

Submission guidance: do not send; сначала нужен локальный two-wallet PoC и конкретный impact, но scope exclusion останется.

Note: This self-check is not an official TON triage decision and does not guarantee a bounty. Invalid or low-quality reports may reduce reviewer trust and review priority.
