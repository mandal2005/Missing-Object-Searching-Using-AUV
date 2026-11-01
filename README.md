# Missing-Object-Searching-Using-AUV


import pygame
import math
import random
import sys

# ---------------------------------------------------------
# Window & Configuration
# ---------------------------------------------------------
WIDTH, HEIGHT = 860, 560
SIM_HEIGHT = 520
FPS = 60

# Palette
BG_DARK = (7, 13, 25)
PANEL_BG = (15, 23, 42)
BORDER_COLOR = (30, 41, 59)
ACCENT_BLUE = (56, 189, 248)
ACCENT_AMBER = (251, 191, 36)
ACCENT_EMERALD = (52, 211, 153)
WHITE = (248, 250, 252)
SLATE_400 = (148, 163, 184)
SLATE_600 = (71, 85, 105)
SLATE_800 = (30, 41, 59)

pygame.init()
pygame.display.set_caption("AUV Reconnaissance & Uplink Telemetry Rig")
screen = pygame.display.set_mode((WIDTH, HEIGHT))
clock = pygame.time.Clock()

font_hud = pygame.font.SysFont("Courier New", 13, bold=True)
font_small = pygame.font.SysFont("monospace", 10)
font_label = pygame.font.SysFont("monospace", 9, bold=True)

# ---------------------------------------------------------
# Simulation Variables
# ---------------------------------------------------------
sim_time = 0

auv = {
    "x": 80.0,
    "y": 375.0,
    "vx": 1.35,
    "state": "SEARCHING",  # SEARCHING, FOUND
    "prop_angle": 0.0,
    "scanning": True,
}

target = {
    "x": 590.0,
    "y": 472.0,
    "width": 30,
    "height": 18,
    "found": False,
    "beacon_flash": 0.0,
}

status_text = "Autonomous Seabed Multibeam Sweep Active..."
hud_color = ACCENT_BLUE

signal_packets = []
particles = []
auv_bubbles = []

# Ambient Marine Snow
for _ in range(50):
    particles.append({
        "x": random.uniform(0, WIDTH),
        "y": random.uniform(115, 480),
        "size": random.uniform(1.0, 2.2),
        "speed_x": -0.15 - random.uniform(0, 0.25),
        "speed_y": 0.08 + random.uniform(0, 0.15),
        "alpha": int(255 * (0.2 + random.uniform(0, 0.35))),
    })

# Precomputed sky & water background gradient
sky_surf = pygame.Surface((WIDTH, 110))
for y in range(110):
    t = y / 110.0
    r = int((1 - t) * 7 + t * 14)
    g = int((1 - t) * 11 + t * 32)
    b = int((1 - t) * 20 + t * 52)
    pygame.draw.line(sky_surf, (r, g, b), (0, y), (WIDTH, y))

sea_surf = pygame.Surface((WIDTH, SIM_HEIGHT - 110))
for y in range(SIM_HEIGHT - 110):
    t = y / (SIM_HEIGHT - 110)
    r = int((1 - t) * 3 + t * 1)
    g = int((1 - t) * 55 + t * 9)
    b = int((1 - t) * 90 + t * 18)
    pygame.draw.line(sea_surf, (r, g, b), (0, y), (WIDTH, y))


def reset_simulation():
    global status_text, hud_color
    auv["x"] = 80.0
    auv["vx"] = 1.35
    auv["scanning"] = True
    auv["state"] = "SEARCHING"
    target["found"] = False
    signal_packets.clear()
    auv_bubbles.clear()
    status_text = "Autonomous Seabed Multibeam Sweep Active..."
    hud_color = ACCENT_BLUE


def relocate_target():
    target["x"] = 220 + random.random() * 410
    reset_simulation()


def trigger_telemetry():
    signal_packets.append({
        "x": auv["x"],
        "y": auv["y"] - 14,
        "dest_y": 124,
        "stage": "ACOUSTIC_UP",
        "radius": 5.0,
    })


# ---------------------------------------------------------
# Detailed Model 1: Autonomous Surface Vessel (ASV)
# ---------------------------------------------------------
def draw_detailed_asv(x):
    # Buoyancy heave & roll motion
    buoy_y = 110 + math.sin((x * 0.02) + (sim_time * 0.04)) * 3.5
    pitch = math.cos((x * 0.02) + (sim_time * 0.04)) * 0.04

    asv_surf = pygame.Surface((120, 90), pygame.SRCALPHA)
    cx, cy = 60, 40

    # Submerged Keel Strut & Acoustic Transducer Bulb
    pygame.draw.rect(asv_surf, (51, 65, 85), (cx - 3, cy + 8, 6, 26))
    pygame.draw.rect(asv_surf, (30, 41, 59), (cx - 4, cy + 28, 8, 4))
    pygame.draw.circle(asv_surf, (30, 41, 59), (cx, cy + 34), 6)
    pygame.draw.circle(asv_surf, ACCENT_BLUE, (cx, cy + 34), 3)

    # Submerged Lower Keel Weight
    pygame.draw.ellipse(asv_surf, (30, 41, 59), (cx - 14, cy + 32, 28, 8))

    # Wave-Piercing Catamaran Hull
    hull_pts = [
        (cx - 38, cy + 2),
        (cx + 36, cy + 2),
        (cx + 42, cy + 8),
        (cx + 28, cy + 12),
        (cx - 28, cy + 12),
        (cx - 34, cy + 6),
    ]
    pygame.draw.polygon(asv_surf, (226, 232, 240), hull_pts)
    pygame.draw.polygon(asv_surf, (148, 163, 184), hull_pts, 1)

    # High-Visibility Survey Orange Trim
    pygame.draw.polygon(asv_surf, (249, 115, 22), [
        (cx - 25, cy + 2), (cx + 25, cy + 2), (cx + 22, cy + 6), (cx - 22, cy + 6)
    ])

    # Superstructure Deckhouse
    pygame.draw.polygon(asv_surf, (203, 213, 225), [
        (cx - 18, cy + 2), (cx + 14, cy + 2), (cx + 10, cy - 7), (cx - 15, cy - 7)
    ])

    # Solar Array Panel on Foredeck
    pygame.draw.polygon(asv_surf, (30, 58, 138), [
        (cx - 34, cy + 2), (cx - 20, cy + 2), (cx - 17, cy - 3), (cx - 30, cy - 3)
    ])
    # Solar Grid Lines
    pygame.draw.line(asv_surf, (96, 165, 250), (cx - 28, cy + 2), (cx - 24, cy - 3), 1)

    # Communications Radome (Satcom Dome)
    pygame.draw.circle(asv_surf, (241, 245, 249), (cx - 4, cy - 11), 6)
    pygame.draw.circle(asv_surf, (148, 163, 184), (cx - 4, cy - 11), 6, 1)
    pygame.draw.rect(asv_surf, (100, 116, 139), (cx - 6, cy - 5, 4, 3))

    # Comms & Sensor Mast
    pygame.draw.line(asv_surf, (100, 116, 139), (cx + 8, cy + 2), (cx + 8, cy - 24), 2)
    pygame.draw.line(asv_surf, (148, 163, 184), (cx + 4, cy - 16), (cx + 12, cy - 16), 1)  # Crossarm
    pygame.draw.line(asv_surf, (100, 116, 139), (cx + 11, cy - 16), (cx + 11, cy - 22), 1)  # VHF Whip

    # Navigation Lights
    pygame.draw.circle(asv_surf, (239, 68, 68), (cx + 4, cy - 16), 2)   # Port Red
    pygame.draw.circle(asv_surf, (34, 197, 94), (cx + 12, cy - 16), 2)  # Starboard Green

    # Flashing Masthead White Strobe
    if math.sin(sim_time * 0.1) > 0.3:
        pygame.draw.circle(asv_surf, (255, 255, 255), (cx + 8, cy - 25), 3)

    # Rotate with surface wave pitch and blit
    rotated_asv = pygame.transform.rotate(asv_surf, math.degrees(-pitch))
    r_rect = rotated_asv.get_rect(center=(int(x), int(buoy_y)))
    screen.blit(rotated_asv, r_rect)

    # Waterline Foam / Ripple
    pygame.draw.ellipse(screen, (186, 230, 253), (int(x - 30), int(buoy_y + 4), 60, 4), 1)


# ---------------------------------------------------------
# Detailed Model 2: Subsea Autonomous Underwater Vehicle (AUV)
# ---------------------------------------------------------
def draw_detailed_auv():
    x, y = auv["x"], auv["y"]
    direction = 1 if auv["vx"] >= 0 else -1

    # Spawn Propeller Wake Bubbles
    if sim_time % 3 == 0:
        auv_bubbles.append({
            "x": x - direction * 44 + random.uniform(-2, 2),
            "y": y + random.uniform(-4, 4),
            "r": random.uniform(1.0, 2.8),
            "life": 30,
            "vx": -direction * random.uniform(0.8, 1.6),
            "vy": random.uniform(-0.3, 0.1),
        })

    # Render Active Bubbles
    for b in auv_bubbles[:]:
        b["x"] += b["vx"]
        b["y"] += b["vy"]
        b["life"] -= 1
        alpha = int(255 * (b["life"] / 30.0))
        b_surf = pygame.Surface((10, 10), pygame.SRCALPHA)
        pygame.draw.circle(b_surf, (224, 242, 254, alpha), (5, 5), b["r"])
        screen.blit(b_surf, (b["x"] - 5, b["y"] - 5))
        if b["life"] <= 0:
            auv_bubbles.remove(b)

    # Sonar Cone Active Field
    if auv["scanning"]:
        sonar_surf = pygame.Surface((WIDTH, SIM_HEIGHT), pygame.SRCALPHA)
        cone_pts = [(x, y + 10), (x - 90, 485), (x + 90, 485)]
        pygame.draw.polygon(sonar_surf, (56, 189, 248, 35), cone_pts)
        for ring in [80, 120, 160]:
            arc_rect = pygame.Rect(x - ring, y + 10 - ring, ring * 2, ring * 2)
            pygame.draw.arc(sonar_surf, (56, 189, 248, 45), arc_rect, 1.25 * math.pi, 1.75 * math.pi, 2)
        screen.blit(sonar_surf, (0, 0))

    # AUV Main Body Surface
    auv_surf = pygame.Surface((120, 60), pygame.SRCALPHA)
    cx, cy = 60, 30

    # 1. Shrouded Propulsor Ducted Nozzle
    prop_x = cx - direction * 35
    pygame.draw.ellipse(auv_surf, (51, 65, 85), (prop_x - 4, cy - 8, 8, 16))
    pygame.draw.ellipse(auv_surf, (30, 41, 59), (prop_x - 3, cy - 7, 6, 14), 1)

    # Rotating Propeller Hub & Blades
    auv["prop_angle"] += 0.45
    b_len = 6 * math.sin(auv["prop_angle"])
    pygame.draw.line(auv_surf, (217, 119, 6), (prop_x, cy - b_len), (prop_x, cy + b_len), 3)
    pygame.draw.circle(auv_surf, (71, 85, 105), (int(prop_x), cy), 3)

    # 2. Cruciform / X-Fin Control Stabilizers
    fin_poly_top = [
        (cx - direction * 22, cy - 8),
        (cx - direction * 32, cy - 18),
        (cx - direction * 38, cy - 17),
        (cx - direction * 28, cy - 8),
    ]
    fin_poly_bot = [
        (cx - direction * 22, cy + 8),
        (cx - direction * 32, cy + 18),
        (cx - direction * 38, cy + 17),
        (cx - direction * 28, cy + 8),
    ]
    pygame.draw.polygon(auv_surf, (71, 85, 105), fin_poly_top)
    pygame.draw.polygon(auv_surf, (51, 65, 85), fin_poly_bot)

    # 3. Streamlined Fuselage Hull (Subsea Survey Yellow)
    hull_pts = [
        (cx + direction * 38, cy),          # Nose tip
        (cx + direction * 32, cy - 12),     # Foredeck curve
        (cx + direction * 10, cy - 14),
        (cx - direction * 20, cy - 12),
        (cx - direction * 28, cy - 6),      # Taper to tail
        (cx - direction * 28, cy + 6),
        (cx - direction * 20, cy + 12),
        (cx + direction * 10, cy + 14),
        (cx + direction * 32, cy + 12),
    ]
    pygame.draw.polygon(auv_surf, (234, 179, 8), hull_pts)

    # 3D Depth Shading: Lower shadow on hull
    lower_shade = [
        (cx + direction * 38, cy),
        (cx + direction * 32, cy + 12),
        (cx + direction * 10, cy + 14),
        (cx - direction * 20, cy + 12),
        (cx - direction * 28, cy + 6),
        (cx - direction * 28, cy),
        (cx + direction * 10, cy + 3),
    ]
    pygame.draw.polygon(auv_surf, (180, 83, 9), lower_shade)

    # 4. Modular Pressure Section Bands & Decals
    pygame.draw.line(auv_surf, (15, 23, 42), (cx - direction * 6, cy - 13), (cx - direction * 6, cy + 13), 2)
    pygame.draw.line(auv_surf, (15, 23, 42), (cx + direction * 14, cy - 13), (cx + direction * 14, cy + 13), 2)

    # Dark Carbon-Fiber Tech Trim Panel
    pygame.draw.rect(auv_surf, (15, 23, 42), (cx - 5, cy - 7, 10, 14), border_radius=2)
    pygame.draw.circle(auv_surf, ACCENT_EMERALD, (cx, cy), 2)  # Internal health telemetry LED

    # 5. Top Recovery Bail / Lift Hook
    pygame.draw.rect(auv_surf, (100, 116, 139), (cx - direction * 8, cy - 17, 12, 3), border_radius=1)

    # 6. Optical Dome & Camera Sensor Face
    nose_x = cx + direction * 36
    pygame.draw.circle(auv_surf, (2, 132, 199), (int(nose_x), cy), 5)
    pygame.draw.circle(auv_surf, (186, 230, 253), (int(nose_x + direction * 2), cy - 1), 2)  # Lens glint

    # Forward Headlight Glare
    pygame.draw.circle(auv_surf, (254, 240, 138), (int(nose_x - direction * 2), cy + 5), 2)

    # 7. Subsea Acoustic Transceiver & DVL Pod on Underbelly
    pygame.draw.rect(auv_surf, (71, 85, 105), (cx + direction * 2, cy + 13, 10, 4), border_radius=1)
    pygame.draw.rect(auv_surf, ACCENT_BLUE, (cx + direction * 4, cy + 16, 6, 2))

    screen.blit(auv_surf, (x - cx, y - cy))


# ---------------------------------------------------------
# Detailed Model 3: HQ Operations Station & Radar Facility
# ---------------------------------------------------------
def draw_detailed_ground_station():
    base_x = 710
    cliff_y = 65

    # 1. Stratified Coastal Rocky Outcrop
    rock_layers = [
        [(675, 110), (710, 78), (840, 78), (WIDTH, 96), (WIDTH, 110)],
        [(685, 110), (722, 85), (835, 85), (WIDTH, 102), (WIDTH, 110)],
        [(695, 110), (730, 92), (830, 92), (WIDTH, 106), (WIDTH, 110)],
    ]
    pygame.draw.polygon(screen, (30, 41, 59), rock_layers[0])
    pygame.draw.polygon(screen, (23, 31, 46), rock_layers[1])
    pygame.draw.polygon(screen, (15, 23, 42), rock_layers[2])

    # Concrete Foundation Pad
    pygame.draw.rect(screen, (71, 85, 105), (base_x - 10, cliff_y + 8, 140, 7), border_radius=1)

    # 2. Reinforced Concrete Ops Bunker
    bunker_w, bunker_h = 70, 32
    bx, by = base_x + 10, cliff_y - 24
    pygame.draw.rect(screen, (51, 65, 85), (bx, by, bunker_w, bunker_h), border_radius=3)
    pygame.draw.rect(screen, (30, 41, 59), (bx, by, bunker_w, bunker_h), 2, border_radius=3)

    # Armor Paneling Seams
    for sx in [bx + 22, bx + 46]:
        pygame.draw.line(screen, (40, 50, 68), (sx, by), (sx, by + bunker_h - 1), 1)

    # Operations Center Observation Window Array
    for win_x in [bx + 8, bx + 30, bx + 52]:
        pygame.draw.rect(screen, (15, 23, 42), (win_x, by + 9, 14, 10), border_radius=1)
        # Tactical Console Cyan Glow
        pygame.draw.rect(screen, (56, 189, 248), (win_x + 2, by + 11, 10, 6))

    # Roof Safety Catwalk & Handrails
    pygame.draw.line(screen, (148, 163, 184), (bx + 2, by - 4), (bx + bunker_w - 2, by - 4), 1)
    for rx in range(bx + 4, bx + bunker_w, 12):
        pygame.draw.line(screen, (100, 116, 139), (rx, by), (rx, by - 4), 1)

    # 3. Communications Tower & Parabolic Radar Assembly
    tower_x = base_x + 105
    tower_top_y = cliff_y - 32

    # Lattice Mast Legs
    pygame.draw.line(screen, (100, 116, 139), (tower_x - 12, cliff_y + 8), (tower_x - 4, tower_top_y), 2)
    pygame.draw.line(screen, (100, 116, 139), (tower_x + 12, cliff_y + 8), (tower_x + 4, tower_top_y), 2)
    # Tower Cross-bracing
    for ly in range(tower_top_y + 8, cliff_y + 5, 8):
        pygame.draw.line(screen, (71, 85, 105), (tower_x - 8, ly), (tower_x + 8, ly), 1)

    # Radar Pedestal Gimbal
    pygame.draw.rect(screen, (51, 65, 85), (tower_x - 7, tower_top_y - 4, 14, 6))

    # Articulated Parabolic Dish (Rotating Back & Forth)
    dish_angle = math.sin(sim_time * 0.04) * 0.35 - 0.75  # Scanning angle
    dish_surf = pygame.Surface((44, 44), pygame.SRCALPHA)
    dcx, dcy = 22, 22

    # Parabolic Curved Shell
    dish_rect = pygame.Rect(dcx - 16, dcy - 16, 32, 32)
    pygame.draw.arc(dish_surf, (226, 232, 240), dish_rect, 0.65 * math.pi, 1.35 * math.pi, 3)
    # Feed Horn Feedhorn Boom & Central Receiver
    pygame.draw.line(dish_surf, (148, 163, 184), (dcx - 4, dcy), (dcx + 10, dcy), 2)
    pygame.draw.circle(dish_surf, (248, 113, 113), (dcx + 11, dcy), 2)

    rot_dish = pygame.transform.rotate(dish_surf, math.degrees(-dish_angle))
    d_rect = rot_dish.get_rect(center=(tower_x, tower_top_y - 5))
    screen.blit(rot_dish, d_rect)

    # Red Aviation Warning Flashing Beacon atop Mast
    beacon_flash = (math.sin(sim_time * 0.15) + 1) / 2
    if beacon_flash > 0.6:
        pygame.draw.circle(screen, (239, 68, 68), (tower_x, tower_top_y - 22), 3)

    # Station Callout Label
    tag = font_label.render("COASTAL RADAR HQ", True, SLATE_400)
    screen.blit(tag, (bx + 8, by + bunker_h + 3))


# ---------------------------------------------------------
# Target Object (Flight Data Recorder)
# ---------------------------------------------------------
def draw_target():
    tx, ty = target["x"], target["y"]
    w, h = target["width"], target["height"]

    # Protective Cylindrical Chassis (Emergency Orange)
    body_rect = pygame.Rect(int(tx - w / 2), int(ty - h / 2), w, h)
    pygame.draw.rect(screen, (234, 88, 12), body_rect, border_radius=4)
    pygame.draw.rect(screen, (154, 52, 18), body_rect, 1, border_radius=4)

    # Retroreflective Zebra Stripes
    pygame.draw.rect(screen, (255, 255, 255), (tx - 8, ty - h / 2, 4, h))
    pygame.draw.rect(screen, (255, 255, 255), (tx + 2, ty - h / 2, 4, h))

    # Underwater Locator Beacon (ULB) Module
    pygame.draw.rect(screen, (148, 163, 184), (tx + w / 2 - 2, ty - 4, 7, 8), border_radius=1)

    # High-intensity Ultrasonic Strobe
    target["beacon_flash"] += 0.05
    flash = math.sin(target["beacon_flash"])
    if flash > 0.65:
        glow_surf = pygame.Surface((28, 28), pygame.SRCALPHA)
        pygame.draw.circle(glow_surf, (56, 189, 248, 120), (14, 14), 10)
        pygame.draw.circle(glow_surf, (224, 242, 254, 255), (14, 14), 3)
        screen.blit(glow_surf, (tx + w / 2 - 4, ty - 10))

    label_col = (74, 222, 128) if target["found"] else (203, 213, 225)
    txt = font_small.render("FDR BEACON", True, label_col)
    screen.blit(txt, (tx - 30, ty + h / 2 + 5))


# ---------------------------------------------------------
# Simulation Physics & Uplink Logic
# ---------------------------------------------------------
def update_simulation():
    global sim_time, status_text, hud_color
    sim_time += 1

    if auv["state"] == "SEARCHING":
        auv["x"] += auv["vx"]

        if auv["x"] > 680:
            auv["vx"] = -abs(auv["vx"])
        elif auv["x"] < 80:
            auv["vx"] = abs(auv["vx"])

        # Acquisition Detection Check
        if abs(auv["x"] - target["x"]) < 32:
            auv["state"] = "FOUND"
            target["found"] = True
            auv["scanning"] = False
            status_text = "Target Acquired! Uplinking acoustic telemetry packet to ASV..."
            hud_color = ACCENT_BLUE
            trigger_telemetry()

    # Telemetry Progression
    for p in signal_packets[:]:
        if p["stage"] == "ACOUSTIC_UP":
            p["y"] -= 2.6
            p["radius"] += 0.35

            arc_rect = pygame.Rect(
                int(p["x"] - p["radius"]), int(p["y"] - p["radius"]),
                int(p["radius"] * 2), int(p["radius"] * 2)
            )
            pygame.draw.arc(screen, ACCENT_BLUE, arc_rect, 0.1 * math.pi, 0.9 * math.pi, 2)

            if p["y"] <= p["dest_y"]:
                p["stage"] = "SATELLITE_TO_HQ"
                p["start_x"] = p["x"]
                p["start_y"] = 90
                p["t"] = 0.0
                status_text = "ASV Surface Vessel relayed telemetry -> HQ Coastal Radar Station..."
                hud_color = ACCENT_AMBER

        elif p["stage"] == "SATELLITE_TO_HQ":
            p["t"] += 0.016
            target_x = 760
            target_y = 52

            cur_x = p["start_x"] + (target_x - p["start_x"]) * p["t"]
            arc_height = 75 * math.sin(p["t"] * math.pi)
            cur_y = (p["start_y"] + (target_y - p["start_y"]) * p["t"]) - arc_height

            # RF Data Pulse
            pygame.draw.circle(screen, ACCENT_BLUE, (int(cur_x), int(cur_y)), 4)
            halo = pygame.Surface((18, 18), pygame.SRCALPHA)
            pygame.draw.circle(halo, (56, 189, 248, 110), (9, 9), 7)
            screen.blit(halo, (int(cur_x - 9), int(cur_y - 9)))

            if p["t"] >= 1.0:
                signal_packets.remove(p)
                status_text = "Telemetry Confirmed: Flight recorder payload recorded at HQ Operations."
                hud_color = ACCENT_EMERALD


def draw_hud():
    pygame.draw.rect(screen, PANEL_BG, (0, SIM_HEIGHT, WIDTH, HEIGHT - SIM_HEIGHT))
    pygame.draw.line(screen, BORDER_COLOR, (0, SIM_HEIGHT), (WIDTH, SIM_HEIGHT), 1)

    dot_r = 4 + int(abs(math.sin(sim_time * 0.08)) * 2)
    pygame.draw.circle(screen, hud_color, (28, SIM_HEIGHT + 20), dot_r)

    hud_surf = font_hud.render(f"Telemetry: {status_text}", True, hud_color)
    screen.blit(hud_surf, (44, SIM_HEIGHT + 13))

    tip = font_small.render("[R] Rerun Mission    [T] Relocate Target", True, SLATE_400)
    screen.blit(tip, (WIDTH - tip.get_width() - 16, SIM_HEIGHT + 15))


# ---------------------------------------------------------
# Main Application Loop
# ---------------------------------------------------------
def main():
    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_r:
                    reset_simulation()
                elif event.key == pygame.K_t:
                    relocate_target()

        # 1. Sky & Oceanic Environment
        screen.blit(sky_surf, (0, 0))
        screen.blit(sea_surf, (0, 110))

        # Dynamic Water Surface Wave
        wave_pts = []
        for x in range(0, WIDTH + 10, 10):
            y = 110 + math.sin((x * 0.02) + (sim_time * 0.04)) * 3
            wave_pts.append((x, y))
        pygame.draw.lines(screen, ACCENT_BLUE, False, wave_pts, 2)

        # Marine Snow Drift
        for p in particles:
            surf = pygame.Surface((int(p["size"] * 2), int(p["size"] * 2)), pygame.SRCALPHA)
            pygame.draw.circle(surf, (224, 242, 254, p["alpha"]), (p["size"], p["size"]), p["size"])
            screen.blit(surf, (p["x"], p["y"]))
            p["x"] += p["speed_x"]
            p["y"] += p["speed_y"]
            if p["x"] < 0:
                p["x"] = WIDTH
            if p["y"] > 480:
                p["y"] = 115

        # Seabed Strata Layers
        seabed_1 = [(0, 480), (240, 465), (520, 490), (WIDTH, 475), (WIDTH, SIM_HEIGHT), (0, SIM_HEIGHT)]
        pygame.draw.polygon(screen, (15, 31, 46), seabed_1)
        seabed_2 = [(0, 490), (300, 480), (560, 500), (WIDTH, 485), (WIDTH, SIM_HEIGHT), (0, SIM_HEIGHT)]
        pygame.draw.polygon(screen, (22, 43, 61), seabed_2)

        # 2. Render Upgraded Detailed Entities
        draw_detailed_ground_station()
        draw_detailed_asv(auv["x"])
        draw_target()
        draw_detailed_auv()

        # 3. Step Physics, Uplink & HUD
        update_simulation()
        draw_hud()

        pygame.display.flip()
        clock.tick(FPS)

    pygame.quit()
    sys.exit()


if __name__ == "__main__":
    main()
