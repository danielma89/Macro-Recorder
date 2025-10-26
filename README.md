#!/usr/bin/env python3
import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, filedialog
import threading, time, json, math
from pynput import mouse, keyboard
from pynput.mouse import Controller as MouseCtl
from pynput.keyboard import Controller as KeyCtl, Key
from pynput.keyboard import GlobalHotKeys

APP_TITLE = "Linux Macro Recorder v3.9"

# ========================== Core ==========================
class Core:
    def __init__(self):
        self.reset()
        # filtros/execução
        self.view_moves  = True
        self.view_clicks = True
        self.view_keys   = True
        self.play_no_sleep = False
        self.play_min_sleep_ms = 5
        self.speed = 1.0
        self.group_clicks = True   # novo: agrupar press+release

    def reset(self):
        self.events_raw = []     # gravação bruta
        self._last_time = None
        self.recording = False
        self._stop_flag = threading.Event()

    # ------------ gravação ------------
    def start_record(self):
        self.reset()
        self._stop_flag.clear()
        self.recording = True
        self._last_time = time.time()
        self.m_listener = mouse.Listener(on_move=self._on_move,
                                         on_click=self._on_click,
                                         on_scroll=self._on_scroll)
        self.k_listener = keyboard.Listener(on_press=self._on_kp,
                                            on_release=self._on_kr)
        self.m_listener.start(); self.k_listener.start()

    def stop_record(self):
        self._stop_flag.set()
        if self.recording:
            self.recording = False
            try: self.m_listener.stop(); self.k_listener.stop()
            except: pass

    def _stamp(self):
        now = time.time()
        dt = now - (self._last_time or now)
        self._last_time = now
        return max(0.0, dt)

    def _push(self, ev):
        self.events_raw.append(ev)

    def _on_move(self, x,y):
        self._push({"type":"mouse_move", "dt": self._stamp(), "x":x, "y":y})
    def _on_click(self, x,y,btn,pressed):
        self._push({"type":"mouse_click", "dt": self._stamp(), "x":x, "y":y,
                    "button": str(btn), "pressed": bool(pressed)})
    def _on_scroll(self, x,y,dx,dy):
        self._push({"type":"mouse_scroll", "dt": self._stamp(), "x":x, "y":y, "dx":dx, "dy":dy})
    def _on_kp(self, key):
        self._push({"type":"key_press", "dt": self._stamp(), "key": str(key)})
    def _on_kr(self, key):
        self._push({"type":"key_release", "dt": self._stamp(), "key": str(key)})

    # ------- filtragem + compactação + dt -------
    def get_filtered_compacted(self):
        """
        Aplica filtros de visibilidade, compõe dt relativo.
        Compacta:
          - movimentos contínuos -> mouse_path
          - (opcional) clique press+release -> click_simple
        """
        kept, total, last_kept = [], 0.0, 0.0
        for e in self.events_raw:
            total += float(e.get("dt",0.0))
            t = e["type"]
            show = ((t in ("mouse_move","mouse_scroll") and self.view_moves) or
                    (t == "mouse_click" and self.view_clicks) or
                    (t.startswith("key_") and self.view_keys))
            if show:
                ev = dict(e); ev["dt"] = round(total - last_kept, 6)
                kept.append(ev); last_kept = total

        # movimentos -> path
        compact = []
        i = 0
        while i < len(kept):
            e = kept[i]
            if e["type"] == "mouse_move":
                x0,y0 = e["x"], e["y"]; x1,y1 = x0,y0; dt_sum = e["dt"]
                j = i+1
                while j < len(kept) and kept[j]["type"]=="mouse_move":
                    dt_sum += kept[j]["dt"]; x1,y1 = kept[j]["x"], kept[j]["y"]; j += 1
                compact.append({"type":"mouse_path","dt":dt_sum,"x0":x0,"y0":y0,"x1":x1,"y1":y1})
                i = j
            else:
                compact.append(e); i += 1

        # cliques -> simples (se habilitado)
        if self.group_clicks:
            grouped = []
            i = 0
            while i < len(compact):
                e = compact[i]
                if e["type"]=="mouse_click" and e.get("pressed") is True:
                    # procurar release correspondente
                    j = i+1
                    found = None
                    while j < len(compact):
                        ee = compact[j]
                        if ee["type"]=="mouse_click" and ee.get("pressed") is False \
                           and ee.get("button")==e.get("button"):
                            # usa as coords do release (normalmente iguais)
                            found = (j, ee); break
                        if ee["type"]!="mouse_click":
                            break
                        j += 1
                    if found:
                        j_idx, ee = found
                        grouped.append({"type":"click_simple","dt":e["dt"],"x":ee["x"],"y":ee["y"],"button":e["button"],"hold":0.02})
                        i = j_idx+1
                        continue
                grouped.append(e); i += 1
            compact = grouped

        return compact

    # -------- util --------
    def _parse_key(self, s):
        s = str(s)
        if s.startswith('Key.'):
            name = s.split('.', 1)[1]
            return getattr(Key, name, s)
        if s.startswith("'") and s.endswith("'") and len(s) == 3:
            return s[1]
        return s

    # --------------- execução ---------------
    def play_once(self, status_cb=lambda s: None,
                  stop_on_mouse_move=False, move_tol_px=20,
                  stop_on_key=False,
                  dt_overrides=None) -> bool:
        """
        Reproduz uma vez.
        Retorna True se foi abortado por input do usuário (mouse/tecla), senão False.
        """
        evs = self.get_filtered_compacted()
        if not evs:
            status_cb("Nada para reproduzir.")
            return False

        # aplica delays editados (se não ignorar delays)
        if isinstance(dt_overrides, dict) and not self.play_no_sleep:
            for i, e in enumerate(evs):
                if i in dt_overrides:
                    e["dt"] = float(dt_overrides[i])

        m = MouseCtl()
        k = KeyCtl()
        time.sleep(0.03)

        ref_pos = m.position
        last_prog_move_ts = 0.0
        GRACE_MS = 80
        min_step = max(0.005, float(self.play_min_sleep_ms) / 1000.0)

        key_flag = {"hit": False}
        kl = None
        if stop_on_key:
            def on_k(_k): key_flag["hit"] = True
            kl = keyboard.Listener(on_press=on_k)
            kl.start()

        def user_moved_now() -> bool:
            if not stop_on_mouse_move:
                return False
            if (time.time() - last_prog_move_ts) * 1000.0 < GRACE_MS:
                return False
            x, y = m.position
            dx, dy = x - ref_pos[0], y - ref_pos[1]
            return math.hypot(dx, dy) > move_tol_px

        def sleep_with_abort(total_s: float) -> bool:
            if total_s <= 0:
                return False
            quantum = 0.01
            remaining = total_s
            while remaining > 0:
                if self._stop_flag.is_set() or key_flag["hit"] or user_moved_now():
                    return True
                dt = quantum if remaining > quantum else remaining
                time.sleep(dt)
                remaining -= dt
            return False

        aborted = False
        try:
            for e in evs:
                if self._stop_flag.is_set() or key_flag["hit"] or user_moved_now():
                    aborted = True
                    break

                raw_dt = float(e.get("dt", 0.0))
                eff = 0.0 if self.play_no_sleep else max(raw_dt / max(0.01, self.speed), min_step)
                if sleep_with_abort(eff):
                    aborted = True
                    break

                t = e["type"]
                try:
                    if t == "mouse_path":
                        m.position = (int(e["x1"]), int(e["y1"]))
                        last_prog_move_ts = time.time()
                        ref_pos = m.position
                    elif t == "click_simple":
                        btn = mouse.Button.left
                        btxt = e.get("button","")
                        if "right" in btxt: btn = mouse.Button.right
                        elif "middle" in btxt: btn = mouse.Button.middle
                        m.position = (int(e["x"]), int(e["y"]))
                        last_prog_move_ts = time.time(); ref_pos = m.position
                        m.press(btn)
                        time.sleep(e.get("hold", 0.02))
                        m.release(btn)
                    elif t == "mouse_click":
                        btn = mouse.Button.left
                        btxt = e.get("button","")
                        if "right" in btxt: btn = mouse.Button.right
                        elif "middle" in btxt: btn = mouse.Button.middle
                        m.position = (int(e["x"]), int(e["y"]))
                        last_prog_move_ts = time.time(); ref_pos = m.position
                        (m.press if e.get("pressed") else m.release)(btn)
                    elif t == "key_press":
                        k.press(self._parse_key(e["key"]))
                    elif t == "key_release":
                        k.release(self._parse_key(e["key"]))
                    elif t == "mouse_scroll":
                        m.scroll(e["dx"], e["dy"])
                except:
                    pass
        finally:
            try:
                if kl:
                    kl.stop()
            except:
                pass

        return aborted

# ========================== App (GUI) ==========================
class App(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title(APP_TITLE); self.geometry("1160x760")
        self.core = Core()
        self.hk_listener = None
        self.marker = None
        self.marker_size = tk.IntVar(value=12)
        self.is_playing = False

        self.last_evs = []
        self.delay_overrides = {}  # idx -> novo delay (s)

        self._build_ui()
        self._apply_hotkeys("<ctrl>+<alt>+r", "<ctrl>+<alt>+p", "<ctrl>+<alt>+s", "<ctrl>+<alt>+o")
        self.bind("<Escape>", lambda e: self.cancel_all())

    # -------- UI --------
    def _build_ui(self):
        pad = {"padx":10,"pady":6}
        root = ttk.Frame(self); root.pack(fill="both", expand=True, padx=12, pady=12)

        r=0
        top = ttk.Frame(root); top.grid(row=r, column=0, columnspan=6, sticky="we")
        self.btn_rec  = ttk.Button(top, text="⏺ Gravar", command=self.on_rec)
        self.btn_stop = ttk.Button(top, text="⏹ Parar", command=self.on_stop, state="disabled")
        self.btn_play = ttk.Button(top, text="▶ Reproduzir", command=self.on_play)
        self.btn_rec.pack(side="left", padx=4); self.btn_stop.pack(side="left", padx=4); self.btn_play.pack(side="left", padx=4)
        ttk.Button(top, text="⚙️ Atalhos…", command=self.win_hotkeys).pack(side="left", padx=10)
        ttk.Button(top, text="➕ Novo comando…", command=self.win_newcmd).pack(side="left", padx=4)
        ttk.Button(top, text="📂 Abrir", command=self.load).pack(side="right", padx=4)
        ttk.Button(top, text="💾 Salvar", command=self.save).pack(side="right", padx=4)

        r+=1
        filt = ttk.LabelFrame(root, text="Filtros / Execução")
        filt.grid(row=r, column=0, columnspan=6, sticky="we", **pad)
        self.var_mv = tk.BooleanVar(value=True)
        self.var_ck = tk.BooleanVar(value=True)
        self.var_k  = tk.BooleanVar(value=True)
        ttk.Checkbutton(filt, text="Mostrar movimentos", variable=self.var_mv, command=self.refresh).pack(side="left", padx=6)
        ttk.Checkbutton(filt, text="Mostrar cliques",   variable=self.var_ck, command=self.refresh).pack(side="left", padx=6)
        ttk.Checkbutton(filt, text="Mostrar teclas",    variable=self.var_k,  command=self.refresh).pack(side="left", padx=6)

        ttk.Label(filt, text="Velocidade:").pack(side="left", padx=(18,4))
        self.speed = tk.DoubleVar(value=1.0)
        sp = ttk.Spinbox(filt, from_=0.1, to=5.0, increment=0.1, textvariable=self.speed, width=6)
        sp.pack(side="left")
        sc = ttk.Scale(filt, from_=0.1, to=5.0, orient="horizontal",
                       command=lambda v: self.speed.set(round(float(v),2)))
        sc.set(1.0); sc.pack(side="left", padx=(8,2), ipadx=40)

        ttk.Label(filt, text="Repetições:").pack(side="left", padx=(18,4))
        self.reps = tk.IntVar(value=1)
        self.reps_box = ttk.Spinbox(filt, from_=1, to=999, increment=1, textvariable=self.reps, width=6)
        self.reps_box.pack(side="left")

        self.var_repeat_until_input = tk.BooleanVar(value=False)
        ttk.Checkbutton(filt, text="Repetir até input", variable=self.var_repeat_until_input,
                        command=self._toggle_repeat_mode).pack(side="left", padx=10)

        # --- Sem delay ---
        self.var_nosleep = tk.BooleanVar(value=False)
        ttk.Checkbutton(filt, text="Sem delay (ignorar delays)", variable=self.var_nosleep,
                        command=self.refresh).pack(side="left", padx=10)
        ttk.Label(filt, text="Sleep mín (ms):").pack(side="left", padx=(8,4))
        self.min_sleep = tk.IntVar(value=5)
        ttk.Spinbox(filt, from_=0, to=200, increment=5, textvariable=self.min_sleep, width=6,
                    command=self.refresh).pack(side="left")

        # --- agrupar cliques
        self.var_group_clicks = tk.BooleanVar(value=True)
        ttk.Checkbutton(filt, text="Clique único (agrupar press/release)",
                        variable=self.var_group_clicks, command=self.refresh).pack(side="left", padx=12)

        # ---- opções manuais de stop
        self.var_stop_on_move = tk.BooleanVar(value=False)
        ttk.Checkbutton(filt, text="Parar se mover o mouse", variable=self.var_stop_on_move).pack(side="left", padx=12)
        self.var_stop_on_key = tk.BooleanVar(value=False)
        ttk.Checkbutton(filt, text="Parar ao pressionar tecla", variable=self.var_stop_on_key).pack(side="left", padx=12)

        r+=1
        tbl = ttk.LabelFrame(root, text="Eventos (duplo clique: editar Delay; duplo clique em Detalhe/X/Y: editar)")
        tbl.grid(row=r, column=0, columnspan=6, sticky="nsew", **pad)
        root.rowconfigure(r, weight=1)
        root.columnconfigure(0, weight=1)

        cols = ("#", "Tipo", "Detalhe", "X", "Y", "Delay (s)")
        self.tv = ttk.Treeview(tbl, columns=cols, show="headings", selectmode="browse")
        for c,w in zip(cols, (60, 140, 240, 180, 180, 120)):
            self.tv.heading(c, text=c); self.tv.column(c, width=w, anchor="w")
        self.tv.pack(fill="both", expand=True, padx=6, pady=6)
        self.tv.bind("<<TreeviewSelect>>", self.on_select_row)
        self.tv.bind("<Double-1>", self.on_double_click_cell)

        # menu de contexto
        self._ctx = tk.Menu(self, tearoff=0)
        self._ctx.add_command(label="Excluir selecionado", command=self.delete_selected)
        self.tv.bind("<Button-3>", self._popup_ctx)

        # mover ↑/↓
        btns = ttk.Frame(root); btns.grid(row=r+1, column=0, sticky="w", padx=10)
        ttk.Button(btns, text="↑ Mover", command=lambda: self.move_selected(-1)).pack(side="left", padx=4)
        ttk.Button(btns, text="↓ Mover", command=lambda: self.move_selected(+1)).pack(side="left", padx=4)

        r+=2
        self.status = tk.StringVar(value="Pronto.  (Esc = cancelar tudo)")
        ttk.Label(root, textvariable=self.status).grid(row=r, column=0, sticky="w", padx=10)

    def _toggle_repeat_mode(self):
        if self.var_repeat_until_input.get():
            self.reps_box.configure(state="disabled")
        else:
            self.reps_box.configure(state="normal")

    # ------ util ------
    @staticmethod
    def _click_side(e):
        if "right" in e.get("button",""): return "direito"
        if "middle" in e.get("button",""): return "meio"
        return "esquerdo"

    # ------ salvar / abrir ------
    def save(self):
        try:
            path = filedialog.asksaveasfilename(
                title="Salvar macro",
                defaultextension=".json",
                filetypes=[("JSON", "*.json")]
            )
            if not path: return
            with open(path, "w") as f:
                json.dump(self.core.events_raw, f, indent=2)
            self.status.set(f"Salvo em {path}")
        except Exception as e:
            messagebox.showerror("Erro ao salvar", str(e))

    def load(self):
        try:
            path = filedialog.askopenfilename(
                title="Abrir macro",
                filetypes=[("JSON", "*.json"), ("Todos", "*.*")]
            )
            if not path: return
            with open(path) as f:
                self.core.events_raw = json.load(f)
            self.status.set(f"Carregado: {len(self.core.events_raw)} eventos")
            self.refresh()
        except Exception as e:
            messagebox.showerror("Erro ao abrir", str(e))

    # ------ ações ------
    def cancel_all(self):
        self.core._stop_flag.set()
        if self.core.recording:
            self.core.stop_record()
        self.is_playing = False
        self.btn_rec.config(state="normal"); self.btn_play.config(state="normal"); self.btn_stop.config(state="disabled")
        self._hide_marker()
        self.status.set("Cancelado.")

    def on_rec(self):
        if self.core.recording: return
        self._hide_marker()
        self.core.start_record()
        self.status.set("Gravando… (Esc = cancelar)")
        self.btn_rec.config(state="disabled"); self.btn_play.config(state="disabled"); self.btn_stop.config(state="normal")

    def on_stop(self):
        if not self.core.recording:
            return
        self.core._stop_flag.set()
        self.core.stop_record()
        self.status.set("Parado.")
        self.btn_rec.config(state="normal"); self.btn_play.config(state="normal"); self.btn_stop.config(state="disabled")
        self.refresh()

    def on_play(self):
        if self.core.recording or self.is_playing: return
        self._hide_marker()
        self.core._stop_flag.clear()
        self.core.view_moves  = self.var_mv.get()
        self.core.view_clicks = self.var_ck.get()
        self.core.view_keys   = self.var_k.get()
        self.core.play_no_sleep = self.var_nosleep.get()
        self.core.play_min_sleep_ms = self.min_sleep.get()
        self.core.speed = float(self.speed.get())
        self.core.group_clicks = self.var_group_clicks.get()

        if self.var_repeat_until_input.get():
            stop_on_move = True
            stop_on_key  = True
            reps = 10**9
        else:
            stop_on_move = self.var_stop_on_move.get()
            stop_on_key  = self.var_stop_on_key.get()
            reps = max(1, int(self.reps.get()))

        self.is_playing = True
        self.btn_rec.config(state="disabled"); self.btn_play.config(state="disabled"); self.btn_stop.config(state="disabled")
        def run():
            try:
                for i in range(reps):
                    if self.core._stop_flag.is_set() or not self.is_playing: break
                    self.status.set(f"PLAY {i+1}… (Esc = cancelar)")
                    overrides = None if self.var_nosleep.get() else self.delay_overrides
                    aborted = self.core.play_once(
                        self.status.set,
                        stop_on_mouse_move=stop_on_move,
                        move_tol_px=20,
                        stop_on_key=stop_on_key,
                        dt_overrides=overrides
                    )
                    if self.var_repeat_until_input.get() and aborted:
                        break
                self.status.set("Reprodução concluída.")
            finally:
                self.is_playing = False
                self.btn_rec.config(state="normal"); self.btn_play.config(state="normal"); self.btn_stop.config(state="disabled")
        threading.Thread(target=run, daemon=True).start()

    # ------ atualizar tabela ------
    def refresh(self):
        self.core.view_moves  = self.var_mv.get()
        self.core.view_clicks = self.var_ck.get()
        self.core.view_keys   = self.var_k.get()
        self.core.group_clicks = self.var_group_clicks.get()
        for i in self.tv.get_children(): self.tv.delete(i)

        evs = self.core.get_filtered_compacted()
        self.last_evs = evs

        show_min = self.var_nosleep.get()
        min_sleep = max(0.0, float(self.min_sleep.get())/1000.0)

        for idx, e in enumerate(evs, start=1):
            t = e["type"]
            if t=="click_simple":
                typ="Clique"; det="único"; x=e["x"]; y=e["y"]
            elif t=="mouse_click":
                typ="Clique"; det="press" if e.get("pressed") else "release"; x=e["x"]; y=e["y"]
            elif t=="mouse_path":
                typ="Movimento"; det=""
                x = f"{int(e['x0'])} \u2192 {int(e['x1'])}"
                y = f"{int(e['y0'])} \u2192 {int(e['y1'])}"
            elif t in ("key_press","key_release"):
                typ="Tecla"; det=str(e.get("key","")).strip("'"); x=""; y=""
            elif t=="mouse_scroll":
                typ="Scroll"; det=f"dx={e['dx']} dy={e['dy']}"; x=e["x"]; y=e["y"]
            else:
                typ=t; det=""; x=""; y=""

            delay = min_sleep if show_min else float(self.delay_overrides.get(idx-1, e.get("dt",0.0)))
            self.tv.insert("", "end", values=(f"{idx:03d}", typ, det, x, y, f"{delay:.3f}"))

    # ------ marcador ------
    def on_select_row(self, _evt=None):
        if self.core.recording or self.is_playing: return
        item = self._selected_item()
        if not item: return
        row = int(self.tv.item(item,"values")[0]) - 1
        evs = self.last_evs
        if not (0 <= row < len(evs)): return
        e = evs[row]
        if e["type"] in ("click_simple","mouse_click"):
            self._show_marker(int(e.get("x")), int(e.get("y")))
        elif e["type"] == "mouse_path":
            self._show_marker(int(e["x1"]), int(e["y1"]))
        else:
            self._hide_marker()

    def _show_marker(self, x, y):
        self._hide_marker()
        s = max(6, int(self.marker_size.get()))
        self.marker = tk.Toplevel(self)
        self.marker.overrideredirect(True); self.marker.attributes("-topmost", True)
        self.marker.geometry(f"{s}x{s}+{x - s//2}+{y - s//2}")
        c = tk.Canvas(self.marker, width=s, height=s, highlightthickness=0, bg="#222")
        c.pack()
        c.create_oval(1,1,s-1,s-1, fill="#ff2d2d", outline="#ff2d2d")

    def _hide_marker(self):
        try:
            if self.marker: self.marker.destroy()
        except: pass
        self.marker = None

    # ------ edição / contexto ------
    def _popup_ctx(self, evt):
        try:
            row_id = self.tv.identify_row(evt.y)
            if row_id:
                self.tv.selection_set(row_id)
                self._ctx.tk_popup(evt.x_root, evt.y_root)
        finally:
            self._ctx.grab_release()

    def delete_selected(self):
        item = self._selected_item()
        if not item: return
        idx = int(self.tv.item(item,"values")[0]) - 1
        if not (0 <= idx < len(self.last_evs)): return
        e = self.last_evs[idx]
        t = e["type"]

        removed = False
        if t == "mouse_path":
            # remove bloco de mouse_move correspondente
            self._remove_mouse_path_block(idx); removed = True
        elif t == "click_simple":
            # remove press+release do RAW
            removed = self._remove_click_simple_block(idx)
        elif t == "mouse_click":
            # remove a ocorrência exata
            x, y, pressed, btn = int(e["x"]), int(e["y"]), bool(e.get("pressed")), str(e.get("button"))
            for i, ev in enumerate(self.core.events_raw):
                if ev.get("type")=="mouse_click" and bool(ev.get("pressed"))==pressed \
                   and int(ev.get("x",-1))==x and int(ev.get("y",-1))==y and str(ev.get("button"))==btn:
                    del self.core.events_raw[i]; removed=True; break
        elif t in ("key_press","key_release"):
            k = str(e.get("key",""))
            for i, ev in enumerate(self.core.events_raw):
                if ev.get("type")==t and str(ev.get("key",""))==k:
                    del self.core.events_raw[i]; removed=True; break

        if removed:
            self.status.set("Linha excluída."); self.refresh()
        else:
            messagebox.showwarning("Não encontrado", "Não consegui localizar o bloco no RAW.")

    def _remove_mouse_path_block(self, idx_comp):
        """Remove do RAW o bloco de mouse_move que forma aquele path compactado."""
        kept = self.core.get_filtered_compacted()
        n_block=-1
        for i,e in enumerate(kept):
            if e["type"]=="mouse_path": n_block += 1
            if i==idx_comp: break
        if n_block<0: return
        # varre RAW e elimina o n-ésimo bloco consecutivo de mouse_move
        blk_idx=-1; in_blk=False; start=None; end=None
        for i,ev in enumerate(self.core.events_raw+ [{"type":"_sentinel"}]):
            if ev.get("type")=="mouse_move":
                if not in_blk: in_blk=True; start=i
            else:
                if in_blk:
                    in_blk=False; end=i
                    blk_idx += 1
                    if blk_idx==n_block:
                        del self.core.events_raw[start:end]
                        return

    def _remove_click_simple_block(self, idx_comp):
        """Remove press+release do RAW para o clique-único apontado pelo compacto."""
        # conta quantos click_simple até idx_comp
        cnt=-1
        for i,e in enumerate(self.last_evs):
            if e["type"]=="click_simple": cnt+=1
            if i==idx_comp: break
        if cnt<0: return False
        # percorre RAW procurando pares press->release
        seen=-1; i=0
        while i < len(self.core.events_raw)-1:
            a=self.core.events_raw[i]; b=self.core.events_raw[i+1]
            if a.get("type")=="mouse_click" and a.get("pressed") is True \
               and b.get("type")=="mouse_click" and b.get("pressed") is False \
               and a.get("button")==b.get("button"):
                seen+=1
                if seen==cnt:
                    del self.core.events_raw[i:i+2]
                    return True
                i+=2; continue
            i+=1
        return False

    def move_selected(self, delta):
        """Move o bloco selecionado (path ou clique-único) ↑/↓ no RAW."""
        item = self._selected_item()
        if not item: return
        idx = int(self.tv.item(item,"values")[0]) - 1
        if not (0 <= idx < len(self.last_evs)): return
        e = self.last_evs[idx]

        # identifica range no RAW
        rng = None
        if e["type"]=="mouse_path":
            rng = self._raw_range_for_mouse_path(idx)
        elif e["type"]=="click_simple":
            rng = self._raw_range_for_click_simple(idx)
        else:
            # evento unitário
            rng = self._raw_range_for_single(idx)

        if not rng: return
        a,b = rng  # [a:b]
        # destino
        target_idx = idx + delta
        if not (0 <= target_idx < len(self.last_evs)): return
        e2 = self.last_evs[target_idx]
        rng2 = None
        if e2["type"]=="mouse_path":
            rng2 = self._raw_range_for_mouse_path(target_idx)
        elif e2["type"]=="click_simple":
            rng2 = self._raw_range_for_click_simple(target_idx)
        else:
            rng2 = self._raw_range_for_single(target_idx)
        if not rng2: return
        c,d = rng2

        raw = self.core.events_raw
        block1 = raw[a:b]
        block2 = raw[c:d]
        if delta < 0:  # mover para cima: coloca block1 antes do block2 anterior
            raw[a:b] = []
            c = c - (b - a)
            raw[c:c] = block1
        else:         # mover para baixo
            raw[c:d] = []
            a = a - (d - c)
            raw[a:a] = block2
        self.refresh()
        self.status.set("Movido.")

    # --- helpers de range ---
    def _raw_range_for_mouse_path(self, idx_comp):
        kept = self.core.get_filtered_compacted()
        n=-1
        for i,e in enumerate(kept):
            if e["type"]=="mouse_path": n+=1
            if i==idx_comp: break
        if n<0: return None
        blk=-1; in_blk=False; start=None
        for i,ev in enumerate(self.core.events_raw+ [{"type":"_sentinel"}]):
            if ev.get("type")=="mouse_move":
                if not in_blk: in_blk=True; start=i
            else:
                if in_blk:
                    in_blk=False; end=i; blk+=1
                    if blk==n:
                        return (start,end)
        return None

    def _raw_range_for_click_simple(self, idx_comp):
        cnt=-1
        for i,e in enumerate(self.last_evs):
            if e["type"]=="click_simple": cnt+=1
            if i==idx_comp: break
        if cnt<0: return None
        seen=-1; i=0
        while i < len(self.core.events_raw)-1:
            a=self.core.events_raw[i]; b=self.core.events_raw[i+1]
            if a.get("type")=="mouse_click" and a.get("pressed") is True \
               and b.get("type")=="mouse_click" and b.get("pressed") is False \
               and a.get("button")==b.get("button"):
                seen+=1
                if seen==cnt:
                    return (i, i+2)
                i+=2; continue
            i+=1
        return None

    def _raw_range_for_single(self, idx_comp):
        """Mapeia índice compacto -> índice RAW aproximado (heurística simples)."""
        # percorre compactos e RAW em paralelo contando eventos 'visíveis'
        count=-1; j=0
        for i,e in enumerate(self.core.events_raw):
            t=e["type"]
            show=((t in ("mouse_move","mouse_scroll") and self.var_mv.get()) or
                  (t=="mouse_click" and self.var_ck.get()) or
                  (t.startswith("key_") and self.var_k.get()))
            if not show: continue
            # quando clicks agrupados estão ativos, pule releases
            if self.core.group_clicks and t=="mouse_click" and e.get("pressed") is False:
                continue
            count += 1
            if count==idx_comp:
                return (i, i+1)
        return None

    # ------ edição (delay/coordenadas) ------
    def on_double_click_cell(self, evt):
        if self.core.recording or self.is_playing: return
        col = self.tv.identify_column(evt.x)
        item = self._selected_item()
        if not item: return
        row = int(self.tv.item(item,"values")[0]) - 1
        evs = self.last_evs
        if not (0 <= row < len(evs)): return
        e = evs[row]

        if col == "#6":
            if self.var_nosleep.get():
                messagebox.showinfo("Info","Com 'Sem delay' ativo os delays são ignorados.")
                return
            old = float(self.tv.item(item,"values")[5])
            try: new = float(simpledialog.askstring("Editar delay", f"Delay atual (s): {old:.3f}\nNovo valor (s):", parent=self) or old)
            except: return
            self.delay_overrides[row] = max(0.0, new)
            self.refresh(); self.status.set("Delay atualizado.")
            return

        if col in ("#3","#4","#5"):
            if e["type"] in ("click_simple","mouse_click"):
                x = simpledialog.askinteger("Editar X", f"X atual: {int(e.get('x'))}", parent=self, initialvalue=int(e.get("x")))
                if x is None: return
                y = simpledialog.askinteger("Editar Y", f"Y atual: {int(e.get('y'))}", parent=self, initialvalue=int(e.get("y")))
                if y is None: return
                # aplica no RAW: procura evento de clique correspondente e atualiza ambos press/release (se existirem)
                self._update_click_coords_in_raw(row, x, y)
                self.refresh(); self.status.set("Clique atualizado.")
            elif e["type"] == "mouse_path":
                x1 = simpledialog.askinteger("Editar destino X", f"X' atual: {int(e['x1'])}", parent=self, initialvalue=int(e["x1"]))
                if x1 is None: return
                y1 = simpledialog.askinteger("Editar destino Y", f"Y' atual: {int(e['y1'])}", parent=self, initialvalue=int(e["y1"]))
                if y1 is None: return
                self._apply_path_edit_to_raw(row, x1, y1)
                self.refresh(); self.status.set("Movimento atualizado.")

    def _update_click_coords_in_raw(self, idx_comp, x, y):
        # tenta achar par press+release do clique-único
        r = self._raw_range_for_click_simple(idx_comp)
        if r:
            a,b = r
            for i in range(a,b):
                if self.core.events_raw[i].get("type")=="mouse_click":
                    self.core.events_raw[i]["x"]=int(x); self.core.events_raw[i]["y"]=int(y)
            return
        # fallback: single
        r = self._raw_range_for_single(idx_comp)
        if r:
            i,_ = r
            if self.core.events_raw[i].get("type")=="mouse_click":
                self.core.events_raw[i]["x"]=int(x); self.core.events_raw[i]["y"]=int(y)

    def _apply_path_edit_to_raw(self, idx_comp, x1, y1):
        r = self._raw_range_for_mouse_path(idx_comp)
        if not r: return
        a,b = r
        # atualiza o último ponto do bloco
        last = b-1
        self.core.events_raw[last]["x"]=int(x1)
        self.core.events_raw[last]["y"]=int(y1)

    # ------ atalhos globais ------
    def win_hotkeys(self):
        w = tk.Toplevel(self); w.title("Atalhos globais"); w.resizable(False,False)
        pad={"padx":8,"pady":6}
        ttk.Label(w, text="Formato: <ctrl>+<alt>+r   (Esc é fixo: cancela tudo)").grid(row=0,column=0,columnspan=2,**pad)
        ttk.Label(w, text="Iniciar gravação:").grid(row=1,column=0,sticky="e",**pad)
        ttk.Label(w, text="Reproduzir:").grid(row=2,column=0,sticky="e",**pad)
        ttk.Label(w, text="Parar (apenas gravando):").grid(row=3,column=0,sticky="e",**pad)
        ttk.Label(w, text="Novo comando:").grid(row=4,column=0,sticky="e",**pad)
        e1=ttk.Entry(w); e2=ttk.Entry(w); e3=ttk.Entry(w); e4=ttk.Entry(w)
        e1.insert(0,getattr(self,"_hk_rec","<ctrl>+<alt>+r"))
        e2.insert(0,getattr(self,"_hk_play","<ctrl>+<alt>+p"))
        e3.insert(0,getattr(self,"_hk_stop","<ctrl>+<alt>+s"))
        e4.insert(0,getattr(self,"_hk_cmd","<ctrl>+<alt>+o"))
        e1.grid(row=1,column=1,**pad); e2.grid(row=2,column=1,**pad)
        e3.grid(row=3,column=1,**pad); e4.grid(row=4,column=1,**pad)
        ttk.Button(w, text="Aplicar", command=lambda:(self._apply_hotkeys(e1.get(),e2.get(),e3.get(),e4.get()), w.destroy())).grid(row=5,column=0,columnspan=2,pady=10)

    def _apply_hotkeys(self, hk_rec, hk_play, hk_stop, hk_cmd):
        try:
            if self.hk_listener: self.hk_listener.stop()
        except: pass
        self._hk_rec, self._hk_play, self._hk_stop, self._hk_cmd = hk_rec, hk_play, hk_stop, hk_cmd
        def stop_wrapper():
            if self.core.recording:
                self.on_stop()
        mapping = {
            hk_rec: self.on_rec,
            hk_play: self.on_play,
            hk_stop: stop_wrapper,
            hk_cmd: self.win_newcmd,
            "<esc>": self.cancel_all,
        }
        try:
            self.hk_listener = GlobalHotKeys(mapping); self.hk_listener.start()
            self.status.set("Atalhos aplicados. (Esc = cancelar tudo)")
        except Exception as e:
            self.status.set(f"Erro nos atalhos: {e}")

    # ------ novo comando (manual) ------
    def win_newcmd(self):
        w = tk.Toplevel(self); w.title("Novo comando"); w.resizable(False, False)
        pad={"padx":8,"pady":6}
        typ = tk.StringVar(value="click")
        ttk.Radiobutton(w,text="Click",value="click",variable=typ).grid(row=0,column=0,sticky="w",**pad)
        ttk.Radiobutton(w,text="Tecla",value="key",variable=typ).grid(row=0,column=1,sticky="w",**pad)
        ttk.Radiobutton(w,text="Delay",value="delay",variable=typ).grid(row=0,column=2,sticky="w",**pad)
        ttk.Label(w,text="X:").grid(row=1,column=0,sticky="e",**pad); ex=ttk.Entry(w,width=8); ex.grid(row=1,column=1,**pad)
        ttk.Label(w,text="Y:").grid(row=1,column=2,sticky="e",**pad); ey=ttk.Entry(w,width=8); ey.grid(row=1,column=3,**pad)
        ttk.Label(w,text="Botão (left/right/middle):").grid(row=2,column=0,columnspan=2,sticky="e",**pad); ebtn=ttk.Entry(w,width=12); ebtn.grid(row=2,column=2,columnspan=2,**pad)
        ttk.Label(w,text="Tecla (ex: a, Key.enter):").grid(row=3,column=0,columnspan=2,sticky="e",**pad); ekey=ttk.Entry(w,width=18); ekey.grid(row=3,column=2,columnspan=2,**pad)
        ttk.Label(w,text="Delay anterior (s):").grid(row=4,column=0,columnspan=2,sticky="e",**pad); edt=ttk.Entry(w,width=10); edt.insert(0,"0.2"); edt.grid(row=4,column=2,columnspan=2,**pad)
        def add():
            try: dt=float(edt.get())
            except: dt=0.2
            if typ.get()=="delay":
                messagebox.showinfo("Info","Delays ficam por linha — edite na coluna Delay.")
            elif typ.get()=="click":
                x=int(ex.get()); y=int(ey.get()); b=ebtn.get().strip().lower() or "left"
                btn="Button.left" if b.startswith("l") else ("Button.right" if b.startswith("r") else "Button.middle")
                self.core.events_raw.append({"type":"mouse_click","dt":dt,"x":x,"y":y,"button":btn,"pressed":True})
                self.core.events_raw.append({"type":"mouse_click","dt":0.02,"x":x,"y":y,"button":btn,"pressed":False})
            else:
                keytxt=ekey.get().strip() or "'a'"
                self.core.events_raw.append({"type":"key_press","dt":dt,"key":keytxt})
                self.core.events_raw.append({"type":"key_release","dt":0.05,"key":keytxt})
            self.refresh(); w.destroy()
        ttk.Button(w,text="Adicionar",command=add).grid(row=5,column=0,columnspan=4,pady=10)

    # ------ helpers gerais ------
    def _selected_item(self):
        sel = self.tv.selection()
        return sel[0] if sel else None

    def destroy(self):
        try:
            if self.hk_listener: self.hk_listener.stop()
        except: pass
        self.core.stop_record()
        super().destroy()


if __name__ == "__main__":
    app = App()
    app.refresh()
    app.mainloop()
