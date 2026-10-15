# CSE423_Final_Project
ECHOES OF THE DEEP – 3D SONAR EXPLORATION SIMULATOR
from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLU import *
import math
import random

# ============================================================
# CAMERA / WINDOW CONSTANTS
# ============================================================
camera_pos = (0, -400, 250)
fovY = 100
GRID_LENGTH = 2000

# ============================================================
# FEATURE 1: SUBMERSIBLE MOVEMENT & SIMPLIFIED PHYSICS
# ============================================================
# Position / orientation
sub_x, sub_y, sub_z = 0.0, 0.0, 100.0
sub_yaw = 0.0     # rotation around Z (turning left/right)
sub_pitch = 0.0   # rotation around X (nose up/down)

# Linear motion state (velocity based, not teleporting)
sub_speed = 0.0          # current forward speed
MAX_SPEED = 300.0
ACCEL = 220.0            # units/sec^2 when thrusting
DECEL = 140.0            # units/sec^2 natural drag when coasting
TURN_RATE = 70.0         # degrees/sec

# Simplified buoyancy (vertical physics)
vertical_velocity = 0.0
BUOYANCY_FORCE = 18.0     # constant upward push (submersible is positively buoyant)
GRAVITY_FORCE = 30.0      # downward pull
WATER_DRAG = 0.9          # damps vertical velocity each frame so it doesn't accelerate forever
FLOOR_Z = 0.0
CEILING_Z = 900.0

move_forward_input = 0    # -1, 0, 1 from W/S
pitch_input = 0           # -1, 0, 1 from Q/E (affects target vertical thrust)




# Last-seen timestamp for each held movement key. GLUT re-fires a keyboard
# event on auto-repeat while a key is held, so if a key hasn't been seen for
# longer than KEY_HOLD_TIMEOUT we treat it as released. This avoids needing
# any GLUT function outside the Lab Assignment 3 template (no key-up callback).
last_w_time = -1000.0
last_s_time = -1000.0
last_q_time = -1000.0
last_e_time = -1000.0
KEY_HOLD_TIMEOUT = 0.15

last_time_ms = 0
elapsed_seconds = 0.0

# ============================================================
# FEATURE 2: DYNAMIC CAVE LAYOUT GENERATION
# ============================================================
CAVE_SEED = random.randint(1, 999999)
random.seed(CAVE_SEED)

CELL_SIZE = 260
GRID_W = 7          # cave grid width  (columns)
GRID_H = 7          # cave grid height (rows, mapped onto world Y)
cave_cells = {}      # (col, row) -> True if this cell is a carved-out chamber/tunnel
formations = []      # list of (x, y, z, kind, scale) decorative stalactite/rock placements
charging_stations = []
heat_vents = []
artifacts = []


# ============================================================
# FEATURE 4: BATTERY & POWER MANAGEMENT
# ============================================================

max_battery = 100.0
battery = max_battery

PASSIVE_BATTERY_DRAIN = 1.0       # battery units per second
MOVEMENT_BATTERY_DRAIN = 2.0      # extra battery units per second while moving
SONAR_BATTERY_COST = 5.0

CHARGING_DISTANCE = 55.0
CHARGING_AMOUNT = 40.0


game_over = False
game_over_reason = ""


# ============================================================
# FEATURE 5: HAZARD AVOIDANCE SYSTEM
# ============================================================

HEAT_VENT_DANGER_RADIUS = 75.0
HEAT_VENT_DAMAGE_RATE = 15.0

hull_health = 100.0
max_hull_health = 100.0

near_heat_vent = False


# ============================================================
# FEATURE 6: ARTIFACT COLLECTION & UPGRADE SYSTEM
# ============================================================

ARTIFACT_COLLECTION_DISTANCE = 55.0

artifacts_collected = 0
artifact_score = 0

# Upgrade levels
sonar_upgrade_level = 0
battery_upgrade_level = 0
hazard_upgrade_level = 0

# Maximum upgrade levels
MAX_SONAR_UPGRADES = 4
MAX_BATTERY_UPGRADES = 4
MAX_HAZARD_UPGRADES = 4

# Artifact respawn system
ARTIFACT_RESPAWN_TIME = 5.0
artifact_respawn_timers = []

def carve_cave():
    """Grid-based procedural cave generation: creates connected chambers and
    tunnel paths while ensuring navigable exploration routes, by randomly
    flood-filling outward from an entry cell so every carved cell stays
    reachable from the start."""
    global cave_cells, formations, charging_stations, heat_vents, artifacts
    global artifact_respawn_timers

    cave_cells = {}
    formations = []
    charging_stations = []
    heat_vents = []
    artifacts = []
    artifact_respawn_timers = []

    start = (GRID_W // 2, GRID_H // 2)
    cave_cells[start] = True
    frontier = [start]
    target_cells = int(GRID_W * GRID_H * 0.55)

    while frontier and len(cave_cells) < target_cells:
        cx, cy = random.choice(frontier)
        dirs = [(1, 0), (-1, 0), (0, 1), (0, -1)]
        random.shuffle(dirs)
        carved_this_round = False

        for dx, dy in dirs:
            nx, ny = cx + dx, cy + dy

            if 0 <= nx < GRID_W and 0 <= ny < GRID_H and (nx, ny) not in cave_cells:
                cave_cells[(nx, ny)] = True
                frontier.append((nx, ny))
                carved_this_round = True
                break

        if not carved_this_round:
            frontier.remove((cx, cy))

    # Decorate carved cells with rock formations, hazards, stations, artifacts
    for (cx, cy) in cave_cells:
        wx = (cx - GRID_W / 2.0) * CELL_SIZE
        wy = (cy - GRID_H / 2.0) * CELL_SIZE

        # Stalactites / stalagmites / rocks (visual clutter, not a feature by itself)
        if random.random() < 0.5:
            kind = random.choice(["stalactite", "stalagmite", "rock"])
            fx = wx + random.uniform(-80, 80)
            fy = wy + random.uniform(-80, 80)
            scale = random.uniform(0.6, 1.4)
            formations.append((fx, fy, kind, scale))

        if random.random() < 0.10 and (cx, cy) != start:
            charging_stations.append([wx, wy, True])   # True = still charged/available

        if random.random() < 0.12 and (cx, cy) != start:
            heat_vents.append([wx, wy])

    available_cells = []

    for (cx, cy) in cave_cells:
        if (cx, cy) != start:
            available_cells.append((cx, cy))

    random.shuffle(available_cells)

    if len(available_cells) >= 3:

        # SONAR ARTIFACT
        cx, cy = available_cells[0]
        wx = (cx - GRID_W / 2.0) * CELL_SIZE
        wy = (cy - GRID_H / 2.0) * CELL_SIZE

        artifacts.append([
            wx,
            wy,
            60.0,
            True,
            "sonar"
        ])

        artifact_respawn_timers.append(0.0)

        # BATTERY ARTIFACT
        cx, cy = available_cells[1]
        wx = (cx - GRID_W / 2.0) * CELL_SIZE
        wy = (cy - GRID_H / 2.0) * CELL_SIZE

        artifacts.append([
            wx,
            wy,
            60.0,
            True,
            "battery"
        ])

        artifact_respawn_timers.append(0.0)

        # HAZARD RESISTANCE ARTIFACT
        cx, cy = available_cells[2]
        wx = (cx - GRID_W / 2.0) * CELL_SIZE
        wy = (cy - GRID_H / 2.0) * CELL_SIZE

        artifacts.append([
            wx,
            wy,
            60.0,
            True,
            "hazard"
        ])

        artifact_respawn_timers.append(0.0)

# ============================================================
# FEATURE 3: SONAR PING SYSTEM
# ============================================================
sonar_active = False
sonar_radius = 0.0
SONAR_MAX_RADIUS = 900.0
SONAR_EXPAND_SPEED = 700.0   # units/sec
SONAR_COOLDOWN_MAX = 1.5     # seconds
sonar_cooldown = 0.0

# revealed[cell] -> remaining visibility time (seconds), fades back to darkness
revealed_cells = {}
REVEAL_HOLD_TIME = 3.0       # seconds an object stays lit once the ring passes it

# Direct, per-object visibility (independent of which cave cell an object's
# world position rounds into). Keyed by (category, index_in_its_list) ->
# remaining reveal time. This is what the review flagged: artifacts/vents/
# stations sitting near a cell boundary were inheriting their cell's
# rounded-center distance instead of being checked against their own exact
# world position, so they could light up (or stay dark) a beat too early or
# late right at cell edges.
revealed_objects = {}


def trigger_sonar():
    global sonar_active, sonar_radius, sonar_cooldown, battery,game_over

    if game_over:
        return

    if sonar_cooldown <= 0.0 and not sonar_active:

        # Make sure there is enough battery for sonar
        if battery < SONAR_BATTERY_COST:
            return
        
        sonar_active = True
        sonar_radius = 0.0

        # Sonar consumes battery when activated
        battery -= SONAR_BATTERY_COST

        # if battery <= 0:
        #     battery = 0
        #     game_over = True
        #     sonar_active = False


def update_sonar(dt):
    global sonar_active, sonar_radius, sonar_cooldown, revealed_cells, revealed_objects

    if sonar_cooldown > 0.0:
        sonar_cooldown -= dt

    if sonar_active:
        sonar_radius += SONAR_EXPAND_SPEED * dt

        # As the ring passes each cave cell's center, mark the cave floor/
        # formations in that cell revealed (fine-grained enough for terrain).
        for (cx, cy) in cave_cells:
            wx = (cx - GRID_W / 2.0) * CELL_SIZE
            wy = (cy - GRID_H / 2.0) * CELL_SIZE
            dist = math.sqrt((wx - sub_x) ** 2 + (wy - sub_y) ** 2)
            if dist <= sonar_radius:
                revealed_cells[(cx, cy)] = REVEAL_HOLD_TIME

        # Distance-based detection straight against each object's own exact
        # position, so artifacts/vents/stations near a cell edge reveal at
        # the moment the ring actually reaches them, not when their cell's
        # center happens to.
        for idx, (wx, wy, charged) in enumerate(charging_stations):
            dist = math.sqrt((wx - sub_x) ** 2 + (wy - sub_y) ** 2)
            if dist <= sonar_radius:
                revealed_objects[("station", idx)] = REVEAL_HOLD_TIME

        for idx, (wx, wy) in enumerate(heat_vents):
            dist = math.sqrt((wx - sub_x) ** 2 + (wy - sub_y) ** 2)
            if dist <= sonar_radius:
                revealed_objects[("vent", idx)] = REVEAL_HOLD_TIME

        for idx, (wx, wy, wz, active, upgrade_type ) in enumerate(artifacts):
            dist = math.sqrt((wx - sub_x) ** 2 + (wy - sub_y) ** 2)
            if dist <= sonar_radius:
                revealed_objects[("artifact", idx)] = REVEAL_HOLD_TIME

        if sonar_radius >= SONAR_MAX_RADIUS:
            sonar_active = False
            sonar_radius = 0.0
            sonar_cooldown = SONAR_COOLDOWN_MAX

    # Fade previously revealed cells back into darkness over time
    dead = []
    for key in revealed_cells:
        revealed_cells[key] -= dt
        if revealed_cells[key] <= 0:
            dead.append(key)
    for key in dead:
        del revealed_cells[key]

    # Fade previously revealed objects back into darkness over time
    dead_objs = []
    for key in revealed_objects:
        revealed_objects[key] -= dt
        if revealed_objects[key] <= 0:
            dead_objs.append(key)
    for key in dead_objs:
        del revealed_objects[key]


def cell_visibility(cx, cy):
    """Returns a 0..1 brightness factor: 1 = fully lit by sonar, fading to 0.
    Used for cave floor tiles and rock/stalactite formations, where
    cell-granularity is enough."""
    if (cx, cy) in revealed_cells:
        return max(0.0, min(1.0, revealed_cells[(cx, cy)] / REVEAL_HOLD_TIME))
    return 0.0


def object_visibility(category, idx):
    """Returns a 0..1 brightness factor for a specific artifact/vent/station,
    based on a direct distance check against that object's own world
    position rather than the cave cell it happens to sit in. Keeps objects
    near a cell boundary from revealing early/late relative to the actual
    sonar ring position."""
    key = (category, idx)
    if key in revealed_objects:
        return max(0.0, min(1.0, revealed_objects[key] / REVEAL_HOLD_TIME))
    return 0.0


# ============================================================
# TEXT HELPER (unchanged pattern from Lab Assignment 3 template)
# ============================================================
def draw_text(x, y, text, font=GLUT_BITMAP_HELVETICA_18):
    glColor3f(1, 1, 1)
    glMatrixMode(GL_PROJECTION)
    glPushMatrix()
    glLoadIdentity()
    gluOrtho2D(0, 1000, 0, 800)

    glMatrixMode(GL_MODELVIEW)
    glPushMatrix()
    glLoadIdentity()

    glRasterPos2f(x, y)
    for ch in text:
        glutBitmapCharacter(font, ord(ch))

    glPopMatrix()
    glMatrixMode(GL_PROJECTION)
    glPopMatrix()
    glMatrixMode(GL_MODELVIEW)


# ============================================================
# DRAWING
# ============================================================
def draw_submersible():
    glPushMatrix()
    glTranslatef(sub_x, sub_y, sub_z)
    glRotatef(sub_yaw, 0, 0, 1)
    glRotatef(sub_pitch, 1, 0, 0)

    glColor3f(1.0, 0.75, 0.1)
    glRotatef(90, 0, 1, 0)
    gluCylinder(gluNewQuadric(), 22, 14, 70, 12, 8)

    glColor3f(0.9, 0.9, 1.0)
    glTranslatef(0, 0, 70)
    gluSphere(gluNewQuadric(), 20, 10, 10)

    glPopMatrix()


def draw_cave():
    for (cx, cy) in cave_cells:
        wx = (cx - GRID_W / 2.0) * CELL_SIZE
        wy = (cy - GRID_H / 2.0) * CELL_SIZE
        vis = cell_visibility(cx, cy)

        # Cave floor tile: dim when unrevealed, bright when lit by sonar
        base = 0.05
        r = base + vis * 0.15
        g = base + vis * 0.25
        b = base + vis * 0.35
        glPushMatrix()
        glTranslatef(wx, wy, FLOOR_Z)
        glColor3f(r, g, b)
        glScalef(CELL_SIZE * 0.95, CELL_SIZE * 0.95, 10)
        glutSolidCube(1)
        glPopMatrix()

    for (fx, fy, kind, scale) in formations:
        cx = int(round(fx / CELL_SIZE + GRID_W / 2.0))
        cy = int(round(fy / CELL_SIZE + GRID_H / 2.0))
        vis = cell_visibility(cx, cy)
        shade = 0.05 + vis * 0.5
        glPushMatrix()
        if kind == "stalactite":
            glTranslatef(fx, fy, CEILING_Z)
            glColor3f(shade * 0.8, shade * 0.6, shade * 0.9)
            glScalef(scale, scale, scale * 2.5)
            glutSolidCube(30)
        elif kind == "stalagmite":
            glTranslatef(fx, fy, FLOOR_Z + 40 * scale)
            glColor3f(shade * 0.8, shade * 0.6, shade * 0.5)
            glScalef(scale, scale, scale * 2.0)
            gluSphere(gluNewQuadric(), 20, 8, 8)
        else:
            glTranslatef(fx, fy, FLOOR_Z + 20 * scale)
            glColor3f(shade * 0.5, shade * 0.5, shade * 0.5)
            glScalef(scale, scale, scale)
            glutSolidCube(40)
        glPopMatrix()

    for idx, (wx, wy, charged) in enumerate(charging_stations):
        vis = object_visibility("station", idx)

        glPushMatrix()
        glTranslatef(wx, wy, FLOOR_Z + 40)

        if charged:
            glColor3f(
                0.1 + vis * 0.1,
                0.1 + vis * 0.8,
                0.1 + vis * 0.2
            )

        else:
            glColor3f(0.08, 0.08, 0.08)

                
        #glColor3f(0.1 + vis * 0.1, 0.1 + vis * 0.8, 0.1 + vis * 0.2)
        gluCylinder(gluNewQuadric(), 18, 18, 80, 8, 4)

        glTranslatef(0, 0, 80)
        gluSphere(gluNewQuadric(), 16, 8, 8)

        glPopMatrix()

    for idx, (wx, wy) in enumerate(heat_vents):
        vis = object_visibility("vent", idx)
        glPushMatrix()
        glTranslatef(wx, wy, FLOOR_Z + 10)
        glColor3f(0.2 + vis * 0.8, 0.05 + vis * 0.1, 0.05)
        gluCylinder(gluNewQuadric(), 25, 10, 40, 10, 6)
        glPopMatrix()

    for idx, (wx, wy, wz, active, upgrade_type) in enumerate(artifacts):
        if not active:
            continue
        vis = object_visibility("artifact", idx)

        glPushMatrix()
        glTranslatef(wx, wy, wz)

        # Different artifact appearance based on upgrade type
        if upgrade_type == "sonar":
            glColor3f(
                0.1 + vis * 0.2,
                0.4 + vis * 0.6,
                0.5 + vis * 0.5,
            )
            

        elif upgrade_type == "battery":
            glColor3f(
                0.4 + vis * 0.5,
                0.05,
                0.1 + vis * 0.2
            )

        elif upgrade_type == "hazard":
            glColor3f(
                0.2+vis*0.8,
                0.2+vis*0.8,
                0.05
            )

        #glColor3f(0.2 + vis * 0.8, 0.2 + vis * 0.8, 0.05)
        gluSphere(gluNewQuadric(), 14, 10, 10)
        glPopMatrix()


def draw_sonar_ring():
    if not sonar_active or sonar_radius <= 0:
        return
    glPushMatrix()
    glColor3f(0.2, 1.0, 1.0)
    glTranslatef(sub_x, sub_y, FLOOR_Z + 5)
    glBegin(GL_LINE_LOOP)
    segments = 48
    for i in range(segments):
        angle = 2.0 * math.pi * i / segments
        vx = sonar_radius * math.cos(angle)
        vy = sonar_radius * math.sin(angle)
        glVertex3f(vx, vy, 0)
    glEnd()
    glPopMatrix()


# ============================================================
# PHYSICS UPDATE (Feature 1)
# ============================================================
def update_physics(dt):
    global sub_x, sub_y, sub_z, sub_speed, vertical_velocity, sub_yaw

    #GM2
    if game_over:
        return

    # Forward/backward thrust with acceleration & drag (not instant teleport)
    if move_forward_input != 0:
        sub_speed += move_forward_input * ACCEL * dt
    else:
        if sub_speed > 0:
            sub_speed = max(0.0, sub_speed - DECEL * dt)
        elif sub_speed < 0:
            sub_speed = min(0.0, sub_speed + DECEL * dt)

    sub_speed = max(-MAX_SPEED * 0.5, min(MAX_SPEED, sub_speed))

    rad = math.radians(sub_yaw)
    sub_x += math.cos(rad) * sub_speed * dt
    sub_y += math.sin(rad) * sub_speed * dt

    # Simplified buoyancy: net vertical force from buoyancy vs gravity,
    # with pitch input acting like ballast tanks nudging vertical thrust
    net_force = BUOYANCY_FORCE - GRAVITY_FORCE + (pitch_input * 45.0)
    vertical_velocity += net_force * dt
    vertical_velocity *= WATER_DRAG
    sub_z += vertical_velocity * dt
    sub_z = max(FLOOR_Z + 15, min(CEILING_Z - 15, sub_z))
    if sub_z in (FLOOR_Z + 15, CEILING_Z - 15):
        vertical_velocity = 0.0



#Feature 4
def update_battery(dt):
    global battery, game_over, game_over_reason

    if game_over:
        return

    # Battery drains slowly all the time
    battery -= PASSIVE_BATTERY_DRAIN * dt

    # Extra battery consumption while moving
    if move_forward_input != 0:
        battery -= MOVEMENT_BATTERY_DRAIN * dt

    # Prevent battery from going below zero
    if battery <= 0:
        battery = 0
        game_over = True
        game_over_reason = "BATTERY DEPLETED"

#Feature 4
def check_charging_stations():
    global battery

    if game_over:
        return

    for station in charging_stations:
        wx, wy, charged = station

        if not charged:
            continue

        distance = math.sqrt(
            (sub_x - wx) ** 2 +
            (sub_y - wy) ** 2
        )

        if distance <= CHARGING_DISTANCE:
            battery = min(max_battery, battery + CHARGING_AMOUNT)

            station[2] = False



#Feature 5
def update_heat_vent_damage(dt):
    global hull_health, near_heat_vent, game_over, game_over_reason

    near_heat_vent = False

    if game_over:
        return

    for wx, wy in heat_vents:

        distance = math.sqrt(
            (sub_x - wx) ** 2 +
            (sub_y - wy) ** 2
        )

        if distance <= HEAT_VENT_DANGER_RADIUS:
            near_heat_vent = True


            damage_multiplier = 1.0 - (hazard_upgrade_level * 0.25)

            hull_health -= (HEAT_VENT_DAMAGE_RATE * damage_multiplier * dt)

            if hull_health <= 0:
                hull_health = 0
                game_over = True
                game_over_reason = "HULL DESTROYED"

            # Only need to detect one nearby vent
            break

#Feature 6
def check_artifact_collection():
    global artifacts_collected, artifact_score
    global sonar_upgrade_level
    global battery_upgrade_level
    global hazard_upgrade_level
    global max_battery
    global battery
    global SONAR_MAX_RADIUS
    global artifact_respawn_timers

    if game_over:
        return

    for index, artifact in enumerate(artifacts):

        wx, wy, wz, active, upgrade_type = artifact

        if not active:
            continue

        distance = math.sqrt(
            (sub_x - wx) ** 2 +
            (sub_y - wy) ** 2 +
            (sub_z - wz) ** 2
        )

        if distance <= ARTIFACT_COLLECTION_DISTANCE:

            artifact[3] = False
            artifact_respawn_timers[artifacts.index(artifact)] = ARTIFACT_RESPAWN_TIME
            artifacts_collected += 1
            artifact_score += 1

            # ==========================================
            # SONAR RANGE UPGRADE
            # ==========================================

            if upgrade_type == "sonar":

                if sonar_upgrade_level < MAX_SONAR_UPGRADES:

                    sonar_upgrade_level += 1

                    # Increase sonar range by 250
                    #global SONAR_MAX_RADIUS
                    SONAR_MAX_RADIUS += 250


            # ==========================================
            # BATTERY CAPACITY UPGRADE
            # ==========================================

            elif upgrade_type == "battery":

                if battery_upgrade_level < MAX_BATTERY_UPGRADES:

                    battery_upgrade_level += 1

                    # Increase maximum battery capacity
                    max_battery += 25

                    # Give the player the extra capacity
                    battery = min(
                        max_battery,
                        battery + 25
                    )


            # ==========================================
            # HAZARD RESISTANCE UPGRADE
            # ==========================================

            elif upgrade_type == "hazard":

                if hazard_upgrade_level < MAX_HAZARD_UPGRADES:

                    hazard_upgrade_level += 1

            break

def update_artifact_respawns(dt):
    global artifact_respawn_timers

    if game_over:
        return

    for i in range(len(artifacts)):

        # Artifact is inactive after being collected
        if not artifacts[i][3]:

            artifact_respawn_timers[i] -= dt

            # Respawn after the timer reaches zero
            if artifact_respawn_timers[i] <= 0:

                wx = artifacts[i][0]
                wy = artifacts[i][1]

                available_cells = []

                start = (GRID_W // 2, GRID_H // 2)

                for (cx, cy) in cave_cells:

                    if (cx, cy) != start:

                        cell_x = (cx - GRID_W / 2.0) * CELL_SIZE
                        cell_y = (cy - GRID_H / 2.0) * CELL_SIZE

                        # Avoid respawning directly on the player
                        distance = math.sqrt(
                            (sub_x - cell_x) ** 2 +
                            (sub_y - cell_y) ** 2
                        )

                        if distance > 200:
                            available_cells.append((cell_x, cell_y))

                if available_cells:

                    new_x, new_y = random.choice(available_cells)

                    artifacts[i][0] = new_x
                    artifacts[i][1] = new_y
                    artifacts[i][2] = 60.0
                    artifacts[i][3] = True

                    artifact_respawn_timers[i] = 0.0

def get_nearby_artifact_type():

    for artifact in artifacts:

        wx, wy, wz, active, upgrade_type = artifact

        if not active:
            continue

        distance = math.sqrt(
            (sub_x - wx) ** 2 +
            (sub_y - wy) ** 2 +
            (sub_z - wz) ** 2
        )

        # Show artifact identity slightly before collection distance
        if distance <= 100:

            if upgrade_type == "sonar":
                return "SONAR RANGE ARTIFACT"

            elif upgrade_type == "battery":
                return "BATTERY CAPACITY ARTIFACT"

            elif upgrade_type == "hazard":
                return "HAZARD RESISTANCE ARTIFACT"

    return None



# ============================================================
# GLUT CALLBACKS
# ============================================================
def keyboardListener(key, x, y):
    global last_w_time, last_s_time, last_q_time, last_e_time
    #GM2
    global battery, game_over, hull_health, game_over_reason, near_heat_vent
    global artifacts_collected, artifact_score, sonar_upgrade_level, battery_upgrade_level
    global hazard_upgrade_level, SONAR_MAX_RADIUS, max_battery

    if key == b'w':
        last_w_time = elapsed_seconds
    if key == b's':
        last_s_time = elapsed_seconds
    if key == b'q':
        last_q_time = elapsed_seconds
    if key == b'e':
        last_e_time = elapsed_seconds
    if key == b'r':

        # Generate a new cave
        carve_cave()
        #reset battery
        max_battery= 100.0
        battery = max_battery
        #reset hull
        hull_health = max_hull_health
        #reset artifacts
        artifacts_collected= 0
        artifact_score=0
        #reset upgrades
        sonar_upgrade_level = 0
        battery_upgrade_level = 0
        hazard_upgrade_level = 0

        # Reset upgrade effects
        SONAR_MAX_RADIUS = 900.0
        #max_battery = 100.0

        # Reset game state
        game_over = False
        game_over_reason = ""
        near_heat_vent = False
    if key == b' ':
        trigger_sonar()


def refresh_held_input_state():
    """Derives move_forward_input / pitch_input from how recently each key's
    auto-repeat event last arrived, so held keys keep accelerating and a
    released key decays to 0 without needing glutKeyboardUpFunc."""
    global move_forward_input, pitch_input

    w_held = (elapsed_seconds - last_w_time) < KEY_HOLD_TIMEOUT
    s_held = (elapsed_seconds - last_s_time) < KEY_HOLD_TIMEOUT
    q_held = (elapsed_seconds - last_q_time) < KEY_HOLD_TIMEOUT
    e_held = (elapsed_seconds - last_e_time) < KEY_HOLD_TIMEOUT

    if w_held and not s_held:
        move_forward_input = 1
    elif s_held and not w_held:
        move_forward_input = -1
    else:
        move_forward_input = 0

    if q_held and not e_held:
        pitch_input = 1
    elif e_held and not q_held:
        pitch_input = -1
    else:
        pitch_input = 0


def specialKeyListener(key, x, y):
    global sub_yaw

    if game_over:
        return

    
    if key == GLUT_KEY_LEFT:
        sub_yaw += TURN_RATE * 0.05
    if key == GLUT_KEY_RIGHT:
        sub_yaw -= TURN_RATE * 0.05


def mouseListener(button, state, x, y):
    if button == GLUT_LEFT_BUTTON and state == GLUT_DOWN:
        trigger_sonar()


def setupCamera():
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    gluPerspective(fovY, 1.25, 0.1, 3000)
    glMatrixMode(GL_MODELVIEW)
    glLoadIdentity()

    # Simple chase camera trailing behind the submersible's yaw
    rad = math.radians(sub_yaw)
    cam_dist = 320
    cx = sub_x - math.cos(rad) * cam_dist
    cy = sub_y - math.sin(rad) * cam_dist
    cz = sub_z + 160

    gluLookAt(cx, cy, cz,
              sub_x, sub_y, sub_z,
              0, 0, 1)


def idle():
    global last_time_ms, elapsed_seconds
    now = glutGet(GLUT_ELAPSED_TIME)
    dt = (now - last_time_ms) / 1000.0
    last_time_ms = now
    dt = min(dt, 0.05)  # clamp to avoid huge jumps on lag spikes
    elapsed_seconds += dt

    refresh_held_input_state()
    update_physics(dt)
    update_sonar(dt)
    #GM2
    update_battery(dt)
    check_charging_stations()
    update_heat_vent_damage(dt)
    check_artifact_collection()
    update_artifact_respawns(dt)

    glutPostRedisplay()


def showScreen():
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)
    glLoadIdentity()
    glViewport(0, 0, 1000, 800)

    setupCamera()

    draw_cave()
    draw_sonar_ring()
    draw_submersible()

    draw_text(300, 680, "Echoes of the Deep - Sonar Exploration Prototype")
    draw_text(10, 670, f"Depth: {sub_z:5.1f}   Speed: {sub_speed:5.1f}")
    #GM2
    draw_text(10, 650, f"Battery: {battery:5.1f}/{max_battery:.0f}")
    draw_text(10, 630, f"Hull: {hull_health:5.1f}%")
    #draw_text(10, 610, f"Artifacts: {artifacts_collected}")
    draw_text(10, 610, f"Artifact Score: {artifact_score}")

    # Upgrade levels
    draw_text(10,530,f"Sonar Upgrade: {sonar_upgrade_level}/4Range: {SONAR_MAX_RADIUS:.0f}")
    draw_text(10,510,f"Battery Upgrade: {battery_upgrade_level}/4Max: {max_battery:.0f}")

    damage_reduction = hazard_upgrade_level * 25

    draw_text(10,490,f"Hazard Resistance: {hazard_upgrade_level}/4 Reduction: {damage_reduction}%")


    draw_text(10, 590, "Charging stations restore 40 battery")
    if near_heat_vent and not game_over:
        draw_text(10, 570, "WARNING: HEAT VENT - MOVE AWAY!")

    if sonar_active:
        draw_text(10, 550, f"SONAR ACTIVE  radius: {sonar_radius:5.0f}")
    elif sonar_cooldown > 0:
        draw_text(10, 550, f"Sonar cooling down: {sonar_cooldown:4.1f}s")
    else:
        draw_text(10, 550, "Sonar ready (SPACE)")

    if game_over:
        draw_text(400, 420, "GAME OVER - " + game_over_reason)
        draw_text(400, 390, "Press R to restart")


    nearby_artifact = get_nearby_artifact_type()
    if nearby_artifact is not None:
        draw_text(20,400,nearby_artifact)

    glutSwapBuffers()


def main():
    global last_time_ms
    carve_cave()

    glutInit()
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB | GLUT_DEPTH)
    glutInitWindowSize(1000, 800)
    glutInitWindowPosition(0, 0)
    glutCreateWindow(b"Echoes of the Deep - Sonar Exploration Simulator")

    last_time_ms = glutGet(GLUT_ELAPSED_TIME)

    glutDisplayFunc(showScreen)
    glutKeyboardFunc(keyboardListener)
    glutSpecialFunc(specialKeyListener)
    glutMouseFunc(mouseListener)
    glutIdleFunc(idle)

    glutMainLoop()


if __name__ == "__main__":
    main()
