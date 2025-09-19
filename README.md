# Quick_Cube
A game about a cube

from kivy.app import App
from kivy.uix.widget import Widget
from kivy.uix.screenmanager import ScreenManager, Screen, SlideTransition
from kivy.properties import NumericProperty, StringProperty, ListProperty, BooleanProperty
from kivy.clock import Clock
from kivy.uix.label import Label
from kivy.uix.button import Button
from kivy.graphics import Color, Rectangle, Ellipse, Triangle
from kivy.core.window import Window
import random


Window.size = (800, 480)


PLAYER_SIZE = 40
PLAYER_SPEED = 8
OBSTACLE_BASE_SPEED = 2
OBSTACLE_SPAWN_INTERVAL = 1.0
LEVEL_DURATION = 9999  


class Player(Widget):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.size = (PLAYER_SIZE, PLAYER_SIZE)
        with self.canvas:
            Color(0.2, 0.6, 1)
            self.rect = Rectangle(pos=self.pos, size=self.size)
        self.bind(pos=self._update_graphics, size=self._update_graphics)

    def _update_graphics(self, *args):
        self.rect.pos = self.pos
        self.rect.size = self.size

class Obstacle(Widget):
    kind = StringProperty('triangle')  
    velocity = ListProperty([0, -1])

    def __init__(self, kind='triangle', size=(40, 40), pos=(0, 0), velocity=(0, -2), **kwargs):
        super().__init__(**kwargs)
        self.kind = kind
        self.size = size
        self.pos = pos
        self.velocity = list(velocity)
        with self.canvas:
            Color(1, 0.2, 0.2)
            if self.kind == 'triangle':
                self.tri = Triangle(points=self._tri_points())
            else:
                self.ell = Ellipse(pos=self.pos, size=self.size)
        self.bind(pos=self._update_graphics, size=self._update_graphics)

    def _tri_points(self):
        x, y = self.pos
        w, h = self.size
        if self.velocity[1] < 0:
            return [x, y + h, x + w / 2.0, y, x + w, y + h]
        else:
            return [x, y, x + w / 2.0, y + h, x + w, y]

    def _update_graphics(self, *args):
        if self.kind == 'triangle':
            self.tri.points = self._tri_points()
        else:
            self.ell.pos = self.pos
            self.ell.size = self.size

    def move(self):
        self.x += self.velocity[0]
        self.y += self.velocity[1]

class MenuScreen(Screen):
    def on_enter(self):
        self.clear_widgets()
        label = Label(text='Quick Cube', font_size=48, pos_hint={'center_x': 0.5, 'center_y': 0.72})
        self.add_widget(label)
        desc = Label(text='A game that will make you lose it', font_size=20, pos_hint={'center_x': 0.5, 'center_y': 0.62})
        self.add_widget(desc)

        start_btn = Button(text='Start', size_hint=(0.2, 0.12), pos_hint={'center_x': 0.35, 'center_y': 0.4})
        exit_btn = Button(text='Exit', size_hint=(0.2, 0.12), pos_hint={'center_x': 0.65, 'center_y': 0.4})
        start_btn.bind(on_release=lambda *_: self.start())
        exit_btn.bind(on_release=lambda *_: App.get_running_app().stop())
        self.add_widget(start_btn)
        self.add_widget(exit_btn)

    def start(self):
        self.manager.transition = SlideTransition(direction='left')
        self.manager.current = 'difficulty'

class DifficultyScreen(Screen):
    def on_enter(self):
        self.clear_widgets()
        title = Label(text='Difficulty', font_size=36, pos_hint={'center_x': 0.5, 'center_y': 0.8})
        self.add_widget(title)

        easy_btn = Button(text='Easy', size_hint=(0.22, 0.12), pos_hint={'x': 0.08, 'y': 0.52})
        easy_desc = Label(text='Objects falling slowly from top to bottom', font_size=14, pos_hint={'x': 0.34, 'y': 0.55})
        easy_btn.bind(on_release=lambda *_: self.choose('easy'))
        self.add_widget(easy_btn)
        self.add_widget(easy_desc)

        med_btn = Button(text='Medium', size_hint=(0.22, 0.12), pos_hint={'x': 0.08, 'y': 0.36})
        med_desc = Label(text='Smaller objects falling faster', font_size=14, pos_hint={'x': 0.34, 'y': 0.38})
        med_btn.bind(on_release=lambda *_: self.choose('medium'))
        self.add_widget(med_btn)
        self.add_widget(med_desc)

        hard_btn = Button(text='Hard', size_hint=(0.22, 0.12), pos_hint={'x': 0.08, 'y': 0.2})
        hard_desc = Label(text='Objects from top and bottom at fast speeds', font_size=14, pos_hint={'x': 0.34, 'y': 0.21})
        hard_btn.bind(on_release=lambda *_: self.choose('hard'))
        self.add_widget(hard_btn)
        self.add_widget(hard_desc)

    def choose(self, level):
        self.manager.transition = SlideTransition(direction='down')
        game = self.manager.get_screen('game')
        game.setup_difficulty(level)
        self.manager.current = 'game'

class WinScreen(Screen):
    def on_enter(self):
        self.clear_widgets()
        self.add_widget(Label(text='You win', font_size=48, pos_hint={'center_x': 0.5, 'center_y': 0.6}))
        menu_btn = Button(text='Menu', size_hint=(0.2, 0.12), pos_hint={'center_x': 0.5, 'center_y': 0.4})
        menu_btn.bind(on_release=lambda *_: self.go_menu())
        self.add_widget(menu_btn)

    def go_menu(self):
        self.manager.transition = SlideTransition(direction='left')
        self.manager.current = 'menu'

class LoseScreen(Screen):
    def on_enter(self):
        self.clear_widgets()
        self.add_widget(Label(text='You lose', font_size=48, pos_hint={'center_x': 0.5, 'center_y': 0.6}))
        menu_btn = Button(text='Menu', size_hint=(0.2, 0.12), pos_hint={'center_x': 0.5, 'center_y': 0.4})
        menu_btn.bind(on_release=lambda *_: self.go_menu())
        self.add_widget(menu_btn)

    def go_menu(self):
        self.manager.transition = SlideTransition(direction='left')
        self.manager.current = 'menu'

class GameScreen(Screen):
    difficulty = StringProperty('easy')
    level_index = NumericProperty(1)
    is_running = BooleanProperty(False)

    def on_enter(self):
        self.clear_widgets()
        self.game_widget = Widget()
        self.add_widget(self.game_widget)
        self.player = Player()
        self.player.pos = (10, (self.game_widget.height - PLAYER_SIZE) / 2) if self.game_widget.height else (10, Window.height/2)
        self.game_widget.add_widget(self.player)
        self.obstacles = []
        self._keyboard = Window.request_keyboard(self._keyboard_closed, self)
        if self._keyboard:
            self._keyboard.bind(on_key_down=self._on_key_down)
        self.is_running = True
        self._spawn_ev = None
        Clock.schedule_once(self._start_level, 0.2)
        Clock.schedule_interval(self._update, 1.0 / 60.0)

    def on_leave(self):
        self.is_running = False
        Clock.unschedule(self._update)
        if hasattr(self, '_spawn_ev') and self._spawn_ev:
            self._spawn_ev.cancel()
        if hasattr(self, '_keyboard') and self._keyboard:
            self._keyboard.unbind(on_key_down=self._on_key_down)
            self._keyboard.release()

    def _keyboard_closed(self):
        pass

    def _on_key_down(self, keyboard, keycode, text, modifiers):
        name = keycode[1] if isinstance(keycode, tuple) else keycode
        if keycode[1] == 'left':
            self.player.x = max(0, self.player.x - PLAYER_SPEED)
        elif keycode[1] == 'right':
            max_x = self.game_widget.width - self.player.width
            self.player.x = min(max_x, self.player.x + PLAYER_SPEED)
        return True

    def setup_difficulty(self, diff):
        self.difficulty = diff
        self.level_index = 1

    def _start_level(self, dt):
        self.player.pos = (10, (self.game_widget.height - PLAYER_SIZE) / 2)
        for o in list(self.obstacles):
            if o.parent:
                self.game_widget.remove_widget(o)
        self.obstacles = []
        if self.difficulty == 'easy':
            self.levels = [1]
        elif self.difficulty == 'medium':
            self.levels = [1, 2]
        else:
            self.levels = [1, 2, 3]
        self.level_index = 1
        self._configure_level(self.level_index)

    def _configure_level(self, level):
        if level == 1:
            self.spawn_interval = 1.2
            self.obstacle_speed = 2
            self.obstacle_size = (80, 80)  
            self.bidirectional = False
        elif level == 2:
            self.spawn_interval = 0.8
            self.obstacle_speed = 4
            self.obstacle_size = (40, 40)  
            self.bidirectional = False
        else: 
            self.spawn_interval = 0.5
            self.obstacle_speed = 6
            self.obstacle_size = (36, 36)
            self.bidirectional = True
        if hasattr(self, '_spawn_ev') and self._spawn_ev:
            try:
                self._spawn_ev.cancel()
            except Exception:
                pass
        self._spawn_ev = Clock.schedule_interval(self._spawn_obstacle, self.spawn_interval)

    def _spawn_obstacle(self, dt):
        w = self.game_widget.width or Window.width
        h = self.game_widget.height or Window.height
        x = random.randint(0, int(max(0, w - self.obstacle_size[0])))
        kind = 'triangle' if random.random() < 0.8 else 'circle'
        if (not self.bidirectional) or random.random() < 0.7:
            y = h
            vel = (0, -self.obstacle_speed)
        else:
            y = -self.obstacle_size[1]
            vel = (0, self.obstacle_speed)
        ob = Obstacle(kind=kind, size=self.obstacle_size, pos=(x, y), velocity=vel)
        self.obstacles.append(ob)
        self.game_widget.add_widget(ob)

    def _update(self, dt):
        if not self.is_running:
            return
        for ob in list(self.obstacles):
            ob.move()
            if ob.y < -200 or ob.y > self.game_widget.height + 200:
                if ob.parent:
                    self.game_widget.remove_widget(ob)
                self.obstacles.remove(ob)
        for ob in self.obstacles:
            if self._collide(self.player, ob):
                self.is_running = False
                self.manager.transition = SlideTransition(direction='left')
                self.manager.current = 'lose'
                return
        if self.player.right >= (self.game_widget.width - 5):
            current_level_pos = self.levels.index(self.level_index) if self.level_index in self.levels else None
            next_index = None
            try:
                idx = self.levels.index(self.level_index)
                if idx + 1 < len(self.levels):
                    next_index = self.levels[idx + 1]
            except ValueError:
                pos = self.level_index
                if pos < max(self.levels):
                    next_index = pos + 1
            if next_index:
                self.level_index = next_index
                self._configure_level(self.level_index)
                self.player.x = 10
                for ob in list(self.obstacles):
                    if ob.parent:
                        self.game_widget.remove_widget(ob)
                self.obstacles = []
            else:
                self.is_running = False
                self.manager.transition = SlideTransition(direction='left')
                self.manager.current = 'win'
                return

    def _collide(self, widget1, widget2):
        x1, y1 = widget1.x, widget1.y
        w1, h1 = widget1.width, widget1.height
        x2, y2 = widget2.x, widget2.y
        w2, h2 = widget2.width, widget2.height
        if (x1 < x2 + w2 and x1 + w1 > x2 and y1 < y2 + h2 and y1 + h1 > y2):
            return True
        return False

class QuickCubeApp(App):
    def build(self):
        sm = ScreenManager()
        sm.add_widget(MenuScreen(name='menu'))
        sm.add_widget(DifficultyScreen(name='difficulty'))
        sm.add_widget(GameScreen(name='game'))
        sm.add_widget(WinScreen(name='win'))
        sm.add_widget(LoseScreen(name='lose'))
        return sm

if __name__ == '__main__':
    QuickCubeApp().run()
