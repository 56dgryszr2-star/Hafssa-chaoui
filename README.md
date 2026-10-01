import streamlit as st
import sympy as sp
import plotly.graph_objects as go
from PIL import Image
from openai import OpenAI
import base64
import io
import random
import time

# ============================================================
# LIMIT & CONTINUITY — BY HAFSA CHAWI
# Advanced Mathematics Web App
# ============================================================

st.set_page_config(
    page_title="Limit & Continuity — Hafsa Chawi",
    page_icon="∞",
    layout="wide",
    initial_sidebar_state="expanded"
)

# ============================================================
# STYLE
# ============================================================

st.markdown("""
<style>

@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap');

html, body, [class*="css"] {
    font-family: 'Inter', sans-serif;
}

.stApp {
    background:
        radial-gradient(circle at 10% 10%, rgba(124,58,237,.15), transparent 30%),
        radial-gradient(circle at 90% 20%, rgba(6,182,212,.12), transparent 30%),
        linear-gradient(135deg,#080b18,#10152b 55%,#080b18);
    color: white;
}

section[data-testid="stSidebar"] {
    background: linear-gradient(180deg,#0b1022,#111936);
    border-right: 1px solid rgba(255,255,255,.08);
}

.hero {
    padding: 35px;
    border-radius: 28px;
    background:
        linear-gradient(135deg,
        rgba(124,58,237,.30),
        rgba(6,182,212,.18));
    border: 1px solid rgba(255,255,255,.12);
    box-shadow: 0 20px 70px rgba(0,0,0,.35);
    margin-bottom: 25px;
}

.hero-title {
    font-size: 48px;
    font-weight: 900;
    background: linear-gradient(90deg,#a78bfa,#22d3ee,#f0abfc);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
}

.hero-sub {
    color: #cbd5e1;
    font-size: 18px;
}

.card {
    padding: 22px;
    border-radius: 22px;
    background: rgba(255,255,255,.055);
    border: 1px solid rgba(255,255,255,.09);
    box-shadow: 0 12px 40px rgba(0,0,0,.22);
    margin-bottom: 18px;
}

.big-number {
    font-size: 38px;
    font-weight: 900;
}

.badge {
    display:inline-block;
    padding:7px 14px;
    border-radius:999px;
    background:rgba(124,58,237,.20);
    border:1px solid rgba(167,139,250,.35);
    color:#ddd6fe;
    font-weight:700;
}

.question {
    font-size: 23px;
    font-weight: 800;
    line-height: 1.5;
}

.correct {
    padding:15px;
    border-radius:15px;
    background:rgba(34,197,94,.13);
    border:1px solid rgba(34,197,94,.3);
}

.wrong {
    padding:15px;
    border-radius:15px;
    background:rgba(239,68,68,.13);
    border:1px solid rgba(239,68,68,.3);
}

.footer {
    text-align:center;
    padding:30px;
    color:#94a3b8;
}

</style>
""", unsafe_allow_html=True)

# ============================================================
# SESSION STATE
# ============================================================

defaults = {
    "xp": 0,
    "level": 1,
    "solved": 0,
    "correct": 0,
    "wrong": 0,
    "streak": 0,
    "history": [],
    "exam_started": False,
    "exam_start": None,
    "exam_score": 0,
    "exam_questions": [],
    "exam_index": 0
}

for key, value in defaults.items():
    if key not in st.session_state:
        st.session_state[key] = value

# ============================================================
# XP SYSTEM
# ============================================================

def add_xp(amount):
    st.session_state.xp += amount
    new_level = max(1, st.session_state.xp // 100 + 1)

    if new_level > st.session_state.level:
        st.session_state.level = new_level
        st.balloons()

def record_result(topic, difficulty, correct):
    st.session_state.solved += 1

    if correct:
        st.session_state.correct += 1
        st.session_state.streak += 1
        add_xp({"Facile": 10, "Moyen": 20, "Difficile": 35}.get(difficulty, 10))
    else:
        st.session_state.wrong += 1
        st.session_state.streak = 0

    st.session_state.history.append({
        "topic": topic,
        "difficulty": difficulty,
        "correct": correct
    })

# ============================================================
# DATA
# ============================================================

questions = [

    {
        "topic": "Limites",
        "difficulty": "Facile",
        "question": "Calculer : lim(x→2) (x² + 3x)",
        "choices": ["10", "8", "6", "12"],
        "answer": "10",
        "solution": [
            "La fonction polynomiale est continue.",
            "On remplace directement x par 2.",
            "2² + 3×2 = 4 + 6.",
            "Donc la limite vaut 10."
        ]
    },

    {
        "topic": "Limites",
        "difficulty": "Moyen",
        "question": "Calculer : lim(x→3) (x² - 9)/(x - 3)",
        "choices": ["3", "6", "9", "∞"],
        "answer": "6",
        "solution": [
            "On obtient une forme indéterminée 0/0.",
            "On factorise : x² - 9 = (x-3)(x+3).",
            "On simplifie par x-3.",
            "Il reste x+3.",
            "Donc la limite en 3 vaut 6."
        ]
    },

    {
        "topic": "Limites",
        "difficulty": "Difficile",
        "question": "Calculer : lim(x→∞) (3x²+2)/(x²-5)",
        "choices": ["0", "2", "3", "∞"],
        "answer": "3",
        "solution": [
            "On divise numérateur et dénominateur par x².",
            "(3 + 2/x²)/(1 - 5/x²).",
            "Lorsque x→∞, 2/x²→0 et 5/x²→0.",
            "La limite vaut donc 3."
        ]
    },

    {
        "topic": "Continuité",
        "difficulty": "Facile",
        "question": "Une fonction polynomiale est-elle continue sur ℝ ?",
        "choices": ["Oui", "Non", "Seulement en 0", "Seulement si elle est positive"],
        "answer": "Oui",
        "solution": [
            "Toute fonction polynomiale est continue sur ℝ.",
            "Il n'existe donc aucun point de discontinuité."
        ]
    },

    {
        "topic": "Continuité",
        "difficulty": "Moyen",
        "question": "Pour que f soit continue en a, quelle condition doit être vérifiée ?",
        "choices": [
            "lim f(x)=0",
            "lim f(x)=f(a)",
            "f(a)=1",
            "f doit être dérivable"
        ],
        "answer": "lim f(x)=f(a)",
        "solution": [
            "La définition fondamentale est :",
            "lim(x→a) f(x) = f(a).",
            "Les deux valeurs doivent donc être égales."
        ]
    },

    {
        "topic": "Continuité",
        "difficulty": "Difficile",
        "question": "Quel théorème permet de garantir l'existence d'un zéro sous certaines conditions ?",
        "choices": [
            "Théorème de Pythagore",
            "TVI",
            "Théorème de Thalès",
            "Binôme de Newton"
        ],
        "answer": "TVI",
        "solution": [
            "Le TVI est le Théorème des Valeurs Intermédiaires.",
            "Pour une fonction continue sur [a,b], toute valeur comprise entre f(a) et f(b) est atteinte."
        ]
    }
]

# ============================================================
# SIDEBAR
# ============================================================

with st.sidebar:

    st.markdown("## ∞ LIMIT & CONTINUITY")

    st.markdown(
        "<span class='badge'>BY HAFSA CHAWI</span>",
        unsafe_allow_html=True
    )

    st.write("")

    page = st.radio(
        "Navigation",
        [
            "🏠 Dashboard",
            "∞ Calculateur de Limites",
            "📈 Graphique",
            "🧠 Exercices",
            "🎯 BAC Exam",
            "🤖 AI Tutor",
            "📊 Ma Progression"
        ]
    )

    st.divider()

    st.metric("XP", st.session_state.xp)
    st.metric("Niveau", st.session_state.level)
    st.metric("Exercices", st.session_state.solved)

# ============================================================
# HERO
# ============================================================

def hero():

    st.markdown("""
    <div class="hero">
        <div class="hero-title">
            LIMIT & CONTINUITY
        </div>
        <div class="hero-sub">
            Une plateforme interactive de mathématiques
            créée par <b>Hafsa Chawi</b>.
        </div>
    </div>
    """, unsafe_allow_html=True)

# ============================================================
# DASHBOARD
# ============================================================

if page == "🏠 Dashboard":

    hero()

    col1, col2, col3, col4 = st.columns(4)

    with col1:
        st.metric("⚡ XP", st.session_state.xp)

    with col2:
        st.metric("🏆 Niveau", st.session_state.level)

    with col3:
        st.metric("✅ Correct", st.session_state.correct)

    with col4:
        st.metric("🔥 Série", st.session_state.streak)

    st.markdown("## 🚀 Centre d'apprentissage")

    a, b, c = st.columns(3)

    with a:
        st.markdown("""
        <div class="card">
        <h2>∞ Limites</h2>
        <p>Calcul, formes indéterminées, factorisation,
        limites à l'infini et méthodes BAC.</p>
        </div>
        """, unsafe_allow_html=True)

    with b:
        st.markdown("""
        <div class="card">
        <h2>🔗 Continuité</h2>
        <p>Continuité en un point, sur un intervalle,
        fonctions usuelles et TVI.</p>
        </div>
        """, unsafe_allow_html=True)

    with c:
        st.markdown("""
        <div class="card">
        <h2>🤖 AI Tutor</h2>
        <p>Photographie un exercice et demande
        une résolution détaillée.</p>
        </div>
        """, unsafe_allow_html=True)

    st.markdown("## 🎯 Objectif du jour")

    target = min(st.session_state.solved, 10)

    st.progress(target / 10)

    st.write(f"**{target}/10 exercices réalisés aujourd'hui**")

# ============================================================
# LIMIT CALCULATOR
# ============================================================

elif page == "∞ Calculateur de Limites":

    hero()

    st.markdown("## ∞ Calculateur de limites")

    st.info(
        "Écris une expression SymPy. Exemple : "
        "(x**2-9)/(x-3)"
    )

    expression = st.text_input(
        "Expression f(x)",
        "(x**2 - 9)/(x - 3)"
    )

    point = st.text_input(
        "x →",
        "3"
    )

    direction = st.selectbox(
        "Direction",
        ["Deux côtés", "+", "-"]
    )

    if st.button("🚀 Calculer la limite", use_container_width=True):

        try:

            x = sp.symbols("x")
            expr = sp.sympify(expression)

            if point.lower() in ["inf", "oo", "infinity"]:
                p = sp.oo
            else:
                p = sp.sympify(point)

            if direction == "Deux côtés":

                result = sp.limit(expr, x, p)

            elif direction == "+":

                result = sp.limit(expr, x, p, dir="+")

            else:

                result = sp.limit(expr, x, p, dir="-")

            st.success(f"Résultat : {sp.pretty(result)}")

            with st.expander("🔍 Voir l'expression simplifiée"):

                st.latex(sp.latex(expr))

                try:
                    simplified = sp.simplify(expr)
                    st.latex(sp.latex(simplified))
                except:
                    pass

        except Exception as e:

            st.error(
                "Expression non reconnue. "
                "Utilise la syntaxe SymPy."
            )

# ============================================================
# GRAPH
# ============================================================

elif page == "📈 Graphique":

    hero()

    st.markdown("## 📈 Visualisation interactive")

    expression = st.text_input(
        "f(x) =",
        "sin(x)/x"
    )

    xmin = st.number_input("x minimum", -20.0, 0.0, -10.0)
    xmax = st.number_input("x maximum", 0.1, 20.0, 10.0)

    if st.button("📊 Afficher le graphique"):

        try:

            x = sp.symbols("x")
            expr = sp.sympify(expression)

            xs = []
            ys = []

            for i in range(1000):

                value = xmin + (xmax - xmin) * i / 999

                try:

                    y = float(expr.subs(x, value))

                    if abs(y) < 1e5:
                        xs.append(value)
                        ys.append(y)

                except:
                    pass

            fig = go.Figure()

            fig.add_trace(
                go.Scatter(
                    x=xs,
                    y=ys,
                    mode="lines",
                    name="f(x)"
                )
            )

            fig.update_layout(
                template="plotly_dark",
                height=600,
                title=f"f(x) = {expression}",
                xaxis_title="x",
                yaxis_title="f(x)"
            )

            st.plotly_chart(
                fig,
                use_container_width=True
            )

        except:

            st.error("Impossible de tracer cette fonction.")

# ============================================================
# EXERCISES
# ============================================================

elif page == "🧠 Exercices":

    hero()

    st.markdown("## 🧠 Training Arena")

    topic = st.selectbox(
        "Thème",
        ["Tous", "Limites", "Continuité"]
    )

    difficulty = st.selectbox(
        "Niveau",
        ["Tous", "Facile", "Moyen", "Difficile"]
    )

    pool = questions

    if topic != "Tous":
        pool = [q for q in pool if q["topic"] == topic]

    if difficulty != "Tous":
        pool = [q for q in pool if q["difficulty"] == difficulty]

    if not pool:

        st.warning("Aucun exercice trouvé.")

    else:

        if "current_question" not in st.session_state:
            st.session_state.current_question = random.choice(pool)

        q = st.session_state.current_question

        st.markdown(
            f"<span class='badge'>{q['topic']}</span> "
            f"<span class='badge'>{q['difficulty']}</span>",
            unsafe_allow_html=True
        )

        st.markdown(
            f"<div class='question'>{q['question']}</div>",
            unsafe_allow_html=True
        )

        answer = st.radio(
            "Choisis ta réponse",
            q["choices"],
            key="answer_radio"
        )

        col1, col2 = st.columns(2)

        with col1:

            if st.button(
                "✅ Vérifier",
                use_container_width=True
            ):

                correct = answer == q["answer"]

                record_result(
                    q["topic"],
                    q["difficulty"],
                    correct
                )

                if correct:

                    st.markdown(
                        "<div class='correct'>"
                        "🎉 Excellent ! Bonne réponse."
                        "</div>",
                        unsafe_allow_html=True
                    )

                else:

                    st.markdown(
                        f"<div class='wrong'>"
                        f"❌ Réponse incorrecte.<br>"
                        f"La bonne réponse est : <b>{q['answer']}</b>"
                        f"</div>",
                        unsafe_allow_html=True
                    )

                st.write("")

                with st.expander("🧠 Correction détaillée"):

                    for i, step in enumerate(q["solution"], 1):

                        st.markdown(
                            f"**Étape {i} :** {step}"
                        )

        with col2:

            if st.button(
                "➡️ Exercice suivant",
                use_container_width=True
            ):

                st.session_state.current_question = random.choice(pool)

                st.rerun()

# ============================================================
# BAC EXAM
# ============================================================

elif page == "🎯 BAC Exam":

    hero()

    st.markdown("## 🎯 BAC EXAM MODE")

    st.write(
        "Simulation rapide : 5 questions — temps limité."
    )

    if not st.session_state.exam_started:

        if st.button(
            "🚀 Commencer l'examen",
            use_container_width=True
        ):

            st.session_state.exam_started = True
            st.session_state.exam_start = time.time()
            st.session_state.exam_score = 0
            st.session_state.exam_index = 0
            st.session_state.exam_questions = random.sample(
                questions,
                min(5, len(questions))
            )

            st.rerun()

    else:

        elapsed = time.time() - st.session_state.exam_start
        remaining = max(0, 600 - int(elapsed))

        mins = remaining // 60
        secs = remaining % 60

        st.metric(
            "⏱ Temps restant",
            f"{mins:02d}:{secs:02d}"
        )

        if remaining <= 0:

            st.warning("⏰ Temps écoulé !")

            st.session_state.exam_started = False

            st.write(
                f"Score final : "
                f"{st.session_state.exam_score}/5"
            )

        else:

            idx = st.session_state.exam_index

            if idx < len(st.session_state.exam_questions):

                q = st.session_state.exam_questions[idx]

                st.progress(
                    idx / len(st.session_state.exam_questions)
                )

                st.markdown(
                    f"### Question {idx + 1}/5"
                )

                st.markdown(
                    f"<div class='question'>{q['question']}</div>",
                    unsafe_allow_html=True
                )

                selected = st.radio(
                    "Réponse",
                    q["choices"],
                    key=f"exam_{idx}"
                )

                if st.button(
                    "Valider",
                    use_container_width=True
                ):

                    if selected == q["answer"]:
                        st.session_state.exam_score += 1

                    st.session_state.exam_index += 1

                    st.rerun()

            else:

                score = st.session_state.exam_score

                st.success(
                    f"🏆 Examen terminé : {score}/5"
                )

                if score == 5:
                    st.balloons()

                st.session_state.exam_started = False

# ============================================================
# AI TUTOR
# ============================================================

elif page == "🤖 AI Tutor":

    hero()

    st.markdown("## 🤖 AI Tutor — Hafsa Math AI")

    st.info(
        "Tu peux écrire ton exercice ou envoyer une photo."
    )

    api_key = st.text_input(
        "OpenAI API Key",
        type="password",
        help="La clé reste uniquement dans cette session."
    )

    uploaded = st.file_uploader(
        "📷 Photographier / importer un exercice",
        type=["png", "jpg", "jpeg", "webp"]
    )

    exercise_text = st.text_area(
        "Ou écris ton exercice ici",
        height=180,
        placeholder="Exemple : Calculer la limite de..."
    )

    level = st.selectbox(
        "Niveau d'explication",
        [
            "Simple",
            "Détaillé",
            "Niveau BAC SM",
            "Expert"
        ]
    )

    if st.button(
        "🧠 Résoudre avec AI",
        use_container_width=True
    ):

        if not api_key:

            st.warning(
                "Ajoute une OpenAI API Key pour activer l'AI Tutor."
            )

        else:

            try:

                client = OpenAI(api_key=api_key)

                instructions = f"""
Tu es Hafsa Math AI, un tuteur spécialisé uniquement
dans LIMITES et CONTINUITÉ pour les élèves de
2ème Bac Sciences Mathématiques.

Niveau demandé : {level}

Règles :
- Ne donne jamais uniquement la réponse.
- Explique chaque étape.
- Identifie la méthode utilisée.
- Explique pourquoi cette méthode fonctionne.
- Signale les erreurs fréquentes.
- Utilise une notation mathématique claire.
- Termine par une mini-règle à retenir.
- Réponds en français.
"""

                content = [
                    {
                        "type": "input_text",
                        "text": instructions
                    }
                ]

                if exercise_text.strip():

                    content.append({
                        "type": "input_text",
                        "text": exercise_text
                    })

                if uploaded:

                    image_bytes = uploaded.read()

                    encoded = base64.b64encode(
                        image_bytes
                    ).decode("utf-8")

                    mime = uploaded.type

                    content.append({
                        "type": "input_image",
                        "image_url":
                            f"data:{mime};base64,{encoded}"
                    })

                response = client.responses.create(
                    model="gpt-5.6-luna",
                    input=[
                        {
                            "role": "user",
                            "content": content
                        }
                    ]
                )

                st.markdown("## 🧠 Correction")

                st.markdown(
                    response.output_text
                )

                add_xp(15)

            except Exception as e:

                st.error(
                    "Une erreur est survenue avec l'AI."
                )

                st.code(str(e))

# ============================================================
# PROGRESS
# ============================================================

elif page == "📊 Ma Progression":

    hero()

    st.markdown("## 📊 Mon évolution")

    total = st.session_state.solved

    if total > 0:

        percentage = (
            st.session_state.correct /
            total
        ) * 100

    else:

        percentage = 0

    c1, c2, c3 = st.columns(3)

    with c1:
        st.metric(
            "Taux de réussite",
            f"{percentage:.1f}%"
        )

    with c2:
        st.metric(
            "Exercices résolus",
            total
        )

    with c3:
        st.metric(
            "Meilleure série",
            st.session_state.streak
        )

    st.progress(
        min(percentage / 100, 1)
    )

    st.markdown("### 🏆 Ton niveau")

    if st.session_state.level >= 10:
        title = "👑 CONTINUITY MASTER"

    elif st.session_state.level >= 7:
        title = "🔥 BAC EXPERT"

    elif st.session_state.level >= 4:
        title = "⚡ MATHS WARRIOR"

    else:
        title = "🌱 FUTURE MATHS MASTER"

    st.markdown(
        f"""
        <div class="card">
            <div class="big-number">{title}</div>
            <p>
            Continue à résoudre des exercices pour
            débloquer les prochains niveaux.
            </p>
        </div>
        """,
        unsafe_allow_html=True
    )

    if st.session_state.history:

        st.markdown("### 📚 Historique")

        for item in reversed(
            st.session_state.history[-10:]
        ):

            icon = "✅" if item["correct"] else "❌"

            st.write(
                f"{icon} {item['topic']} — "
                f"{item['difficulty']}"
            )

# ============================================================
# FOOTER
# ============================================================

st.markdown("""
<div class="footer">

━━━━━━━━━━━━━━━━━━━━━━━━━━━━

<h3>∞ LIMIT & CONTINUITY</h3>

<b>by Hafsa Chawi</b>

<p>
Master Limits. Master Continuity. Master the BAC.
</p>

<p>
© 2026 Hafsa Chawi — Educational Project
</p>

</div>
""", unsafe_allow_html=True)
