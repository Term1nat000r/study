# Архитектура: Мультиагентная система генерации статей

## Граф агентов (LangGraph)

```mermaid
graph TD
    START(["▶ Вход<br/>topic + language + channel"]) --> R

    R["🔍 Исследователь<br/><i>researcher.py</i><br/>DuckDuckGo поиск → факты"]
    R -->|"research"| P

    P["📋 Планировщик<br/><i>planner.py</i><br/>План + SEO-ключевики"]
    P -->|"plan"| W

    W["✍️ Автор<br/><i>writer.py</i><br/>Полный текст в Markdown"]
    W -->|"draft"| E

    E["🔎 Редактор<br/><i>editor.py</i><br/>Проверка качества"]

    E -->|"VERDICT: approved"| F
    E -->|"VERDICT: needs_revision<br/>(макс 2 раза)"| W

    F["🎨 Форматтер<br/><i>formatter.py</i><br/>Заголовок + теги + финал"]
    F --> DONE(["✅ Выход<br/>output/*.md с YAML frontmatter"])

    style START fill:#4a9eff,color:#fff,stroke:none
    style DONE fill:#22c55e,color:#fff,stroke:none
    style R fill:#f59e0b,color:#fff,stroke:none
    style P fill:#8b5cf6,color:#fff,stroke:none
    style W fill:#3b82f6,color:#fff,stroke:none
    style E fill:#ef4444,color:#fff,stroke:none
    style F fill:#06b6d4,color:#fff,stroke:none
```

## Структура проекта

```mermaid
graph LR
    subgraph "📁 multiagent-blog/"
        M["main.py<br/><i>точка входа, CLI</i>"]
        G["graph.py<br/><i>StateGraph + conditional edges</i>"]
        S["state.py<br/><i>BlogState TypedDict</i>"]
        C["config.py<br/><i>LLM + утилиты</i>"]

        subgraph "📁 agents/"
            A1["researcher.py"]
            A2["planner.py"]
            A3["writer.py"]
            A4["editor.py"]
            A5["formatter.py"]
        end

        subgraph "📁 prompts/"
            P1["researcher.py<br/><i>ru + en</i>"]
            P2["planner.py"]
            P3["writer.py"]
            P4["editor.py"]
            P5["formatter.py"]
        end

        subgraph "📁 tools/"
            T1["search.py<br/><i>DuckDuckGo</i>"]
        end
    end

    M --> G
    G --> S
    G --> A1 & A2 & A3 & A4 & A5
    A1 --> P1 & T1
    A2 --> P2
    A3 --> P3
    A4 --> P4
    A5 --> P5
    A1 & A2 & A3 & A4 & A5 --> C

    style M fill:#4a9eff,color:#fff,stroke:none
    style G fill:#22c55e,color:#fff,stroke:none
    style S fill:#8b5cf6,color:#fff,stroke:none
    style C fill:#f59e0b,color:#fff,stroke:none
```

## Поток данных (State)

```mermaid
flowchart LR
    subgraph BlogState
        direction TB
        IN["topic · language · channel"]
        RES["research"]
        PLN["plan"]
        DRF["draft"]
        ED["editor_verdict · editor_feedback · revision_count"]
        OUT["final_article · zen_title · zen_description · zen_tags"]
    end

    R["Исследователь"] -->|заполняет| RES
    P["Планировщик"] -->|заполняет| PLN
    W["Автор"] -->|заполняет| DRF
    E["Редактор"] -->|заполняет| ED
    F["Форматтер"] -->|заполняет| OUT

    style IN fill:#e2e8f0,stroke:#94a3b8
    style OUT fill:#dcfce7,stroke:#86efac
```

## Зачем LangGraph?

LangGraph решает одну конкретную задачу: **оркестрация цепочки LLM-вызовов с ветвлениями и циклами**.

Без LangGraph пришлось бы писать:
```python
research = call_researcher(topic)
plan = call_planner(research)
draft = call_writer(plan, research)
feedback = call_editor(draft)
if feedback == "bad" and count < 2:
    draft = call_writer(plan, research, feedback)  # а если опять bad?
    feedback = call_editor(draft)                   # вложенные if...
```

С LangGraph это **декларативный граф**: добавил узлы, рёбра, conditional edge — и `app.stream()` сам гоняет данные по конвейеру, включая возврат автору на ревизию.
