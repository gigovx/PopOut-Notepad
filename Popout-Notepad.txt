import tkinter as tk
from tkinter import scrolledtext
import pyautogui
import winreg
import json
import os

def is_light_theme():
    """
    Checks the Windows registry for AppsUseLightTheme.
    Returns True if the system uses the light theme.
    (Used only if no configuration is set yet.)
    """
    try:
        key = winreg.OpenKey(
            winreg.HKEY_CURRENT_USER,
            r"Software\Microsoft\Windows\CurrentVersion\Themes\Personalize"
        )
        value, _ = winreg.QueryValueEx(key, "AppsUseLightTheme")
        winreg.CloseKey(key)
        return bool(value)
    except Exception:
        return True
        
class NotepadApp:
    def __init__(self, root):
        self.root = root
        self.root.overrideredirect(True)
        self.root.attributes("-topmost", True)
        
        # Use a unique configuration filename.
        self.config_file = "popout_notepad_config.json"
        self.load_config()  # Loads button_size, side, current_font, theme, x_pos, y_pos, pages, current_page, text_font_size
        
        # For backwards compatibility: if pages aren’t defined but "content" exists, use that.
        if not hasattr(self, "pages") or self.pages is None:
            if hasattr(self, "content") and self.content != "":
                self.pages = [self.content]
            else:
                self.pages = [""]
        if not hasattr(self, "current_page"):
            self.current_page = 0
        
        # Set defaults if missing.
        if not hasattr(self, "button_size"):
            self.button_size = 48
        if not hasattr(self, "side"):
            self.side = "right"
        if not hasattr(self, "current_font"):
            self.current_font = "Arial"
        if not hasattr(self, "theme"):
            self.theme = "light" if is_light_theme() else "dark"
        if not hasattr(self, "text_font_size"):
            self.text_font_size = 12  # Default text editor font size
        
        # Tkinter variables for the right-click menu.
        self.size_var = tk.IntVar(value=self.button_size)
        self.font_var = tk.StringVar(value=self.current_font)
        self.side_var = tk.StringVar(value=self.side)
        self.theme_var = tk.StringVar(value=self.theme)
        self.text_font_size_var = tk.IntVar(value=self.text_font_size)
        
        self.update_colors()
        
        # The permanent bar / hidden area dimension.
        self.hidden_size = 20
        
        self.screen_width = self.root.winfo_screenwidth()
        self.screen_height = self.root.winfo_screenheight()
        self.is_expanded = False
        
        # Set up geometry and build the UI.
        self.set_geometry_parameters()
        self.build_ui()
        self.root.bind("<Button-3>", self.show_settings_menu)
        self.check_hover()
        
        # Save text on exit.
        self.root.protocol("WM_DELETE_WINDOW", self.on_exit)
    
    def update_colors(self):
        if self.theme == "light":
            self.bg_color = "#ffffff"
            self.btn_color = "#f0f0f0"
            self.fg_color = "#000000"
            self.handle_color = "#d0d0d0"
        else:
            self.bg_color = "#333333"
            self.btn_color = "#555555"
            self.fg_color = "#ffffff"
            self.handle_color = "#444444"
        self.root.configure(bg=self.bg_color)
    
    def load_config(self):
        if os.path.exists(self.config_file):
            try:
                with open(self.config_file, "r") as f:
                    config = json.load(f)
                self.button_size = config.get("button_size", 48)
                self.side = config.get("side", "right")
                self.current_font = config.get("current_font", "Arial")
                self.theme = config.get("theme", "light")
                self.x_pos = config.get("x_pos", None)
                self.y_pos = config.get("y_pos", None)
                # If multi-page configuration exists, load that; otherwise load content.
                self.pages = config.get("pages", None)
                if self.pages is None:
                    self.content = config.get("content", "")
                    self.pages = [self.content]
                self.current_page = config.get("current_page", 0)
                self.text_font_size = config.get("text_font_size", 12)
            except Exception:
                self.button_size = 48
                self.side = "right"
                self.current_font = "Arial"
                self.theme = "light"
                self.x_pos = None
                self.y_pos = None
                self.pages = [""]
                self.current_page = 0
                self.text_font_size = 12
        else:
            self.button_size = 48
            self.side = "right"
            self.current_font = "Arial"
            self.theme = "light"
            self.x_pos = None
            self.y_pos = None
            self.pages = [""]
            self.current_page = 0
            self.text_font_size = 12
    
    def save_config(self):
        # Save current page content without trailing newline.
        if hasattr(self, "text_widget"):
            content = self.text_widget.get("1.0", tk.END).rstrip("\n")
            self.pages[self.current_page] = content
        config = {
            "button_size": self.button_size,
            "side": self.side,
            "current_font": self.current_font,
            "theme": self.theme,
            "x_pos": self.x_pos,
            "y_pos": self.y_pos,
            "pages": self.pages,
            "current_page": self.current_page,
            "text_font_size": self.text_font_size
        }
        with open(self.config_file, "w") as f:
            json.dump(config, f)
    def set_geometry_parameters(self):
        bs = self.button_size
        self.handle_height = bs // 4
        self.copy_height = bs // 2
        # Define text area height (e.g. 6 rows tall)
        self.text_height = 6 * bs
        
        # Add extra space (40 pixels) for navigation buttons.
        if self.side in ["right", "left"]:
            self.sliding_width = 4 * bs
            self.total_width = self.sliding_width + self.hidden_size
            self.full_height = self.handle_height + self.copy_height + self.text_height + 40
            if self.y_pos is None:
                self.y_pos = int((self.screen_height - self.full_height) / 2)
            if self.side == "right":
                self.x_visible = self.screen_width - self.total_width
                self.x_hidden = self.screen_width - self.hidden_size
            else:
                self.x_visible = self.hidden_size
                self.x_hidden = self.sliding_width
            self.current_offset = self.x_visible if self.is_expanded else self.x_hidden
        elif self.side in ["top", "bottom"]:
            self.sliding_height = self.handle_height + self.copy_height + self.text_height
            self.total_height = self.sliding_height + self.hidden_size
            self.full_width = 4 * bs
            if self.x_pos is None:
                self.x_pos = int((self.screen_width - self.full_width) / 2)
            if self.side == "top":
                self.y_visible = self.hidden_size
                self.y_hidden = self.sliding_height
            else:
                self.y_visible = self.screen_height - self.total_height
                self.y_hidden = self.screen_height - self.hidden_size
            self.current_offset = self.y_visible if self.is_expanded else self.y_hidden

    def update_geometry(self):
        if self.side in ["right", "left"]:
            self.root.geometry(f"{self.total_width}x{self.full_height}+{self.current_offset}+{self.y_pos}")
            if self.side == "right":
                if self.current_offset == self.x_hidden:
                    self.sideLabel.place(x=self.sliding_width, y=0, width=self.hidden_size, height=self.full_height)
                else:
                    self.sideLabel.place_forget()
            else:
                if self.current_offset == self.x_hidden:
                    self.sideLabel.place(x=0, y=0, width=self.hidden_size, height=self.full_height)
                else:
                    self.sideLabel.place_forget()
        elif self.side in ["top", "bottom"]:
            self.root.geometry(f"{self.full_width}x{self.total_height}+{self.x_pos}+{self.current_offset}")
            if self.side == "top":
                if self.current_offset == self.y_hidden:
                    self.sideLabel.place(x=0, y=self.sliding_height, width=self.full_width, height=self.hidden_size)
                else:
                    self.sideLabel.place_forget()
            else:
                if self.current_offset == self.y_hidden:
                    self.sideLabel.place(x=0, y=0, width=self.full_width, height=self.hidden_size)
                else:
                    self.sideLabel.place_forget()

    def build_ui(self):
        bs = self.button_size
        # Create the side label (e.g., "C\nA\nL\nC")
        self.sideLabel = tk.Label(
            self.root,
            text="C\nA\nL\nC",
            font=(self.current_font, 10),
            bg=self.bg_color,
            fg=self.fg_color
        )
        copy_font_size = max(bs // 3 - 2, 8)
        
        if self.side in ["right", "left"]:
            sliding_x = 0 if self.side == "right" else self.hidden_size
            # Draggable handle
            self.drag_handle = tk.Frame(self.root, bg=self.handle_color, height=self.handle_height, cursor="fleur")
            self.drag_handle.place(x=0, y=0, width=self.total_width, height=self.handle_height)
            self.drag_handle.bind("<ButtonPress-1>", self.start_move)
            self.drag_handle.bind("<B1-Motion>", self.do_move)
            
            # Copy and Paste buttons
            self.copy_button = tk.Button(
                self.root, text="Copy", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color, command=self.copy_text
            )
            self.copy_button.place(x=sliding_x, y=self.handle_height,
                                   width=self.sliding_width // 2, height=self.copy_height)
            self.paste_button = tk.Button(
                self.root, text="Paste", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color, command=self.paste_text
            )
            self.paste_button.place(x=sliding_x + self.sliding_width // 2, y=self.handle_height,
                                    width=self.sliding_width // 2, height=self.copy_height)
            
            text_y = self.handle_height + self.copy_height
            # Multi-page text widget
            self.text_widget = scrolledtext.ScrolledText(
                self.root, font=(self.current_font, self.text_font_size),
                bg=self.bg_color, fg=self.fg_color, wrap="word"
            )
            self.text_widget.place(x=sliding_x, y=text_y,
                                   width=self.sliding_width, height=self.text_height)
            self.text_widget.delete("1.0", tk.END)
            self.text_widget.insert(tk.END, self.pages[self.current_page])
            
            # Navigation buttons: Prev, Delete, New, Next
            nav_y = text_y + self.text_height + 5
            nav_x = sliding_x
            button_width = int(self.sliding_width / 4)
            self.prev_button = tk.Button(
                self.root, text="<<", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.prev_page()]
            )
            self.prev_button.place(x=nav_x, y=nav_y, width=button_width, height=30)
            self.delete_button = tk.Button(
                self.root, text="del", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.delete_page()]
            )
            self.delete_button.place(x=nav_x + button_width, y=nav_y, width=button_width, height=30)
            self.new_page_button = tk.Button(
                self.root, text="+", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.add_page()]
            )
            self.new_page_button.place(x=nav_x + 2 * button_width, y=nav_y, width=button_width, height=30)
            self.next_button = tk.Button(
                self.root, text=">>", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.next_page()]
            )
            self.next_button.place(x=nav_x + 3 * button_width, y=nav_y, width=button_width, height=30)
            
        elif self.side in ["top", "bottom"]:
            sliding_y = 0 if self.side == "top" else self.hidden_size
            self.drag_handle = tk.Frame(self.root, bg=self.handle_color, height=self.handle_height, cursor="fleur")
            self.drag_handle.place(x=0, y=sliding_y, width=self.full_width, height=self.handle_height)
            self.drag_handle.bind("<ButtonPress-1>", self.start_move)
            self.drag_handle.bind("<B1-Motion>", self.do_move)
            
            self.copy_button = tk.Button(
                self.root, text="Copy", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color, command=self.copy_text
            )
            self.copy_button.place(x=0, y=sliding_y + self.handle_height,
                                   width=self.full_width // 2, height=self.copy_height)
            self.paste_button = tk.Button(
                self.root, text="Paste", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color, command=self.paste_text
            )
            self.paste_button.place(x=self.full_width // 2, y=sliding_y + self.handle_height,
                                    width=self.full_width // 2, height=self.copy_height)
            
            text_y = sliding_y + self.handle_height + self.copy_height
            self.text_widget = scrolledtext.ScrolledText(
                self.root, font=(self.current_font, self.text_font_size),
                bg=self.bg_color, fg=self.fg_color, wrap="word"
            )
            self.text_widget.place(x=0, y=text_y,
                                   width=self.full_width, height=self.text_height)
            self.text_widget.delete("1.0", tk.END)
            self.text_widget.insert(tk.END, self.pages[self.current_page])
            
            nav_y = text_y + self.text_height + 5
            nav_x = 0
            button_width = int(self.full_width / 4)
            self.prev_button = tk.Button(
                self.root, text="<<", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.prev_page()]
            )
            self.prev_button.place(x=nav_x, y=nav_y, width=button_width, height=30)
            self.delete_button = tk.Button(
                self.root, text="del", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.delete_page()]
            )
            self.delete_button.place(x=nav_x + button_width, y=nav_y, width=button_width, height=30)
            self.new_page_button = tk.Button(
                self.root, text="+", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.add_page()]
            )
            self.new_page_button.place(x=nav_x + 2 * button_width, y=nav_y, width=button_width, height=30)
            self.next_button = tk.Button(
                self.root, text=">>", bg=self.btn_color, fg=self.fg_color,
                font=(self.current_font, copy_font_size), bd=0, relief="flat",
                activebackground=self.bg_color,
                command=lambda: [self.save_current_page(), self.next_page()]
            )
            self.next_button.place(x=nav_x + 3 * button_width, y=nav_y, width=button_width, height=30)
        
        self.update_geometry()
    def rebuild_ui(self):
        self.set_geometry_parameters()
        self.update_colors()
        for widget in self.root.winfo_children():
            widget.destroy()
        self.build_ui()
    
    def copy_text(self):
        try:
            selection = self.text_widget.get(tk.SEL_FIRST, tk.SEL_LAST)
        except tk.TclError:
            selection = self.text_widget.get("1.0", tk.END)
        self.root.clipboard_clear()
        self.root.clipboard_append(selection)
    
    def paste_text(self):
        try:
            clip = self.root.clipboard_get()
            self.text_widget.insert(tk.INSERT, clip)
        except tk.TclError:
            pass
    
    def start_move(self, event):
        if self.side in ["right", "left"]:
            self.drag_start_y = event.y_root
            self.initial_y = self.y_pos
        elif self.side in ["top", "bottom"]:
            self.drag_start_x = event.x_root
            self.initial_x = self.x_pos
    
    def do_move(self, event):
        if self.side in ["right", "left"]:
            delta = event.y_root - self.drag_start_y
            new_y = self.initial_y + delta
            new_y = max(0, min(new_y, self.screen_height - self.full_height))
            self.y_pos = new_y
        elif self.side in ["top", "bottom"]:
            delta = event.x_root - self.drag_start_x
            new_x = self.initial_x + delta
            new_x = max(0, min(new_x, self.screen_width - self.full_width))
            self.x_pos = new_x
        self.update_geometry()
        self.save_config()
    
    def on_exit(self):
        self.save_config()
        self.root.destroy()
    
    def check_hover(self):
        mouse_x, mouse_y = pyautogui.position()
        if self.side in ["right", "left"]:
            if self.is_expanded:
                full_left = self.x_visible if self.side == "right" else 0
                full_right = self.screen_width if self.side == "right" else self.total_width
                full_top = self.y_pos
                full_bottom = self.y_pos + self.full_height
                if not (full_left <= mouse_x <= full_right and full_top <= mouse_y <= full_bottom):
                    self.slide_out()
                    self.is_expanded = False
            else:
                if self.side == "right":
                    bar_left = self.screen_width - self.hidden_size
                    bar_right = self.screen_width
                else:
                    bar_left = 0
                    bar_right = self.hidden_size
                bar_top = self.y_pos
                bar_bottom = self.y_pos + self.full_height
                if (bar_left <= mouse_x <= bar_right and bar_top <= mouse_y <= bar_bottom):
                    self.slide_in()
                    self.is_expanded = True
        elif self.side in ["top", "bottom"]:
            if self.is_expanded:
                full_left = self.x_pos
                full_right = self.x_pos + self.full_width
                full_top = self.y_visible
                full_bottom = self.y_visible + self.total_height
                if not (full_left <= mouse_x <= full_right and full_top <= mouse_y <= full_bottom):
                    self.slide_out()
                    self.is_expanded = False
            else:
                if self.side == "top":
                    bar_top = self.sliding_height
                    bar_bottom = self.sliding_height + self.hidden_size
                else:
                    bar_top = self.screen_height - self.hidden_size
                    bar_bottom = self.screen_height
                bar_left = self.x_pos
                bar_right = self.x_pos + self.full_width
                if (bar_left <= mouse_x <= bar_right and bar_top <= mouse_y <= bar_bottom):
                    self.slide_in()
                    self.is_expanded = True
        self.root.after(100, self.check_hover)
    
    def slide_in(self):
        step = 10
        if self.side == "right":
            if self.current_offset > self.x_visible:
                self.current_offset = max(self.current_offset - step, self.x_visible)
                self.update_geometry()
                self.root.after(10, self.slide_in)
        elif self.side == "left":
            if self.current_offset < self.x_visible:
                self.current_offset = min(self.current_offset + step, self.x_visible)
                self.update_geometry()
                self.root.after(10, self.slide_in)
        elif self.side == "top":
            if self.current_offset < self.y_visible:
                self.current_offset = min(self.current_offset + step, self.y_visible)
                self.update_geometry()
                self.root.after(10, self.slide_in)
        elif self.side == "bottom":
            if self.current_offset > self.y_visible:
                self.current_offset = max(self.current_offset - step, self.y_visible)
                self.update_geometry()
                self.root.after(10, self.slide_in)
    
    def slide_out(self):
        step = 10
        if self.side == "right":
            if self.current_offset < self.x_hidden:
                self.current_offset = min(self.current_offset + step, self.x_hidden)
                self.update_geometry()
                self.root.after(10, self.slide_out)
        elif self.side == "left":
            if self.current_offset > self.x_hidden:
                self.current_offset = max(self.current_offset - step, self.x_hidden)
                self.update_geometry()
                self.root.after(10, self.slide_out)
        elif self.side == "top":
            if self.current_offset > self.y_hidden:
                self.current_offset = max(self.current_offset - step, self.y_hidden)
                self.update_geometry()
                self.root.after(10, self.slide_out)
        elif self.side == "bottom":
            if self.current_offset < self.y_hidden:
                self.current_offset = min(self.current_offset + step, self.y_hidden)
                self.update_geometry()
                self.root.after(10, self.slide_out)
    
    # ---------- Update Methods for Right-Click Menu ----------
    def update_theme(self, new_theme):
        self.theme = new_theme
        self.theme_var.set(new_theme)
        self.update_colors()
        self.rebuild_ui()
        self.save_config()
    
    def update_size(self, new_size):
        self.button_size = new_size
        self.size_var.set(new_size)
        self.rebuild_ui()
        self.save_config()
    
    def update_side(self, new_side):
        self.side = new_side
        self.side_var.set(new_side)
        self.rebuild_ui()
        self.save_config()
    
    def update_font(self, new_font):
        self.current_font = new_font
        self.font_var.set(new_font)
        self.rebuild_ui()
        self.save_config()
    
    def update_text_font_size(self, new_size):
        self.text_font_size = new_size
        self.text_font_size_var.set(new_size)
        if hasattr(self, "text_widget"):
            self.text_widget.config(font=(self.current_font, self.text_font_size))
        self.save_config()
    
    def show_settings_menu(self, event):
        menu = tk.Menu(self.root, tearoff=0)
        # Change Size submenu.
        size_menu = tk.Menu(menu, tearoff=0)
        for size in [48, 64, 128, 256]:
            size_menu.add_radiobutton(
                label=f"{size}x{size}",
                variable=self.size_var,
                value=size,
                command=lambda s=size: self.update_size(s)
            )
        menu.add_cascade(label="Change Size", menu=size_menu)
        # Change Font submenu.
        font_menu = tk.Menu(menu, tearoff=0)
        for font in ["Arial", "Tahoma", "Calibri", "Verdana", "Segoe UI"]:
            font_menu.add_radiobutton(
                label=font,
                variable=self.font_var,
                value=font,
                command=lambda f=font: self.update_font(f)
            )
        menu.add_cascade(label="Change Font", menu=font_menu)
        # Change Text Editor Font Size submenu.
        text_font_menu = tk.Menu(menu, tearoff=0)
        for ts in [10, 12, 14, 16, 18]:
            text_font_menu.add_radiobutton(
                label=str(ts),
                variable=self.text_font_size_var,
                value=ts,
                command=lambda ts=ts: self.update_text_font_size(ts)
            )
        menu.add_cascade(label="Text Editor Font Size", menu=text_font_menu)
        # Change Side submenu.
        side_menu = tk.Menu(menu, tearoff=0)
        for dock in ["left", "right", "top", "bottom"]:
            side_menu.add_radiobutton(
                label=dock.capitalize(),
                variable=self.side_var,
                value=dock,
                command=lambda d=dock: self.update_side(d)
            )
        menu.add_cascade(label="Change Side", menu=side_menu)
        # Change Theme submenu.
        theme_menu = tk.Menu(menu, tearoff=0)
        for th in ["light", "dark"]:
            theme_menu.add_radiobutton(
                label=th.capitalize(),
                variable=self.theme_var,
                value=th,
                command=lambda t=th: self.update_theme(t)
            )
        menu.add_cascade(label="Change Theme", menu=theme_menu)
        menu.add_command(label="Exit", command=self.on_exit)
        try:
            menu.tk_popup(event.x_root, event.y_root)
        finally:
            menu.grab_release()

    # ===== Multi-Page Functions =====
    def save_current_page(self):
        content = self.text_widget.get("1.0", tk.END).rstrip("\n")
        self.pages[self.current_page] = content

    def next_page(self):
        if self.current_page < len(self.pages) - 1:
            self.current_page += 1
            self.update_text()

    def prev_page(self):
        if self.current_page > 0:
            self.current_page -= 1
            self.update_text()

    def add_page(self):
        self.pages.append("")
        self.current_page = len(self.pages) - 1
        self.update_text()

    def delete_page(self):
        if len(self.pages) > 1:
            del self.pages[self.current_page]
            if self.current_page >= len(self.pages):
                self.current_page = len(self.pages) - 1
            self.update_text()

    def update_text(self):
        self.text_widget.delete("1.0", tk.END)
        self.text_widget.insert(tk.END, self.pages[self.current_page])

# ================= Main Execution Block =================
if __name__ == "__main__":
    root = tk.Tk()
    app = NotepadApp(root)
    root.mainloop()