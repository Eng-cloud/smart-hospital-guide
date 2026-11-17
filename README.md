# Smart Hospital Guide | مرشد المستشفى الذكي

**English** · [العربية](#arabic)

---

## English

A bilingual (Arabic/English) **hospital triage agent**. The patient describes a complaint, and the agent:

1. routes them to the right hospital **department** (it never diagnoses),
2. flags **urgent** cases,
3. lists the **3 nearest hospitals or clinics/health centers** from OpenStreetMap data, with a Google Maps link for each.

> Sample student project. Not medical advice. In an emergency, go to the nearest emergency department.

Built as the **Day 2 homework** of the atomcamp Arabia × SDA Agentic AI Bootcamp: prompt engineering + a LangChain agent with tools and structured output + a Gradio interface.

### How it works

```
User (Arabic/English) → Gradio UI → ask()
                                      │
                         LangChain agent (Groq model)
                          ├─ tool 1: triage_lookup            → department + urgent flag (keyword rules)
                          └─ tool 2: find_nearest_facilities  → nearest hospitals/clinics (OpenStreetMap)
                                      │
                    Pydantic parser → Result(department, urgent, reason, next_steps)
                                      │
                       Gradio result screen (answer + facility list)
```

The model decides **when** to call each tool. Facility details shown to the user come from the tool output, not from the model's text.

### Code walkthrough (`app.py`)

The file is one script, in this order:

| Section | What it does |
|---|---|
| **Model** | `ChatOpenAI` pointed at Groq's OpenAI-compatible endpoint, model `openai/gpt-oss-120b`. The key is read from the `GROQ_API_KEY` environment variable. |
| **System prompt** (`SYSTEM_PROMPT`) | Engineered prompt with clear sections: Role, Audience, Context, Task and tools, Reasoning, Constraints, Output format. It tells the model not to diagnose, not to suggest medicines, not to invent phone numbers, hours or crowd levels, and to answer in the requested language. |
| **Rules** (`KEYWORDS`, `find_department`) | Keyword rules (Arabic and English) that map a complaint to a department: Emergency, Dermatology, Dental, Ophthalmology, Orthopedics, General Practice. Urgent keywords (e.g. chest pain, difficulty breathing, heavy bleeding, unconscious) always give Emergency. These are sample rules for a student project, not official medical guidance. |
| **Tool 1** (`triage_lookup`) | A LangChain `@tool` that returns the rules' result as JSON. |
| **Tool 2** (`find_nearest_facilities`) | A LangChain `@tool` that finds nearby hospitals or clinics. See "Facility lookup" below. |
| **Structured output** (`Result`, `parser`) | A Pydantic model and `PydanticOutputParser`, so the answer always has the same fields. |
| **Agent** (`create_agent`, `ask`) | Builds the agent with both tools and runs it. `ask()` retries once if the JSON cannot be parsed, takes facility data from the tool's output, and has a safety net: if a location was given but the model skipped the tool or asked for the wrong type, the lookup runs directly. |
| **Interface** (Gradio) | Three screens: language, complaint (plus optional location and facility type), result. All texts live in the `UI` dictionary for both languages. |

**Safety rules run in code, not only in the prompt:** if the rules or the model mark a case as urgent, the app shows the urgent banner and the Emergency department, whatever else the model wrote.

### Facility lookup (`find_nearest_facilities`)

1. **Geocoding:** the place name becomes coordinates through Nominatim, with Open-Meteo as a backup.
2. **Search:** an Overpass query for OpenStreetMap objects tagged as hospitals (`amenity=hospital`, `healthcare=hospital`) or clinics/health centers (`amenity=clinic`, `healthcare=clinic`, `healthcare=centre`). The radius starts small and grows: hospitals 15 km then 50 km, clinics 5, 15 then 40 km.
3. **Public servers can be slow or down**, so three Overpass servers are tried in a staggered race (a new one starts if the previous has not answered in 3 seconds). If all fail, a Nominatim search is used as a fallback.
4. **Speed:** the search starts in the background as soon as the user submits (`prefetch_facilities`). Successful results are cached, and a failed lookup is remembered for 20 seconds.
5. **Results:** the 3 nearest, one entry per name, with distance, a Google Maps link, and opening hours/phone only if OpenStreetMap has them.
6. **If everything fails**, the app shows the technical reason instead of an empty list.

### Interface behavior

- Language choice changes every text, the examples, the form labels and the answer language.
- A loading card appears immediately; a 120-second watchdog makes sure it never stays forever, and errors are shown as a card.
- Arabic is shown right-to-left.

### Files

| File | Purpose |
|---|---|
| `app.py` | The whole project in one script. |
| `requirements.txt` | Python dependencies. |
| `Day2_Homework_Starter_2_(2).ipynb` | The Colab notebook used to build it. |

### Run it yourself

```bash
pip install -r requirements.txt
export GROQ_API_KEY="your-key"     # Windows PowerShell: $env:GROQ_API_KEY="your-key"
python app.py
```

Then open `http://127.0.0.1:7860`. Never put your key inside the code or commit it to GitHub.

### Limitations

- It routes to a department. It does not diagnose, prescribe, or replace a doctor.
- The rules cover a small sample of complaints; anything else goes to General Practice.
- Facility data comes from OpenStreetMap and can be incomplete or outdated.
- Public map servers can be slow or unavailable.
- No crowd levels or waiting times are shown, because no verified data source is used for them.

### Data credit

Map data © OpenStreetMap contributors (ODbL). See https://www.openstreetmap.org/copyright

### Credits

Built by **Abdullah Abdali** as the Day 2 homework of the atomcamp Arabia × SDA Agentic AI Bootcamp (trainer: Razan Mohammed Otayn).

---

<a id="arabic"></a>

<div dir="rtl">

## العربية

أيجنت **ثنائي اللغة** (عربي/إنجليزي) لتوجيه المرضى داخل المستشفى. يكتب المريض شكواه، والأيجنت:

1. يوجّهه إلى **القسم** المناسب في المستشفى (بدون تشخيص).
2. ينبّه إذا كانت الحالة **عاجلة**.
3. يعرض **أقرب 3 مستشفيات أو مستوصفات/مراكز صحية** من بيانات OpenStreetMap، مع رابط خرائط Google لكل مرفق.

> مشروع تعليمي، وليس نصيحة طبية. في الحالات الطارئة توجّه إلى أقرب طوارئ.

بُني كواجب **اليوم الثاني** من معسكر atomcamp Arabia × SDA لـ Agentic AI: هندسة الـ Prompt + أيجنت LangChain فيه أدوات ومخرجات منظمة + واجهة Gradio.

### كيف يعمل

المستخدم يختار اللغة ويكتب شكواه (وموقعه إذا أراد) في واجهة Gradio، فتُرسل الشكوى إلى الدالة `ask()` التي تشغّل أيجنت LangChain بنموذج من Groq. الأيجنت يقرر **متى** يستدعي كل أداة:

- **الأداة 1 `triage_lookup`:** تحدد القسم وهل الحالة عاجلة، بقواعد كلمات مفتاحية.
- **الأداة 2 `find_nearest_facilities`:** تبحث عن أقرب المستشفيات أو المستوصفات من OpenStreetMap.

بعدها تُحوَّل الإجابة إلى نموذج Pydantic اسمه `Result` فيه: `department` و`urgent` و`reason` و`next_steps`، وتُعرض في شاشة النتيجة. بيانات المرافق اللي تظهر للمستخدم تؤخذ من مخرجات الأداة، مو من نص النموذج.

### شرح الكود (`app.py`)

الملف سكربت واحد بهذا الترتيب:

| القسم | وظيفته |
|---|---|
| **النموذج** | `ChatOpenAI` موجّه إلى رابط Groq المتوافق مع OpenAI، والموديل `openai/gpt-oss-120b`. المفتاح يُقرأ من متغير البيئة `GROQ_API_KEY`. |
| **الـ System prompt** (`SYSTEM_PROMPT`) | Prompt مهندس من عناصر: الدور، الجمهور، السياق، المهمة والأدوات، التفكير، القيود، شكل المخرجات. يمنع النموذج من التشخيص ومن اقتراح أدوية ومن اختلاق أرقام هواتف أو ساعات دوام أو مستويات زحمة، ويلزمه بالإجابة بلغة المستخدم. |
| **القواعد** (`KEYWORDS`, `find_department`) | قواعد كلمات مفتاحية (عربي وإنجليزي) تربط الشكوى بقسم: الطوارئ، الجلدية، الأسنان، العيون، العظام، الطب العام. الكلمات العاجلة (مثل ألم الصدر وضيق التنفس ونزيف شديد وفقدان الوعي) تعطي دائماً الطوارئ. هذي قواعد نموذجية لمشروع طلابي، وليست إرشاداً طبياً رسمياً. |
| **الأداة 1** (`triage_lookup`) | أداة LangChain بـ `@tool` ترجّع نتيجة القواعد بصيغة JSON. |
| **الأداة 2** (`find_nearest_facilities`) | أداة LangChain بـ `@tool` تبحث عن أقرب المستشفيات أو المستوصفات. التفاصيل في قسم «البحث عن المرافق». |
| **المخرجات المنظمة** (`Result`, `parser`) | نموذج Pydantic مع `PydanticOutputParser` عشان تجي الإجابة دائماً بنفس الحقول. |
| **الأيجنت** (`create_agent`, `ask`) | يبني الأيجنت بالأداتين ويشغّله. الدالة `ask()` تعيد المحاولة مرة وحدة إذا تعذّر قراءة الـ JSON، وتأخذ بيانات المرافق من مخرجات الأداة، وفيها شبكة أمان: إذا أُعطي موقع وتجاوز النموذج الأداة أو طلب نوعاً غلط، يُشغَّل البحث مباشرة. |
| **الواجهة** (Gradio) | ثلاث شاشات: اللغة، الشكوى (مع الموقع ونوع المرفق اختيارياً)، النتيجة. كل النصوص باللغتين في القاموس `UI`. |

**قواعد الأمان تعمل في الكود، مو في الـ Prompt بس:** إذا اعتبرت القواعد أو النموذج الحالة عاجلة، يعرض التطبيق تنبيه العاجل وقسم الطوارئ مهما كتب النموذج غير ذلك.

### البحث عن المرافق (`find_nearest_facilities`)

1. **تحديد الموقع:** يتحول اسم المكان إلى إحداثيات عبر Nominatim، وOpen-Meteo كخيار احتياطي.
2. **البحث:** استعلام Overpass عن عناصر OpenStreetMap الموسومة كمستشفيات (`amenity=hospital` و`healthcare=hospital`) أو مستوصفات/مراكز صحية (`amenity=clinic` و`healthcare=clinic` و`healthcare=centre`). يبدأ نصف القطر صغيراً ثم يكبر: المستشفيات 15 كم ثم 50 كم، والمستوصفات 5 ثم 15 ثم 40 كم.
3. **السيرفرات العامة ممكن تبطّئ أو تتعطل**، فتُجرَّب ثلاثة سيرفرات Overpass تباعاً (يبدأ سيرفر جديد إذا ما رد السابق خلال 3 ثوانٍ). وإذا فشلت كلها يُستخدم بحث Nominatim كحل احتياطي.
4. **السرعة:** يبدأ البحث في الخلفية بمجرد إرسال المستخدم (`prefetch_facilities`)، وتُحفظ النتائج الناجحة، وتُتذكَّر العملية الفاشلة 20 ثانية.
5. **النتائج:** أقرب 3 مرافق، بدون تكرار الأسماء، مع المسافة ورابط خرائط Google، ومواعيد الدوام والهاتف فقط إذا كانت موجودة في OpenStreetMap.
6. **إذا فشل كل شي**، يعرض التطبيق السبب التقني بدل قائمة فاضية.

### سلوك الواجهة

- اختيار اللغة يغيّر كل النصوص والأمثلة وعناوين الحقول ولغة الإجابة.
- تظهر بطاقة تحميل فوراً، ومؤقت مراقبة (120 ثانية) يضمن أنها ما تبقى للأبد، والأخطاء تُعرض في بطاقة.
- العربية تُعرض من اليمين إلى اليسار.

### الملفات

| الملف | الغرض |
|---|---|
| `app.py` | المشروع كله في سكربت واحد. |
| `requirements.txt` | مكتبات Python المطلوبة. |
| `Day2_Homework_Starter_2_(2).ipynb` | دفتر Colab اللي بُني فيه المشروع. |

### تشغيله بنفسك

```bash
pip install -r requirements.txt
export GROQ_API_KEY="your-key"     # في ويندوز PowerShell: $env:GROQ_API_KEY="your-key"
python app.py
```

ثم افتح `http://127.0.0.1:7860`. لا تكتب مفتاحك داخل الكود ولا ترفعه على GitHub أبداً.

### حدود المشروع

- يوجّه إلى قسم، ولا يشخّص ولا يصف دواء ولا يغني عن الطبيب.
- القواعد تغطي عينة صغيرة من الشكاوى، وغيرها يروح للطب العام.
- بيانات المرافق من OpenStreetMap وقد تكون ناقصة أو قديمة.
- سيرفرات الخرائط العامة قد تبطّئ أو تتوقف.
- ما يعرض مستويات زحمة ولا أوقات انتظار، لأنه ما فيه مصدر بيانات موثوق لها.

### مصدر البيانات

بيانات الخرائط © مساهمو OpenStreetMap (رخصة ODbL). انظر https://www.openstreetmap.org/copyright

### الشكر

بناه **Abdullah Abdali** كواجب اليوم الثاني من معسكر atomcamp Arabia × SDA لـ Agentic AI (المدربة: رزان عتين).

</div>
