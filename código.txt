import customtkinter as ctk
from PIL import Image, ImageDraw, ImageFont
from tkinter import filedialog
import os

# =================================================================
# CONFIGURAÇÕES DE CAMINHOS E DIRETÓRIOS
# =================================================================

# Caminhos fixos
ASSETS    = r"c:\Users\vitor.silva\Downloads\NBR_Assinatura\assets"
FONTS_DIR = r"c:\Users\vitor.silva\Downloads\NBR_Assinatura\assets\fonts"
BASE_IMG  = r"c:\Users\vitor.silva\Downloads\NBR_Assinatura\assets\assinatura_base.png"
ICON_PATH = r"c:\Users\vitor.silva\Downloads\NBR_Assinatura\assets\icon_nbr.png"
LOGO_PATH = r"c:\Users\vitor.silva\Downloads\NBR_Assinatura\assets\logo_nbr.png"
# Pasta padrão onde o Windows armazena assinaturas do Outlook
OUTLOOK_SIG_DIR = os.path.join(os.environ.get("APPDATA", ""), "Microsoft", "Signatures")

# =================================================================
# IDENTIDADE VISUAL E CORES
# =================================================================

RED     = "#DA2128"
DK_RED  = "#981B1E"
GR_DARK = "#595C5C"
GR_LITE = "#B4BBBF"
GR_TRUE = "#787878"
BG_MAIN = ["#F0EDE9", "#111010"] # [Modo Claro, Modo Escuro]
BG_SIDE = ["#FFFFFF", "#1C1A18"]

# Caminhos dos arquivos de fonte TTF
_AVENIR_REG  = os.path.join(FONTS_DIR, "avenir-next-lt-pro.ttf")
_AVENIR_BOLD = os.path.join(FONTS_DIR, "avenir-next-lt-pro-bold.ttf")

# =================================================================
# FUNÇÕES DE APOIO (LÓGICA INTERNA)
# =================================================================

def _carregar_fonte_ui() -> str:
    """Define a fonte padrão para a interface gráfica."""
    return "Calibri"

def _pil_fonte(size: int, bold: bool = False) -> ImageFont.FreeTypeFont:
    """
    Tenta carregar a fonte Avenir para desenhar na imagem.
    Se não encontrar, tenta fontes do sistema (Calibri/Arial).
    """
    candidates = (
        [_AVENIR_BOLD, "C:/Windows/Fonts/calibrib.ttf", "C:/Windows/Fonts/arialbd.ttf"]
        if bold else
        [_AVENIR_REG,  "C:/Windows/Fonts/calibri.ttf",  "C:/Windows/Fonts/arial.ttf"]
    )
    for p in candidates:
        if os.path.exists(p):
            return ImageFont.truetype(p, size)
    return ImageFont.load_default()

def _remover_fundo(img: Image.Image, thresh: int = 235) -> Image.Image:
    """
    Torna pixels quase brancos (acima do thresh) transparentes.
    Utilizado para limpar bordas de logos.
    """
    img = img.convert("RGBA")
    px  = img.load()
    for y in range(img.height):
        for x in range(img.width):
            r, g, b, a = px[x, y]
            if r > thresh and g > thresh and b > thresh:
                px[x, y] = (r, g, b, 0)
    return img

def _preparar_assets():
    """
    Verifica se o logo e o ícone existem. Caso não existam,
    tenta recortá-los automaticamente da imagem base de assinatura.
    """
    base_ok = os.path.exists(BASE_IMG)
    if not os.path.exists(ICON_PATH) and base_ok:
        base = Image.open(BASE_IMG).convert("RGBA")
        w, h = base.size
        _remover_fundo(base.crop((int(w * 0.80), 0, w, int(h * 0.295)))).save(ICON_PATH, "PNG")
    if not os.path.exists(LOGO_PATH) and base_ok:
        base = Image.open(BASE_IMG).convert("RGBA")
        w, h = base.size
        _remover_fundo(base.crop((int(w * 0.69), 0, w, int(h * 0.72)))).save(LOGO_PATH, "PNG")

# =================================================================
# LÓGICA DE GERAÇÃO DA ASSINATURA
# =================================================================

# Coordenadas e tamanhos de texto na imagem final
NOME_XY    = (35, 22)
CARGO_XY   = (35, 95)
NOME_SIZE  = 55
CARGO_SIZE = 50
# Dimensões para o preview na tela da aplicação
PREVIEW_W  = 624
PREVIEW_H  = int(PREVIEW_W * 713 / 1600)

imagem_para_salvar: Image.Image | None = None

def _gerar_preview(img: Image.Image) -> ctk.CTkImage:
    """Redimensiona a imagem para caber na interface do app."""
    resized = img.resize((PREVIEW_W, PREVIEW_H), Image.LANCZOS)
    return ctk.CTkImage(light_image=resized, dark_image=resized, size=(PREVIEW_W, PREVIEW_H))

def gerar():
    """
    Lê os campos de texto, carrega a imagem base e desenha
    o Nome e o Cargo com as cores e fontes corretas.
    """
    global imagem_para_salvar
    nome  = campo_nome.get().strip()
    cargo = campo_cargo.get().strip()
   
    if not nome or not cargo:
        _status("Preencha Nome e Cargo antes de gerar.", "warn")
        return
    if not os.path.exists(BASE_IMG):
        _status("Imagem base nao encontrada.", "err")
        return

    # Abre a base da assinatura
    img  = Image.open(BASE_IMG).convert("RGBA")
    draw = ImageDraw.Draw(img)
    fn_n = _pil_fonte(NOME_SIZE,  bold=True)
    fn_c = _pil_fonte(CARGO_SIZE, bold=False)

    # Lógica de cores: Primeiro nome em cinza escuro, restante em vermelho
    partes = nome.split(None, 1)
    draw.text(NOME_XY, partes[0], font=fn_n, fill=GR_TRUE) # Cor a ser ajustada para o nome
    if len(partes) > 1:
        # Calcula onde o primeiro nome terminou para colar o sobrenome na frente
        bbox = draw.textbbox(NOME_XY, partes[0] + " ", font=fn_n)
        draw.text((bbox[2], NOME_XY[1]), partes[1], font=fn_n, fill=RED)

    # Desenha o cargo logo abaixo
    draw.text(CARGO_XY, cargo, font=fn_c, fill=GR_TRUE) # Cor a ser ajustada para o cargo

    imagem_para_salvar = img
    lbl_preview.configure(image=_gerar_preview(img), text="")
    _status("Assinatura gerada. Clique em SALVAR.", "ok")

def salvar():
    """Abre uma caixa de diálogo para salvar a imagem como PNG no PC."""
    if imagem_para_salvar is None:
        _status("Gere a assinatura antes de salvar.", "warn")
        return
    path = filedialog.asksaveasfilename(
        defaultextension=".png",
        filetypes=[("PNG", "*.png"), ("Todos", "*.*")],
        title="Salvar assinatura",
        initialfile="assinatura_nbr.png",
    )
    if path:
        imagem_para_salvar.convert("RGB").save(path, "PNG")
        _status(f"Salvo: {os.path.basename(path)}", "ok")

# =================================================================
# INTERFACE GRÁFICA (UI)
# =================================================================

def _on_switch_tema():
    """Alterna entre modo claro e escuro."""
    ctk.set_appearance_mode("Dark" if switch_tema.get() else "Light")

def _status(msg: str, level: str = "ok"):
    """Atualiza o texto de feedback na barra lateral."""
    colors = {"ok": "#4CAF50", "warn": "#E08020", "err": "#E05050"}
    lbl_status.configure(text=msg, text_color=colors.get(level, "#888"))

# Inicialização de recursos
_preparar_assets()
UI_FONT = _carregar_fonte_ui()

# Configuração da janela principal
ctk.set_appearance_mode("Light")
ctk.set_default_color_theme("blue")

app = ctk.CTk()
app.title("NBR Group — Gerador de Assinatura Digital")
app.geometry("960x560")
# 960x560
app.resizable(False, False)

# Ícone da janela
_ico = os.path.join(ASSETS, "nbr.ico")
if os.path.exists(_ico):
    app.iconbitmap(_ico)

def _f(size: int, bold: bool = False) -> ctk.CTkFont:
    """Atalho para criar fontes da UI."""
    return ctk.CTkFont(family=UI_FONT, size=size, weight="bold" if bold else "normal")

# --- BARRA LATERAL (MENU DE ENTRADA) ---
sidebar = ctk.CTkFrame(app, width=288, corner_radius=0, fg_color=BG_SIDE)
sidebar.pack(side="left", fill="y")
sidebar.pack_propagate(False)

# Logo no topo da sidebar
_hdr = ctk.CTkFrame(sidebar, fg_color="transparent")
_hdr.pack(fill="x", padx=20, pady=(22, 0))

if os.path.exists(LOGO_PATH):
    _li = Image.open(LOGO_PATH).convert("RGBA")
    _lw, _lh = _li.size
    _tw = 130
    _th = int(_lh * _tw / _lw)
    ctk.CTkLabel(_hdr, image=ctk.CTkImage(_li, _li, size=(_tw, _th)), text="").pack(anchor="center")
else:
    ctk.CTkLabel(_hdr, text="nbr group", font=_f(18, bold=True), text_color=RED).pack(anchor="center")

ctk.CTkLabel(sidebar, text="ASSINATURA DIGITAL", font=_f(9),
             text_color=["#A8A39E", GR_LITE]).pack(pady=(5, 12))
ctk.CTkFrame(sidebar, height=2, fg_color=RED).pack(fill="x", padx=18, pady=(0, 18))

# Campos de entrada (Nome)
ctk.CTkLabel(sidebar, text="NOME COMPLETO", font=_f(10),
             text_color=["#A8A39E", GR_LITE]).pack(anchor="w", padx=22)
campo_nome = ctk.CTkEntry(
    sidebar, placeholder_text="Digite seu nome completo",
    height=38, corner_radius=7, font=_f(12),
    border_color=["#D5D0CB", GR_DARK],
    fg_color=["#F5F2EE", "#2A2825"],
)
campo_nome.pack(fill="x", padx=22, pady=(4, 14))

# Campos de entrada (Cargo)
ctk.CTkLabel(sidebar, text="CARGO / DEPARTAMENTO", font=_f(10),
             text_color=["#A8A39E", GR_LITE]).pack(anchor="w", padx=22)
campo_cargo = ctk.CTkEntry(
    sidebar, placeholder_text="Digite seu departamento",
    height=38, corner_radius=7, font=_f(12),
    border_color=["#D5D0CB", GR_DARK],
    fg_color=["#F5F2EE", "#2A2825"],
)
campo_cargo.pack(fill="x", padx=22, pady=(4, 18))

# Botão Principal
ctk.CTkButton(
    sidebar, text="GERAR ASSINATURA",
    fg_color=RED, hover_color=DK_RED,
    font=_f(12, bold=True), height=44, corner_radius=8,
    command=gerar,
).pack(fill="x", padx=22, pady=(0, 8))

# Botões secundários (Salvar e Outlook)
_row = ctk.CTkFrame(sidebar, fg_color="transparent")
_row.pack(fill="x", padx=22)
_row.columnconfigure(0, weight=1)
_row.columnconfigure(1, weight=1)

ctk.CTkButton(
    _row, text="SALVAR",
    fg_color="transparent", border_width=1,
    border_color=["#C8C3BE", GR_DARK],
    text_color=["#4A4643", "#D6D2CE"],
    hover_color=["#EDE9E4", "#262422"],
    font=_f(11), height=36, corner_radius=7,
    command=salvar,
).grid(row=0, column=0, columnspan=2, sticky="ew")

# Switch de Tema (Claro/Escuro)
_trow = ctk.CTkFrame(sidebar, fg_color="transparent")
_trow.pack(side="bottom", pady=(0, 18))

ctk.CTkLabel(_trow, text="☀", font=ctk.CTkFont(size=16),
             text_color=["#888580", "#C8C4C0"]).pack(side="left", padx=(0, 6))

switch_tema = ctk.CTkSwitch(
    _trow, text="", width=48, height=22,
    command=_on_switch_tema,
    fg_color=["#D5D0CB", "#3A3836"],
    progress_color=RED,
    button_color="#FFFFFF",
    button_hover_color="#F0EDEA",
)
switch_tema.pack(side="left")

ctk.CTkLabel(_trow, text="🌙", font=ctk.CTkFont(size=16),
             text_color=["#888580", "#C8C4C0"]).pack(side="left", padx=(6, 0))

# Feedback de status
lbl_status = ctk.CTkLabel(sidebar, text="", font=_f(10), wraplength=244, justify="left")
lbl_status.pack(side="bottom", padx=22, pady=(0, 6), anchor="w")

# --- ÁREA PRINCIPAL (PREVISÃO DA IMAGEM) ---
main = ctk.CTkFrame(app, corner_radius=0, fg_color=BG_MAIN)
main.pack(side="right", fill="both", expand=True)

preview_frame = ctk.CTkFrame(
    main, fg_color="#FFFFFF", corner_radius=6,
    width=PREVIEW_W + 4, height=PREVIEW_H + 4,
    border_width=1, border_color=["#DDD9D4", "#2C2A28"],
)
preview_frame.pack(anchor="center", expand=True)
preview_frame.pack_propagate(False)

lbl_preview = ctk.CTkLabel(preview_frame, text="")
lbl_preview.place(relx=0.5, rely=0.5, anchor="center")

# Carrega preview inicial se a imagem base existir
if os.path.exists(BASE_IMG):
    _base = Image.open(BASE_IMG)
    _tk   = _gerar_preview(_base)
    lbl_preview.configure(image=_tk)
    lbl_preview.image = _tk
else:
    lbl_preview.configure(text="assets/assinatura_base.png nao encontrada", text_color="#999")

app.mainloop()