# TON Bug Bounty Self-Check Report

## Short Assessment

DO NOT SEND!!!
Status: out-of-scope but technically valid.
Confidence: 96.
Fit bug bounty confidence: 99.
Component: TON Multisig V1 (`multisig-code.fc`).
Class: out-of-scope known peculiarity.
Self-check report: `reports/F-02/self-check-report.md`.

## Repository State

- Analysis date UTC: 2026-07-13
- Input: `reports/F-02/report.md`
- Bug bounty rules repository: `https://github.com/ton-blockchain/bug-bounty`
- Bug bounty rules commit: `fc351b9a114922ef5dd704ab53711f583df18df5`
- Repositories analyzed:
  - `https://github.com/xlabtg/multisig-contract`, branch `issue-1-189ab3244a81`, pre-report commit `cf2eefb7b3b7c03919e9b90ce6286a3e4090f53c`, submodules not present
  - `https://github.com/ton-blockchain/ton`, branch `master`, commit `6308c96bc8c4e95d4d8c598a84d648534331e92b`, submodules updated recursively
- Local modification warnings: только создаваемые отчёты; не используются как upstream evidence
- Fetch/build limitations: нет `func`/`fift`/emulator/`acton`; количественная gas trace отсутствует

## Scope Validation

- Target component: Multisig V1
- In scope: no
- Eligible category: no
- Redirect required: none
- Relevant exclusions or warnings: `multisig-code.fc` прямо исключён официальным skill; актуальный README bounty перечисляет Multisig V2. Риск около 100 orders уже документирован README самого V1.

## Technical Finding Summary

Неограниченный глобально `pending_queries` может увеличить pre-acceptance стоимость до `out of gas credit` и блокировать валидные внешние запросы.

## Vulnerability Existence

- Exact files/functions: `multisig-code.fc`: `unpack_state`, `recv_external`; `README.md:9`
- Verified code path: state/dict/signature operations происходят до `set_gas_limit`; cleanup после acceptance; README предупреждает о примерно 100 live orders
- Attacker-controlled input path: только валидно подписанные owner requests; один owner одновременно ограничен десятью новыми orders
- Assumptions: для быстрого достижения ~100 нужны несколько owner keys либо временные циклы; точный gas threshold зависит от environment
- Already fixed: no для V1; documented operational limit существует

## Reproducibility

- Reproduced: partially
- Reproduction method: static
- Reproduction confidence: 85
- Missing reproduction evidence: исполняемый emulator test, gas measurements, exit trace и доказательство recovery duration
- Live-target testing avoided: yes; state-changing testing публичных contracts запрещено

## Bug Bounty Eligibility

- Technical validity: valid
- Bounty eligibility: not eligible
- Realistic attacker prerequisites: uncertain; owner control реалистичен в threat model, но достижение порога одним ключом ограничено
- Security impact: временная/потенциально длительная недоступность wallet; permanent brick не доказан
- Low-priority notes: известное и документированное ограничение deprecated/out-of-scope design

## Severity and Claim Validation

- Claimed impact/severity: Medium, wallet DoS около 100 orders
- Validated impact: plausible wallet DoS; точный порог и длительность не измерены
- Overclaiming or downgrade notes: не заявлять «cheaply by one key», «permanent» или точный порог без локальной trace

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
- Proof of concept or evidence: present (source/README), executable PoC missing
- Security impact: present
- Suggested remediation: present
- Environment details: present

## Common Error Scan

Отсутствует quantitative reproduction. Первоначальная формулировка переоценивала возможность одного owner быстро накопить ~100 записей и могла называть отказ перманентным. Риск документирован самим проектом и target исключён.

## Final Verdict

Final verdict: REJECTED

Detailed reasoning: source path и README подтверждают технический риск, но доказательная база не фиксирует точный threshold/impact, а Multisig V1 прямо не допускается текущим bounty.

Submission guidance: do not send; использовать как эксплуатационное предупреждение V1 и при необходимости сначала добавить локальный gas-trace PoC.

Note: This self-check is not an official TON triage decision and does not guarantee a bounty. Invalid or low-quality reports may reduce reviewer trust and review priority.
