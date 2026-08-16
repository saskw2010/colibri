# Sky365 / Colibri — الخلاصة المرجعية وخطة MoE Router

**التاريخ:** 2026-08-16  
**النطاق:** Router + Mixture-of-LoRA Experts + Native MoE + Harness  
**الحالة العامة:** `PARTIAL` — اكتملت مرحلة بحث وتجميع كود ومخزون؛ لم يبدأ تدريب ولم تُنزّل أوزان كبيرة جديدة.

## القرار التنفيذي

نعتمد بنية **Dense Chat Base مجمّد + عدة LoRA Experts + Router قابل للتدريب** كأول مسار عملي. هذا ليس تناقضًا: في MixLoRA يكون الـBase كثيفًا، بينما الـMoE مصنوع من Adapters صغيرة موزعة داخل طبقات الـFFN ويختار Router عددًا قليلًا منها لكل Token في كل طبقة.

الاختيارات المرتبة:

1. **Router Lab:** `Qwen/Qwen2.5-0.5B-Instruct` — أصغر Chat Base مناسب، عربي ومتعدد اللغات، و`model_type=qwen2` مدعوم في نسخة MoE-PEFT الموجودة لدينا. `PLANNED`.
2. **Chat Pilot:** `Qwen/Qwen2.5-1.5B-Instruct` — المرشح الأساسي بعد نجاح المختبر الصغير؛ 1.54B ومعرفة محادثة وعربية أفضل من TinyLlama. `PLANNED`.
3. **Native Router Reference:** `OLMoE-1B-7B` الموجود محليًا — لفهم Router أصلي وخبراء FFN حقيقيين، بلا إعادة تنزيل. `VERIFIED` مرجع.
4. **Micro Router Mechanics:** `google/switch-base-8` — لتعلم Top-1، capacity، load balancing وrouter logits، وليس Chat Base للمنتج. `PLANNED`.
5. **GLM-Edge-1.5B-Chat:** خفيف ومفيد كـInference Benchmark، لكنه **ليس Base البداية لـMixLoRA** لأن MoE-PEFT الحالي يدعم `chatglm` وGLM4 القديم، بينما GLM-Edge يعرّف نفسه `model_type=glm` وبنية مختلفة. يحتاج Backend Integration جديد. `PARTIAL`.
6. **Kimi K3 وMoST:** مراجع معمارية متقدمة فقط في هذه المرحلة. `PARTIAL` / `BLOCKED` للتجربة الكاملة.

## هل الـDense Base يصلح لـAdapter Mixture of Experts؟

نعم، وهو تصميم MixLoRA الأصلي:

```text
Frozen Dense Chat Base
        +
LoRA Expert 0..N داخل FFN في طبقات مختارة
        +
Router مستقل في كل طبقة يختار Top-k لكل Token
        +
Shared/Fallback behavior + telemetry
```

التصحيح المهم: الـToken لا «يفتش في كل Adapters مرة واحدة» على مستوى الموديل كله. في MixLoRA يمر عبر طبقات الـTransformer، وفي كل طبقة مفعّل بها MoE يحسب Router محلي درجات الخبراء ثم يختار `Top-k`. وقد يختار نفس الـToken خبراء مختلفين في طبقات مختلفة.

## قرار GLM-Edge مقابل Qwen Chat

| البند | GLM-Edge-1.5B-Chat | Qwen2.5-0.5B-Instruct | Qwen2.5-1.5B-Instruct |
|---|---:|---:|---:|
| النوع | Dense Chat | Dense Chat | Dense Chat |
| الحجم الكامل | 1.593B parameter؛ Safetensors 3,186,882,328 bytes | 0.49B | 1.54B؛ Safetensors نحو 3.09 GB |
| الموجود محليًا | GGUF Q4_K_M بحجم 980,470,144 bytes | غير مفحوص/غير منزّل | غير مفحوص/غير منزّل |
| العربية | غير مثبتة لدينا باختبار قبول | مذكورة رسميًا ضمن +29 لغة | مذكورة رسميًا ضمن +29 لغة |
| MoE-PEFT الحالي | غير مدعوم مباشرة (`glm`) | مدعوم عبر `qwen2` | مدعوم عبر `qwen2` |
| القرار | Benchmark استدلال | مختبر Router أول | Chat Pilot بعد المختبر |
| الحالة | `PARTIAL` | `PLANNED` | `PLANNED` |

**الخلاصة:** GLM-Edge ليس ثقيلًا للاستدلال، لكنه مكلف هندسيًا لمسار MixLoRA الحالي. Qwen2.5 هو الطريق الأقصر والأكثر فائدة للعربية. TinyLlama يبقى Fallback لتشغيل مثال MoE-PEFT كما هو، لا اختيارنا الأول للمحادثة العربية.

## الاستركشر المتفق عليه

### 1. طبقة الأصول

نفصل دائمًا بين:

- **GGUF:** استدلال داخل LM Studio فقط؛ ليس وزن تدريب MixLoRA.
- **Safetensors Base:** البيز الكامل الذي تتصل به الـAdapters أثناء التدريب/التحميل.
- **Adapter Weights:** LoRA Experts + Router weights؛ مرتبطة ببنية ونسخة Base محددة.
- **Source Code:** MixLoRA، MoE-PEFT، MoST، MoE-LoRA، MoLA، mLoRA، K3.
- **Datasets:** مستقلة بعقود provenance وTrain/Validation/Test/Challenge.
- **Runtime/Harness:** توجيه طلبات، state، tools، telemetry؛ منفصل عن Token Router.

### 2. طبقة الموديل

```text
Input
  → Chat Template / Tokenizer
  → Dense Base مجمّد
  → Per-layer MixLoRA Router
  → Top-k LoRA Experts
  → Shared/Fallback path
  → Output
```

### 3. طبقة الروترات

لدينا مستويان لا يجب خلطهما:

| المستوى | ما الذي يختاره؟ | المرجع |
|---|---|---|
| Request/Session Router | موديل أو Adapter Profile واحد للطلب/الأداة/الجلسة | Macaron / LLMRouter |
| Token/Layer Router | LoRA Experts أو Native Experts لكل Token داخل كل طبقة | MixLoRA / Switch / OLMoE / MoST |

### 4. عقد الـExpert

كل Expert مستقبلي يجب أن يمتلك:

- `expert_id` ثابتًا واسمًا وظيفيًا لا دعائيًا.
- Base ID + revision/hash متوافقًا.
- Dataset manifest + provenance + license.
- target modules وrank/alpha وعدد الخبراء وTop-k.
- Validation وChallenge مستقلين.
- Router telemetry: التوزيع، entropy، overload، fallback، والـper-layer selection.

لا نعتبر Expert متخصصًا في ERP أو العربية لمجرد تسميته؛ التخصص يجب أن يثبت من الداتا والاختبار.

## المراجع والقرار لكل موديول

| الموديول/الموديل | ما نتعلمه | الموجود | القرار | الحالة |
|---|---|---|---|---|
| MixLoRA | Dense Base + LoRA experts + Top-k + aux loss | Source على Q وF؛ Adapter Llama2 لم يُنزّل | المرجع الأساسي لـAdapter MoE | `VERIFIED` source / `PARTIAL` run |
| MoE-PEFT | Runtime تدريب/تقييم/استدلال متعدد PEFT | `F:\AI-RESOURCES\REFERENCES\MOE\MoE-PEFT` @ `40f8cb7` | Runtime الأساسي للتجربة | `VERIFIED` source |
| Qwen2.5-0.5B-Instruct | أصغر Chat dense متوافق وعربي | ليس ضمن المخزون المثبت | Base مختبر أول بعد بوابة تنزيل | `PLANNED` |
| Qwen2.5-1.5B-Instruct | Chat Pilot عربي أقوى | ليس ضمن المخزون المثبت | Base المرحلة الثانية | `PLANNED` |
| GLM-Edge-1.5B-Chat | Chat edge خفيف | GGUF Q4 محلي 0.91 GiB | Benchmark فقط؛ لا MixLoRA الآن | `PARTIAL` |
| TinyLlama 1.1B Chat | أقصر مثال جاهز في MoE-PEFT | Source recipe فقط | Fallback تقني | `PARTIAL` |
| Switch-base-8 | Top-1/capacity/router losses | لم يُنزّل | Micro-lab تعليمي | `PLANNED` |
| OLMoE-1B-7B | Native sparse experts وrouter | أوزان سابقة موجودة؛ ممنوع إعادة التنزيل | Native Router reference | `VERIFIED` asset |
| MoST | modality masks + 64 routed + 2 shared experts | Source @ `9c13577`؛ checkpoints/data غير منشورة | مرجع Modality Router | `VERIFIED` source / `BLOCKED` full run |
| MoLA | Layer-wise expert allocation | Source @ `01785c8` | مرجع مقارنة | `VERIFIED` source |
| MoE-LoRA | Training entry points أبسط | Source @ `8b1ef25` | مرجع مساعد | `VERIFIED` source |
| mLoRA | إدارة عدة LoRA بكفاءة | Source @ `89aa53f` | مرجع بنية Adapter factory | `VERIFIED` source |
| Macaron Harness | request routing + state + tool stickiness | Source على Q | Harness reference فقط | `VERIFIED` source |
| Router-R1 | router data/reward/RL/eval | Source على Q | مرحلة متأخرة بعد Router v0 | `VERIFIED` source |
| LLMRouter | request/model routing | clone غير مكتمل | لا نعتمد عليه الآن | `BLOCKED` |
| Kimi K3 | MoE ضخم وshared/routed experts | Source مرجعي @ `3cb39df` بلا تدريب عملي محلي | Architecture horizon فقط | `PARTIAL` |
| Llama-2-7B MixLoRA Adapter | أعلى مثال Adapter منشور مطابق للكود | Hub adapter ~0.758 GiB؛ Base gated ~12.55 GiB | Remote smoke فقط بموافقتين منفصلتين | `PARTIAL` / `BLOCKED` محليًا |
| Golden Gemma / MK7 Dataset | المسار المحمي الحالي | موجود خارج هذا البحث | لا لمس ولا خلط مع Adapter Llama2 | `VERIFIED` guardrail |

## ما تم فعله

- `VERIFIED` قراءة Handovers ونتائج MixLoRA وRouter/Dataset/Model lanes.
- `VERIFIED` تجميع Source repositories على F مع commits: K3 `3cb39df`، MixLoRA `d59d09b`، mLoRA `89aa53f`، MoE-LoRA `8b1ef25`، MoE-PEFT `40f8cb7`، MoLA `01785c8`، MoST `9c13577`.
- `VERIFIED` وجود مصادر إضافية على `Q:\Colibri\research\external`، منها MixLoRA وMoE-LoRA وRouter-R1 وMacaron.
- `VERIFIED` توحيد رؤية LM Studio عبر Junction: `Q:\Users\mosta\.lmstudio\models` → `F:\Colibri-Models\lmstudio`.
- `VERIFIED` GLM-Edge GGUF المحلي: `F:\Colibri-Models\lmstudio\zai-org\glm-edge-1.5b-chat-gguf\ggml-model-Q4_K_M.gguf`، 980,470,144 bytes.
- `VERIFIED` فحص GLM-Edge الرسمي: 28 layer، hidden 2048، 1.593B parameter، 8K config context، Safetensors 3,186,882,328 bytes.
- `VERIFIED` أن نسخة MoE-PEFT الحالية تربط `chatglm` فقط ولا تربط `glm`، لذلك GLM-Edge ليس Plug-and-Play.
- `VERIFIED` مراجعة نقدية لمقترح K3: صالح كفكرة عامة، لكن 2×3090 وحقن 8 خبراء بالطريقة المعروضة ليس Recipe صحيحًا ولا قابلًا للتنفيذ.
- `VERIFIED` فصل Request Router عن Token Router وفصل GGUF عن Safetensors عن Source.
- `VERIFIED` لم يبدأ تدريب، ولم تُنزّل Dataset ضخمة، ولم تُمس Golden/Gemma/MK7 Dataset، ولم يُعاد تنزيل OLMoE.

## حالة التخزين الحالية

لقطة الفحص في 2026-08-16:

| المسار | الدور | الحالة |
|---|---|---|
| `F:\Colibri-Models\lmstudio` | Active inference models | `VERIFIED`؛ F متاح وبنحو 546.70 GiB free |
| `F:\AI-RESOURCES\REFERENCES\MOE` | Source references | `VERIFIED` |
| `Q:\Colibri\docs\session-map` | Canonical docs | `VERIFIED` |
| `Q:\Colibri\research\external` | Research source copies | `VERIFIED` |
| `Q:\Users\mosta\.lmstudio\models` | Junction فقط | `VERIFIED` → F |
| `H:` | Archive محتمل | `UNRESOLVED`؛ لم يظهر في آخر `Get-PSDrive`، فلا نعتمد عليه في أي خطوة |

## الخطة القادمة

### المرحلة 0 — توثيق واعتماد القرار

- اعتماد هذه الحزمة كسجل قرار.
- لا تنزيل ولا تدريب في هذه المرحلة.

### المرحلة 1 — Router Mechanics بلا تدريب منتج

- فحص/تنزيل `switch-base-8` فقط بعد تقرير الحجم والموافقة.
- Inference قصير يسجل router logits، Top-1، capacity، dropped/overflow والـload balance.
- Gate: نفهم القياسات ونستطيع إعادة إنتاجها.

### المرحلة 2 — Adapter-MoE Micro Lab

- تقرير تنزيل منفصل لـ`Qwen2.5-0.5B-Instruct` Safetensors؛ لا GGUF للتدريب.
- تشغيل اختبارات MoE-PEFT dummy/config فقط أولًا.
- إنشاء 2–4 Experts اصطناعية صغيرة وRouter Top-1/Top-2 على Dataset مصغرة منفصلة تمامًا عن MK7.
- Gate: reload صحيح، expert usage غير منهار، fallback معروف.

### المرحلة 3 — Chat Pilot

- الانتقال إلى `Qwen2.5-1.5B-Instruct` بعد نجاح 0.5B.
- اختبار عربي/إنجليزي وJSON/tool schema، ثم تحديد هل الجودة تكفي.
- P2000 للاختبارات المحدودة فقط؛ التدريب الجاد على NVIDIA Remote 16–24GB+.

### المرحلة 4 — Native MoE Comparison

- استخدام OLMoE الموجود فقط لاستخراج router telemetry ومقارنة per-layer routing.
- لا إعادة تنزيل ولا fine-tuning قبل بوابة موارد مستقلة.

### المرحلة 5 — أعلى مثال مرجعي كامل

- Adapter `TUDB-Labs/alpaca-mixlora-7b` تنزيل مستقل صغير بعد موافقة.
- Base `Llama-2-7B` له تقرير حجم وترخيص وموافقة منفصلة، ويفضل Remote NVIDIA 24GB.
- Smoke inference فقط؛ لا تدريب.

### المرحلة 6 — تصميم MK7 بلا لمس البيانات

- Expert Registry + Router Contract + deterministic Router v0.
- تعريف labels/fallback/Unknown/telemetry/challenge.
- لا نستخدم Adapter Llama2 مع Gemma ولا نعدل Golden Dataset.

### المرحلة 7 — بوابة التدريب

لا يبدأ أي تدريب قبل: Dataset contract، hashes، license، hardware plan، baseline، rollback، وقرار Owner صريح.

## توزيع الأجهزة

| الجهاز | المناسب | غير المناسب |
|---|---|---|
| P2000 4GB | كود، unit tests، GGUF inference، وربما Switch/Qwen 0.5B smoke محدود | MixLoRA 7B، تدريب Chat Pilot جاد، K3 |
| Remote NVIDIA 16–24GB | Qwen 0.5/1.5B pilot، MixLoRA صغير، Llama2-7B smoke 24GB | K3 الكامل |
| Datacenter multi-GPU | K3-like native MoE research | ليس مطلوبًا لمسارنا الحالي |

## المصادر الأساسية

- [MixLoRA](https://github.com/TUDB-Labs/MixLoRA)
- [MoE-PEFT](https://github.com/TUDB-Labs/MoE-PEFT)
- [Qwen2.5-0.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct)
- [Qwen2.5-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct)
- [GLM-Edge official](https://github.com/zai-org/GLM-Edge)
- [GLM-Edge-1.5B-Chat](https://huggingface.co/zai-org/glm-edge-1.5b-chat)
- [Switch Transformers](https://huggingface.co/docs/transformers/model_doc/switch_transformers)
- [OLMoE](https://huggingface.co/allenai/OLMoE-1B-7B-0924)
- [MoST](https://github.com/ictnlp/MoST)
- [Kimi K3](https://github.com/MoonshotAI/Kimi-K3)

