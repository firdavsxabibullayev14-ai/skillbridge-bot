import pygame
import random
import sys
import math

# Pygame va Mikserni ishga tushirish
pygame.init()
pygame.mixer.init()

# ----------------------------------------------------
# GLOBAL KONSTANTALAR VA SOZLAMALAR
# ----------------------------------------------------
SCREEN_WIDTH = 900
SCREEN_HEIGHT = 500
FPS = 60

# Ranglar Palitrasi (RGB)
COLOR_BLACK = (10, 10, 15)
COLOR_WHITE = (255, 255, 255)
COLOR_CYAN = (0, 245, 255)
COLOR_GOLD = (255, 215, 0)
COLOR_RED = (255, 50, 60)
COLOR_GREEN = (50, 220, 100)
COLOR_PURPLE = (140, 40, 220)
COLOR_GRAY = (40, 40, 50)
COLOR_DARK_GRAY = (25, 25, 30)

screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
pygame.display.set_caption("Geometry Dash: Ultimate Expanded Edition")
clock = pygame.time.Clock()

# ----------------------------------------------------
# O'YINCHI MA'LUMOTLARI VA PROFILI
# ----------------------------------------------------
PLAYER_PROFILE = {
    "diamonds": 200,
    "unlocked_levels": [1],
    "selected_skin": "Standard Cube",
    "unlocked_skins": ["Standard Cube"]
}

SKIN_DATABASE = {
    "Standard Cube": {
        "price": 0,
        "color": COLOR_CYAN,
        "type": "cube",
        "desc": "Boshlang'ich standart kubik"
    },
    "Neon Mouse": {
        "price": 50,
        "color": (255, 100, 200),
        "type": "mouse",
        "desc": "Tezkor sichqoncha shakli"
    },
    "Racer Arrow": {
        "price": 100,
        "color": COLOR_GOLD,
        "type": "arrow",
        "desc": "Poyga rejimi uchun ideal"
    },
    "Emerald Shield": {
        "price": 150,
        "color": COLOR_GREEN,
        "type": "circle",
        "desc": "Aylanuvchi yorqin doira"
    }
}

# 20 ta Level Generatsiyasi
LEVEL_DATABASE = {}
for lvl_idx in range(1, 21):
    base_speed = 7.0 + (lvl_idx * 0.4)
    if lvl_idx == 5:
        base_speed = 16.0  # 5-bo'lim: Poyga rejimi
    
    LEVEL_DATABASE[lvl_idx] = {
        "title": f"Level {lvl_idx}" if lvl_idx != 5 else "Level 5: POYGA SPEED",
        "speed": base_speed,
        "length": 3500 + (lvl_idx * 450),
        "best_progress": 0,
        "coins_collected": 0,
        "is_poyga": (lvl_idx == 5)
    }

# ----------------------------------------------------
# ZARRA (PARTICLE) EFFEKTI SINFI
# ----------------------------------------------------
class TrailParticle:
    def __init__(self, x_pos, y_pos, particle_color):
        self.x = x_pos
        self.y = y_pos
        self.color = particle_color
        self.radius = random.uniform(3.0, 7.0)
        self.vel_x = -random.uniform(2.0, 5.0)
        self.vel_y = random.uniform(-1.5, 1.5)
        self.alpha = 255

    def update_particle(self):
        self.x += self.vel_x
        self.y += self.vel_y
        self.alpha -= 8
        if self.radius > 0.3:
            self.radius -= 0.1

    def render_particle(self, target_surface):
        if self.alpha > 0 and self.radius > 0:
            surf_size = int(self.radius * 2) + 2
            p_surf = pygame.Surface((surf_size, surf_size), pygame.SRCALPHA)
            pygame.draw.circle(
                p_surf, 
                (*self.color, max(0, self.alpha)), 
                (int(self.radius), int(self.radius)), 
                int(self.radius)
            )
            target_surface.blit(p_surf, (self.x, self.y))

# ----------------------------------------------------
# O'YINCHI (PLAYER) SINFI
# ----------------------------------------------------
class GamePlayer:
    def __init__(self):
        self.width = 40
        self.height = 40
        self.pos_x = 120
        self.ground_y = SCREEN_HEIGHT - 100
        self.pos_y = self.ground_y - self.height
        
        self.vel_y = 0.0
        self.gravity_force = 1.15
        self.jump_power = -16.5
        self.is_in_air = False
        self.rotation_angle = 0

    def trigger_jump(self):
        if not self.is_in_air:
            self.vel_y = self.jump_power
            self.is_in_air = True

    def update_physics(self):
        self.vel_y += self.gravity_force
        self.pos_y += self.vel_y

        if self.pos_y >= self.ground_y - self.height:
            self.pos_y = self.ground_y - self.height
            self.vel_y = 0.0
            self.is_in_air = False
            self.rotation_angle = 0
            
        if self.is_in_air:
            self.rotation_angle += 7

    def draw_player(self, target_surface):
        skin_meta = SKIN_DATABASE[PLAYER_PROFILE["selected_skin"]]
        draw_color = skin_meta["color"]
        shape_type = skin_meta["type"]
        
        p_canvas = pygame.Surface((self.width, self.height), pygame.SRCALPHA)
        
        if shape_type == "cube":
            pygame.draw.rect(p_canvas, draw_color, (0, 0, self.width, self.height))
            pygame.draw.rect(p_canvas, COLOR_WHITE, (0, 0, self.width, self.height), 2)
        elif shape_type == "mouse":
            pts = [(self.width, self.height // 2), (0, 0), (0, self.height)]
            pygame.draw.polygon(p_canvas, draw_color, pts)
            pygame.draw.polygon(p_canvas, COLOR_WHITE, pts, 1)
        elif shape_type == "arrow":
            pts = [(self.width // 2, 0), (0, self.height), (self.width, self.height)]
            pygame.draw.polygon(p_canvas, draw_color, pts)
        else:
            pygame.draw.circle(p_canvas, draw_color, (self.width // 2, self.height // 2), self.width // 2)
            pygame.draw.circle(p_canvas, COLOR_WHITE, (self.width // 2, self.height // 2), self.width // 2, 2)

        rotated_canvas = pygame.transform.rotate(p_canvas, self.rotation_angle)
        rect_center = rotated_canvas.get_rect(
            center=(self.pos_x + self.width // 2, self.pos_y + self.height // 2)
        )
        target_surface.blit(rotated_canvas, rect_center.topleft)

# ----------------------------------------------------
# O'YIN ELEMENTLARI (TO'SIQ, TANGA, OLMOS)
# ----------------------------------------------------
class MapElement:
    def __init__(self, x_position, elem_type, base_ground):
        self.x = x_position
        self.type = elem_type
        self.ground = base_ground
        self.is_collected = False
        
        if self.type == "spike":
            self.w, self.h = 32, 42
            self.y = self.ground - self.h
        else:
            self.w, self.h = 26, 26
            self.y = self.ground - 75 - random.randint(0, 45)

    def move_element(self, speed):
        self.x -= speed

    def draw_element(self, target_surface):
        if self.is_collected:
            return
            
        if self.type == "spike":
            pts = [
                (self.x, self.y + self.h),
                (self.x + self.w // 2, self.y),
                (self.x + self.w, self.y + self.h)
            ]
            pygame.draw.polygon(target_surface, COLOR_RED, pts)
            pygame.draw.polygon(target_surface, COLOR_WHITE, pts, 1)
        elif self.type == "coin":
            cx, cy = int(self.x + self.w // 2), int(self.y + self.h // 2)
            pygame.draw.circle(target_surface, COLOR_GOLD, (cx, cy), 13)
            pygame.draw.circle(target_surface, COLOR_WHITE, (cx, cy), 13, 2)
        elif self.type == "diamond":
            cx, cy = self.x + self.w // 2, self.y + self.h // 2
            pts = [(cx, cy - 13), (cx + 10, cy), (cx, cy + 13), (cx - 10, cy)]
            pygame.draw.polygon(target_surface, COLOR_CYAN, pts)

# ----------------------------------------------------
# MENYU VA INTERFEYS FUNKSIYALARI
# ----------------------------------------------------
def draw_ui_button(surface, rect, text, bg_color, font_obj):
    pygame.draw.rect(surface, bg_color, rect, border_radius=8)
    pygame.draw.rect(surface, COLOR_WHITE, rect, width=2, border_radius=8)
    txt_surf = font_obj.render(text, True, COLOR_WHITE)
    txt_rect = txt_surf.get_rect(center=rect.center)
    surface.blit(txt_surf, txt_rect)

def run_main_menu():
    title_font = pygame.font.SysFont("Verdana", 36, bold=True)
    btn_font = pygame.font.SysFont("Verdana", 18)
    info_font = pygame.font.SysFont("Verdana", 14)

    while True:
        screen.fill(COLOR_BLACK)
        
        title_txt = title_font.render("GEOMETRY DASH ULTRA", True, COLOR_CYAN)
        screen.blit(title_txt, (SCREEN_WIDTH // 2 - title_txt.get_width() // 2, 35))
        
        dia_txt = btn_font.render(f"Olmoslar: {PLAYER_PROFILE['diamonds']} 💎", True, COLOR_GOLD)
        screen.blit(dia_txt, (30, 20))

        play_btn = pygame.Rect(100, 160, 280, 50)
        shop_btn = pygame.Rect(100, 230, 280, 50)
        exit_btn = pygame.Rect(100, 300, 280, 50)

        draw_ui_button(screen, play_btn, "1. O'YINNI BOSHLASH", COLOR_PURPLE, btn_font)
        draw_ui_button(screen, shop_btn, "2. SICHQONCHA DO'KONI", COLOR_PURPLE, btn_font)
        draw_ui_button(screen, exit_btn, "3. CHIQUV", COLOR_RED, btn_font)

        # Level 1 va Level 2 holatlari haqida info
        screen.blit(btn_font.render("DARASYALAR HOLATI:", True, COLOR_WHITE), (450, 150))
        
        lvl1_meta = LEVEL_DATABASE[1]
        l1_str = f"Lvl 1: {lvl1_meta['best_progress']}% | Tangalar: {lvl1_meta['coins_collected']}/3"
        screen.blit(info_font.render(l1_str, True, COLOR_GREEN), (450, 190))

        is_l2_open = 2 in PLAYER_PROFILE["unlocked_levels"]
        l2_str = "Lvl 2: OCHIQ 🔓" if is_l2_open else "Lvl 2: YOPYIQ 🔒 (Shart: Lvl1 100% + 3 Tanga)"
        screen.blit(info_font.render(l2_str, True, COLOR_GOLD if is_l2_open else COLOR_RED), (450, 220))

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                click_pos = pygame.mouse.get_pos()
                if play_btn.collidepoint(click_pos):
                    select_level_menu()
                elif shop_btn.collidepoint(click_pos):
                    open_skin_shop()
                elif exit_btn.collidepoint(click_pos):
                    pygame.quit()
                    sys.exit()

def open_skin_shop():
    font_main = pygame.font.SysFont("Verdana", 16)
    running_shop = True
    
    while running_shop:
        screen.fill(COLOR_DARK_GRAY)
        
        screen.blit(font_main.render(f"Olmoslaringiz: {PLAYER_PROFILE['diamonds']} 💎", True, COLOR_GOLD), (30, 20))
        screen.blit(font_main.render("Sichqoncha Turlari Do'koni (Chiqish uchun ESC)", True, COLOR_WHITE), (30, 50))

        y_offset = 110
        item_buttons = {}
        
        for skin_key, skin_val in SKIN_DATABASE.items():
            is_owned = skin_key in PLAYER_PROFILE["unlocked_skins"]
            is_selected = PLAYER_PROFILE["selected_skin"] == skin_key
            
            if is_selected:
                lbl = "JIHOZLANGAN"
                btn_col = COLOR_GREEN
            elif is_owned:
                lbl = "TANLASH"
                btn_col = COLOR_PURPLE
            else:
                lbl = f"SOTIB OLISH ({skin_val['price']} 💎)"
                btn_col = COLOR_RED

            action_btn = pygame.Rect(500, y_offset, 250, 40)
            draw_ui_button(screen, action_btn, lbl, btn_col, font_main)
            
            pygame.draw.rect(screen, skin_val["color"], (40, y_offset + 5, 30, 30))
            screen.blit(font_main.render(f"{skin_key} - {skin_val['desc']}", True, COLOR_WHITE), (90, y_offset + 8))
            
            item_buttons[skin_key] = action_btn
            y_offset += 65

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                running_shop = False
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                c_pos = pygame.mouse.get_pos()
                for key, rect_btn in item_buttons.items():
                    if rect_btn.collidepoint(c_pos):
                        if key in PLAYER_PROFILE["unlocked_skins"]:
                            PLAYER_PROFILE["selected_skin"] = key
                        elif PLAYER_PROFILE["diamonds"] >= SKIN_DATABASE[key]["price"]:
                            PLAYER_PROFILE["diamonds"] -= SKIN_DATABASE[key]["price"]
                            PLAYER_PROFILE["unlocked_skins"].append(key)
                            PLAYER_PROFILE["selected_skin"] = key

def select_level_menu():
    font_lbl = pygame.font.SysFont("Verdana", 14)
    running_sel = True
    
    while running_sel:
        screen.fill(COLOR_BLACK)
        screen.blit(font_lbl.render("DUNYO 1: Bo'limni Tanlang (Orqaga: ESC)", True, COLOR_WHITE), (30, 20))

        level_btns = {}
        for idx in range(1, 21):
            c_col = (idx - 1) % 5
            c_row = (idx - 1) // 5
            bx = 40 + c_col * 165
            by = 70 + c_row * 80
            
            rect_obj = pygame.Rect(bx, by, 145, 55)
            is_open = idx in PLAYER_PROFILE["unlocked_levels"]
            
            b_color = COLOR_GREEN if is_open else COLOR_GRAY
            pygame.draw.rect(screen, b_color, rect_obj, border_radius=6)
            
            title_str = f"Lvl {idx}" if idx != 5 else "Lvl 5 🏎️"
            screen.blit(font_lbl.render(title_str, True, COLOR_WHITE), (bx + 10, by + 10))
            
            if not is_open:
                screen.blit(font_lbl.render("🔒", True, COLOR_RED), (bx + 115, by + 10))
                
            level_btns[idx] = rect_obj

        pygame.display.flip()

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN and event.key == pygame.K_ESCAPE:
                running_sel = False
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                m_pos = pygame.mouse.get_pos()
                for num, b_rect in level_btns.items():
                    if b_rect.collidepoint(m_pos) and num in PLAYER_PROFILE["unlocked_levels"]:
                        launch_gameplay(num)
                        running_sel = False

# ----------------------------------------------------
# ASOSIY O'YIN JARAYONI (GAMEPLAY)
# ----------------------------------------------------
def launch_gameplay(level_id):
    lvl_data = LEVEL_DATABASE[level_id]
    player_obj = GamePlayer()
    particles_list = []
    map_elements = []

    # Map generatsiyasi
    curr_dist = 450
    placed_coins = 0
    while curr_dist < lvl_data["length"]:
        rnd = random.random()
        if rnd < 0.55:
            map_elements.append(MapElement(curr_dist, "spike", player_obj.ground_y))
        elif rnd < 0.78 and placed_coins < 3:
            map_elements.append(MapElement(curr_dist, "coin", player_obj.ground_y))
            placed_coins += 1
        else:
            map_elements.append(MapElement(curr_dist, "diamond", player_obj.ground_y))
            
        curr_dist += random.randint(280, 480)

    session_coins = 0
    session_diamonds = 0
    traveled_px = 0
    is_dead = False
    is_win = False
    
    font_hud = pygame.font.SysFont("Consolas", 18, bold=True)
    font_msg = pygame.font.SysFont("Verdana", 28, bold=True)

    game_running = True
    while game_running:
        clock.tick(FPS)
        screen.fill(COLOR_BLACK)

        # Inputlar (Klaviatura va Mishka)
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN:
                if event.key in (pygame.K_SPACE, pygame.K_UP):
                    if is_dead or is_win:
                        game_running = False
                    else:
                        player_obj.trigger_jump()
            if event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
                if is_dead or is_win:
                    game_running = False
                else:
                    player_obj.trigger_jump()

        if not is_dead and not is_win:
            player_obj.update_physics()
            traveled_px += lvl_data["speed"]

            # Zarralar generatsiyasi
            if not player_obj.is_in_air:
                sk_col = SKIN_DATABASE[PLAYER_PROFILE["selected_skin"]]["color"]
                particles_list.append(TrailParticle(player_obj.pos_x, player_obj.ground_y - 8, sk_col))

            for pt in particles_list[:]:
                pt.update_particle()
                if pt.alpha <= 0:
                    particles_list.remove(pt)

            # To'qnashuvlar tekshiruvi
            p_rect = pygame.Rect(player_obj.pos_x, player_obj.pos_y, player_obj.width, player_obj.height)
            
            for elem in map_elements:
                elem.move_element(lvl_data["speed"])
                if not elem.is_collected:
                    e_rect = pygame.Rect(elem.x, elem.y, elem.w, elem.h)
                    if p_rect.colliderect(e_rect):
                        if elem.type == "spike":
                            is_dead = True
                        elif elem.type == "coin":
                            elem.is_collected = True
                            session_coins += 1
                        elif elem.type == "diamond":
                            elem.is_collected = True
                            session_diamonds += 1

            # Progress foizi
            curr_progress = min(100, int((traveled_px / lvl_data["length"]) * 100))
            lvl_data["best_progress"] = max(lvl_data["best_progress"], curr_progress)

            if curr_progress >= 100:
                is_win = True
                lvl_data["coins_collected"] = max(lvl_data["coins_collected"], session_coins)
                PLAYER_PROFILE["diamonds"] += session_diamonds + 15

                # 2-bo'lim uchun qat'iy shart: Lvl 1 100% va 3 ta tanga yig'ilishi shart
                if level_id == 1 and curr_progress == 100 and lvl_data["coins_collected"] == 3:
                    if 2 not in PLAYER_PROFILE["unlocked_levels"]:
                        PLAYER_PROFILE["unlocked_levels"].append(2)
                elif level_id in PLAYER_PROFILE["unlocked_levels"] and level_id < 20:
                    if (level_id + 1) not in PLAYER_PROFILE["unlocked_levels"]:
                        PLAYER_PROFILE["unlocked_levels"].append(level_id + 1)

        # Chizishlar
        pygame.draw.rect(screen, COLOR_DARK_GRAY, (0, player_obj.ground_y, SCREEN_WIDTH, 100))
        pygame.draw.line(screen, COLOR_CYAN, (0, player_obj.ground_y), (SCREEN_WIDTH, player_obj.ground_y), 3)

        for pt in particles_list:
            pt.render_particle(screen)
            
        player_obj.draw_player(screen)
        
        for elem in map_elements:
            elem.draw_element(screen)

        # HUD ko'rsatkichlari
        pct_display = min(100, int((traveled_px / lvl_data["length"]) * 100))
        screen.blit(font_hud.render(f"PROGRESS: {pct_display}%", True, COLOR_WHITE), (20, 20))
        screen.blit(font_hud.render(f"TANGALAR: {session_coins}/3 🟡", True, COLOR_GOLD), (240, 20))
        screen.blit(font_hud.render(f"OLMOSLAR: +{session_diamonds} 💎", True, COLOR_CYAN), (470, 20))

        if is_dead:
            m_txt = font_msg.render("YUTQAZDINGIZ! (Qayta urunish uchun mishkani bosing)", True, COLOR_RED)
            screen.blit(m_txt, (SCREEN_WIDTH // 2 - m_txt.get_width() // 2, SCREEN_HEIGHT // 2 - 20))
        elif is_win:
            w_txt = font_msg.render("G'ALABA! KEYINGI BOSHQICH OCHILDI", True, COLOR_GREEN)
            screen.blit(w_txt, (SCREEN_WIDTH // 2 - w_txt.get_width() // 2, SCREEN_HEIGHT // 2 - 20))

        pygame.display.flip()

# ----------------------------------------------------
# ISHGA TUSHIRISH
# ----------------------------------------------------
if __name__ == "__main__":
    run_main_menu()
