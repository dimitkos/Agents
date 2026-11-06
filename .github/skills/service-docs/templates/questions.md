# Questions for the previous team — <Service name>

Things the code doesn't answer. Go through these in the handover sessions, starting with **High**.
Write the answer under each question and, if it changes the docs, update the page.

| # | Priority | Area | Question | Why it matters | Found in |
|---|----------|------|----------|----------------|----------|
| 1 | High | Operations | Who monitors the dead-letter queue of `purchase-orders/landed-cost`, and what is done with those messages? | Failed landed-cost calculations would be lost silently | `src/Functions/LandedCostFunction.cs:24` |
| 2 | <…> | <Business / Architecture / Operations / Security / Deployment / Data / Testing> | <…> | <…> | <file:line> |

## Answers

### 1. <short title>

> <answer, who answered, date>
