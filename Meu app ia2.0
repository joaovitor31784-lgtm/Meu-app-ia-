import streamlit as st
import fitz
from bs4 import BeautifulSoup
from collections import defaultdict
from PIL import Image, ImageDraw
import io
import json
import os

# =========================
# CONFIG
# =========================
st.set_page_config(page_title="Leitor Inteligente com IA", layout="wide")

st.title("📄🔍 Leitor Inteligente de Documentos com IA")
st.markdown("""
Faça upload de um arquivo **PDF ou HTML** e utilize:
- 🔍 Busca inteligente com navegação
- 🧠 Palavras-chave automáticas
- 💬 Chat com IA baseado no documento

👉 Ideal para estudar, analisar contratos ou resumir conteúdos rapidamente.
""")

# =========================
# API KEY (SEGURO)
# =========================
api_key = None

# prioridade: st.secrets
if "OPENAI_API_KEY" in st.secrets:
    api_key = st.secrets["OPENAI_API_KEY"]

# fallback: input manual
if not api_key:
    with st.sidebar:
        st.warning("⚠️ Insira sua API Key da OpenAI")
        api_key = st.text_input("API Key", type="password")

# =========================
# OPENAI
# =========================
def get_client():
    from openai import OpenAI
    return OpenAI(api_key=api_key)


# =========================
# LIMITADOR TEXTO
# =========================
def limitar_texto(texto, max_chars=15000):
    if len(texto) <= max_chars:
        return texto
    metade = max_chars // 2
    return texto[:metade] + "\n...\n" + texto[-metade:]


# =========================
# LEITOR PDF
# =========================
def ler_pdf(file):
    try:
        doc = fitz.open(stream=file.read(), filetype="pdf")

        if doc.is_encrypted and not doc.authenticate(""):
            st.error("🔒 PDF protegido por senha.")
            return None, None

        indice = defaultdict(list)

        for i, pagina in enumerate(doc):
            for w in pagina.get_text("words"):
                x0, y0, x1, y1, palavra, *_ = w
                indice[palavra.lower()].append({
                    "tipo": "pdf",
                    "pagina": i,
                    "bbox": (x0, y0, x1, y1)
                })

        return doc, dict(indice)

    except Exception as e:
        st.error(f"Erro ao ler PDF: {e}")
        return None, None


# =========================
# LEITOR HTML
# =========================
def ler_html(file):
    try:
        html = file.read().decode("utf-8", errors="ignore")
        soup = BeautifulSoup(html, "html.parser")

        for tag in soup(["script", "style"]):
            tag.decompose()

        paragrafos = [p.get_text(strip=True) for p in soup.find_all("p") if p.get_text(strip=True)]

        indice = defaultdict(list)

        for i, p in enumerate(paragrafos):
            for palavra in p.split():
                indice[palavra.lower()].append({
                    "tipo": "html",
                    "paragrafo": i,
                    "texto": p
                })

        return paragrafos, dict(indice)

    except Exception as e:
        st.error(f"Erro ao ler HTML: {e}")
        return None, None


# =========================
# IA: KEYWORDS
# =========================
def extrair_keywords(texto):
    try:
        client = get_client()
        texto = limitar_texto(texto)

        prompt = f"""
Retorne apenas JSON:
{{ "keywords": ["10 palavras-chave"] }}

Texto:
{texto}
"""

        resp = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.2
        )

        return json.loads(resp.choices[0].message.content)

    except:
        return {"keywords": []}


# =========================
# IA: CONTEXTO
# =========================
def explicar_contexto(texto, pergunta):
    try:
        client = get_client()
        texto = limitar_texto(texto)

        prompt = f"""
Contexto:
{texto}

Pergunta:
{pergunta}
"""

        resp = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.3
        )

        return resp.choices[0].message.content

    except:
        return "Erro ao consultar IA."


# =========================
# BUSCA
# =========================
def buscar(indice, termo):
    termo = termo.lower()
    resultados = []

    for palavra in indice:
        if termo in palavra:
            resultados.extend(indice[palavra])

    return resultados


# =========================
# RENDER PDF
# =========================
def render_pdf(doc, pagina, destaques=None):
    try:
        pix = doc[pagina].get_pixmap(matrix=fitz.Matrix(2, 2))
        img = Image.open(io.BytesIO(pix.tobytes())).convert("RGBA")

        if destaques:
            overlay = Image.new("RGBA", img.size, (255, 255, 255, 0))
            draw = ImageDraw.Draw(overlay)

            for bbox in destaques:
                scaled = [b * 2 for b in bbox]
                draw.rectangle(scaled, fill=(255, 255, 0, 100))

            img = Image.alpha_composite(img, overlay)

        return img.convert("RGB")

    except:
        st.error("Erro ao renderizar PDF")
        return None


# =========================
# UPLOAD
# =========================
file = st.file_uploader("📤 Envie um PDF ou HTML", type=["pdf", "html"])

if file:

    if not file.name.endswith(("pdf", "html")):
        st.warning("⚠️ Formato não suportado. Envie apenas PDF ou HTML.")
        st.stop()

    if "indice" not in st.session_state:

        with st.spinner("Processando documento..."):

            if file.name.endswith(".pdf"):
                doc, indice = ler_pdf(file)
                st.session_state.tipo = "pdf"
                st.session_state.doc = doc

            else:
                paragrafos, indice = ler_html(file)
                st.session_state.tipo = "html"
                st.session_state.paragrafos = paragrafos

            if indice is None:
                st.stop()

            st.session_state.indice = indice
            st.session_state.resultados = []
            st.session_state.idx = 0
            st.session_state.pagina = 0
            st.session_state.messages = []

            # texto completo
            if st.session_state.tipo == "pdf":
                texto = "\n".join([p.get_text() for p in st.session_state.doc])
            else:
                texto = "\n".join(st.session_state.paragrafos)

            st.session_state.keywords = extrair_keywords(texto)["keywords"]

    indice = st.session_state.indice

    # =========================
    # SIDEBAR
    # =========================
    st.sidebar.subheader("🧠 Palavras-chave")

    for kw in st.session_state.keywords:
        if st.sidebar.button(kw):
            st.session_state.busca = kw

    # =========================
    # BUSCA
    # =========================
    termo = st.text_input("🔍 Buscar", value=st.session_state.get("busca", ""))

    if termo:
        st.session_state.resultados = buscar(indice, termo)
        st.session_state.idx = 0

    resultados = st.session_state.resultados

    col1, col2 = st.columns([1, 3])

    # =========================
    # NAVEGAÇÃO
    # =========================
    if resultados:
        total = len(resultados)
        idx = st.session_state.idx
        item = resultados[idx]

        with col1:
            if st.button("⬅️ Anterior"):
                st.session_state.idx = (idx - 1) % total

            if st.button("Próximo ➡️"):
                st.session_state.idx = (idx + 1) % total

            st.write(f"{idx+1}/{total}")

        with col2:
            if item["tipo"] == "pdf":
                st.session_state.pagina = item["pagina"]
                img = render_pdf(st.session_state.doc, item["pagina"], [item["bbox"]])
                if img:
                    st.image(img, use_container_width=True)

            else:
                texto = item["texto"]
                termo_lower = termo.lower()

                palavras = texto.split()
                highlight = [
                    f"**:yellow[{p}]**" if termo_lower in p.lower() else p
                    for p in palavras
                ]

                st.write(" ".join(highlight))

    # =========================
    # CHAT
    # =========================
    st.divider()
    st.subheader("💬 Chat com o documento")

    for msg in st.session_state.messages:
        with st.chat_message(msg["role"]):
            st.write(msg["content"])

    pergunta = st.chat_input("Pergunte algo sobre o documento...")

    if pergunta:
        st.session_state.messages.append({"role": "user", "content": pergunta})

        if st.session_state.tipo == "pdf":
            texto = st.session_state.doc[st.session_state.pagina].get_text()
        else:
            texto = st.session_state.paragrafos[0]

        resposta = explicar_contexto(texto, pergunta)

        st.session_state.messages.append({"role": "assistant", "content": resposta})

        st.rerun()
