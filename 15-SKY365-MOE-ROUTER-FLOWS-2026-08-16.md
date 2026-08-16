# Sky365 / Colibri — MoE Router Flows

**التاريخ:** 2026-08-16  
**الحالة:** `PLANNED` architecture مبنية على مراجع `VERIFIED`، بلا Training run.

## 1. مسار التعلم والقرار

```mermaid
flowchart LR
    A["Switch-base-8<br/>فهم Top-1 وCapacity"] --> B["Qwen2.5 0.5B Chat<br/>MixLoRA Micro Lab"]
    B --> C["Qwen2.5 1.5B Chat<br/>Arabic Chat Pilot"]
    C --> D["OLMoE الموجود<br/>Native Router Comparison"]
    D --> E["Llama2 + MixLoRA Adapter<br/>Remote Reference Smoke"]
    E --> F["MK7 Router Contract<br/>Experts + Challenge"]
    F -. "Horizon فقط" .-> G["MoST / Kimi K3<br/>Advanced Architecture"]

    classDef learn fill:#e0f2fe,stroke:#0284c7,color:#082f49;
    classDef pilot fill:#ecfccb,stroke:#65a30d,color:#1a2e05;
    classDef gate fill:#fef3c7,stroke:#d97706,color:#451a03;
    classDef horizon fill:#f3e8ff,stroke:#9333ea,color:#3b0764;
    class A,D learn;
    class B,C pilot;
    class E,F gate;
    class G horizon;
```

## 2. Runtime Architecture المتفق عليها

```mermaid
flowchart TD
    U["User / Agent Input"] --> H["Request-level Harness Router"]
    H -->|"chat profile + session + tool stickiness"| T["Chat Template / Tokenizer"]
    T --> L0["Frozen Dense Chat Base"]

    subgraph BLOCK["كل Transformer Layer مفعّل بها MixLoRA"]
      X["Token Hidden State"] --> R["Per-layer Router"]
      R -->|"Top-k"| E1["LoRA Expert A"]
      R -->|"Top-k"| E2["LoRA Expert B"]
      R -->|"Top-k"| EN["LoRA Expert N"]
      R --> M["Shared / Fallback Path"]
      E1 --> C["Weighted Combine"]
      E2 --> C
      EN --> C
      M --> C
    end

    L0 --> X
    C --> O["Next Layer / LM Head"]
    O --> Y["Response"]
    R --> Z["Telemetry: logits, usage, entropy, overload"]
    H --> Z
    Y --> V["Validation + Challenge + Human Review"]
```

## 3. فصل Request Router عن Token Router

```mermaid
flowchart LR
    Q["طلب واحد"] --> RR["Request Router<br/>Macaron-style"]
    RR --> P1["Adapter/Profile: Knowledge"]
    RR --> P2["Adapter/Profile: Tools"]
    RR --> P3["Adapter/Profile: Coding"]

    P1 --> TR["Token Routers داخل الطبقات<br/>MixLoRA-style"]
    P2 --> TR
    P3 --> TR
    TR --> K["Top-k Experts لكل Token ولكل Layer"]
```

الـRequest Router يحدد سياق/بروفايل الطلب. الـToken Router يعمل داخل الموديل. يمكن جمعهما لاحقًا، لكن لا نعتبرهما Router واحدًا.

## 4. شجرة الأصول والمسارات

```mermaid
flowchart TD
    ROOT["Colibri AI Assets"] --> INF["F:\\Colibri-Models\\lmstudio<br/>GGUF Inference"]
    ROOT --> SRCF["F:\\AI-RESOURCES\\REFERENCES\\MOE<br/>Source References"]
    ROOT --> SRCQ["Q:\\Colibri\\research\\external<br/>Research Copies"]
    ROOT --> DOC["Q:\\Colibri\\docs\\session-map<br/>Canonical Decisions"]
    ROOT --> LMS["Q:\\Users\\mosta\\.lmstudio\\models<br/>Junction → F"]
    ROOT -. "لا نعتمد عليه حاليًا" .-> ARCH["H:\\ Archive<br/>UNRESOLVED"]

    INF --> GGUF["GGUF فقط"]
    SRCF --> CODE["Git Source + Commit"]
    DOC --> REG["Reports + Decision Registry"]
```

## 5. بوابات التنفيذ

```mermaid
flowchart TD
    S0["Research + Source Audit"] --> G0{"Source/License/Hash واضح؟"}
    G0 -->|"لا"| B0["BLOCKED / UNRESOLVED"]
    G0 -->|"نعم"| S1["Download Size Report"]
    S1 --> G1{"مساحة + موافقة مستقلة؟"}
    G1 -->|"لا"| B1["توقف بلا تنزيل"]
    G1 -->|"نعم"| S2["Smoke بلا تدريب"]
    S2 --> G2{"Reload + Router Telemetry صحيح؟"}
    G2 -->|"لا"| B2["PARTIAL: أصلح Runtime"]
    G2 -->|"نعم"| S3["Dataset Contract + Frozen Challenge"]
    S3 --> G3{"Owner Training Approval؟"}
    G3 -->|"لا"| B3["PLANNED فقط"]
    G3 -->|"نعم"| S4["Controlled Training"]
```

## 6. قرار البيز

```mermaid
flowchart TD
    START{"نحتاج Chat + Arabic + MixLoRA؟"}
    START -->|"نعم، أصغر تجربة"| Q05["Qwen2.5-0.5B-Instruct"]
    START -->|"نعم، Pilot أفضل"| Q15["Qwen2.5-1.5B-Instruct"]
    START -->|"Router mechanics فقط"| SW["Switch-base-8"]
    START -->|"Native MoE"| OL["OLMoE الموجود"]
    START -->|"Inference benchmark"| GLM["GLM-Edge-1.5B-Chat GGUF"]
    START -->|"K3-scale architecture"| K3["مرجع فقط — ليس تشغيلًا محليًا"]

    GLM -. "MoE-PEFT backend غير موجود" .-> DEV["Custom integration لاحقًا إن استحق"]
```

