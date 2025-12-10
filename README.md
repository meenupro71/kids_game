import json
import os
import random
import threading
try:
    import winsound
except Exception:
    winsound = None
import tkinter as tk
from tkinter import ttk, messagebox
kids_game_data= {"high_scores": {"Evaan":50, "Lucky": 0, "Evaa": 0, "Enaa": 0,"Jiya":0}, "sounds": True}
DATA_FILE = kids_game_data
def play_beep(freq=800, dur=120):
    if winsound:
        try:
            winsound.Beep(int(freq), int(dur))
        except Exception:
            pass
def load_data():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except Exception:
            return {}
    return {}
def save_data(data):
    try:
        with open(DATA_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f)
    except Exception:
        pass
class KidsGameApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Kids Learning Fun")
        self.geometry("800x500")
        self.configure(bg="#fefefe")
        self.resizable(False, False)
        self.data = load_data()
        self.sounds = self.data.get("sounds", True)
        self.high_scores = self.data.get("high_scores", {})
        self.player_names = ["Evaan","Evaa","Enaa","Lucky","Jiya"]
        self.current_player = self.player_names[0]
        self.style = ttk.Style(self)
        try:
            self.style.theme_use("clam")
        except Exception:
            pass
        self.container = tk.Frame(self, bg="#fefefe")
        self.container.pack(fill="both", expand=True)
        self.frames = {}
        for F in (StartPage, SettingsPage, GamePage, EndPage):
            page_name = F.__name__
            frame = F(parent=self.container, controller=self)
            self.frames[page_name] = frame
            frame.grid(row=0, column=0, sticky="nsew")
        self.show_frame("StartPage")
    def show_frame(self, name):
        frame = self.frames[name]
        frame.tkraise()
    def toggle_sounds(self):
        self.sounds = not self.sounds
        self.data["sounds"] = self.sounds
        save_data(self.data)
class StartPage(tk.Frame):
    def __init__(self, parent, controller: KidsGameApp):
        super().__init__(parent, bg="#fffbf0")
        self.controller = controller
        header = tk.Label(self, text="Kids Learning Fun", font=("arial", 28, "italic","bold"), bg="#fffbf0")
        header.pack(pady=(20, 10))
        subtitle = tk.Label(self, text="Play mini-games: shapes • Colors • Counting", font=("Helvetica", 14), bg="#fffbf0")
        subtitle.pack(pady=(0, 20))
        btn_frame = tk.Frame(self, bg="#fffbf0")
        btn_frame.pack()
        # Player selection dropdown
        tk.Label(btn_frame, text="Choose your name:", font=("", 12), bg="#fffbf0").grid(row=0, column=0, padx=8)
        self.player_var = tk.StringVar(value=controller.player_names[0])
        player_menu = ttk.Combobox(btn_frame, textvariable=self.player_var, values=controller.player_names, state="readonly", font=("Helvetica", 12), width=8)
        player_menu.grid(row=0, column=1, padx=8)
        start_btn = tk.Button(btn_frame, text="Start Game", width=16, height=2, bg="#62c370", fg="white",
                              font=("Helvetica", 14), command=self.start_game)
        start_btn.grid(row=0, column=2, padx=10, pady=10)
        settings_btn = tk.Button(btn_frame, text="Settings", width=12, height=2, bg="#5196d8", fg="white",
                                 font=("Helvetica", 12), command=lambda: controller.show_frame("SettingsPage"))
        settings_btn.grid(row=0, column=3, padx=10, pady=10)
        readme_btn = tk.Button(btn_frame, text="How to Play", width=12, height=2, bg="#ffb86b", fg="white",
                               font=("Helvetica", 12), command=self.show_help)
        readme_btn.grid(row=0, column=4, padx=10, pady=10)
        footer = tk.Label(self, text="Designed for ages 3-8", font=("comic sans", 10), bg="#fffbf0")
        footer.pack(side="bottom", pady=12)
    def start_game(self):
        self.controller.current_player = self.player_var.get()
        self.controller.show_frame("GamePage")
    def show_help(self):
        messagebox.showinfo("How to Play", "Choose one of the mini-games. Each round gives simple questions and large buttons.\n"
                            "Correct answers give a star and short sound (if enabled).\n"
                            "Try to get as many correct as you can!")
class SettingsPage(tk.Frame):
    def __init__(self, parent, controller: KidsGameApp):
        super().__init__(parent, bg="#fffaf6")
        self.controller = controller
        tk.Label(self, text="Settings", font=("Helvetica", 22, "bold"), bg="#fffaf6").pack(pady=12)
        self.sound_var = tk.BooleanVar(value=self.controller.sounds)
        sound_cb = tk.Checkbutton(self, text="Enable sounds", variable=self.sound_var, bg="#fffaf6",
                                  font=("Helvetica", 14), command=self.on_toggle)
        sound_cb.pack(pady=6)
        tk.Label(self, text="High Scores of all players", font=("Helvetica", 16, "underline"), bg="#fffaf6").pack(pady=(12, 6))
        self.scores_text = tk.Text(self, width=40, height=8, font=("Helvetica", 12))
        self.scores_text.pack()
        self.scores_text.configure(state="disabled")
        back = tk.Button(self, text="Back", command=lambda: controller.show_frame("StartPage"), bg="#d3d3d3",
                         font=("Helvetica", 12))
        back.pack(pady=10)
        self.update_scores()
    def on_toggle(self):
        self.controller.sounds = self.sound_var.get()
        self.controller.data["sounds"] = self.controller.sounds
        save_data(self.controller.data)
    def update_scores(self):
        data=self.controller.high_scores
        lines = []
        for name in self.controller.player_names:
            score = data.get(name, 0)
            lines.append(f"{name}: {score}")
        self.scores_text.configure(state="normal")
        self.scores_text.delete("1.0", tk.END)
        self.scores_text.insert(tk.END, "\n".join(lines) if lines else "No scores yet")
        self.scores_text.configure(state="disabled")
class GamePage(tk.Frame):
    def __init__(self, parent, controller: KidsGameApp):
        super().__init__(parent, bg="#fffef9")
        self.controller = controller
        self.score = 0
        self.rounds = 0
        # None means infinite levels (keeps going until player quits)
        self.max_rounds = None
        # level increases every few rounds to make it gradually harder
        self.level = 1
        top = tk.Frame(self, bg="#fffef9")
        top.pack(fill="x", pady=6)
        self.score_label = tk.Label(top, text="Score: 0", font=("Helvetica", 16), bg="#fffef9")
        self.score_label.pack(side="left", padx=8)
        self.level_label = tk.Label(top, text="Level: 1", font=("Helvetica", 14), bg="#fffef9")
        self.level_label.pack(side="left", padx=8)
        quit_btn = tk.Button(top, text="Quit", command=self.finish, bg="#ff7b7b")
        quit_btn.pack(side="right", padx=12)
        self.mode_var = tk.StringVar(value="colors")
        self.canvas = tk.Canvas(self, width=760, height=360, bg="#ffffff", highlightthickness=0)
        self.canvas.pack(pady=6)
        self.choice_frame = tk.Frame(self, bg="#fffef9")
        self.choice_frame.pack(pady=8)
        self.next_round()
    def clear_choices(self):
        for w in self.choice_frame.winfo_children():
            w.destroy()
    def next_round(self):
        self.rounds += 1
        # update level every 2 rounds
        self.level = 1 + (self.rounds - 1) // 2
        self.level_label.config(text=f"Level: {self.level}")
        # if max_rounds is set (not None) enforce end, otherwise keep going
        if self.max_rounds is not None and self.rounds > self.max_rounds:
            self.finish()
            return
        self.canvas.delete("all")
        self.clear_choices()
        # Route by level: 1-10 shapes, 11-20 colors, 21+ counting
        if self.level <= 10:
            self.play_shape_round()
        elif self.level <= 20:
            self.play_color_round()
        else:
            self.play_counting_round()
    def finish(self):
        name = self.controller.current_player
        prev = self.controller.high_scores.get(name, 0)
        if self.score > prev:
            self.controller.high_scores[name] = self.score
            self.controller.data["high_scores"] = self.controller.high_scores
            save_data(self.controller.data)
        self.controller.frames["EndPage"].set_results(self.score, self.rounds, name)
        self.controller.show_frame("EndPage")
    def do_correct(self):
        self.score += 1
        self.score_label.config(text=f"Score: {self.score}")
        if self.controller.sounds:
            threading.Thread(target=play_beep, args=(950, 120), daemon=True).start()
        self.canvas.create_text(380, 180, text="✅", font=("Helvetica", 80), fill="#147a19")
        self.after(700, self.next_round)
    def do_wrong(self):
        if self.score > 0:
            self.score -= 1
        self.score_label.config(text=f"Score: {self.score}")
        if self.controller.sounds:
            threading.Thread(target=play_beep, args=(400, 170), daemon=True).start()
        self.canvas.create_text(380, 180, text="✖", font=("Helvetica", 80), fill="#ff0000")
        self.after(700,self.finish)
    # --- Rounds ---
    def play_color_round(self):
        colors = {
            "Red": "#e63946",
            "Blue": "#3a86ff",
            "Green": "#299a38",
            "Yellow": "#ffd166",
            "Purple": "#9d4edd",
            "Orange": "#f4a261",
            "Brown": "#613817",
            "Gray": "#6c757d",
            "Cyan": "#0aa8bc",
            "dark cyan":"#103c42",
            "dark red": "#480808",
            "dark blue":"#0c2d62",
            "dark green":"#14431b",
            "dark orange":"#924605",
            "dark yellow":"#705108",
            "Pink": "#ff69b4"
        }
        correct_name, correct_hex = random.choice(list(colors.items()))
        # draw big square with the color
        self.canvas.create_rectangle(180, 80, 580, 280, fill=correct_hex, outline="")
        # create choices with shuffled names
        names = list(colors.keys())
        random.shuffle(names)
        options = names[:3]
        if correct_name not in options:
            options[random.randrange(len(options))] = correct_name
        for i, opt in enumerate(options):
            btn = tk.Button(self.choice_frame, text=opt, width=12, height=2, font=("Helvetica", 12),
                            command=(lambda o=opt: self.do_correct() if o == correct_name else self.do_wrong()))
            btn.grid(row=0, column=i, padx=10)
    def play_shape_round(self):
        import math
        shapes = ["Circle", "Square", "Triangle", "Star", "Rectangle", "Pentagon", "Hexagon", "Diamond"]
        correct = random.choice(shapes)
        cx, cy = 380, 175
        size = 90
        # draw the selected shape clearly
        if correct == "Circle":
            self.canvas.create_oval(cx - size, cy - size, cx + size, cy + size, fill="#6ee7b7", outline="#081C6C")
        elif correct == "Square":
            self.canvas.create_rectangle(cx - size, cy - size, cx + size, cy + size, fill="#ffd6a5", outline="#081C6C")
        elif correct == "Triangle":
            self.canvas.create_polygon(cx, cy - size, cx - size, cy + size, cx + size, cy + size, fill="#ffd6a5", outline="#081C6C")
        elif correct == "Rectangle":
            self.canvas.create_rectangle(150, 100, 610, 250, fill="#4e3a21", outline="#081C6C")
        elif correct == "Star":
            # draw a 5-point star using math
            points = []
            outer_r = size
            inner_r = size * 0.45
            for i in range(10):
                ang = math.pi / 2 + i * math.pi / 5
                r = outer_r if i % 2 == 0 else inner_r
                x = cx + r * math.cos(ang)
                y = cy - r * math.sin(ang)
                points.extend((x, y))
            self.canvas.create_polygon(points, fill="#ffd1dc", outline="")
        elif correct == "Pentagon":
            # draw a 5-sided polygon
            points = []
            for i in range(5):
                ang = 2 * math.pi * i / 5 - math.pi / 2
                x = cx + size * math.cos(ang)
                y = cy + size * math.sin(ang)
                points.extend((x, y))
            self.canvas.create_polygon(points, fill="#90ee90", outline="#081C6C")
        elif correct == "Hexagon":
            # draw a 6-sided polygon
            points = []
            for i in range(6):
                ang = 2 * math.pi * i / 6
                x = cx + size * math.cos(ang)
                y = cy + size * math.sin(ang)
                points.extend((x, y))
            self.canvas.create_polygon(points, fill="#ffb3ba", outline="#081C6C")
        elif correct == "Diamond":
            # draw a diamond (rotated square)
            points = [cx, cy - size, cx + size, cy, cx, cy + size, cx - size, cy]
            self.canvas.create_polygon(points, fill="#e6ccff", outline="#081C6C")
        # build options: include the correct shape plus two distinct distractors
        others = [s for s in shapes if s != correct]
        distractors = random.sample(others, k=2)
        options = [correct] + distractors
        random.shuffle(options)
        for i, opt in enumerate(options):
            btn = tk.Button(self.choice_frame, text=opt, width=12, height=2, font=("Helvetica", 12),
                            command=(lambda o=opt: self.do_correct() if o == correct else self.do_wrong()))
            btn.grid(row=0, column=i, padx=10)
    def play_counting_round(self):
        n = random.randint(1,40)
        spacing = 760 // (n + 1)
        for i in range(n):
            x = spacing * (i + 1)
            y = random.randint(80, 300)
            r = 20
            self.canvas.create_oval(x - r, y - r, x + r, y + r,
                                    fill=random.choice(["#e63946","#3a86ff","#299a38"]))
        # Prefer plausible distractors (n-1, n+1), bounded between 1 and 9
        plausible = []
        if n - 1 >= 1:
            plausible.append(n - 1)
        if n + 1 <= 9:
            plausible.append(n + 1)
        if n + 2 <= 9:
            plausible.append(n + 2)
        if n - 2 >= 1:
            plausible.append(n - 2)
        # pick up to two distinct distractors
        distractors = []
        for val in plausible:
            if val != n and len(distractors) < 2:
                distractors.append(val)
        # if not enough plausible distractors, fill randomly (but not equal to n)
        while len(distractors) < 2:
            v = random.randint(1,70)
            if v != n and v not in distractors:
                distractors.append(v)
        options = [n] + distractors
        random.shuffle(options)
        for i, c in enumerate(options[:3]):  
            btn = tk.Button(self.choice_frame, text=str(c), width=8, height=2, font=("comic sans", 14),
                            command=(lambda v=c: self.do_correct() if v == n else self.do_wrong()))
            btn.grid(row=0, column=i, padx=10)
class EndPage(tk.Frame):
    def __init__(self,parent,controller:KidsGameApp):
        super().__init__()
        super().__init__(parent, bg="#fffdf0")
        self.controller = controller
        tk.Label(self, text="Good job!", font=("comic sans", 26, "bold"), bg="#fffdf0").pack(pady=18)
        self.summary = tk.Label(self, text="", font=("comic sans", 16), bg="#fffdf0")
        self.summary.pack(pady=12)
        btn_frame = tk.Frame(self, bg="#fffdf0")
        btn_frame.pack(pady=8)
        again = tk.Button(btn_frame, text="Play Again", width=12, bg="#62c370", fg="white",
                          command=self.play_again)
        again.grid(row=0, column=0, padx=8)
        home = tk.Button(btn_frame, text="Home", width=12, bg="#FFFFFF",
                         command=lambda: controller.show_frame("StartPage"))
        home.grid(row=0, column=1, padx=8)
    def set_results(self, score, rounds, name):
        self.summary.config(text=f"{name}'s Score: {score}  —  Rounds: {rounds}")
    def play_again(self):
        gp = self.controller.frames["GamePage"]
        gp.score = 0
        gp.rounds = 0
        self.rounds=0
        self.level = 1 + (self.rounds - 1) // 2
        gp.score_label.config(text="Score: 0")
        self.controller.show_frame("GamePage")
def main():
    app = KidsGameApp()
    app.mainloop()
if __name__ == "__main__":
    main()
