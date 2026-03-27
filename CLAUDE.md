I want to create a set of basic orchestrator workflows for each "deployment channel".
I want the new orchestrator workflows created to be very simple with stubs for now as this is only to test out the environment workflow.
Each job can have a single step that just echos "Deploying into <environment>"

Environments (in order):
US-Dev-DB
US-Dev-App
EU-QA-DB
US-QA-DB
EU-QA-App
US-QA-App
US-PRD-DB
EU-PRD-DB
US-PRD-App
EU-PRD-App


- orchestrator-main.yml workflow pulls artifacts (stub) and deploys through US-Dev-DB -> US-Dev-Ap
- orchestrator-rc.yml workflow pulls artifacts and deploys to ((EU-QA-DB -> EU->QA-App) and (US-QA-DB -> US-QA-App)) -> PreRelease-PRD -> EU-PRD-DB -> EU-PRD-App -> US-PRD-DB -> US-PRD-App
    - EU->US in PRD is sequential intentionally, not parallel like in QA
- orchestrator-hotfix-rc.yml workflow in devops-demo repository pulls artifacts and deploys to ((EU-QA-DB -> EU->QA-App) and (US-QA-DB -> EU-QA-App)) -> PreRelease-PRD -> EU-PRD-DB -> EU-PRD-App -> US-PRD-DB -> US-PRD-App
    - NOTE: In this case *-QA-* environments are optional
    - EU->US in PRD is sequential intentionally, not parallel like in QA



